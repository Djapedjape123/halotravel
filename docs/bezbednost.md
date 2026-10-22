# Bezbednost — Halo Travel platforma

> Plan bezbednosti za razvoj i lansiranje. Dopuna za `CLAUDE.md`, čuva se kao `docs/bezbednost.md`.
> Verzija 1 · 21.10.2026 · Predrag Radić (prWeb)
> Ovo su tehničke smernice, ne pravni savet. Pravna pitanja (ZZPL, rokovi čuvanja, obrađivači) proverava pravnik klijenta.

---

## 0. Kako se koristi ovaj fajl

- Pre svakog koraka iz Roadmap-a otvori sekciju 17 (plan po koracima) i štikliraj stavke za taj korak.
- Claude Code: kratka pravila idu u `CLAUDE.md` (Prilog A). Ovaj fajl čita pre rada na prijavi, ulogama, RLS-u, migracijama, uploadu, Excel uvozu, formi zahteva, mejlovima i headerima.
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
- [ ] Sopstveni SMTP (Resend) za mejlove Auth-a (poziv, reset). Podrazumevani Supabase SMTP nije za produkciju.
- [ ] Site URL i Redirect URLs: samo produkcioni domen i tačan obrazac preview adresa ovog projekta. Nikad `https://*.vercel.app/**`.
- [ ] Asimetrični JWT ključevi za potpisivanje uključeni (tada `getClaims()` proverava potpis lokalno, bez poziva Auth serveru).
- [ ] Trajanje sesije (ograničenje trajanja i odjava posle neaktivnosti) ako plan dozvoljava, npr. 12 h / 2 h. TODO(itum) + Halo.

### 3.2 MFA (TOTP)
- Obavezan za sve osoblje, jer se prijavljuje samo osoblje. Minimum su admin i sales. TODO(halo): da li svi imaju telefon za aplikaciju autentifikatora.
- Sprovodi se na dva mesta:
  - u aplikaciji: `requireRole()` odbija ako `aal !== 'aal2'` i šalje na ekran za MFA,
  - u bazi: RESTRICTIVE politika na svakoj tabeli i na `storage.objects` za `authenticated` (Prilog C.2).
- Izgubljen telefon: samo admin resetuje faktor drugom korisniku, posle provere uživo ili telefonom, uz upis u `audit_log`. Ako admin izgubi pristup, Predrag resetuje preko Supabase dashboarda.

### 3.3 Provera identiteta na serveru
- Na serveru uvek `supabase.auth.getClaims()` (proverava potpis JWT-a). `getSession()` se nikad ne koristi za odluke o pristupu.
- `getClaims()` ne zna da li je sesija u međuvremenu ugašena na serveru. Zato se uz svaki zahtev iz baze čitaju i `profiles.role` i `profiles.is_active`. Deaktivacija naloga deluje odmah.
- Jedna funkcija `requireRole()` u `src/lib/auth/require-role.ts` (Prilog B.1). Svi admin ulazi je koriste.
- `proxy.ts` matcher pokriva samo `/admin` (i stranicu prijave). Javne, keširane stranice ne smeju da osvežavaju sesiju, jer bi keširan odgovor mogao da ponese tuđi `Set-Cookie`. Javni podaci se čitaju klijentom bez kolačića.
- `?next=` posle prijave prihvata samo relativne putanje unutar `/admin` (Prilog B.3).
- Odjava briše sesiju. Admin ima i „Odjavi sa svih uređaja".
- Svaka osoba ima svoj nalog. Zajednički nalozi su zabranjeni, inače `audit_log` gubi smisao.

### 3.4 Kad neko od osoblja ode
`is_active = false` → odjava svih sesija → uklanjanje MFA faktora → pregled `audit_log` za poslednjih 30 dana. TODO(halo): ko javlja odlazak zaposlenog.

---

## 4. Autorizacija

### 4.1 Uloge
Tabela „Uloge — ko sme šta" iz `CLAUDE.md` važi. Dodaci:
- `profiles.role` i `profiles.is_active` menja samo admin, kroz funkciju `set_user_role()` koja proverava admin + aal2, ne dozvoljava uklanjanje poslednjeg aktivnog admina i upisuje `audit_log` (Prilog C.4).
- Korisnik u svom redu u `profiles` sme da menja samo dozvoljene kolone (npr. `full_name`), preko kolonskog GRANT-a (Prilog C.3).
- Uloga se nikad ne čita iz `user_metadata`, jer ga korisnik sam menja. Ako se ikad stavi u JWT, samo preko `app_metadata` ili Auth hook-a.
- Editor ne vidi zahteve, putnike, cenovnike ni uvoze.

### 4.2 Aplikacioni sloj (Next.js)
- Redosled u svakoj server akciji: zod (`.strict()`) → `requireRole()` → posao → `audit_log` → `updateTag` (Prilog B.2).
- Admin stranice zovu `requireRole()` u layout-u **i** u svakoj akciji i funkciji koja čita podatke. Zaštita layout-a ne štiti server akcije.
- Route handleri isto. Za POST sa kolačićima dodatno proveri `Origin`, jer server akcije to rade same, a route handleri ne.
- Čitanje podataka za admin ide kroz funkcije sa `import 'server-only'` koje vraćaju samo potrebna polja, ne cele redove.
- `"use cache"` samo za javne podatke. Nikad za funkcije koje čitaju kolačiće, sesiju ili lične podatke.
- Pregled nacrta je dinamički, poziva `requireRole()`, ima `noindex` i ne kešira se.
- Admin koristi Supabase klijent sa JWT-om prijavljenog korisnika, pa RLS važi. Secret ključ samo za: poziv osoblja, odjavu drugih korisnika, upis zahteva kupca (4.4) i, ako ODLUKA-1 tako ispadne, server-only čitanje cenovnika.
- `serverActions.allowedOrigins` ne podešavati osim ako je baš potrebno, i nikad ne dodavati `null`.

### 4.3 Baza: RLS, GRANT, funkcije, view-ovi
- RLS uključen na svakoj tabeli u `public`. pgTAP test pada ako nije (Prilog C.5).
- Politike se pišu po operaciji i po ulozi (`to authenticated`), uz helper `private.has_role(...)` (Prilog C.1).
- Tabele sa ličnim podacima (`booking_requests`, `travelers`, `quotes`, `proformas`, `audit_log`): pored RLS-a i `revoke all ... from anon`.
- **Funkcije.** Postgres podrazumevano daje EXECUTE svima, a na Supabase-u je svaka funkcija u `public` dostupna preko `/rest/v1/rpc/`, i anonimnim korisnicima. Zato:
  - helperi (`has_role`, rate limit) idu u šemu `private`, koja nije izložena kroz API,
  - svaka `SECURITY DEFINER` funkcija ima `set search_path = ''`, potpuno kvalifikovana imena tabela, proveru uloge **unutar** funkcije, `revoke execute ... from public, anon` i `grant` samo onima kojima treba,
  - pgTAP proverava da anon ne može da izvrši nijednu `SECURITY DEFINER` funkciju u `public`.
- **View-ovi** zaobilaze RLS ako nemaju `with (security_invoker = true)`. Svaki view ga mora imati (test).
- Nepromenljivi zapisi:
  - poslata ponuda (posle `quote_sent`): trigger zabranjuje UPDATE,
  - `booking_events` i `audit_log`: samo INSERT, bez UPDATE i DELETE za bilo koju ulogu,
  - redovi objavljenog cenovnika se ne menjaju; nova verzija = novi uvoz; korekcije su zasebna tabela.
- Supabase Security Advisor (dashboard) bez upozorenja pre svakog merge-a koji menja bazu i pre lansiranja.

### 4.4 Javni (anon) pristup — konačna lista

| Šta | Anon | Napomena |
|---|---|---|
| Objavljene destinacije, ponude, sadržaj, slike | SELECT | samo `status = 'published'` |
| Aktivni popusti | SELECT | samo polja koja sajt prikazuje |
| Cena „od" | SELECT | kolona na ponudi, računa je motor pri objavi |
| Redovi cenovnika | **ODLUKA-1** | vidi ispod |
| Zahtevi, putnici, ponude, predračuni, uvozi, audit | ništa | |
| INSERT u bilo koju tabelu | ništa | zahtev kupca upisuje server |

**Upis zahteva kupca.** Anon nema INSERT politiku. Server akcija posle validacije, Turnstile-a i rate limita poziva funkciju `create_booking_request(...)` preko admin klijenta. Funkcija je `SECURITY DEFINER` i izvršiva samo za `service_role`. U jednoj transakciji upisuje zahtev i prvi red u `booking_events`. Tako niko ne može da zaobiđe formu direktnim pozivom API-ja.

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
  2. Cloudflare Turnstile: token se proverava na serveru tajnim ključem, uz proveru `success` i `hostname`; token je jednokratan,
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
- **Sloj 2 — brojač u Postgres-u:** tabela `private.rate_limits` i funkcija `private.hit_rate_limit(key, limit, window)`. Početni ključevi:
  - `zahtev:ip:<hmac>` — npr. 5 na 10 min,
  - `zahtev:mejl:<hmac>` — npr. 3 na 24 h (štiti tuđe inboxe),
  - `kalkulator:ip:<hmac>` — npr. 120 na minut.
- IP i mejl se ne čuvaju sirovi, nego kao HMAC-SHA256 sa tajnom `RATE_LIMIT_SECRET`. Redovi stariji od 48 h se brišu.
- IP na Vercelu se čita iz `x-forwarded-for` (prva vrednost), koji postavlja platforma. Proveriti u Vercel dokumentaciji.
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
  2. browser uploaduje direktno u Storage (politika: samo editor/sales/admin, samo prefiks `offers/`, samo INSERT, bez upsert-a i UPDATE-a),
  3. server akcija `registerImage()` čita prvih 16 bajtova i proverava potpis: JPEG `FF D8 FF`, PNG `89 50 4E 47 0D 0A 1A 0A`, WebP `RIFF????WEBP`. Ako ne odgovara, briše fajl i vraća grešku. Tek tada upisuje sliku u tabelu.
- Fajl u Storage-u bez reda u bazi, stariji od 24 h, je smeće (kasnije automatsko čišćenje).
- `next/image` `remotePatterns`: samo `https://<ref>.supabase.co/storage/v1/object/public/images/**`. `dangerouslyAllowSVG` ostaje isključen.
- Anon nema SELECT politiku na `storage.objects` (nema listanja). Javne slike se čitaju preko javnog URL-a sa uuid putanjom. Slike iz nacrta vidi ko zna URL, što je prihvatljivo.
- Uklanjanje EXIF podataka (GPS) je opciono i nisko prioritetno za hotelske fotografije.

### 7.2 Excel (bucket `imports`)
- Privatan. INSERT i SELECT samo sales/admin (+ aal2). Bez UPDATE i DELETE, jer je fajl dokaz.
- Dozvoljeni `.xlsx` i `.xls`; limit veličine npr. 15 MB (podesiti kad stignu pravi fajlovi). Browser za `.xls` ponekad šalje `application/octet-stream`, pa je prava provera potpis na serveru (sekcija 8).

### 7.3 Dokumenti (bucket `documents`)
- Privatan, samo sales/admin. Osoblje pristupa preko signed URL-a kratkog roka (npr. 60 s).
- Kupac dobija PDF ponude i predračuna **kao prilog mejla**, ne kao signed URL. Kratak rok bi istekao pre nego što kupac otvori mejl, a dug rok je javni link do ličnih podataka. Primenjeno u `CLAUDE.md` (§18, tačka 4).

---

## 8. Excel uvoz

- Pre parsiranja proveri potpis fajla: `.xlsx` je ZIP (`50 4B 03 04`), `.xls` je OLE2 (`D0 CF 11 E0 A1 B1 1A E1`). SheetJS sam prepoznaje i druge formate (CSV, HTML, SYLK...), pa bi bez ove provere prihvatio i HTML preimenovan u `.xls`.
- SheetJS opcije čitanja: bez generisanja HTML-a iz ćelija (`cellHTML: false`), bez formula (`cellFormula: false`), bez VBA (podrazumevano isključeno), limit redova (`sheetRows`), `dense: true`. Formule se nikad ne izvršavaju, čitaju se samo sačuvane vrednosti. Tačna imena opcija proveriti u dokumentaciji verzije 0.20.3.
- Ograničenja: veličina fajla, broj listova, broj redova, `maxDuration` route handlera. Prekoračenje daje jasnu grešku, nikad tihu obradu dela fajla.
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
- U runbook-u (sekcija 15) stoji spisak ključeva, gde se koriste i kako se rotiraju: Supabase secret, Resend, Turnstile secret, `RATE_LIMIT_SECRET`.
- Rotacija odmah ako ključ završi u git-u, logu, chatu ili AI alatu, ili ako ode osoba koja mu je imala pristup.
- Pre lansiranja produkcija dobija nove ključeve; razvojni se ne prenose.

---

## 11. Zavisnosti i lanac nabavke

- `package-lock.json` je u repou; CI koristi `npm ci`, ne `npm install`.
- Dependabot alerts i security updates. Ako je dostupno, period čekanja (`cooldown`) od nekoliko dana za nove verzije, jer se kompromitovani paketi obično otkriju u prvih par dana.
- `npm audit --audit-level=high` u CI.
- Nova biblioteka = prvo pitanje (već u `CLAUDE.md`). Pre dodavanja proveriti održavanje, popularnost i install skripte.
- Next.js i React uvek na poslednjem patch-u svoje verzije. U 2025–2026. bilo je ozbiljnih ranjivosti baš u delovima koje koristimo (zaobilaženje middleware-a posebnim headerom, RCE u React Server Components, CSRF u server akcijama sa `Origin: null`). Pratiti GitHub Security Advisories za `vercel/next.js` i `facebook/react`.
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

- `.claude/settings.json` u repou (Prilog B.7): zabrana čitanja `.env` fajlova, potvrda za `git push`, zabrana komandi nad produkcijom. Proveriti u dokumentaciji Claude Code-a kako se pravila primenjuju na Bash komande.
- Supabase MCP: samo lokalna ili dev baza, u read-only režimu i ograničen na dev projekat. **Ne povezivati ga sa produkcijom u MVP-u** (predlog izmene sekcije 11 u `CLAUDE.md`). Napomena kupca, tekst iz Worda i Excel ćelije mogu sadržati „uputstva" za agenta (prompt injection), a produkcija sadrži lične podatke.
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
- Logovi: nikad telo zahteva, ime, telefon, mejl ni uzrast. Greške se loguju sa ID-jem zahteva; klijent dobija opštu poruku.
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

### 15.3 Runbook (`docs/runbook.md`, kratko)

| Situacija | Prvi koraci |
|---|---|
| Sumnja na ukraden nalog osoblja | `is_active = false`, odjava svih sesija, uklanjanje MFA faktora, pregled `audit_log`, nova lozinka posle provere |
| Procureo ključ | nov ključ → Vercel env → redeploy → opoziv starog; provera logova korišćenja |
| Objavljena pogrešna cena | „Vrati prethodnu verziju"; pregled zahteva primljenih u međuvremenu (imaju zapamćenu cenu) |
| Talas spama na formi | pooštriti WAF pravilo ili uključiti Attack Challenge Mode na Vercelu, sniziti limite |
| Curenje ličnih podataka | zaustaviti curenje, sačuvati dokaze, obavestiti Baneta i Stefana, pravnik → Poverenik |
| Kompromitovana zavisnost | vratiti prethodnu verziju, rotirati ključeve koje je build video |

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
- [ ] Storage: anon i editor ne čitaju `imports` i `documents`; upload u `images` van `offers/` je odbijen; prepisivanje fajla je odbijeno

### Vitest
- [ ] `safeRedirectPath` i `safeHref`, uključujući `javascript:`, `  javascript:`, `java\tscript:`, `data:`, `//evil.com`
- [ ] Tiptap validator odbija nepoznate čvorove i loše linkove; renderer ne prikazuje loš link
- [ ] zod šema forme: granice; `.strict()` odbija višak polja
- [ ] provera potpisa slika i Excel fajlova: pravi fajl, preimenovan HTML, prazan fajl
- [ ] logika rate limita

### Playwright
- [ ] `/admin` bez prijave vodi na prijavu; posle lozinke bez MFA vodi na MFA ekran i ne prikazuje podatke
- [ ] direktan poziv admin server akcije bez sesije je odbijen
- [ ] forma: popunjen honeypot ne pravi zapis; prekoračen limit daje poruku; bez Turnstile tokena zahtev je odbijen

### Ručno pre lansiranja
- [ ] Supabase Security Advisor bez upozorenja
- [ ] `curl` sa publishable ključem na svaku tabelu i RPC funkciju: prazno ili greška tamo gde treba
- [ ] provera headera (npr. Mozilla Observatory)
- [ ] pretraga build-a za tajnama
- [ ] test vraćanja backup-a

---

## 17. Plan po koracima Roadmap-a

### Korak 1 — Inicijalizacija
- [ ] 2FA na svim nalozima
- [ ] GitHub: privatan repo, zaštita `main`, Dependabot, gitleaks (pre-commit + CI)
- [ ] CI: `npm ci`, typecheck, lint, test, `npm audit`, pretraga build-a za tajnama
- [ ] `.gitignore` za `.env*`, `.env.example` bez vrednosti
- [ ] `.claude/settings.json` (Prilog B.7)
- [ ] Vercel: proverena Deployment Protection, env promenljive po okruženju, region `fra1`
- [ ] Supabase: dev projekat odvojen od budućeg produkcionog; `enable_signup = false`; novi API ključevi
- [ ] Osnovni security headeri (CSP kasnije)
- [ ] ESLint: `react/no-danger` kao greška
- [ ] SheetJS sa CDN-a ili iz `vendor/`, upisano u `docs/odluke.md`

### Korak 2 — Šema v1
- [ ] ODLUKA-1; dok se ne odluči, anon nema pristup cenama
- [ ] Šema `private` i `private.has_role()`
- [ ] RLS na svakoj tabeli; `revoke ... from anon` na tabelama sa ličnim podacima
- [ ] `profiles`: `role` (null dok admin ne dodeli), `is_active`, kolonski GRANT, `set_user_role()`
- [ ] `audit_log`, samo INSERT
- [ ] Restrictive aal2 politike (najkasnije u koraku 7, svakako pre prvih pravih podataka)
- [ ] Storage bucketi sa `allowed_mime_types` i `file_size_limit`
- [ ] uuid primarni ključevi
- [ ] pgTAP: RLS svuda, `SECURITY DEFINER` i anon, view-ovi

### Korak 3a
- [ ] Ništa posebno; fast-check invarijante već štite od negativnih iznosa

### Korak 7 — Prijava, uloge, RLS
- [ ] Prijava + TOTP MFA (upis faktora i provera); Turnstile na prijavi i resetu
- [ ] `requireRole()` i `safeRedirectPath()`
- [ ] Admin layout i svaka akcija proveravaju ulogu; `proxy.ts` samo preusmerava, matcher samo `/admin`
- [ ] Poziv osoblja (samo admin), deaktivacija, reset MFA
- [ ] Supabase Auth: SMTP preko Resend-a, Redirect URLs, trajanje sesije
- [ ] pgTAP negativni testovi po ulogama (sekcija 16)
- [ ] Strogi CSP sa nonce-om na `/admin` (prvo Report-Only)

### Korak 8 — Sadržaj aranžmana
- [ ] 8b: Tiptap validator na serveru, `safeHref` u rendereru, testovi
- [ ] 8c: upload sa proverom potpisa, Storage politike + pgTAP, `remotePatterns`
- [ ] 8d: pregled nacrta dinamički + `noindex`; kopiranje aranžmana ne kopira ništa iz prodaje

### Koraci 4–6 i 9 — Excel uvoz (kad se odblokira)
- [ ] Provera potpisa fajla, opcije SheetJS-a, limiti, zod po ćeliji
- [ ] Provere pre objave; zaštita od istovremene objave
- [ ] „Objavi" i „Vrati prethodnu verziju" u bazi, sa proverom uloge i `audit_log`
- [ ] Bucket `imports` bez UPDATE/DELETE

### Koraci 8e i 8f — Ručne cene, popusti, korekcije
- [ ] Samo sales/admin; `audit_log` za svaku izmenu; iznosi u centima sa granicama

### Koraci 11–12 — Javni sajt
- [ ] `"use cache"` samo nad javnim podacima; javni podaci klijentom bez kolačića
- [ ] Kalkulator: zod, rate limit, vraća samo `PriceResult`

### Koraci 13–14 — Zahtev
- [ ] `create_booking_request()` samo za `service_role`; anon bez INSERT-a
- [ ] zod granice, honeypot, Turnstile, rate limit (IP + mejl), WAF pravilo
- [ ] Mejl bez slobodnog teksta, subject bez korisničkog unosa; SPF/DKIM/DMARC
- [ ] Statusi kroz funkciju; `booking_events` samo INSERT

### Korak 15 — PDF
- [ ] Bucket `documents` privatan; osoblje preko kratkog signed URL-a; kupac dobija PDF kao prilog
- [ ] Poslata ponuda nepromenljiva (trigger)

### Koraci 16–17 — Završetak i lansiranje
- [ ] CSP za javni sajt (Report-Only → prava politika); odluka o HSTS-u
- [ ] Politika privatnosti sa spiskom obrađivača; GA4 i Meta posle pristanka, bez ličnih podataka
- [ ] Vercel Pro + WAF pravila; Supabase Pro + backup + kopija Storage-a + test vraćanja
- [ ] Novi ključevi za produkciju; razvojni se ne prenose
- [ ] Runbook, kontakti, postupak kad zaposleni ode
- [ ] Ručne provere iz sekcije 16
- [ ] Uputstvo za osoblje: lozinke, MFA, lažni mejlovi (npr. „cenovnik od partnera" u prilogu sa sumnjive adrese)

---

## 18. Predlozi izmena `CLAUDE.md`

**Status: primenjeno 22.10.2026.** Brojevi su iz ranije verzije; u zagradi je naziv sekcije u `CLAUDE.md`.

1. **Sekcija 11 („Baza i Supabase“), RLS:** „javno čitanje objavljene verzije cenovnika" zavisi od ODLUKA-1; do tada anon nema pristup cenama.
2. **Sekcija 11 („Baza i Supabase“):** „Anon sme samo da UBACI rezervacioni zahtev (kroz server)" → „Anon nema INSERT politiku; zahtev upisuje server kroz `create_booking_request()`".
3. **Sekcija 11 („Baza i Supabase“), Supabase MCP:** „na produkciju SAMO read-only" → „ne povezivati sa produkcijom u MVP-u".
4. **Sekcija 7 („Arhitektura“ → „Ponuda i predračun“):** kupac dobija PDF kao prilog mejla; signed URL samo za osoblje.
5. **Sekcija 12 („Next.js 16 — pravila“):** dodati „`proxy.ts` samo preusmerava, matcher samo `/admin`; svaka akcija sama proverava ulogu (`requireRole`)" i „na serveru `getClaims()`, nikad `getSession()`".
6. **Sekcija 13 („Next.js 16 — pravila“):** CSP strategija (nonce na `/admin`, statički CSP na javnom sajtu).
7. **Sekcija 14 („Bezbednost i lični podaci“):** „proveri MIME tip i veličinu u Storage politici" → podešavanja bucketa (`allowed_mime_types`, `file_size_limit`) + provera potpisa na serveru, jer Storage politika ne vidi sadržaj fajla. Dodati Turnstile i limit po mejlu.
8. **Sekcija 3 („Stack“):** Turnstile, ako ODLUKA-2 prođe.
9. **Sekcija 18 („Roadmap“):** bezbednosne stavke po koracima iz sekcije 17; korak 6 dobija „Vrati prethodnu verziju".
10. **Sekcija 20 („Promenljive okruženja“):** env promenljive iz Priloga D.
11. **Sekcija 5 („Struktura projekta“):** `src/lib/auth/` (`require-role.ts`, `safe-redirect.ts`) i `src/lib/security/` (`safe-href.ts`, `file-signature.ts`, `rate-limit.ts`, `turnstile.ts`).
12. **Kraj fajla:** kratka sekcija iz Priloga A.

---

## 19. Otvorene odluke i pitanja

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
| TODO(itum) | Trajanje sesije osoblja | Stefan + Halo |

---

## Prilog A — kratka sekcija za kraj `CLAUDE.md`

```md
## Bezbednost (detalji: docs/bezbednost.md)
Pre rada na prijavi, ulogama, RLS-u, migracijama, uploadu, Excel uvozu, formi zahteva, mejlovima ili headerima pročitaj docs/bezbednost.md.
- Server akcija / route handler: zod (.strict()) → requireRole() → posao → audit_log. proxy.ts samo preusmerava (matcher /admin).
- Na serveru getClaims(), nikad getSession(). Uloga i is_active se čitaju iz profiles pri svakom zahtevu.
- Nova tabela: RLS + politike po operaciji i ulozi + pgTAP. Nova SECURITY DEFINER funkcija: set search_path = '', provera uloge unutar funkcije, revoke execute from public, anon. View: with (security_invoker = true).
- Anon: samo SELECT objavljenog sadržaja, nikad INSERT. Cene po ODLUKA-1.
- Secret ključ samo u src/lib/supabase/admin.ts.
- Bez dangerouslySetInnerHTML (izuzetak: JSON-LD komponenta). Linkovi samo http/https/mailto/tel.
- Upload: provera potpisa fajla na serveru, uuid imena, bez SVG-a.
- "use cache" samo za javne podatke.
- Lični podaci ne idu u logove, URL-ove, analitiku ni AI alate.
- RLS, GRANT-ove, Storage politike i Auth podešavanja ne menjaj bez testa i bez da korisnik pregleda izmenu.
```

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
  if (!next.startsWith('/admin') || next.startsWith('//') || next.includes('\\')) return fallback;
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
Restrictive politika sama ne daje pristup; samo sužava ono što daju obične (permissive) politike. Ponoviti za svaku tabelu i za `storage.objects`.
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
