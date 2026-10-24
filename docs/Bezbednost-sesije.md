# Halo Travel: dobar plan, sedam rupa za zatvaranje

## Kratak zaključak

Plan je iznad proseka za Supabase + Next.js: `getClaims()` umesto `getSession()`, uloga i `is_active` iz baze na svakom zahtevu, aal2 proveren i u aplikaciji i u RLS-u, proxy samo za redirekcije. Sesije su dobro zamišljene, ali fale `Secure` flag, `no-store` zaglavlja, brzi opoziv kompromitovane sesije i ispravan offboarding (`auth.admin.signOut` ne prima ID korisnika). Najveći rizici nisu u sesijama nego oko njih: Supabase od **30. 10. 2026** više ne daje tabelama automatske GRANT-ove, `next` mora biti najmanje **16.4.1**, a nonce CSP ne radi sa statičkim shell-om koji pravi Cache Components. Sve navedeno se popravlja pre Koraka 2 (šema).

## Kako rade sesije i da li je pristup dobar

**Skladište.** `@supabase/ssr` čuva celu sesiju (access JWT, refresh token, user objekat) kao jedan JSON u kolačiću `sb-<project-ref>-auth-token`, kodiran kao base64url sa prefiksom `base64-`. Kad pređe 3180 znakova, deli se na `.0`, `.1` itd. Sa TOTP faktorom to će se često desiti, pa kolačić uvek čitajte kroz Supabase klijent, nikad ručno. Podrazumevano je `path=/`, `SameSite=Lax`, `httpOnly: false`, `maxAge` 400 dana i **bez `Secure`** ([ssr constants.ts](https://github.com/supabase/ssr/blob/main/src/utils/constants.ts), [chunker.ts](https://github.com/supabase/ssr/blob/main/src/utils/chunker.ts)).

**Životni ciklus.** Access JWT traje 1 h. Refresh token ne ističe sam, ali je jednokratan i rotira se. Ponovna upotreba posle 10 s (osim „roditelja“ aktivnog tokena) gasi celu sesiju ([Sessions](https://supabase.com/docs/guides/auth/sessions)). `getClaims()` prvo zove `getSession()` i osvežava token ako ističe za manje od 90 s. Zatim lokalno proverava potpis preko JWKS-a (keš 10 min). Kod HS256 tokena ili kad nema `kid`, tiho prelazi na mrežni `getUser()` ([GoTrueClient.ts](https://github.com/supabase/supabase-js/blob/master/packages/core/auth-js/src/GoTrueClient.ts)). Server Components ne mogu da upisuju kolačiće, pa `proxy.ts` mora da osveži sesiju pre njih. U `setAll` piše kolačiće i u request i u novi `NextResponse.next({ request })` ([zvanični primer](https://github.com/supabase/supabase/blob/master/examples/auth/nextjs/lib/supabase/proxy.ts)).

**MFA.** `aal` je claim u JWT-u. Ako ga nema, nivo je aal1. Put do aal2 je `enroll` → `challenge` → `verify`, a ssr upisuje novu sesiju na događaj `MFA_CHALLENGE_VERIFIED`. `aal` se čuva po sesiji u `auth.sessions`, pa refresh zadržava aal2. To je izvedeno iz modela, nije eksplicitno dokumentovano. Restriktivna politika C.2 doslovno prati Supabase šablon ([MFA](https://supabase.com/docs/guides/auth/auth-mfa)).

**Opoziv.** `getClaims()` ne vidi opoziv na serveru: posle odjave, promene lozinke ili reseta faktora access token važi do `exp` (≤ 1 h). `is_active` po zahtevu rešava deaktivaciju zaposlenog, ali ne rešava ukradenu sesiju aktivnog zaposlenog.

**Ocena: pristup je dobar.** Dopune:
- Dodati `cookieOptions: { secure: true }` u produkciji, i na browser i na server klijentu.
- U proxy `setAll` primeniti prosleđena cache zaglavlja. Na svaki `/admin` odgovor, uključujući redirekcije, postaviti `Cache-Control: private, no-store`.
- Dodati `private.session_is_valid()`: proverava da `session_id` iz JWT-a postoji u `auth.sessions` i da `not_after` nije prošao. Zvati je iz `private.has_role()` i `loadStaff`. Razmotriti JWT od 10–15 min.
- Obično dugme „Odjava“ treba da zove `signOut({ scope: 'local' })`, jer je podrazumevano `'global'`.
- Proveriti da projekat zaista izdaje ES256 tokene. U suprotnom je svaki `getClaims()` mrežni poziv.
- Opciono: HttpOnly kolačići, ako se upload prebaci na `createSignedUploadUrl` koji izdaje server. To je zaobilaznica zajednice, a ne zvanično rešenje ([diskusija #12303](https://github.com/orgs/supabase/discussions/12303)).
- „Inactivity timeout“ znači da nije bilo refresh-a. Otvoren admin tab sa browser klijentom ga zato nikad ne aktivira.

## Propusti po prioritetu

### Visoko

1. **GRANT model se menja 30. 10. 2026.** Nove tabele više nemaju automatske privilegije, a Prilog C nema nijedan `GRANT`, pa će `requireRole()` dobiti 42501 ([changelog](https://supabase.com/changelog/45329-breaking-change-tables-not-exposed-to-data-and-graphql-api-automatically)). Popravka u prvoj migraciji:
   - `alter default privileges … revoke all` ([Data API](https://supabase.com/docs/guides/database/data-api));
   - eksplicitni `grant` po tabeli i funkciji, uključujući `grant select on public.profiles to authenticated`;
   - `revoke truncate, references, trigger` za sve uloge, jer TRUNCATE zaobilazi RLS (lokalni test);
   - pgTAP testove za privilegije, ne samo za RLS.
2. **`create_booking_request` ostaje izvršiv za `authenticated`.** Plan radi `revoke … from public, anon`, ali direktan grant za `authenticated` preživljava (lokalni PG16 test). Popravka: `revoke … from public, anon, authenticated`, a funkciju napraviti kao `SECURITY INVOKER`.
3. **`set_user_role` može da ostavi 0 admina.** Dva admina koja istovremeno ražaluju jedan drugog oba prolaze, a `p_active = NULL` zaobilazi proveru. Popravka:
   - `is_active boolean not null default false`;
   - `if p_active is null then raise`;
   - `pg_advisory_xact_lock(hashtext('public.set_user_role'))` pre provere uloge i brojanja admina.
4. **Next.js verzija.** Bezbednosne zakrpe stižu samo na Active LTS minor. 16.3.x nije dobio ispravku kritičnog RCE-a u `use cache` od 14. 10. 2026 ([Next.js blog](https://nextjs.org/blog/nextjs-security-update-october-14-2026)). Popravka: `next >= 16.4.1` (trenutno 16.4.3), `react` ≥ 19.2.9, CI provera minimalne verzije, zakrpa u roku od 48 h za Critical.
5. **Nonce CSP i Cache Components.** PPR shell nema nonce i „nije kompatibilan“ sa nonce CSP-om ([CSP vodič](https://nextjs.org/docs/app/guides/content-security-policy)). Popravka:
   - poseban root layout za `/admin` sa `await connection()` van `<Suspense>`;
   - isključiti `/admin` iz globalnog CSP-a u `next.config`;
   - nonce generisati pre `createServerClient` i proslediti iste request headere u oba `NextResponse.next(...)` poziva, jer ga `setAll` inače briše.
6. **Offboarding.** Ova stavka se javlja u dve beleške. `auth.admin.signOut` prima JWT korisnika, ne ID ([referenca](https://supabase.com/docs/reference/javascript/auth-admin-signout)). Ispravan redosled:
   - `is_active=false`;
   - `updateUserById(id, { ban_duration })`;
   - brisanje redova iz `auth.sessions` kroz definer funkciju dostupnu samo service_role ulozi (neprovereno kao zvaničan API);
   - `mfa.deleteFactor`;
   - upis u `audit_log`.

   Uz `session_is_valid()` opoziv deluje odmah.

### Srednje

- **Server akcije su dostupne sa bilo koje putanje.** Proxy matcher i WAF pravila po putanji ih ne štite ([proxy.js](https://nextjs.org/docs/app/api-reference/file-conventions/proxy)). Rešenje: `requireRole()` prvo u svakoj admin akciji, CI grep koji to proverava, WAF uslov „Server Action Name“.
- **Invite i reset nisu PKCE tok.** Šablone mejla preusmeriti na `/admin/auth/confirm?token_hash=…` koji zove `verifyOtp`. Ruta mora biti u matcheru i imati `no-store` (neprovereno na trenutnim šablonima).
- **aal1 bootstrap.** MFA ekran gradite samo iz claims-a ili izuzmite SELECT sopstvenog reda u `profiles`. Uvedite `requireAal1Session()` za `/admin/mfa*`. `is_active=true` postaviti tek posle potvrđenog TOTP-a.
- **Javni kod ne sme da koristi cookie klijent.** ESLint `no-restricted-imports` treba da zabrani `@/lib/supabase/server` i `createBrowserClient` u javnim rutama. Javne akcije koriste klijent bez kolačića (`persistSession: false`).
- **Brisanje slika.** Trenutno nemoguće: postoji samo INSERT politika. Dodati DELETE i SELECT politiku za osoblje ili brisati na serveru secret ključem. Definisati i politike za `documents` bucket.
- **Append-only nije sproveden.** `audit_log` i `booking_events` treba da imaju `before update or delete` i `before truncate` trigere koji bacaju grešku.
- **Rate limit.** `private.hit_rate_limit()` nije dostupan preko `rpc()`, pa ga zvati unutar `create_booking_request` i koristiti atomični `insert … on conflict`. Unutar funkcije ponovo proveriti `published`, termin i granice broja putnika.
- **Excel zip bomb.** Pre `XLSX.read` proveriti nekompresovane veličine u ZIP central directory-ju ([SheetJS](https://docs.sheetjs.com/docs/api/parse-options)).
- **Turnstile.** Proveravati i `action`, dužinu ≤ 2048 i rok od 300 s. Na grešku siteverify-a odbiti zahtev ([Cloudflare](https://developers.cloudflare.com/turnstile/get-started/server-side-validation/)).
- **Lični podaci.** Ne logovati PostgREST `details`, jer CHECK greška ispisuje ceo red. Anonimizovati `tests/golden` pre commita. S3 ključeve za noćnu kopiju dodati u inventar tajni.
- **Claude Code deny pravila nisu bezbednosna granica** ([dokumentacija](https://code.claude.com/docs/en/permissions)). Uključiti sandbox ili PreToolUse hook i ne držati prod kredencijale na dev mašini.

### Nisko

- `safeHref` prihvata `//evil.com` i `/\evil.com`, a `safeRedirectPath` dozvoljava `/admin/../login`. Rešenje: vraćati kanonski `url.href` i zahtevati `/admin` ili `/admin/` prefiks.
- Deaktivirati legacy JWT ključeve. `drop extension if exists pg_graphql`. Isključiti Realtime „Allow public access“.
- Leaked-password zaštita, timebox i single-session zahtevaju Pro plan. Uneti ih u checklist za lansiranje.
- Na Vercelu ne postavljati `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY`.
- Ispraviti pgTAP testove koji prolaze na praznim tabelama, ne pokrivaju materialized view-ove i lažno javljaju grešku za `security_invoker=yes`.
- Dve Supabase Auth zakrpe iz 2026 (replay AAL1 tokena i provera bana) već su na hosted servisu. Lokalni CLI treba ažurirati ([CT4ZXP4Z](https://red.anthropic.com/2026/cvd/findings/ANT-2026-CT4ZXP4Z), [GQ8G77M4](https://red.anthropic.com/2026/cvd/findings/ANT-2026-GQ8G77M4)).

## Šta je već dobro

Pretnja je ispravno postavljena: Supabase API je direktan ulaz, a RLS i GRANT-ovi su poslednja linija odbrane. Helperi su u neizloženoj `private` šemi, sa `search_path=''` i `(select auth.uid())`. Restriktivna aal2 `FOR ALL` politika štiti i INSERT, a kolona-grant na `profiles` sprečava da korisnik sam sebi menja ulogu (oba lokalno testirana). Nema anon INSERT-a. Ostalo što je dobro: Turnstile, HMAC ključevi za rate limit i WAF; MIME i limit veličine na nivou bucket-a uz proveru magic bajtova; `"use cache"` samo za javne podatke; matcher ograničen na `/admin`; MCP isključen sa produkcije; svest da Storage ne ulazi u DB backup.

## Sources

- Supabase: [Sessions](https://supabase.com/docs/guides/auth/sessions), [Advanced SSR guide](https://supabase.com/docs/guides/auth/server-side/advanced-guide), [Next.js SSR](https://supabase.com/docs/guides/auth/server-side/nextjs), [MFA](https://supabase.com/docs/guides/auth/auth-mfa), [JWT Signing Keys](https://supabase.com/docs/guides/auth/signing-keys), [admin.signOut](https://supabase.com/docs/reference/javascript/auth-admin-signout), [Changelog 45329](https://supabase.com/changelog/45329-breaking-change-tables-not-exposed-to-data-and-graphql-api-automatically), [Data API hardening](https://supabase.com/docs/guides/database/data-api), [RLS](https://supabase.com/docs/guides/database/postgres/row-level-security), [Storage access control](https://supabase.com/docs/guides/storage/security/access-control)
- Izvorni kod: [supabase/ssr](https://github.com/supabase/ssr), [auth-js GoTrueClient.ts](https://github.com/supabase/supabase-js/blob/master/packages/core/auth-js/src/GoTrueClient.ts), [Supabase Next.js primer](https://github.com/supabase/supabase/tree/master/examples/auth/nextjs), [diskusija #12303](https://github.com/orgs/supabase/discussions/12303)
- Next.js: [Security update 14. 10. 2026](https://nextjs.org/blog/nextjs-security-update-october-14-2026), [CSP vodič](https://nextjs.org/docs/app/guides/content-security-policy), [proxy.js](https://nextjs.org/docs/app/api-reference/file-conventions/proxy), [Data Security](https://nextjs.org/docs/app/guides/data-security)
- Ostalo: [PostgreSQL CREATE POLICY](https://www.postgresql.org/docs/current/sql-createpolicy.html), [Cloudflare Turnstile](https://developers.cloudflare.com/turnstile/get-started/server-side-validation/), [SheetJS parse options](https://docs.sheetjs.com/docs/api/parse-options), [Vercel request headers](https://vercel.com/docs/headers/request-headers), [Claude Code permissions](https://code.claude.com/docs/en/permissions), [ANT-2026-CT4ZXP4Z](https://red.anthropic.com/2026/cvd/findings/ANT-2026-CT4ZXP4Z), [ANT-2026-GQ8G77M4](https://red.anthropic.com/2026/cvd/findings/ANT-2026-GQ8G77M4)
- Lokalni testovi na PostgreSQL 16 (grant-ovi, `set_user_role`, TRUNCATE) iz istraživačkih beleški, nisu ponovljeni na hosted Supabase-u.
