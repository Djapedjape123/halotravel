# Bezbednost — prilozi (skice koda)

> Dopuna za `docs/bezbednost.md`. Skice su polazna tačka; tipove i API-je proveri u `node_modules/` pre upotrebe.
> Kad dođe odgovarajući korak, kod ide u pravi repo (tamo ga čuvaju testovi), a ovde ostaje samo ono što se ne implementira.

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
const loadStaff = cache(async () => {
  const supabase = await createClient();
  const { data, error } = await supabase.auth.getClaims();
  const claims = data?.claims;
  if (error || !claims) return { supabase, claims: null, profile: null };

  const { data: profile } = await supabase
    .from('profiles')
    .select('role, is_active')
    .eq('id', claims.sub)
    .maybeSingle();

  return { supabase, claims, profile };
});

export async function requireRole(allowed: StaffRole[]): Promise<Allowed | Denied> {
  const { supabase, claims, profile } = await loadStaff();
  if (!claims) return { ok: false, error: 'unauthenticated' };
  if (claims.aal !== 'aal2') return { ok: false, error: 'mfa_required' };

  const role = profile?.role as StaffRole | null | undefined;
  if (!profile?.is_active || !role || !allowed.includes(role)) {
    return { ok: false, error: 'forbidden' };
  }
  return { ok: true, userId: claims.sub, role, supabase };
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

  // 2. ko je i sme li
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
export function safeRedirectPath(next: string | null | undefined, fallback = '/admin'): string {
  if (!next) return fallback;
  // samo naše admin putanje; odbij //host, /\host i apsolutne URL-ove
  const inAdmin = next === '/admin' || next.startsWith('/admin/');
  if (!inAdmin || next.startsWith('//') || next.includes('\\')) return fallback;
  return next;
}
```

### B.4 `src/lib/security/safe-href.ts`
```ts
const ALLOWED_PROTOCOLS = new Set(['http:', 'https:', 'mailto:', 'tel:']);

export function safeHref(raw: unknown): string | null {
  if (typeof raw !== 'string') return null;
  const value = raw.trim();
  if (value.length === 0 || value.length > 2048) return null;
  try {
    // relativne putanje se tumače kao https; javascript:, data:, vbscript: padaju
    const url = new URL(value, 'https://halotravel.rs');
    return ALLOWED_PROTOCOLS.has(url.protocol) ? value : null;
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
    ];
  },
};

export default nextConfig;
```

### B.6 Skice CSP-a
Domene GA4 i Meta piksela uzeti iz njihove dokumentacije. U razvoju Next.js traži i `'unsafe-eval'`.

Javni sajt (statički, iz `next.config.ts`):
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

Admin (nonce iz `proxy.ts`, novi za svaki zahtev):
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

---

## Prilog C — skice SQL-a

U primerima je `profiles.role` tipa `text` sa CHECK ograničenjem. Ako bude enum, dodati cast.

### C.1 Šema `private` i `has_role`
```sql
create schema if not exists private;
revoke all on schema private from public;
grant usage on schema private to authenticated;

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
  );
$$;

revoke all on function private.has_role(text[]) from public;
grant execute on function private.has_role(text[]) to authenticated;

-- primer politike
create policy "prodaja čita zahteve"
on public.booking_requests
for select
to authenticated
using ((select private.has_role('sales', 'admin')));
```

### C.2 MFA (aal2) kao restrictive politika
Restrictive politika sama ne daje pristup; samo sužava ono što daju obične (permissive) politike. Ponoviti za svaku tabelu i za `storage.objects`. Važi samo za tabele i Storage, ne i za Auth API, pa upis prvog MFA faktora radi pri `aal1` (vidi 3.2 u `bezbednost.md`).
```sql
create policy "osoblje mora imati MFA"
on public.booking_requests
as restrictive
for all
to authenticated
using ((select auth.jwt() ->> 'aal') = 'aal2');
```

### C.3 `profiles` — korisnik ne može da promeni svoju ulogu
```sql
alter table public.profiles enable row level security;

revoke all on public.profiles from anon;
revoke insert, update, delete on public.profiles from authenticated;
grant update (full_name) on public.profiles to authenticated;

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
  v_old text;
begin
  if not private.has_role('admin')
     or (select auth.jwt() ->> 'aal') is distinct from 'aal2' then
    raise exception 'forbidden' using errcode = '42501';
  end if;

  if p_role is not null and p_role not in ('admin', 'sales', 'editor') then
    raise exception 'invalid role' using errcode = '22023';
  end if;

  select p.role into v_old
  from public.profiles p
  where p.id = p_user_id
  for update;

  if not found then
    raise exception 'profile not found' using errcode = 'P0002';
  end if;

  -- sistem ne sme ostati bez aktivnog admina
  if v_old = 'admin'
     and (p_role is distinct from 'admin' or not p_active)
     and (select count(*) from public.profiles where role = 'admin' and is_active) <= 1 then
    raise exception 'last admin' using errcode = '23514';
  end if;

  update public.profiles
     set role = p_role, is_active = p_active
   where id = p_user_id;

  insert into public.audit_log (actor_id, action, entity, entity_id, details)
  values ((select auth.uid()), 'user.role_changed', 'profile', p_user_id,
          jsonb_build_object('from', v_old, 'to', p_role, 'active', p_active));
end;
$$;

revoke all on function public.set_user_role(uuid, text, boolean) from public, anon;
grant execute on function public.set_user_role(uuid, text, boolean) to authenticated;
```

### C.5 pgTAP — strukturne provere
```sql
begin;
select plan(3);

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
  (select count(*)::int
     from pg_class c
     join pg_namespace n on n.oid = c.relnamespace
    where n.nspname = 'public'
      and c.relkind = 'v'
      and coalesce(array_to_string(c.reloptions, ','), '') !~ 'security_invoker=(true|on|1)'),
  0,
  'svi view-ovi u public su security_invoker'
);

select * from finish();
rollback;
```

Primer testa po ulozi:
```sql
set local role anon;
select throws_ok(
  'select id from public.booking_requests',
  '42501',
  null,
  'anon nema pristup zahtevima'
);
reset role;

-- prijavljen korisnik: postavi claims pa ulogu
select set_config(
  'request.jwt.claims',
  '{"sub":"<uuid-editora>","role":"authenticated","aal":"aal2"}',
  true
);
set local role authenticated;
select is_empty('select id from public.booking_requests', 'editor ne vidi zahteve');
reset role;
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
```
