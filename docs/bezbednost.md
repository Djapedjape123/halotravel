# Bezbednost — Halo Travel platforma

> Plan bezbednosti za razvoj i lansiranje. Dopuna za `CLAUDE.md`, čuva se kao `docs/bezbednost.md`.
> Verzija 2 · 24.10.2026 · Predrag Radić (prWeb). Verzija 2 unosi nalaze spoljnog pregleda plana: GRANT model, opoziv sesije, nonce CSP i Cache Components, offboarding, `set_user_role`, `create_booking_request`, Storage politike.
> Ovo su tehničke smernice, ne pravni savet. Pravna pitanja (ZZPL, rokovi čuvanja, obrađivači) proverava pravnik klijenta.

---

## 0. Kako se koristi ovaj fajl

- Pre svakog koraka iz Roadmap-a otvori sekciju 17 (plan po koracima) i štikliraj stavke za taj korak.
- Claude Code: kratka pravila su već u `CLAUDE.md` (sekcija „Bezbednost“). Ovaj fajl čita pre rada na prijavi, ulogama, RLS-u, migracijama, uploadu, Excel uvozu, formi zahteva, mejlovima i headerima.
- Prilozi B–D (skice koda, SQL, env promenljive) su u `docs/bezbednost-prilozi.md`. Kad dođe odgovarajući korak, kod ide u pravi repo (tamo ga čuvaju testovi), a skica se briše iz priloga.
- Oznake: `ODLUKA-n` je odluka koja čeka; `TODO(halo)` i `TODO(itum)` znače isto kao u `CLAUDE.md`.
- Kad se nešto promeni, ažuriraj ovaj fajl i upiši odluku u `docs/odluke.md`.
- Skice koda u prilozima su polazna tačka. Tipove i API-je proveri u `node_modules/` pre upotrebe (isto pravilo kao u `CLAUDE.md`).

---

## 1. Šta štitimo i od koga

| Šta štitimo | Zašto |
|---|---|
| Lični podaci kupaca: imena, telefoni, mejlovi, uzrast i datumi rođenja dece | ZZPL, poverenje kupaca |
| Integritet objavljenih cena | pogrešna cena = gubitak novca ili spor sa kupcem |
| Netto cene i marže iz Excela partnera | poslovna tajna klijenta |
| Nalozi osoblja | nalog daje pristup cenama i podacima kupaca |
| Ključevi (Supabase secret, Resend, Turnstile) | secret ključ zaobilazi ceo RLS |
| Domen halotravel.rs i reputacija mejla | zloupotreba = spam sa njihovog domena |
| Dokazi (uvezeni fajlovi, poslate ponude) | Zakon o turizmu, eventualni sporovi |

**Ko napada / šta može da krene naopako**
- botovi i spameri (forma zahteva, prijava),
- konkurencija (skidanje cena),
- neko sa ukradenom lozinkom osoblja,
- pokvaren ili zlonamerni Excel fajl,
- kompromitovan npm paket,
- XSS kroz sadržaj (link u editoru, tekst iz Worda),
- prompt injection kroz AI alate (Claude Code + MCP čita tekst koji je uneo napadač),
- greška u kodu ili migraciji (najčešći realan uzrok).

**Ulazne tačke (svaka mora da se brani sama)**
1. Javne stranice i kalkulator (server akcija bez prijave)
2. Forma zahteva (server akcija bez prijave)
3. Prijava, reset lozinke, MFA
4. Admin server akcije i route handleri
5. **Supabase API direktno.** Publishable (anon) ključ je javan, pa svako može da zove `/rest/v1/<tabela>`, `/rest/v1/rpc/<funkcija>` i Storage mimo našeg sajta. Na tom putu jedina zaštita su RLS i GRANT-ovi.
6. Upload u Storage direktno iz browsera (slike, Excel)
7. Preview deploymenti na `*.vercel.app`
8. Nalozi: GitHub, Vercel, Supabase, Resend, DNS, mejl osoblja

---

## 2. Zlatna pravila

1. Svaka server akcija i route handler je javni endpoint: validacija (zod) → identitet → uloga → tek onda posao.
2. `proxy.ts` služi samo za preusmeravanje na prijavu. Nikad nije jedina zaštita.
3. RLS i GRANT-ovi su poslednja linija odbrane. Pišu se kao da napadač ima publishable ključ i nalog editora.
4. Sve je zabranjeno dok se izričito ne dozvoli: nova tabela, funkcija, bucket ili view nema pristup dok ga ne dodelimo i ne testiramo.
5. Server ne veruje klijentu: ni ceni, ni ulozi, ni MIME tipu, ni Tiptap JSON-u, ni imenu fajla.
6. Secret ključ samo u `src/lib/supabase/admin.ts` (`import 'server-only'`) i samo tamo gde RLS ne može drugačije.
7. Nema sirovog HTML-a. Svaki link prolazi kroz listu dozvoljenih protokola.
8. Lični podaci ne idu u logove, URL-ove, analitiku ni AI alate.
9. Svaka promena politika, GRANT-ova i `SECURITY DEFINER` funkcija dobija pgTAP test i ljudski pregled.
10. Nepoznato = odbij. Isto kao „nepoznato pravilo = greška" kod cena.

---

## 3. Autentikacija (prijavljuje se samo osoblje)

### 3.1 Podešavanja Supabase Auth-a
- [ ] Javna registracija isključena („Allow new users to sign up" off; lokalno `enable_signup = false` u `supabase/config.toml`). Nalozi samo po pozivu admina (`auth.admin.inviteUserByEmail` iz server akcije koju sme samo admin).
- [ ] Nov nalog nema ulogu ni pristup dok ga admin ne aktivira (`profiles.role = null`, `is_active = false`).
- [ ] Prijava: mejl + lozinka + TOTP. Bez magic linka i OAuth-a u MVP-u (manje ulaznih tačaka).
- [ ] Minimalna dužina lozinke 12. Zaštita od procurelih lozinaka ako je dostupna na planu (proveriti na Pro).
- [ ] Turnstile CAPTCHA na prijavi i resetu lozinke (Supabase Auth ga podržava). Zavisi od ODLUKA-2.
- [ ] Pregledani rate limiti za Auth endpoint-e u Supabase podešavanjima.
- [ ] Bez enumeracije naloga: prijava i reset lozinke vraćaju isti opšti odgovor („Pogrešan mejl ili lozinka“, „Ako nalog postoji, poslali smo mejl“) i za postojeći i za nepostojeći mejl, bez vidljive razlike u vremenu odgovora. Registracije nema, pa je poziv osoblja jedini način da nalog nastane.
- [ ] Sopstveni SMTP (Resend) za mejlove Auth-a (poziv, reset). Podrazumevani Supabase SMTP nije za produkciju.
- [ ] Site URL i Redirect URLs: samo produkcioni domen i tačan obrazac preview adresa ovog projekta. Nikad `https://*.vercel.app/**`.
- [ ] Asimetrični JWT ključevi za potpisivanje uključeni (tada `getClaims()` proverava potpis lokalno, bez poziva Auth serveru). Proveriti da projekat zaista izdaje ES256 tokene (`kid` u headeru tokena): kod HS256 tokena `getClaims()` tiho prelazi na mrežni `getUser()` pri svakom pozivu. Posle prelaska deaktivirati legacy JWT ključ.
- [ ] Mejlovi za poziv i reset lozinke vode na `/admin/auth/confirm?token_hash={{ .TokenHash }}&type=…`, koji na serveru zove `verifyOtp`. Podrazumevani `{{ .ConfirmationURL }}` vraća token u `#` delu adrese, koji server ne vidi. Ruta je u matcheru `proxy.ts` i ima `Cache-Control: no-store`. Proveriti na trenutnim šablonima.
- [ ] Kolačići sesije: `cookieOptions: { secure: true }` u produkciji, i na browser i na server klijentu. `@supabase/ssr` podrazumevano ne postavlja `Secure` (ostalo: `SameSite=Lax`, nije HttpOnly, `maxAge` 400 dana). Kolačić se uvek čita kroz Supabase klijent, nikad ručno: kad je velik (npr. sa TOTP faktorom), deli se na `.0`, `.1`…
- [ ] Trajanje sesije (ograničenje trajanja i odjava posle neaktivnosti) ako plan dozvoljava, npr. 12 h / 2 h. TODO(itum) + Halo. Timebox, inactivity timeout i single-session traže Pro plan. „Inactivity timeout“ meri vreme bez refresh-a, pa ga otvoren admin tab nikad ne aktivira. Razmotriti i kraći access JWT (10–15 min umesto 1 h).

### 3.2 MFA (TOTP)
- Obavezan za sve osoblje, jer se prijavljuje samo osoblje. Minimum su admin i sales. TODO(halo): da li svi imaju telefon za aplikaciju autentifikatora.
- Sprovodi se na dva mesta:
  - u aplikaciji: `requireRole()` odbija ako `aal !== 'aal2'` i šalje na ekran za MFA,
  - u bazi: RESTRICTIVE politika na svakoj tabeli i na `storage.objects` za `authenticated` (Prilog C.2).
- **Upis prvog faktora (enrollment).** Nov korisnik posle prve prijave ima `aal1` i nema faktor, pa restrictive aal2 politike blokiraju sve tabele, uključujući `profiles`. Zato:
  - postoji jedna `aal1` stranica `/admin/mfa` koja zove samo `supabase.auth.mfa.*` (enroll, challenge, verify) i ne čita nijednu tabelu,
  - `requireRole()` proverava `aal` iz `getClaims()` **pre** bilo kog čitanja `profiles` i vraća `mfa_required` (vodi na `/admin/mfa`),
  - restrictive politike važe samo za tabele i Storage, ne i za Auth API, pa upis faktora radi pri `aal1`,
  - posle upisa i potvrde koda sesija dobija `aal2` i tek tada se čita profil,
  - `/admin/mfa*` koristi `requireAal1Session()` (Prilog B.1): traži prijavu, ali ne `aal2`, i ne čita tabele,
  - admin aktivira nalog (`is_active = true`) tek kad korisnik ima potvrđen TOTP faktor; `set_user_role()` to sama proverava (Prilog C.4).
- Izgubljen telefon: samo admin resetuje faktor drugom korisniku, posle provere uživo ili telefonom, uz upis u `audit_log`. Ako admin izgubi pristup, Predrag resetuje preko Supabase dashboarda.

### 3.3 Provera identiteta na serveru
- Na serveru uvek `supabase.auth.getClaims()` (proverava potpis JWT-a). `getSession()` se nikad ne koristi za odluke o pristupu.
- `getClaims()` ne zna da li je sesija u međuvremenu ugašena na serveru. Zato se uz svaki zahtev iz baze čitaju i `profiles.role` i `profiles.is_active`. Deaktivacija naloga deluje odmah.
- To ne rešava ukradenu sesiju aktivnog zaposlenog: posle odjave, promene lozinke ili reseta faktora access token važi do `exp` (do 1 h). Zato `private.session_is_valid()` proverava da `session_id` iz JWT-a još postoji u `auth.sessions` i da `not_after` nije prošao. Zove se iz restrictive politike i iz `private.has_role()` (Prilog C.1, C.2), pa brisanje sesije (3.4) odmah deluje i na RLS i na `requireRole()`.
- Jedna funkcija `requireRole()` u `src/lib/auth/require-role.ts` (Prilog B.1). Svi admin ulazi je koriste.
- `proxy.ts` matcher pokriva samo `/admin` (i stranicu prijave). Javne, keširane stranice ne smeju da osvežavaju sesiju, jer bi keširan odgovor mogao da ponese tuđi `Set-Cookie`. Javni podaci se čitaju klijentom bez kolačića: ESLint `no-restricted-imports` u javnim rutama zabranjuje `@/lib/supabase/server` i `createBrowserClient`, a javne akcije koriste klijent sa `persistSession: false`.
- `proxy.ts` osvežava sesiju pre Server Components (one ne mogu da upisuju kolačiće). Na svaki `/admin` odgovor, uključujući preusmerenja, stavlja `Cache-Control: private, no-store` i zaglavlja koja `@supabase/ssr` prosledi u `setAll` (Prilog B.8).
- `?next=` posle prijave prihvata samo relativne putanje unutar `/admin`, proverene u kanonskom obliku (Prilog B.3).
- Obično dugme „Odjava“ zove `signOut({ scope: 'local' })`, jer je podrazumevano `'global'` (odjava sa svih uređaja). „Odjavi sa svih uređaja“ je posebno dugme sa `'global'`.
- Svaka osoba ima svoj nalog. Zajednički nalozi su zabranjeni, inače `audit_log` gubi smisao.

### 3.4 Kad neko od osoblja ode
Redosled (jedna server akcija, samo za admina):
1. `set_user_role(id, role, false)`: `is_active = false`, svaki sledeći zahtev je odbijen.
2. `auth.admin.updateUserById(id, { ban_duration })`: nalog više ne dobija nove tokene. `auth.admin.signOut()` ovde ne pomaže, jer prima JWT korisnika, a ne njegov ID.
3. `revoke_user_sessions(id)` (Prilog C.8): briše redove iz `auth.sessions`. Refresh tokeni prestaju da važe, a uz `session_is_valid()` i postojeći access token. Neprovereno kao zvaničan API; proveriti lokalno i na hosted verziji.
4. `auth.admin.mfa.deleteFactor` za sve faktore korisnika.
5. Upis u `audit_log`, pa pregled njegovih događaja za poslednjih 30 dana.

TODO(halo): ko javlja odlazak zaposlenog.

---

## 4. Autorizacija

### 4.1 Uloge
Tabela „Uloge — ko sme šta" iz `CLAUDE.md` važi. Dodaci:
- `profiles.role` i `profiles.is_active` menja samo admin, kroz funkciju `set_user_role()` (Prilog C.4). Funkcija proverava admin + aal2 i ne dozvoljava uklanjanje poslednjeg aktivnog admina. Promene uloga zaključava advisory lock-om, jer bi inače dva admina koja istovremeno ražaluju jedan drugog oba prošla. Odbija `p_active = null`, a aktivaciju dozvoljava samo uz potvrđen TOTP faktor. Svaku promenu upisuje u `audit_log`. `is_active` je `not null default false`.
- Korisnik u svom redu u `profiles` sme da menja samo dozvoljene kolone (npr. `full_name`), preko kolonskog GRANT-a (Prilog C.3).
- Uloga se nikad ne čita iz `user_metadata`, jer ga korisnik sam menja. Ako se ikad stavi u JWT, samo preko `app_metadata` ili Auth hook-a.
- Editor ne vidi zahteve, putnike, cenovnike ni uvoze.

### 4.2 Aplikacioni sloj (Next.js)
- Redosled u svakoj server akciji: zod (`.strict()`) → `requireRole()` → posao → `audit_log` → `updateTag` (Prilog B.2).
- Admin stranice zovu `requireRole()` u layout-u **i** u svakoj akciji i funkciji koja čita podatke. Zaštita layout-a ne štiti server akcije.
- Server akcije su dostupne sa bilo koje putanje (POST sa `Next-Action` headerom), ne samo sa stranice na kojoj se koriste. Zato ih proxy matcher i WAF pravila po putanji ne štite. `requireRole()` je prvi poziv posle zod-a u svakoj admin akciji. CI grep pada ako fajl sa `'use server'` u admin delu ne zove `requireRole(`. WAF pravila za akcije koriste uslov „Server Action Name“.
- `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` se ne postavlja na Vercelu; Next pravi ključ po build-u.
- Route handleri isto. Za POST sa kolačićima dodatno proveri `Origin`, jer server akcije to rade same, a route handleri ne.
- Čitanje podataka za admin ide kroz funkcije sa `import 'server-only'` koje vraćaju samo potrebna polja, ne cele redove.
- `"use cache"` samo za javne podatke. Nikad za funkcije koje čitaju kolačiće, sesiju ili lične podatke.
- Pregled nacrta je dinamički, poziva `requireRole()`, ima `noindex` i ne kešira se.
- Admin koristi Supabase klijent sa JWT-om prijavljenog korisnika, pa RLS važi. Secret ključ samo za: poziv osoblja, odjavu drugih korisnika, upis zahteva kupca (4.4) i, ako ODLUKA-1 tako ispadne, server-only čitanje cenovnika.
- `serverActions.allowedOrigins` ne podešavati osim ako je baš potrebno, i nikad ne dodavati `null`.

### 4.3 Baza: RLS, GRANT, funkcije, view-ovi
- **GRANT model.** Supabase novim projektima od 30.05.2026, a postojećim od 30.10.2026, ne daje tabelama u `public` automatske privilegije. Prva migracija to piše i eksplicitno (`alter default privileges … revoke`, Prilog C.0), da lokalna baza, dev i produkcija rade isto.
  - Svaka tabela u istoj migraciji dobija `grant` samo za operacije za koje ima politiku, po ulozi. Bez `grant select on public.profiles to authenticated` `requireRole()` dobija grešku 42501.
  - TRUNCATE (zaobilazi RLS), REFERENCES i TRIGGER ne dobija niko. PostgREST TRUNCATE ne izlaže, ali to je odbrana u dubini.
  - pgTAP proverava privilegije, ne samo politike (Prilog C.5).
- RLS uključen na svakoj tabeli u `public`. pgTAP test pada ako nije (Prilog C.5).
- Politike se pišu po operaciji i po ulozi (`to authenticated`), uz helper `private.has_role(...)` (Prilog C.1).
- Tabele sa ličnim podacima (`booking_requests`, `travelers`, `quotes`, `proformas`, `audit_log`): pored RLS-a i `revoke all ... from anon`.
- **Funkcije.** Postgres podrazumevano daje EXECUTE svima, a na Supabase-u je svaka funkcija u `public` dostupna preko `/rest/v1/rpc/`, i anonimnim korisnicima. Zato:
  - helperi (`has_role`, rate limit) idu u šemu `private`, koja nije izložena kroz API,
  - svaka `SECURITY DEFINER` funkcija ima `set search_path = ''`, potpuno kvalifikovana imena tabela, proveru uloge **unutar** funkcije, `revoke execute ... from public, anon, authenticated` i `grant` samo onima kojima treba. Direktan grant za `authenticated` preživljava revoke od `public` (lokalni test na PG16),
  - `SECURITY DEFINER` funkcije zaobilaze RLS, pa i restrictive aal2 politiku; zato i one proveravaju aal i važeću sesiju (kroz `private.has_role()`, Prilog C.1),
  - pgTAP proverava da anon ne može da izvrši nijednu `SECURITY DEFINER` funkciju u `public`, a da `authenticated` izvršava samo funkcije sa izričite liste.
- **View-ovi** zaobilaze RLS ako nemaju `with (security_invoker = true)`. Svaki view ga mora imati (test).
- **Materialized view** ne podržava ni RLS ni `security_invoker`. U `public` nema privilegija ni za `anon` ni za `authenticated` (test).
- Nepromenljivi zapisi:
  - poslata ponuda (posle `quote_sent`): trigger zabranjuje UPDATE,
  - `booking_events` i `audit_log`: samo INSERT, bez UPDATE i DELETE za bilo koju ulogu. Pored izostanka politika, `before update or delete` i `before truncate` trigeri bacaju grešku (Prilog C.7), jer `service_role` i vlasnik tabele zaobilaze RLS. Posledica: brisanje ličnih podataka na zahtev (§14) anonimizuje zahtev umesto da briše red, a `booking_events` ne sadrži lične podatke,
  - redovi objavljenog cenovnika se ne menjaju; nova verzija = novi uvoz; korekcije su zasebna tabela.
- Supabase Security Advisor (dashboard) bez upozorenja pre svakog merge-a koji menja bazu i pre lansiranja.
- **Površina Data API-ja.** Svaka uključena Supabase usluga je dodatna ulazna tačka za nekoga sa publishable ključem:
  - `pg_graphql` je podrazumevano uključen i izlaže šemu (introspekciju) preko `/graphql/v1`; ne koristimo ga, pa `drop extension if exists pg_graphql`,
  - Realtime isključiti (ili ostaviti bez ijedne tabele u publikaciji) dok ga ne koristimo, i isključiti „Allow public access“ u podešavanjima Realtime-a,
  - izložena samo šema `public` (Data API → Exposed schemas); `private` nikad,
  - `db-extra-search-path` ostaje na podrazumevanoj vrednosti, bez dodatnih šema,
  - ručna provera pre lansiranja: `curl` sa publishable ključem na `/graphql/v1` mora da vrati grešku.

### 4.4 Javni (anon) pristup — konačna lista

| Šta | Anon | Napomena |
|---|---|---|
| Objavljene destinacije, ponude, sadržaj, slike | SELECT | samo `status = 'published'` |
| Aktivni popusti | SELECT | samo polja koja sajt prikazuje |
| Cena „od" | SELECT | kolona na ponudi, računa je motor pri objavi |
| Redovi cenovnika | **ODLUKA-1** | vidi ispod |
| Zahtevi, putnici, ponude, predračuni, uvozi, audit | ništa | |
| INSERT u bilo koju tabelu | ništa | zahtev kupca upisuje server |

**Upis zahteva kupca.** Anon nema INSERT politiku. Server akcija posle validacije i Turnstile-a poziva funkciju `create_booking_request(...)` preko admin klijenta (Prilog C.6).
- Funkcija je `SECURITY INVOKER`: poziva je samo `service_role`, koji ionako zaobilazi RLS, pa definer ne donosi ništa osim rizika.
- Izvršiva je samo za `service_role`: `revoke … from public, anon, authenticated`. Bez revoke-a od `authenticated` svaki prijavljen korisnik bi mogao da je pozove direktno, mimo Turnstile-a i rate limita.
- U jednoj transakciji: rate limit (6.3), ponovna provera da je ponuda objavljena, da termin pripada ponudi i nije stop sale i granica broja putnika, upis zahteva i prvog reda u `booking_events`.

Tako niko ne može da zaobiđe formu direktnim pozivom API-ja, pa ni prijavljen korisnik.

**ODLUKA-1 — da li su cenovnici javno čitljivi?**
`CLAUDE.md` je ranije (sekcija „Baza i Supabase“) govorio da anon čita objavljene verzije cenovnika. To je u redu samo ako u bazi stoje krajnje cene. Ako Excel partnera sadrži netto cene (otvoreno pitanje iz sekcije „Otvorena pitanja“ u `CLAUDE.md`), javno čitanje otkriva nabavne cene i maržu svakome ko ima publishable ključ.
- Predlog dok ne stigne odgovor: anon **nema** pristup tabelama cena. Kalkulator (server akcija) čita objavljeni cenovnik kroz server-only modul `src/lib/pricing/load-published.ts` sa secret ključem, samo SELECT objavljenih verzija, i vraća samo rezultat motora.
- Ne rešavati preko view-a: view bez `security_invoker` zaobilazi RLS, a sa njim anon i dalje mora da čita osnovnu tabelu.
- Privremena odluka (anon bez pristupa cenama) upisana je 22.10.2026. u `docs/odluke.md` i u `CLAUDE.md` („Baza i Supabase“). Konačnu odluku upisati isto tamo.

---

## 5. XSS i sadržaj

### 5.1 Tiptap
- Pri čuvanju, na serveru: zod šema dozvoljenih čvorova i oznaka (`doc`, `paragraph`, `heading` 2–3, `bulletList`, `orderedList`, `listItem`, `text`, `hardBreak`, `bold`, `italic`, `link`). Sve ostalo se odbija. Ograničiti dubinu i veličinu JSON-a (npr. 200 KB).
- Linkovi: `href` samo `http:`, `https:`, `mailto:`, `tel:` (Prilog B.4). Proverava se i pri čuvanju i pri renderovanju, jer je renderer poslednja linija. Spoljni linkovi dobijaju `rel="noopener noreferrer"`.
- Renderer su naše React komponente. `dangerouslySetInnerHTML` je zabranjen u celom projektu (ESLint `react/no-danger` kao greška). Jedini izuzetak je jedna komponenta za JSON-LD koja escapuje `<` kao `\u003c`.
- Lepljenje iz Worda: Tiptap šema odbacuje nepoznato. Dodati test sa pravim Word HTML-om (stilovi, `<o:p>`, komentari).

### 5.2 Ostali izlazi
- Sav tekst iz baze (naziv hotela iz Excela, napomene, imena) React renderuje kao tekst, nikad kao HTML.
- React-PDF ne tumači tekst kao HTML. Dovoljno je da se iz korisničkog unosa ne prave klikabilni linkovi.
- Mejl šabloni: tekst kroz komponente (escapuje se). Bez ručnog spajanja HTML stringova.
- Eventualni CSV izvoz (kasnije): ćelije koje počinju sa `=`, `+`, `-`, `@` dobijaju apostrof ispred (zaštita od formula u Excelu osoblja).

### 5.3 Zašto je XSS u adminu posebno opasan
Kolačići sesije iz `@supabase/ssr` po defaultu nisu HttpOnly, jer ih browser klijent koristi (između ostalog za direktan upload). XSS u adminu znači krađu sesije osoblja, a sa njom i pristup ličnim podacima i cenama. Zato admin dobija najstroži CSP (sekcija 9).

---

## 6. Forma zahteva, kalkulator i mejl

### 6.1 Forma zahteva
- zod šema sa `.strict()`, ista na klijentu i serveru, sa granicama:
  - ime i prezime: slova (`\p{L}`), razmak, crtica, apostrof, 2–80 znakova (bez tačaka i kosih crta, pa ni linkova u mejlu),
  - telefon: cifre, `+`, razmak, `/`, `-`, 6–20 znakova,
  - mejl: standardna provera, do 254 znaka,
  - odrasli 1–9 (TODO(halo): maksimum), deca 0–6, uzrast celi broj 0–17,
  - napomena: čist tekst, do 1000 znakova,
  - izbor iz kalkulatora: samo ID-jevi. Server proverava da postoje, da pripadaju toj ponudi, da je ponuda objavljena i da termin nije stop sale.
- Cena iz forme se ignoriše (već u `CLAUDE.md`).
- Odbrana od botova, u slojevima:
  1. honeypot polje (skriveno, `autocomplete="off"`, `tabIndex={-1}`),
  2. Cloudflare Turnstile: token se proverava na serveru tajnim ključem, uz proveru `success`, `hostname` i `action`. Token je jednokratan, najviše 2048 znakova i ne stariji od 300 s. Ako poziv siteverify-a ne uspe (mreža, timeout), zahtev se odbija,
  3. rate limit (6.3),
  4. Vercel WAF pravilo za POST zahteve forme (6.3).
- **ODLUKA-2:** Turnstile je nova zavisnost (skripta + serverski poziv, bez obaveznog npm paketa). Treba ga odobriti i dodati u Stack tabelu i u politiku privatnosti.
- Kupac dobija opštu poruku o grešci, bez razloga odbijanja. Detalj ide u log, bez ličnih podataka.
- Stranica potvrde prikazuje samo broj zahteva, bez ličnih podataka iz baze.

### 6.2 Broj zahteva
- Primarni ključ je uuid. Broj za kupca može biti čitljiv (npr. `HT-2026-00123`), jer u MVP-u ne postoji javna stranica za proveru zahteva.
- Ako se ikad doda javni pregled statusa: link sa dugim nasumičnim tokenom, nikad samo broj.

### 6.3 Rate limit
Vercel funkcije nemaju zajedničku memoriju, pa brojač ne može biti u promenljivoj.
- **Sloj 1 — Vercel WAF:** pravilo za POST na rute forme i kalkulatora (server akcije šalju POST na putanju stranice, sa `Next-Action` headerom). Proveriti limite plana.
- **Sloj 2 — brojač u Postgres-u:** tabela `private.rate_limits` i funkcija `private.hit_rate_limit(key, limit, window)`. Brojač je atomičan (`insert … on conflict do update … returning`, Prilog C.6). Šema `private` nije izložena kroz API, pa se funkcija ne zove preko `rpc()`: forma je zove unutar `create_booking_request`, a kalkulator kroz tanak omotač `public.check_rate_limit()`, izvršiv samo za `service_role`. Početni ključevi:
  - `zahtev:ip:<hmac>` — npr. 5 na 10 min,
  - `zahtev:mejl:<hmac>` — npr. 3 na 24 h (štiti tuđe inboxe),
  - `kalkulator:ip:<hmac>` — npr. 120 na minut.
- IP i mejl se ne čuvaju sirovi, nego kao HMAC-SHA256 sa tajnom `RATE_LIMIT_SECRET`. Redovi stariji od 48 h se brišu.
- **IP adresa je obavezna stavka koraka 14, ne preporuka.** `x-forwarded-for` klijent može da lažira ako ga platforma samo dopuni, pa bi napadač svakim zahtevom menjao ključ brojača. Pre pisanja `rate-limit.ts` u Vercel dokumentaciji potvrditi koji header platforma sama postavlja i prepisuje (`x-real-ip` ili `x-vercel-forwarded-for`), i čitati samo njega. Test: zahtev sa lažnim `x-forwarded-for` ne menja ključ. Dok se ne potvrdi, brojač po IP-u ne vredi, pa se oslanjamo na limit po mejlu i WAF.
- Brojevi su početni predlog i podešavaju se posle prvih nedelja rada.

### 6.4 Mejl
- Potvrda kupcu ide na adresu koju je kupac upisao i koja nije proverena. To je kanal za zloupotrebu (slanje sa halotravel.rs na tuđe adrese). Zato:
  - limit po mejlu (6.3),
  - u mejl kupcu ne ide slobodan tekst (napomena); samo broj zahteva, ponuda, termin, stavke cene i kontakt agencije,
  - ime ide u mejl tek posle validacije iz 6.1,
  - subject je fiksan tekst + broj zahteva, bez korisničkog unosa.
- Resend: API ključ samo za slanje, ograničen na domen; odvojeni ključevi za dev i produkciju.
- DNS: SPF, DKIM (Resend) i DMARC (prvo `p=none` sa izveštajima, zatim `quarantine`). TODO(halo): pristup DNS-u.
- Mejl osoblju sadrži lične podatke. TODO(halo): da li mejl zajedničkog inboxa ima 2FA.

### 6.5 Kalkulator
- Javna server akcija bez prijave: zod sa istim granicama, rate limit (6.3), keširano učitavanje objavljenog cenovnika.
- Vraća samo `PriceResult` (stavke i total), nikad sirove redove cenovnika.

---

## 7. Upload fajlova

### 7.1 Slike (bucket `images`)
- Podešavanja bucketa u migraciji/configu: `allowed_mime_types` = `image/jpeg`, `image/png`, `image/webp`; `file_size_limit` = 10 MB. SVG nikad.
- Bucket proverava MIME prema `Content-Type` koji šalje klijent, a to može da se lažira. Storage politika (RLS) ne vidi sadržaj fajla. Zato je tok:
  1. browser traži od servera putanju `offers/<offer_id>/<uuid>.<ext>`; server proverava ulogu i da ponuda postoji,
  2. browser uploaduje direktno u Storage (INSERT politika: samo editor/sales/admin, samo prefiks `offers/`; bez UPDATE politike, pa ni upsert-a),
  3. server akcija `registerImage()` čita prvih 16 bajtova i proverava potpis: JPEG `FF D8 FF`, PNG `89 50 4E 47 0D 0A 1A 0A`, WebP `RIFF????WEBP`. Ako ne odgovara, briše fajl i vraća grešku. Tek tada upisuje sliku u tabelu.
- Fajl u Storage-u bez reda u bazi, stariji od 24 h, je smeće (kasnije automatsko čišćenje).
- Brisanje slike briše i fajl (`CLAUDE.md`). `remove()` traži DELETE **i** SELECT politiku na `storage.objects` za osoblje, ograničenu na bucket `images` i prefiks `offers/` (Prilog C.9). Bez njih brisanje ne radi.
- `next/image` `remotePatterns`: samo `https://<ref>.supabase.co/storage/v1/object/public/images/**`. `dangerouslyAllowSVG` ostaje isključen.
- Anon nema SELECT politiku na `storage.objects` (nema listanja). Javne slike se čitaju preko javnog URL-a sa uuid putanjom. Slike iz nacrta vidi ko zna URL, što je prihvatljivo.
- Uklanjanje EXIF podataka (GPS) je opciono i nisko prioritetno za hotelske fotografije.

### 7.2 Excel (bucket `imports`)
- Privatan. INSERT i SELECT samo sales/admin (+ aal2). Bez UPDATE i DELETE, jer je fajl dokaz.
- Dozvoljeni `.xlsx` i `.xls`; limit veličine npr. 15 MB (podesiti kad stignu pravi fajlovi). Browser za `.xls` ponekad šalje `application/octet-stream`, pa je prava provera potpis na serveru (sekcija 8).

### 7.3 Dokumenti (bucket `documents`)
- Privatan, samo sales/admin. Osoblje pristupa preko signed URL-a kratkog roka (npr. 60 s).
- Politike: INSERT i SELECT za sales/admin; bez UPDATE i DELETE, jer je poslata ponuda dokaz (Prilog C.9).
- Kupac dobija PDF ponude i predračuna **kao prilog mejla**, ne kao signed URL. Kratak rok bi istekao pre nego što kupac otvori mejl, a dug rok je javni link do ličnih podataka. Primenjeno u `CLAUDE.md` („Ponuda i predračun“).

---

## 8. Excel uvoz

- Pre parsiranja proveri potpis fajla: `.xlsx` je ZIP (`50 4B 03 04`), `.xls` je OLE2 (`D0 CF 11 E0 A1 B1 1A E1`). SheetJS sam prepoznaje i druge formate (CSV, HTML, SYLK...), pa bi bez ove provere prihvatio i HTML preimenovan u `.xls`.
- SheetJS opcije čitanja: bez generisanja HTML-a iz ćelija (`cellHTML: false`), bez formula (`cellFormula: false`), bez VBA (podrazumevano isključeno), limit redova (`sheetRows`), `dense: true`. Formule se nikad ne izvršavaju, čitaju se samo sačuvane vrednosti. Tačna imena opcija proveriti u dokumentaciji verzije 0.20.3.
- Ograničenja: veličina fajla, broj listova, broj redova, `maxDuration` route handlera. Prekoračenje daje jasnu grešku, nikad tihu obradu dela fajla.
- **Zip bomb.** `.xlsx` je ZIP, pa mali fajl (limit 15 MB) može da se raspakuje u gigabajte i sruši funkciju pre nego što limit redova išta znači. Pre SheetJS-a pročitati ZIP direktorijum (bez raspakivanja) i odbiti fajl ako zbir `uncompressed size` zapisa, broj zapisa ili odnos kompresije prelaze limit (konkretne brojeve podesiti na pravim fajlovima). `sheetRows` ograničava redove, ali ne štiti od ogromnog raspakovanog XML-a. Za `.xls` (OLE2) važi samo limit veličine i `maxDuration`.
- Svaka ćelija je nepouzdan ulaz: zod validacija (brojevi u opsegu, ispravni datumi, nazivi ograničene dužine).
- Bez `eval` i bez dinamičkih ključeva iz ćelija (`obj[vrednostCelije] = ...`), zbog prototype pollution-a. Koristiti `Map` ili `Object.create(null)`.
- Provere pre „Objavi" (štite od pokvarenog fajla i od greške u profilu):
  - cena ≤ 0 blokira objavu,
  - promena cene veća od ±X% (npr. 30%, TODO(halo)) je posebno označena i traži drugu potvrdu,
  - uklonjeno više od N hotela u jednom update-u daje upozorenje,
  - objava se odbija ako se objavljena verzija promenila otkad je nacrt napravljen (dva čoveka istovremeno).
- Funkcija „Objavi" u bazi sama proverava sales/admin + aal2 (ne veruje aplikaciji) i upisuje `audit_log`.
- Predlog za korak 6: dugme „Vrati prethodnu verziju" (prethodna `archived` verzija se ponovo objavljuje kao nova). To je najbrži odgovor na pogrešno objavljene cene. Poslate ponude se ne menjaju, jer su snapshot.
- SheetJS se instalira sa njihovog CDN-a (`CLAUDE.md`), pa ga Dependabot ne prati. Pri svakom većem koraku ručno proveriti `https://cdn.sheetjs.com/advisories/`. Opcija: tarball u `vendor/` i instalacija preko `file:` (opisano u SheetJS dokumentaciji), da build ne zavisi od dostupnosti CDN-a.

---

## 9. HTTP headeri i CSP

### 9.1 Headeri za sve rute
U `next.config.ts` kroz `headers()` (Prilog B.5): `X-Content-Type-Options`, `Referrer-Policy`, `X-Frame-Options`, `Permissions-Policy`, HSTS, `poweredByHeader: false`, a za `/admin` još i `X-Robots-Tag: noindex`.
- HSTS: `max-age=31536000`. `includeSubDomains` i `preload` tek kad se proveri da svi poddomeni halotravel.rs rade preko HTTPS-a (TODO(halo): DNS). Proveriti šta Vercel već šalje.

### 9.2 CSP — dve strategije
- CSP sa nonce-om zahteva dinamičko renderovanje, a to se kosi sa keširanjem javnih stranica (`"use cache"`, cilj LCP < 2,5 s). Zato:
  - **`/admin`** (ionako dinamički, a tu su sesije i lični podaci): strogi CSP sa nonce-om iz `proxy.ts` (`script-src 'self' 'nonce-…' 'strict-dynamic'`), bez GA4 i Meta piksela.
    - Nonce ne radi sa statičkim shell-om koji pravi Cache Components. CSP vodič u `node_modules/next/dist/docs/` kaže da je PPR „incompatible with nonce-based CSP“. Zato `/admin` ima sopstveni root layout (`src/app/(admin)/admin/layout.tsx`) sa `export const instant = false` i `await connection()` van `<Suspense>`, pa se ceo admin renderuje po zahtevu (Prilog B.9). Proveriti na build-u u koraku 7.
    - Globalni CSP iz `next.config.ts` ne važi za `/admin` (izuzet u `source`, Prilog B.6), da se dve politike ne seku.
    - Nonce se generiše pre `createServerClient`, a isti request headeri (sa nonce-om) idu u oba `NextResponse.next(...)` poziva, jer `setAll` pravi nov odgovor i inače bi ih izgubio (Prilog B.8).
  - **Javni sajt:** statički CSP bez nonce-a iz `next.config.ts`. `script-src` tu mora da ima `'unsafe-inline'` zbog Next.js inline skripti, pa je zaštita od XSS-a slabija i glavna odbrana su pravila iz sekcije 5. CSP i dalje vredi za `frame-ancestors`, `object-src`, `base-uri`, `form-action` i `connect-src`.
- Prvo `Content-Security-Policy-Report-Only` na preview-u, pa prava politika kad prijave prestanu.
- Pre pisanja: pročitati CSP vodič u `node_modules/next/dist/docs/` i tačne domene GA4, Meta piksela i Turnstile-a iz njihove dokumentacije.
- Skice obe politike su u Prilogu B.6.

---

## 10. Tajne, ključevi i okruženja

### 10.1 Okruženja

| Okruženje | Supabase | Podaci | Ko pristupa |
|---|---|---|---|
| Lokalno | Docker (`supabase start`) | seed | developer, Claude Code |
| Preview (`*.vercel.app`) | poseban dev projekat | seed i test podaci, bez pravih kupaca | developer, Stefan, Halo za pregled |
| Produkcija | produkcioni projekat | pravi podaci | samo produkcioni deploy |

- Preview nikad ne dobija produkcione ključeve. Vercel env promenljive se podešavaju po okruženju (Production / Preview / Development).
- Pravi lični podaci kupaca nikad ne idu u dev ili preview bazu.

### 10.2 Ključevi
- Preći na nove Supabase ključeve: `sb_publishable_…` (javni) i `sb_secret_…` (server). Secret ključ može da se odmah opozove i zameni, a iz browsera uvek vraća 401.
- Prefiks `NEXT_PUBLIC_` samo za publishable ključ, URL, GA/Meta ID-jeve i Turnstile site key.
- U CI posle build-a: `grep -R "sb_secret" .next/static` mora biti prazno.
- `.env*` u `.gitignore`, osim `.env.example` bez vrednosti.
- Predlog spiska env promenljivih je u Prilogu D.

### 10.3 Rotacija
- U runbook-u (sekcija 15) stoji spisak ključeva, gde se koriste i kako se rotiraju: Supabase secret, Resend, Turnstile secret, `RATE_LIMIT_SECRET`, S3 ključevi za noćnu kopiju Storage-a (15.2).
- Rotacija odmah ako ključ završi u git-u, logu, chatu ili AI alatu, ili ako ode osoba koja mu je imala pristup.
- Pre lansiranja produkcija dobija nove ključeve; razvojni se ne prenose.

---

## 11. Zavisnosti i lanac nabavke

- `package-lock.json` je u repou; CI koristi `npm ci`, ne `npm install`.
- Dependabot alerts i security updates. Ako je dostupno, period čekanja (`cooldown`) od nekoliko dana za nove verzije, jer se kompromitovani paketi obično otkriju u prvih par dana.
- `npm audit --audit-level=high` u CI.
- Nova biblioteka = prvo pitanje (već u `CLAUDE.md`). Pre dodavanja proveriti održavanje, popularnost i install skripte.
- Next.js i React uvek na poslednjem patch-u svoje verzije. U 2025–2026. bilo je ozbiljnih ranjivosti baš u delovima koje koristimo (zaobilaženje middleware-a posebnim headerom, RCE u React Server Components, CSRF u server akcijama sa `Origin: null`). Pratiti GitHub Security Advisories za `vercel/next.js` i `facebook/react`.
  - Minimum: `next` ≥ 16.4.1 (izdanje od 14.10.2026 ispravlja kritični RCE u `use cache`, a mi koristimo Cache Components) i `react` ≥ 19.2.9. Trenutno: `next` 16.4.2, `react` 19.3.0.
  - Zakrpe stižu samo na Active LTS minor, pa ne ostajemo na starom minor-u.
  - CI pada ako je verzija ispod minimuma. Critical zakrpa se primenjuje u roku od 48 h.
- Lokalni Supabase CLI se redovno ažurira. Dve Auth zakrpe iz 2026 (replay AAL1 tokena, provera bana) već su na hosted servisu, ali ne i u starom lokalnom image-u.
- GitHub Actions: `permissions: contents: read` po defaultu, akcije zakucane na commit SHA, bez produkcionih tajni u CI.

---

## 12. Nalozi i infrastruktura

- [ ] 2FA na GitHub-u, Vercel-u, Supabase-u, Resend-u, kod registra domena i na DNS-u (i kod Halo-a i kod ITum-a).
- [ ] GitHub: privatan repo, zaštita `main` grane (obavezan PR, zelen CI: typecheck, lint, test, pgTAP), bez force push-a.
- [ ] Secret scanning i push protection ako su dostupni; u svakom slučaju gitleaks kao pre-commit i u CI.
- [ ] Vercel Deployment Protection (Standard Protection, dostupna na svim planovima) proverena za preview adrese. Za Stefana i Halo deljivi linkovi ili Vercel nalog.
- [ ] Vercel funkcije u `fra1`, Supabase u `eu-central-1`.
- [ ] Supabase organizacija: što manje članova. Produkcioni projekat nije povezan sa Claude Code-om.
- [ ] Prenos naloga na klijenta ili ITum (TODO) uključuje predaju 2FA i kodova za oporavak i rotaciju svih ključeva.

---

## 13. AI alati (Claude Code, MCP)

- `.claude/settings.json` u repou (Prilog B.7): zabrana čitanja `.env` fajlova, potvrda za `git push`, zabrana komandi nad produkcijom.
- `deny` pravila **nisu bezbednosna granica** (dokumentacija Claude Code-a): `Read(./.env)` ne sprečava `cat .env` kroz Bash. Za pravu zaštitu: sandbox za Bash ili PreToolUse hook, i bez produkcionih kredencijala na dev mašini.
- Supabase MCP: samo lokalna ili dev baza, u read-only režimu i ograničen na dev projekat. **Ne povezivati ga sa produkcijom u MVP-u** (primenjeno u `CLAUDE.md`, „Baza i Supabase“). Napomena kupca, tekst iz Worda i Excel ćelije mogu sadržati „uputstva" za agenta (prompt injection), a produkcija sadrži lične podatke.
- **ODLUKA-3:** kad Claude Code čita `tests/fixtures/`, sadržaj pravih Excel fajlova i starih obračuna ide modelu. Potvrditi sa Halo-om da je to u redu, ili raditi sa anonimizovanim kopijama gde je moguće. Proveriti podešavanja privatnosti naloga.
- Svaku migraciju, RLS politiku, GRANT, Storage politiku i `SECURITY DEFINER` funkciju koju napiše AI čovek pregleda liniju po liniju. Tipične greške: `using (true)`, zaboravljen `revoke`, politika za pogrešnu ulogu, `security definer` bez `search_path`.
- Lični podaci kupaca nikad ne idu u prompt, chat, issue ni log.

---

## 14. Lični podaci (ZZPL)

Nije pravni savet; proverava pravnik klijenta.
- Minimum podataka: forma traži samo ono što je navedeno u `CLAUDE.md`. Datume rođenja unosi osoblje tek kad unosi putnike.
- Pristup: samo sales i admin (RLS). Editor nema pristup.
- Rokovi čuvanja: TODO(halo) + pravnik. Na primer, nerealizovani zahtevi se brišu ili anonimizuju posle X meseci; realizovani se čuvaju po propisima o računovodstvu i turizmu. Kad se dogovori, dodati zakazano čišćenje.
- Prava lica (uvid, ispravka, brisanje): ručni postupak u adminu, samo za admina, uz upis u `audit_log`.
- Obrađivači podataka koje treba navesti u politici privatnosti: Supabase, Vercel, Resend, Google (GA4), Meta, Cloudflare (Turnstile). TODO pravnik: ugovori o obradi i prenos podataka.
- Logovi: nikad telo zahteva, ime, telefon, mejl ni uzrast. Ni PostgREST polje `details`, jer CHECK greška u njemu ispisuje ceo red. Greške se loguju sa ID-jem zahteva i kodom greške; klijent dobija opštu poruku.
- Stari obračuni za `tests/golden` dobijaju anonimizovana imena i kontakte pre prvog commita. Cene i ostali ulazi motora ostaju netaknuti, jer su golden vrednosti istina. Vidi i ODLUKA-3.
- Brisanje na zahtev znači anonimizaciju reda, jer su `booking_events` i `audit_log` append-only (4.3).
- Analitika: GA4 i Meta tek posle pristanka (već u `CLAUDE.md`). Bez ličnih podataka u URL-ovima i događajima. Meta „automatic advanced matching" isključen.
- Curenje podataka: ZZPL predviđa obaveštavanje Poverenika (rok 72 h od saznanja). Postupak je u runbook-u; potvrditi sa pravnikom.

---

## 15. Audit, backup i incidenti

### 15.1 `audit_log`
Tabela `audit_log (id, at, actor_id, action, entity, entity_id, details jsonb)`. Samo INSERT; čita samo admin; bez ličnih podataka u `details`. Upisuje se:
- promena uloge, aktivacija i deaktivacija naloga, reset MFA,
- uvoz Excela, objava i vraćanje cenovnika,
- korekcije cena i popusti (napravljeno / izmenjeno / obrisano),
- objava i povlačenje aranžmana,
- slanje ponude i predračuna, brisanje ličnih podataka na zahtev.

Statusi zahteva već idu u `booking_events`, a prijave beleži Supabase Auth.

### 15.2 Backup
- Supabase Pro pravi dnevni backup baze. **Backup baze ne sadrži fajlove iz Storage-a**, samo njihove metapodatke. Bucketi `imports` (dokaz) i `documents` (poslate ponude) moraju se posebno kopirati, npr. noćna kopija preko S3-kompatibilnog endpoint-a Supabase Storage-a. TODO(itum) + Halo: gde se čuvaju kopije.
- Test vraćanja pre lansiranja, pa bar jednom u 3 meseca (baza + fajlovi, u odvojen projekat).
- Tokom razvoja: `supabase db dump` po potrebi; pravih podataka ionako nema.

### 15.3 Monitoring i alarmi
Runbook reaguje, ali neko mora da primeti incident. Minimum pre lansiranja:
- **Prijave (Supabase Auth logovi):** alarm na veći broj neuspelih prijava u kratkom roku, na prijavu sa novog uređaja za admina i na reset MFA faktora.
- **Forma zahteva:** alarm na skok broja zahteva ili odbijenih zahteva (rate limit, honeypot, Turnstile) u odnosu na uobičajeno.
- **Greške:** Vercel log drain ili alert na skok 5xx i grešaka server akcija; Supabase alert na greške funkcija i skok 401/403 na `/rest/v1/`.
- **Poslovni događaji:** `audit_log` događaji „objava cenovnika“ i „promena uloge“ šalju kratko obaveštenje adminu (mejl), da se pogrešna objava vidi odmah.
- Obaveštenja idu na adresu koja nije zajednički inbox sa ličnim podacima. TODO(itum): ko ih prima i ko reaguje van radnog vremena.
- Alarmi ne sadrže lične podatke (samo broj, vreme, ID).

### 15.4 Runbook (`docs/runbook.md`, kratko)

| Situacija | Prvi koraci |
|---|---|
| Sumnja na ukraden nalog osoblja | `is_active = false`, odjava svih sesija, uklanjanje MFA faktora, pregled `audit_log`, nova lozinka posle provere |
| Procureo ključ | nov ključ → Vercel env → redeploy → opoziv starog; provera logova korišćenja |
| Objavljena pogrešna cena | „Vrati prethodnu verziju"; pregled zahteva primljenih u međuvremenu (imaju zapamćenu cenu) |
| Talas spama na formi | pooštriti WAF pravilo ili uključiti Attack Challenge Mode na Vercelu, sniziti limite |
| Curenje ličnih podataka | zaustaviti curenje, sačuvati dokaze, obavestiti Baneta i Stefana, pravnik → Poverenik |
| Kompromitovana zavisnost | vratiti prethodnu verziju, rotirati ključeve koje je build video |
| Alarm: neuspele prijave ili nova prijava admina | proveriti u Auth logu da li je to osoblje; ako nije, postupak za ukraden nalog |
| Alarm: skok zahteva ili grešaka | proveriti da li je napad (WAF, rate limit) ili greška posle deploy-a; po potrebi vratiti prethodni deploy |

Kontakti: Predrag (tehnika), Stefan (ITum), Bane (odluke). TODO(halo): telefoni i ko menja koga.

---

## 16. Bezbednosni testovi

### pgTAP (`supabase/tests/`)
- [ ] RLS uključen na svim tabelama u `public`
- [ ] anon ne može da izvrši nijednu `SECURITY DEFINER` funkciju u `public` (osim izričite liste izuzetaka)
- [ ] svi view-ovi su `security_invoker`
- [ ] anon vidi samo objavljen sadržaj; ne vidi nacrte, zahteve, putnike, ponude, uvoze ni audit; nigde nema INSERT/UPDATE/DELETE
- [ ] prijavljen korisnik bez uloge ili sa `is_active = false` ne vidi ništa
- [ ] korisnik sa `aal1` je odbijen svuda (restrictive politike)
- [ ] editor ne vidi zahteve, putnike, cenovnike i uvoze; ne menja cene; ne menja svoju ulogu
- [ ] sales ne menja uloge i ne briše `audit_log` ni `booking_events`
- [ ] admin ne može da ukloni poslednjeg aktivnog admina
- [ ] zabranjeni prelazi statusa bacaju grešku
- [ ] poslata ponuda ne može da se izmeni
- [ ] objava cenovnika odbija korisnika bez sales/admin i bez aal2
- [ ] Storage: anon i editor ne čitaju `imports` i `documents`; upload u `images` van `offers/` je odbijen; prepisivanje fajla je odbijeno; osoblje briše sliku iz `images`; niko ne briše iz `imports` i `documents`
- [ ] privilegije: niko nema TRUNCATE, REFERENCES ni TRIGGER u `public`; anon nema INSERT/UPDATE/DELETE nigde i SELECT samo na tabelama iz 4.4
- [ ] `authenticated` izvršava samo funkcije sa izričite liste; `create_booking_request` i `revoke_user_sessions` samo `service_role`
- [ ] materialized view-ovi nisu dostupni ni anon ni authenticated
- [ ] `set_user_role`: `p_active = null` odbijen; aktivacija bez potvrđenog TOTP-a odbijena; dva admina koja ražaluju jedan drugog iz dve konekcije (npr. `dblink`) ostavljaju bar jednog aktivnog admina
- [ ] opozvana sesija (red obrisan iz `auth.sessions`) ne vidi ništa, iako je JWT još važeći
- [ ] `audit_log` i `booking_events`: UPDATE, DELETE i TRUNCATE bacaju grešku i za `service_role`
- [ ] svaki test po ulozi prvo ubaci red (kao vlasnik) i potvrdi da postoji; `is_empty` nad praznom tabelom ne dokazuje ništa

### Vitest
- [ ] `safeRedirectPath` i `safeHref`, uključujući `javascript:`, `  javascript:`, `java\tscript:`, `data:`, `//evil.com`, `/\evil.com`, `/administrator`, `/admin.evil`, `/admin/..//evil.com`, `/admin/../login`. `safeRedirectPath` sve to odbija; `safeHref` za `//evil.com` i `/\evil.com` vraća kanonski `https://evil.com/`, nikad ulaz koji izgleda kao relativna putanja
- [ ] Turnstile: pogrešan `action`, token stariji od 300 s i greška siteverify-a odbijaju zahtev
- [ ] rate limit: zahtev sa lažnim `x-forwarded-for` ne menja ključ brojača
- [ ] prijava i reset lozinke: isti odgovor za postojeći i nepostojeći mejl
- [ ] Excel: ZIP sa premalim fajlom a ogromnim raspakovanim sadržajem se odbija pre parsiranja
- [ ] Tiptap validator odbija nepoznate čvorove i loše linkove; renderer ne prikazuje loš link
- [ ] zod šema forme: granice; `.strict()` odbija višak polja
- [ ] provera potpisa slika i Excel fajlova: pravi fajl, preimenovan HTML, prazan fajl
- [ ] logika rate limita

### Playwright
- [ ] `/admin` bez prijave vodi na prijavu; posle lozinke bez MFA vodi na MFA ekran i ne prikazuje podatke
- [ ] korisnik bez upisanog faktora stiže do `/admin/mfa`, upiše faktor i tek tada dobija pristup podacima
- [ ] direktan poziv admin server akcije bez sesije je odbijen
- [ ] forma: popunjen honeypot ne pravi zapis; prekoračen limit daje poruku; bez Turnstile tokena zahtev je odbijen
- [ ] svi `/admin` odgovori, i preusmerenja, imaju `Cache-Control: private, no-store`
- [ ] admin stranice: CSP Report-Only ne prijavljuje nijednu blokiranu skriptu (nonce je na svim skriptama)
- [ ] link za poziv iz mejla prolazi kroz `/admin/auth/confirm` i vodi na postavljanje lozinke
- [ ] posle offboardinga (3.4) otvoren tab tog korisnika odmah dobija odbijanje, ne tek posle isteka tokena

### Ručno pre lansiranja
- [ ] Supabase Security Advisor bez upozorenja
- [ ] `curl` sa publishable ključem na svaku tabelu i RPC funkciju: prazno ili greška tamo gde treba
- [ ] `curl` sa publishable ključem na `/graphql/v1` i Realtime: greška (isključeni)
- [ ] `curl` sa JWT-om prijavljenog korisnika na `/rest/v1/rpc/create_booking_request`: greška
- [ ] JWT iz produkcije ima `alg: ES256` i `kid`
- [ ] alarmi iz 15.3 podešeni i probani (lažna neuspela prijava daje obaveštenje)
- [ ] provera headera (npr. Mozilla Observatory)
- [ ] pretraga build-a za tajnama
- [ ] test vraćanja backup-a

---

## 17. Plan po koracima Roadmap-a

### Korak 1 — Inicijalizacija
- [ ] 2FA na svim nalozima
- [ ] GitHub: privatan repo, zaštita `main`, Dependabot, gitleaks (pre-commit + CI)
- [ ] CI: `npm ci`, typecheck, lint, test, `npm audit`, pretraga build-a za tajnama
- [ ] CI: provera minimalne verzije `next` (≥ 16.4.1) i `react` (≥ 19.2.9)
- [ ] `.gitignore` za `.env*`, `.env.example` bez vrednosti
- [ ] `.claude/settings.json` (Prilog B.7) + sandbox za Bash ili PreToolUse hook (deny pravila nisu granica); bez produkcionih kredencijala na dev mašini
- [ ] Vercel: proverena Deployment Protection, env promenljive po okruženju, region `fra1`; bez `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY`
- [ ] Supabase: dev projekat odvojen od budućeg produkcionog; `enable_signup = false`; novi API ključevi; ES256 ključevi aktivni, legacy JWT ključ deaktiviran; lokalni CLI ažuran
- [ ] Supabase: `drop extension if exists pg_graphql`; Realtime isključen, uključujući „Allow public access“; izložena samo šema `public`
- [ ] Osnovni security headeri (CSP kasnije)
- [ ] ESLint: `react/no-danger` kao greška
- [ ] SheetJS sa CDN-a ili iz `vendor/`, upisano u `docs/odluke.md`

### Korak 2 — Šema v1
- [ ] ODLUKA-1; dok se ne odluči, anon nema pristup cenama
- [ ] Prva migracija: `alter default privileges … revoke` (Prilog C.0); eksplicitni `grant` po tabeli i funkciji, samo za operacije sa politikom; nikome TRUNCATE/REFERENCES/TRIGGER
- [ ] Šema `private`, `private.has_role()` i `private.session_is_valid()`
- [ ] RLS na svakoj tabeli; `revoke ... from anon` na tabelama sa ličnim podacima
- [ ] `profiles`: `role` (null dok admin ne dodeli), `is_active not null default false`, `grant select` + kolonski GRANT za `full_name`, `set_user_role()` sa advisory lock-om, odbijanjem `null` i proverom TOTP-a
- [ ] `audit_log`, samo INSERT + append-only trigeri (Prilog C.7)
- [ ] Restrictive aal2 politike sa proverom sesije (najkasnije u koraku 7, svakako pre prvih pravih podataka)
- [ ] Storage bucketi sa `allowed_mime_types` i `file_size_limit`
- [ ] uuid primarni ključevi
- [ ] pgTAP: RLS svuda, privilegije, `SECURITY DEFINER` i anon/authenticated, view-ovi i materialized view-ovi; testovi po ulozi sa ubačenim redovima (Prilog C.5)

### Korak 3a
- [ ] Ništa posebno; fast-check invarijante već štite od negativnih iznosa

### Korak 7 — Prijava, uloge, RLS
- [ ] Prijava + TOTP MFA (upis faktora i provera); Turnstile na prijavi i resetu
- [ ] `aal1` stranica `/admin/mfa` za upis prvog faktora (ne čita tabele, `requireAal1Session()`); `requireRole()` proverava `aal` pre čitanja `profiles`; aktivacija naloga tek posle potvrđenog TOTP-a
- [ ] Prijava i reset lozinke: isti odgovor za postojeći i nepostojeći mejl
- [ ] Poziv i reset kroz `/admin/auth/confirm` (`token_hash` + `verifyOtp`), u matcheru, `no-store`
- [ ] `requireRole()` i `safeRedirectPath()` (kanonski oblik, Prilog B.3)
- [ ] Admin layout i svaka akcija proveravaju ulogu; `proxy.ts` samo preusmerava, matcher samo `/admin`; CI grep za `requireRole(` u svakom `'use server'` fajlu admina; ESLint zabrana cookie klijenta u javnim rutama
- [ ] Kolačići `secure: true` u produkciji; `Cache-Control: private, no-store` na svim `/admin` odgovorima; „Odjava“ sa `scope: 'local'`
- [ ] Poziv osoblja (samo admin), reset MFA; offboarding po 3.4 (ban, `revoke_user_sessions`, brisanje faktora, `audit_log`)
- [ ] Supabase Auth: SMTP preko Resend-a, Redirect URLs, trajanje sesije i access JWT-a
- [ ] pgTAP negativni testovi po ulogama (sekcija 16), uključujući opozvanu sesiju i trku u `set_user_role`
- [ ] Strogi CSP sa nonce-om na `/admin` (prvo Report-Only): sopstveni root layout sa `instant = false` + `await connection()`, `/admin` izuzet iz globalnog CSP-a, nonce pre `createServerClient` (Prilog B.8, B.9)

### Korak 8 — Sadržaj aranžmana
- [ ] 8b: Tiptap validator na serveru, `safeHref` u rendereru, testovi
- [ ] 8c: upload sa proverom potpisa, Storage politike (INSERT, SELECT i DELETE za osoblje, Prilog C.9) + pgTAP, `remotePatterns`
- [ ] 8d: pregled nacrta dinamički + `noindex`; kopiranje aranžmana ne kopira ništa iz prodaje

### Koraci 4–6 i 9 — Excel uvoz (kad se odblokira)
- [ ] Provera potpisa fajla, opcije SheetJS-a, limiti, zod po ćeliji
- [ ] Zip bomb: provera ZIP direktorijuma (raspakovana veličina, broj zapisa, odnos kompresije) pre SheetJS-a
- [ ] Provere pre objave; zaštita od istovremene objave
- [ ] „Objavi" i „Vrati prethodnu verziju" u bazi, sa proverom uloge i `audit_log`
- [ ] Bucket `imports` bez UPDATE/DELETE

### Koraci 8e i 8f — Ručne cene, popusti, korekcije
- [ ] Samo sales/admin; `audit_log` za svaku izmenu; iznosi u centima sa granicama

### Koraci 11–12 — Javni sajt
- [ ] `"use cache"` samo nad javnim podacima; javni podaci klijentom bez kolačića
- [ ] Kalkulator: zod, rate limit, vraća samo `PriceResult`

### Koraci 13–14 — Zahtev
- [ ] `create_booking_request()` kao `SECURITY INVOKER`, samo za `service_role` (revoke i od `authenticated`); rate limit i ponovne provere unutar funkcije (Prilog C.6); anon bez INSERT-a
- [ ] zod granice, honeypot, Turnstile (`action`, rok 300 s, odbij na grešku siteverify-a), rate limit (IP + mejl, atomičan brojač), WAF pravilo
- [ ] IP za rate limit iz headera koji Vercel sam postavlja (potvrđeno u dokumentaciji), ne iz slobodnog `x-forwarded-for`; test sa lažnim headerom
- [ ] Mejl bez slobodnog teksta, subject bez korisničkog unosa; SPF/DKIM/DMARC
- [ ] Statusi kroz funkciju; `booking_events` samo INSERT + append-only trigeri; bez `on delete cascade` sa zahteva

### Korak 15 — PDF
- [ ] Bucket `documents` privatan, politike INSERT/SELECT za sales/admin, bez UPDATE i DELETE; osoblje preko kratkog signed URL-a; kupac dobija PDF kao prilog
- [ ] Poslata ponuda nepromenljiva (trigger)

### Koraci 16–17 — Završetak i lansiranje
- [ ] CSP za javni sajt (Report-Only → prava politika); odluka o HSTS-u
- [ ] Politika privatnosti sa spiskom obrađivača; GA4 i Meta posle pristanka, bez ličnih podataka
- [ ] Vercel Pro + WAF pravila; Supabase Pro + backup + kopija Storage-a + test vraćanja
- [ ] Na Pro planu: zaštita od procurelih lozinaka, timebox i inactivity timeout sesije, single-session (po odluci TODO(itum))
- [ ] Novi ključevi za produkciju; razvojni se ne prenose
- [ ] Runbook, kontakti, postupak kad zaposleni ode
- [ ] Monitoring i alarmi iz 15.3 podešeni i probani
- [ ] Ručne provere iz sekcije 16
- [ ] Uputstvo za osoblje: lozinke, MFA, lažni mejlovi (npr. „cenovnik od partnera" u prilogu sa sumnjive adrese)

---

## 18. Otvorene odluke i pitanja

| Oznaka | Pitanje | Ko odlučuje |
|---|---|---|
| ODLUKA-1 | Javno čitanje cenovnika (zavisi od toga da li Excel sadrži netto cene) | Halo (sadržaj Excela) + Predrag |
| ODLUKA-2 | Turnstile kao nova zavisnost | Predrag + Stefan |
| ODLUKA-3 | Da li pravi Excel fajlovi i obračuni smeju u AI alate | Halo |
| TODO(halo) | MFA za sve osoblje (telefoni za aplikaciju) | Bane |
| TODO(halo) | Rokovi čuvanja ličnih podataka | Bane + pravnik |
| TODO(halo) | 2FA na mejlu zajedničkog inboxa; ko javlja odlazak zaposlenog | Milica / Dimitrije |
| TODO(halo) | Pristup DNS-u (SPF/DKIM/DMARC, HSTS) | Bane |
| TODO(halo) | Prag upozorenja za promenu cene pri uvozu (%) | Milica / Dimitrije |
| TODO(itum) | Gde se čuvaju kopije Storage-a; ko prima obaveštenja o incidentima | Stefan |
| TODO(itum) | Trajanje sesije osoblja; access JWT 10–15 min umesto 1 h | Stefan + Halo |

---

## Izvori (provereno 21.10.2026)

- Supabase — SSR klijent i `getClaims()`: https://supabase.com/docs/guides/auth/server-side/creating-a-client
- Supabase — napredni SSR vodič (keš i `Set-Cookie`): https://supabase.com/docs/guides/auth/server-side/advanced-guide
- Supabase — MFA i restrictive politike: https://supabase.com/docs/guides/auth/auth-mfa
- Supabase — API ključevi (publishable / secret): https://supabase.com/docs/guides/api/api-keys
- Supabase — backup (Storage nije uključen): https://supabase.com/docs/guides/platform/backups
- Supabase linter — `SECURITY DEFINER` izvršiv za anon: https://github.com/supabase/splinter/pull/160
- Next.js — CSP: https://nextjs.org/docs/app/guides/content-security-policy
- Next.js — server akcije i bezbednost: https://nextjs.org/docs/app/guides/server-actions
- OWASP — Next.js Security Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Nextjs_Security_Cheat_Sheet.html
- Vercel — Deployment Protection: https://vercel.com/docs/deployment-protection
- Vercel — WAF rate limiting: https://vercel.com/docs/vercel-firewall/vercel-waf/rate-limiting
- SheetJS — instalacija i CDN: https://docs.sheetjs.com/docs/getting-started/installation/nodejs

## Izvori za verziju 2 (24.10.2026)

Provereno 24.10.2026:
- Supabase — breaking change, tabele bez automatskih GRANT-ova (novi projekti od 30.05.2026, postojeći od 30.10.2026): https://supabase.com/changelog/45329-breaking-change-tables-not-exposed-to-data-and-graphql-api-automatically
- Next.js — bezbednosno izdanje od 14.10.2026 (RCE u `use cache`, ispravka u 16.4.1): https://nextjs.org/blog/nextjs-security-update-october-14-2026
- Next.js — CSP vodič, PPR i nonce (lokalno: `node_modules/next/dist/docs/01-app/02-guides/content-security-policy.md`), `instant = false` i `connection()` (lokalno: `migrating-to-cache-components.md`)

Iz spoljnog pregleda, nije ponovo provereno:
- Supabase — Sessions: https://supabase.com/docs/guides/auth/sessions
- Supabase — Data API hardening: https://supabase.com/docs/guides/database/data-api
- Supabase — `auth.admin.signOut`: https://supabase.com/docs/reference/javascript/auth-admin-signout
- Supabase — Storage access control: https://supabase.com/docs/guides/storage/security/access-control
- `@supabase/ssr` izvorni kod (podrazumevane opcije kolačića): https://github.com/supabase/ssr
- Next.js — proxy (server akcije i matcher): https://nextjs.org/docs/app/api-reference/file-conventions/proxy
- Cloudflare — Turnstile server-side validation: https://developers.cloudflare.com/turnstile/get-started/server-side-validation/
- SheetJS — parse options: https://docs.sheetjs.com/docs/api/parse-options
- Claude Code — permissions: https://code.claude.com/docs/en/permissions
