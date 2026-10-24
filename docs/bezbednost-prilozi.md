# Bezbednost — prilozi (skice koda)

> Dopuna za `docs/bezbednost.md`. Skice su polazna tačka; tipove i API-je proveri u `node_modules/` pre upotrebe.
> Kad dođe odgovarajući korak, kod ide u pravi repo (tamo ga čuvaju testovi), a ovde ostaje samo ono što se ne implementira.
> Revizija 24.10.2026: GRANT model (C.0), opoziv sesije (C.1, C.2), ispravljen `set_user_role` (C.4), `create_booking_request` i rate limit (C.6), append-only trigeri (C.7), offboarding (C.8), Storage politike (C.9), proxy sa nonce-om i admin root layout (B.8, B.9), ispravljeni `safeRedirectPath` i `safeHref` (B.3, B.4).

---

## Prilog B — skice koda (TypeScript / konfiguracija)

### B.1 `src/lib/auth/require-role.ts`
```ts
import 'server-only';
import { cache } from 'react';
import { createClient } from '@/lib/supabase/server';

export type StaffRole = 'admin' | 'sales' | 'editor';

type Denied = { ok: false; error: 'unauthenticated' | 'mfa_required' | 'forbidden' };
type Allowed = {
  ok: true;
  userId: string;
  role: StaffRole;
  supabase: Awaited<ReturnType<typeof createClient>>;
};

// Jednom po zahtevu, da layout i funkcije za podatke ne ponavljaju upit.
const loadClaims = cache(async () => {
  const supabase = await createClient();
  const { data, error } = await supabase.auth.getClaims();
  return { supabase, claims: error ? null : (data?.claims ?? null) };
});

// Profil se čita tek posle provere aal: pri aal1 restrictive politike ionako vraćaju prazno.
// Ako je sesija opozvana (red obrisan iz auth.sessions), RLS vraća null i korisnik je odbijen (C.2).
const loadProfile = cache(async () => {
  const { supabase, claims } = await loadClaims();
  if (!claims || claims.aal !== 'aal2') return null;

  const { data } = await supabase
    .from('profiles')
    .select('role, is_active')
    .eq('id', claims.sub)
    .maybeSingle();
  return data;
});

export async function requireRole(allowed: StaffRole[]): Promise<Allowed | Denied> {
  const { supabase, claims } = await loadClaims();
  if (!claims) return { ok: false, error: 'unauthenticated' };
  if (claims.aal !== 'aal2') return { ok: false, error: 'mfa_required' };

  const profile = await loadProfile();
  const role = profile?.role as StaffRole | null | undefined;
  if (!profile?.is_active || !role || !allowed.includes(role)) {
    return { ok: false, error: 'forbidden' };
  }
  return { ok: true, userId: claims.sub, role, supabase };
}

// Samo za /admin/mfa*: prijavljen korisnik koji još nema aal2. Ne čita nijednu tabelu.
export async function requireAal1Session() {
  const { supabase, claims } = await loadClaims();
  if (!claims) return { ok: false, error: 'unauthenticated' } as const;
  return { ok: true, userId: claims.sub, aal: claims.aal ?? 'aal1', supabase } as const;
}
```

### B.2 Obrazac server akcije
```ts
'use server';
import { z } from 'zod';
import { updateTag } from 'next/cache';
import { requireRole } from '@/lib/auth/require-role';

const Input = z.object({ offerId: z.string().uuid() }).strict();

export async function publishOffer(raw: unknown) {
  // 1. ulaz
  const parsed = Input.safeParse(raw);
  if (!parsed.success) return { ok: false, error: 'Neispravni podaci.' } as const;

  // 2. ko je i sme li (server akcija je dostupna sa bilo koje putanje, ne samo sa ove stranice)
  const staff = await requireRole(['editor', 'sales', 'admin']);
  if (!staff.ok) return { ok: false, error: 'Nemate dozvolu za ovu akciju.' } as const;

  // 3. posao kroz klijent prijavljenog korisnika: RLS i funkcija u bazi proveravaju ponovo
  //    (funkcija publish_offer sama upisuje audit_log)
  const { error } = await staff.supabase.rpc('publish_offer', { p_offer_id: parsed.data.offerId });
  if (error) return { ok: false, error: 'Objava nije uspela. Pokušajte ponovo.' } as const;

  // 4. keš
  updateTag(`offer:${parsed.data.offerId}`);
  updateTag('offers');
  return { ok: true, data: null } as const;
}
```

### B.3 `src/lib/auth/safe-redirect.ts`
```ts
const BASE = 'https://halotravel.invalid';

export function safeRedirectPath(next: string | null | undefined, fallback = '/admin'): string {
  if (!next || !next.startsWith('/') || next.includes('\\')) return fallback;
  try {
    // URL parser uklanja tabove i nove redove i razrešava ../, pa proveravamo kanonski oblik:
    // //host i /\host menjaju origin, /admin/../login postaje /login
    const url = new URL(next, BASE);
    if (url.origin !== BASE) return fallback;
    if (url.pathname !== '/admin' && !url.pathname.startsWith('/admin/')) return fallback;
    return url.pathname + url.search + url.hash;
  } catch {
    return fallback;
  }
}
```

### B.4 `src/lib/security/safe-href.ts`
```ts
const ALLOWED_PROTOCOLS = new Set(['http:', 'https:', 'mailto:', 'tel:']);
const BASE = 'https://halotravel.rs';

export function safeHref(raw: unknown): string | null {
  if (typeof raw !== 'string') return null;
  const value = raw.trim();
  if (value.length === 0 || value.length > 2048) return null;
  try {
    // relativne putanje se tumače kao https; javascript:, data:, vbscript: padaju
    const url = new URL(value, BASE);
    if (!ALLOWED_PROTOCOLS.has(url.protocol)) return null;

    // vraćamo kanonski oblik, nikad ulaz: //evil.com i /\evil.com izgledaju kao
    // relativne putanje, a vode na drugi domen, pa postaju vidljivo https://evil.com/
    const looksRelative = !/^[a-z][a-z\d+.-]*:/i.test(value);
    if (looksRelative && url.origin === BASE) return url.pathname + url.search + url.hash;
    return url.href;
  } catch {
    return null;
  }
}
```

### B.5 Headeri u `next.config.ts` (deo)
```ts
import type { NextConfig } from 'next';

const securityHeaders = [
  { key: 'X-Content-Type-Options', value: 'nosniff' },
  { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
  { key: 'X-Frame-Options', value: 'DENY' },
  { key: 'Permissions-Policy', value: 'camera=(), microphone=(), geolocation=(), payment=(), usb=()' },
  // includeSubDomains / preload tek posle provere DNS-a
  { key: 'Strict-Transport-Security', value: 'max-age=31536000' },
];

const nextConfig: NextConfig = {
  poweredByHeader: false,
  async headers() {
    return [
      { source: '/:path*', headers: securityHeaders },
      { source: '/admin/:path*', headers: [{ key: 'X-Robots-Tag', value: 'noindex, nofollow' }] },
      // Cache-Control za /admin postavlja proxy.ts (B.8), ne ovde
    ];
  },
};

export default nextConfig;
```

### B.6 Skice CSP-a
Domene GA4 i Meta piksela uzeti iz njihove dokumentacije. U razvoju Next.js traži i `'unsafe-eval'`.

Javni sajt (statički, iz `next.config.ts`). `/admin` je izuzet, jer ima svoju politiku iz `proxy.ts`:
```ts
{ source: '/:path((?!admin(?:/|$)).*)', headers: [{ key: 'Content-Security-Policy-Report-Only', value: publicCsp }] }
```
```text
default-src 'self';
script-src 'self' 'unsafe-inline' <GA4> <META> https://challenges.cloudflare.com;
style-src 'self' 'unsafe-inline';
img-src 'self' data: blob: https://<ref>.supabase.co <GA4> <META>;
font-src 'self';
connect-src 'self' https://<ref>.supabase.co <GA4> <META>;
frame-src https://challenges.cloudflare.com;
frame-ancestors 'none';
base-uri 'self';
form-action 'self';
object-src 'none';
upgrade-insecure-requests;
```

Admin (nonce iz `proxy.ts`, novi za svaki zahtev; vidi B.8 i B.9):
```text
default-src 'self';
script-src 'self' 'nonce-<NONCE>' 'strict-dynamic';
style-src 'self' 'unsafe-inline';
img-src 'self' data: blob: https://<ref>.supabase.co;
connect-src 'self' https://<ref>.supabase.co wss://<ref>.supabase.co;
frame-src https://challenges.cloudflare.com;
frame-ancestors 'none';
base-uri 'self';
form-action 'self';
object-src 'none';
```

### B.7 `.claude/settings.json`
```json
{
  "permissions": {
    "deny": [
      "Read(./.env)",
      "Read(./.env.local)",
      "Read(./.env.*.local)",
      "Read(./.env.production)",
      "Bash(npx supabase db push:*)",
      "Bash(npx supabase db reset --linked:*)",
      "Bash(vercel --prod:*)",
      "Bash(npx vercel --prod:*)"
    ],
    "ask": [
      "Bash(git push:*)",
      "Bash(npx supabase link:*)"
    ]
  }
}
```
`deny` pravila nisu bezbednosna granica: `Read(./.env)` ne sprečava `cat .env` kroz Bash. Prava zaštita je sandbox za Bash ili PreToolUse hook (tačna podešavanja proveriti u dokumentaciji Claude Code-a) i to da na dev mašini nema produkcionih kredencijala.

### B.8 `src/proxy.ts` — osvežavanje sesije, nonce, `no-store`
Potpis `setAll` (drugi argument sa cache zaglavljima) i `cookieOptions` proveriti u `node_modules/@supabase/ssr` posle instalacije. Ime stranice prijave potvrditi u koraku 7.
```ts
import { NextResponse, type NextRequest } from 'next/server';
import { createServerClient } from '@supabase/ssr';

function adminCsp(nonce: string) {
  const isDev = process.env.NODE_ENV === 'development';
  return [
    "default-src 'self'",
    `script-src 'self' 'nonce-${nonce}' 'strict-dynamic'${isDev ? " 'unsafe-eval'" : ''}`,
    "style-src 'self' 'unsafe-inline'",
    `img-src 'self' data: blob: ${process.env.NEXT_PUBLIC_SUPABASE_URL}`,
    `connect-src 'self' ${process.env.NEXT_PUBLIC_SUPABASE_URL}`,
    "frame-ancestors 'none'",
    "base-uri 'self'",
    "form-action 'self'",
    "object-src 'none'",
  ].join('; ');
}

export async function proxy(request: NextRequest) {
  // nonce pre Supabase klijenta: setAll pravi nov odgovor i bez ovoga bi ga izgubio
  const nonce = Buffer.from(crypto.randomUUID()).toString('base64');
  const csp = adminCsp(nonce);

  // isti request headeri u oba NextResponse.next(...) poziva;
  // posle request.cookies.set(...) request.headers već nosi osvežene kolačiće
  const forward = () => {
    const headers = new Headers(request.headers);
    headers.set('x-nonce', nonce);
    headers.set('Content-Security-Policy', csp); // odavde Next čita nonce pri renderovanju
    return NextResponse.next({ request: { headers } });
  };

  let response = forward();

  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY!,
    {
      cookieOptions: { secure: process.env.NODE_ENV === 'production' },
      cookies: {
        getAll: () => request.cookies.getAll(),
        setAll(cookiesToSet, cacheHeaders) {
          cookiesToSet.forEach(({ name, value }) => request.cookies.set(name, value));
          response = forward();
          cookiesToSet.forEach(({ name, value, options }) => response.cookies.set(name, value, options));
          // zaglavlja koja ssr šalje uz nove kolačiće, da keš ne ponese tuđi Set-Cookie
          Object.entries(cacheHeaders ?? {}).forEach(([key, value]) => response.headers.set(key, value));
        },
      },
    },
  );

  // osvežava sesiju pre Server Components; ovo NIJE provera pristupa (to radi requireRole)
  const { data } = await supabase.auth.getClaims();

  const path = request.nextUrl.pathname;
  const isAuthPath = path === '/prijava' || path.startsWith('/admin/auth/');
  if (!data?.claims && !isAuthPath) {
    const url = request.nextUrl.clone();
    url.pathname = '/prijava';
    url.search = `?next=${encodeURIComponent(path)}`;
    const redirect = NextResponse.redirect(url);
    response.cookies.getAll().forEach((cookie) => redirect.cookies.set(cookie));
    response = redirect;
  }

  // na svaki odgovor, i na preusmerenje
  response.headers.set('Content-Security-Policy-Report-Only', csp); // posle provere: Content-Security-Policy
  response.headers.set('Cache-Control', 'private, no-store');
  return response;
}

export const config = { matcher: ['/admin/:path*', '/prijava'] };
```

### B.9 Root layout samo za `/admin`
Nonce ne radi sa statičkim shell-om koji pravi Cache Components (CSP vodič: PPR „is incompatible with nonce-based CSP“). Zato `/admin` ima sopstveni root layout i renderuje se ceo po zahtevu. Javni sajt ima svoj root layout u `(public)`.
```text
src/app/
  (public)/layout.tsx               # root layout javnog sajta (statički CSP, "use cache")
  (admin)/admin/layout.tsx          # root layout admina: connection(), requireAal1Session()
  (admin)/admin/mfa/                # upis i provera TOTP-a (aal1)
  (admin)/admin/auth/confirm/route.ts  # token_hash + verifyOtp (poziv, reset)
  (admin)/admin/(osoblje)/layout.tsx   # requireRole(); sve ostale admin stranice
```
```tsx
// src/app/(admin)/admin/layout.tsx
import type { Metadata } from 'next';
import { connection } from 'next/server';

// admin sme da blokira; bez statičkog shell-a nema skripti bez nonce-a
export const instant = false;
export const metadata: Metadata = { robots: { index: false, follow: false } };

export default async function AdminRootLayout({ children }: { children: React.ReactNode }) {
  await connection(); // van <Suspense>: ceo admin se renderuje po zahtevu, pa Next dodaje nonce svuda
  return (
    <html lang="sr-Latn">
      <body>{children}</body>
    </html>
  );
}
```
Proveriti na build-u u koraku 7: `next build` ne sme da prijavi statički shell za `/admin`, a CSP Report-Only ne sme da prijavi blokiranu skriptu.

---

## Prilog C — skice SQL-a

U primerima je `profiles.role` tipa `text` sa CHECK ograničenjem. Ako bude enum, dodati cast.

### C.0 Prva migracija — bez automatskih GRANT-ova
Supabase od 30.05.2026. novim projektima, a postojećim od 30.10.2026, ne daje tabelama u `public` automatske privilegije ([changelog](https://supabase.com/changelog/45329-breaking-change-tables-not-exposed-to-data-and-graphql-api-automatically)). Pišemo to i eksplicitno, da lokalna baza, dev i produkcija rade isto bez obzira na verziju CLI-ja.
```sql
-- ništa novo u public nije dostupno dok se izričito ne dodeli
alter default privileges for role postgres in schema public
  revoke all on tables from anon, authenticated, service_role;
alter default privileges for role postgres in schema public
  revoke all on sequences from anon, authenticated, service_role;
alter default privileges for role postgres in schema public
  revoke execute on functions from anon, authenticated, service_role;
-- EXECUTE za PUBLIC je globalni podrazumevani grant Postgres-a: revoke "in schema" ga ne uklanja
alter default privileges for role postgres
  revoke execute on functions from public;
```
Obrazac za svaku tabelu (u istoj migraciji: tabela, RLS, grant, politike):
```sql
create table public.destinations (
  id uuid primary key default gen_random_uuid()
  -- ...
);
alter table public.destinations enable row level security;

-- samo operacije za koje postoji politika; TRUNCATE (zaobilazi RLS), REFERENCES i TRIGGER nikome
grant select on public.destinations to anon;                                  -- javno (4.4)
grant select, insert, update, delete on public.destinations to authenticated; -- politike po ulozi
grant select on public.destinations to service_role;                          -- samo ako ga server koristi
```
Primarni ključevi su uuid, pa sekvence ne trebaju. Ako se ipak pojavi `identity` kolona, proveriti da li uloga koja upisuje treba `usage` na sekvenci.

### C.1 Šema `private`, `has_role` i `session_is_valid`
```sql
create schema if not exists private;
revoke all on schema private from public;
grant usage on schema private to authenticated, service_role;

-- sesija iz JWT-a još postoji i nije istekla (opoziv deluje odmah, ne tek na exp tokena)
-- neprovereno: kolone auth.sessions i pravo vlasnika funkcije da ih čita na hosted verziji
create or replace function private.session_is_valid()
returns boolean
language sql
stable
security definer
set search_path = ''
as $$
  select exists (
    select 1
    from auth.sessions s
    where s.id = ((select auth.jwt()) ->> 'session_id')::uuid
      and s.user_id = (select auth.uid())
      and (s.not_after is null or s.not_after > now())
  );
$$;

create or replace function private.has_role(variadic roles text[])
returns boolean
language sql
stable
security definer
set search_path = ''
as $$
  select exists (
    select 1
    from public.profiles p
    where p.id = (select auth.uid())
      and p.is_active
      and p.role = any (roles)
  )
  -- SECURITY DEFINER funkcije zaobilaze RLS (i C.2), pa i ovde proveravamo sesiju
  and (select private.session_is_valid());
$$;

revoke all on function private.session_is_valid() from public, anon;
revoke all on function private.has_role(text[]) from public, anon;
grant execute on function private.session_is_valid() to authenticated;
grant execute on function private.has_role(text[]) to authenticated;

-- primer politike
create policy "prodaja čita zahteve"
on public.booking_requests
for select
to authenticated
using ((select private.has_role('sales', 'admin')));
```

### C.2 MFA (aal2) i važeća sesija kao restrictive politika
Restrictive politika sama ne daje pristup; samo sužava ono što daju obične (permissive) politike. Ponoviti za svaku tabelu i za `storage.objects`. Važi samo za tabele i Storage, ne i za Auth API, pa upis prvog MFA faktora radi pri `aal1` (vidi 3.2 u `bezbednost.md`).
```sql
create policy "osoblje mora imati MFA i važeću sesiju"
on public.booking_requests
as restrictive
for all
to authenticated
using (
  (select auth.jwt() ->> 'aal') = 'aal2'
  and (select private.session_is_valid())
);
```
Posledica za pgTAP: testovi po ulozi moraju da ubace red u `auth.sessions` i stave njegov `session_id` u `request.jwt.claims` (C.5).

### C.3 `profiles` — korisnik ne može da promeni svoju ulogu
```sql
create table public.profiles (
  id uuid primary key references auth.users (id) on delete cascade,
  full_name text check (char_length(full_name) <= 120),
  role text check (role in ('admin', 'sales', 'editor')),  -- null dok admin ne dodeli
  is_active boolean not null default false,
  updated_at timestamptz not null default now()
);
alter table public.profiles enable row level security;

revoke all on public.profiles from anon, authenticated;
grant select on public.profiles to authenticated;              -- bez ovoga requireRole() dobija 42501
grant update (full_name) on public.profiles to authenticated;  -- uloga i is_active samo kroz set_user_role()

create policy "korisnik vidi svoj profil"
on public.profiles for select to authenticated
using (id = (select auth.uid()));

create policy "admin vidi sve profile"
on public.profiles for select to authenticated
using ((select private.has_role('admin')));

create policy "korisnik menja svoje ime"
on public.profiles for update to authenticated
using (id = (select auth.uid()))
with check (id = (select auth.uid()));
```

### C.4 `set_user_role` — jedini način promene uloge
```sql
create or replace function public.set_user_role(p_user_id uuid, p_role text, p_active boolean)
returns void
language plpgsql
security definer
set search_path = ''
as $$
declare
  v_old_role text;
  v_old_active boolean;
begin
  -- NULL bi pretvorio proveru poslednjeg admina u NULL i tiho je preskočio
  if p_active is null then
    raise exception 'p_active is required' using errcode = '22004';
  end if;

  -- jedna promena uloge u isto vreme, pre provere uloge i brojanja admina:
  -- bez ovoga dva admina koja istovremeno ražaluju jedan drugog oba prolaze (for update zaključava samo ciljni red)
  perform pg_advisory_xact_lock(hashtext('public.set_user_role'));

  if not private.has_role('admin')
     or (select auth.jwt() ->> 'aal') is distinct from 'aal2' then
    raise exception 'forbidden' using errcode = '42501';
  end if;

  if p_role is not null and p_role not in ('admin', 'sales', 'editor') then
    raise exception 'invalid role' using errcode = '22023';
  end if;

  select p.role, p.is_active into v_old_role, v_old_active
  from public.profiles p
  where p.id = p_user_id
  for update;

  if not found then
    raise exception 'profile not found' using errcode = 'P0002';
  end if;

  -- aktivan nalog samo uz potvrđen TOTP faktor (neprovereno: kolone auth.mfa_factors na hosted verziji)
  if p_active and not exists (
    select 1
    from auth.mfa_factors f
    where f.user_id = p_user_id
      and f.factor_type = 'totp'
      and f.status = 'verified'
  ) then
    raise exception 'mfa not enrolled' using errcode = '23514';
  end if;

  -- sistem ne sme ostati bez aktivnog admina
  if v_old_role = 'admin' and v_old_active
     and (p_role is distinct from 'admin' or not p_active)
     and (select count(*) from public.profiles where role = 'admin' and is_active) <= 1 then
    raise exception 'last admin' using errcode = '23514';
  end if;

  update public.profiles
     set role = p_role, is_active = p_active, updated_at = now()
   where id = p_user_id;

  insert into public.audit_log (actor_id, action, entity, entity_id, details)
  values ((select auth.uid()), 'user.role_changed', 'profile', p_user_id,
          jsonb_build_object('from', v_old_role, 'to', p_role, 'active', p_active));
end;
$$;

revoke all on function public.set_user_role(uuid, text, boolean) from public, anon, authenticated;
grant execute on function public.set_user_role(uuid, text, boolean) to authenticated;
```
Prvog admina postavlja Predrag ručno (seed lokalno, SQL na produkciji) posle njegovog upisa TOTP-a, uz red u `audit_log`.

### C.5 pgTAP — strukturne provere
```sql
begin;
select plan(7);

select is(
  (select count(*)::int
     from pg_tables
    where schemaname = 'public' and not rowsecurity),
  0,
  'RLS je uključen na svim tabelama u public'
);

select is(
  (select count(*)::int
     from pg_proc p
     join pg_namespace n on n.oid = p.pronamespace
    where n.nspname = 'public'
      and p.prosecdef
      and has_function_privilege('anon', p.oid, 'EXECUTE')
      and not exists (            -- preskoči funkcije iz ekstenzija
        select 1 from pg_depend d
        where d.objid = p.oid and d.deptype = 'e'
      )),
  0,
  'anon ne može da izvrši nijednu SECURITY DEFINER funkciju u public'
);

select is(
  array(select p.proname::text
          from pg_proc p
          join pg_namespace n on n.oid = p.pronamespace
         where n.nspname = 'public'
           and has_function_privilege('authenticated', p.oid, 'EXECUTE')
           and not exists (select 1 from pg_depend d where d.objid = p.oid and d.deptype = 'e')
         order by 1),
  array['set_user_role']::text[],   -- izričita lista; dopunjuje se sa svakom novom funkcijom
  'authenticated izvršava samo funkcije sa liste'
);

select is(
  (select count(*)::int
     from pg_class c
     join pg_namespace n on n.oid = c.relnamespace
    where n.nspname = 'public'
      and c.relkind = 'v'
      and coalesce(array_to_string(c.reloptions, ','), '') !~* 'security_invoker=(true|on|1|yes)'),
  0,
  'svi view-ovi u public su security_invoker'
);

-- materialized view ne podržava ni RLS ni security_invoker
select is(
  (select count(*)::int
     from pg_class c
     join pg_namespace n on n.oid = c.relnamespace
    where n.nspname = 'public'
      and c.relkind = 'm'
      and (has_table_privilege('anon', c.oid, 'SELECT')
           or has_table_privilege('authenticated', c.oid, 'SELECT'))),
  0,
  'materialized view-ovi nisu dostupni anon ni authenticated'
);

select is(
  (select count(*)::int
     from pg_class c
     join pg_namespace n on n.oid = c.relnamespace
    where n.nspname = 'public'
      and c.relkind in ('r', 'p', 'v', 'm')
      and (has_table_privilege('anon', c.oid, 'TRUNCATE, REFERENCES, TRIGGER')
           or has_table_privilege('authenticated', c.oid, 'TRUNCATE, REFERENCES, TRIGGER'))),
  0,
  'anon i authenticated nemaju TRUNCATE, REFERENCES ni TRIGGER'
);

select is(
  (select count(*)::int
     from pg_class c
     join pg_namespace n on n.oid = c.relnamespace
    where n.nspname = 'public'
      and c.relkind in ('r', 'p', 'v', 'm')
      and has_table_privilege('anon', c.oid, 'INSERT, UPDATE, DELETE')),
  0,
  'anon nema INSERT, UPDATE ni DELETE nigde'
);

select * from finish();
rollback;
```
Dodati i test da anon ima SELECT samo na tabelama iz 4.4 (lista se piše kad šema v1 bude gotova).

Primer testa po ulozi. Test bez redova ne dokazuje ništa (`is_empty` nad praznom tabelom uvek prolazi), pa se prvo ubacuje red kao vlasnik:
```sql
-- kontrola: red postoji
insert into public.booking_requests (id /* , ostale obavezne kolone */)
values ('00000000-0000-0000-0000-000000000001' /* , ... */);
select isnt_empty('select id from public.booking_requests', 'kontrola: red postoji');

set local role anon;
select throws_ok(
  'select id from public.booking_requests',
  '42501',
  null,
  'anon nema pristup zahtevima'
);
reset role;

-- prijavljen korisnik: važeća sesija u auth.sessions (C.2), pa claims, pa uloga
insert into auth.sessions (id, user_id, aal)
values ('00000000-0000-0000-0000-0000000000a1', '<uuid-editora>', 'aal2');
select set_config(
  'request.jwt.claims',
  '{"sub":"<uuid-editora>","role":"authenticated","aal":"aal2","session_id":"00000000-0000-0000-0000-0000000000a1"}',
  true
);
set local role authenticated;
select is_empty('select id from public.booking_requests', 'editor ne vidi zahteve');
reset role;
```
Kolone `auth.sessions` proveriti na lokalnoj verziji.

### C.6 `create_booking_request` i rate limit
Kolone zahteva su ilustracija; prave dolaze iz šeme u koraku 13. Ključeve za rate limit (HMAC od IP-a i mejla) računa server.
```sql
create table private.rate_limits (
  key text not null,
  window_start timestamptz not null,
  hits int not null default 0,
  primary key (key, window_start)
);

-- atomično: jedan upit broji i odgovara (bez read-then-write trke)
create or replace function private.hit_rate_limit(p_key text, p_limit int, p_window interval)
returns boolean
language sql
volatile
security definer
set search_path = ''
as $$
  insert into private.rate_limits as r (key, window_start, hits)
  values (p_key, date_bin(p_window, now(), timestamptz 'epoch'), 1)
  on conflict (key, window_start) do update set hits = r.hits + 1
  returning hits <= p_limit;
$$;
revoke all on function private.hit_rate_limit(text, int, interval) from public, anon, authenticated;
grant execute on function private.hit_rate_limit(text, int, interval) to service_role;

create or replace function public.create_booking_request(
  p_offer_id uuid,
  p_departure_id uuid,
  p_adults int,
  p_child_ages int[],
  p_contact jsonb,
  p_ip_key text,
  p_email_key text
)
returns uuid
language plpgsql
security invoker          -- poziva ga samo service_role, koji ionako zaobilazi RLS
set search_path = ''
as $$
declare
  v_id uuid;
begin
  -- private.* nije dostupan kroz rpc(), pa se rate limit zove ovde, u istoj transakciji.
  -- Odbijen pokušaj vraća transakciju, pa se ne broji; limit broji uspešne zahteve.
  if not private.hit_rate_limit(p_ip_key, 5, interval '10 minutes')
     or not private.hit_rate_limit(p_email_key, 3, interval '24 hours') then
    raise exception 'rate_limited' using errcode = 'HT429';
  end if;

  -- ponovna provera, ne veruje aplikaciji
  if p_adults is null or p_adults not between 1 and 9
     or coalesce(cardinality(p_child_ages), 0) > 6
     or exists (select 1 from unnest(p_child_ages) a where a is null or a not between 0 and 17) then
    raise exception 'invalid travelers' using errcode = '22023';
  end if;

  if not exists (
    select 1 from public.offers o
    where o.id = p_offer_id and o.status = 'published'
  ) then
    raise exception 'offer not available' using errcode = 'P0002';
  end if;
  -- TODO kad šema cena bude poznata: termin pripada ponudi i nije stop sale

  insert into public.booking_requests (offer_id, departure_id, adults, child_ages, contact)
  values (p_offer_id, p_departure_id, p_adults, p_child_ages, p_contact)
  returning id into v_id;

  insert into public.booking_events (booking_request_id, from_status, to_status)
  values (v_id, null, 'new');

  return v_id;
end;
$$;

revoke all on function public.create_booking_request(uuid, uuid, int, int[], jsonb, text, text)
  from public, anon, authenticated;   -- direktan grant za authenticated inače preživljava
grant execute on function public.create_booking_request(uuid, uuid, int, int[], jsonb, text, text)
  to service_role;
```
Kalkulator ne upisuje ništa, pa rate limit zove kroz tanak omotač `public.check_rate_limit(p_key, p_limit, p_window)` (`security invoker`, izvršiv samo za `service_role`) koji vraća `private.hit_rate_limit(...)`. Redove starije od 48 h briše zakazani posao (kasnije, `pg_cron`).

### C.7 Append-only: `audit_log` i `booking_events`
Izostanak UPDATE/DELETE politika ne štiti od `service_role` (secret ključ) ni od vlasnika tabele, jer oni zaobilaze RLS. Trigeri štite od greške u našem kodu i od zloupotrebe secret ključa; vlasnik baze ih i dalje može isključiti.
```sql
create or replace function private.forbid_change()
returns trigger
language plpgsql
set search_path = ''
as $$
begin
  raise exception '% je samo za upis (append-only)', tg_table_name using errcode = '42501';
end;
$$;

create trigger audit_log_append_only
  before update or delete on public.audit_log
  for each row execute function private.forbid_change();
create trigger audit_log_no_truncate
  before truncate on public.audit_log
  for each statement execute function private.forbid_change();

-- isto za public.booking_events (korak 13)
```
Posledica: `booking_events.booking_request_id` ne sme imati `on delete cascade` koji bi brisao događaje. Brisanje ličnih podataka na zahtev (§14) anonimizuje red u `booking_requests` i `travelers`, ne briše ga.

### C.8 Offboarding — brisanje sesija drugog korisnika
`auth.admin.signOut()` prima JWT korisnika, ne njegov ID, pa ne može da odjavi nekog drugog. Redosled je u `bezbednost.md` 3.4.
```sql
-- neprovereno kao zvaničan API: proveriti na lokalnoj i hosted verziji da brisanje
-- iz auth.sessions gasi i refresh tokene (FK) i da vlasnik funkcije sme da briše
create or replace function public.revoke_user_sessions(p_user_id uuid)
returns void
language sql
security definer
set search_path = ''
as $$
  delete from auth.sessions where user_id = p_user_id;
$$;

revoke all on function public.revoke_user_sessions(uuid) from public, anon, authenticated;
grant execute on function public.revoke_user_sessions(uuid) to service_role;
```
Zove je server akcija koja je prethodno prošla `requireRole(['admin'])`, preko admin klijenta, i upisuje `audit_log`.

### C.9 Storage politike
`remove()` traži i DELETE i SELECT politiku. Bez UPDATE politike nema ni prepisivanja ni upsert-a. Restrictive aal2 politika iz C.2 važi i za `storage.objects`.
```sql
-- images: javno čitanje ide preko javnog URL-a; SELECT politika je samo za osoblje (potrebna za brisanje)
create policy "osoblje dodaje slike"
on storage.objects for insert to authenticated
with check (
  bucket_id = 'images'
  and (storage.foldername(name))[1] = 'offers'
  and (select private.has_role('editor', 'sales', 'admin'))
);

create policy "osoblje vidi slike"
on storage.objects for select to authenticated
using (bucket_id = 'images' and (select private.has_role('editor', 'sales', 'admin')));

create policy "osoblje briše slike"
on storage.objects for delete to authenticated
using (
  bucket_id = 'images'
  and (storage.foldername(name))[1] = 'offers'
  and (select private.has_role('editor', 'sales', 'admin'))
);

-- imports i documents: dokaz, pa bez UPDATE i DELETE
create policy "prodaja dodaje uvoze i dokumente"
on storage.objects for insert to authenticated
with check (bucket_id in ('imports', 'documents') and (select private.has_role('sales', 'admin')));

create policy "prodaja čita uvoze i dokumente"
on storage.objects for select to authenticated
using (bucket_id in ('imports', 'documents') and (select private.has_role('sales', 'admin')));
```

---

## Prilog D — predlog env promenljivih

```text
NEXT_PUBLIC_SITE_URL=
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY=   # sb_publishable_...
SUPABASE_SECRET_KEY=                    # sb_secret_..., samo src/lib/supabase/admin.ts
RESEND_API_KEY=                         # samo slanje, ograničen na domen
EMAIL_FROM=
EMAIL_STAFF_INBOX=
NEXT_PUBLIC_TURNSTILE_SITE_KEY=         # ODLUKA-2
TURNSTILE_SECRET_KEY=                   # ODLUKA-2
RATE_LIMIT_SECRET=                      # HMAC za IP i mejl u brojaču
NEXT_PUBLIC_GA_ID=                      # učitava se tek posle pristanka
NEXT_PUBLIC_META_PIXEL_ID=              # učitava se tek posle pristanka
# NEXT_SERVER_ACTIONS_ENCRYPTION_KEY se ne postavlja na Vercelu (Next ga pravi po build-u)
# S3 ključevi za noćnu kopiju Storage-a nisu env aplikacije; vode se u inventaru tajni (bezbednost.md 10.3)
```
