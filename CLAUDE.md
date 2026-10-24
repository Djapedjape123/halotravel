@AGENTS.md

# CLAUDE.md — Halo Travel platforma

## O projektu

Nova web platforma za turističku agenciju **Halo Travel** (Niš; organizator putovanja, licenca OTP 217/2021, kategorija A).
Izvođač: ITum Digital (Stefan Pavlović). PM i developer: Predrag Radić (prWeb).
Kontakti kod klijenta: Milica i Dimitrije (operativa), Bane (vlasnik, donosi poslovne odluke).

**Problem koji rešavamo:** osoblje ručno preračunava cene. Primer: 24 hotela u Egiptu posle svakog Excel update-a od egipatskog partnera (stiže 1–2 puta dnevno, često delimično, plus posebne stop sale tabele). Na starom WordPress/Elementor sajtu cene su samo „informativne“ i menjaju se posle dogovora sa kupcem („ispadamo dosta neprofesionalno“). Sav sadržaj (tekst, slike, cene) danas kači eksterni developer iz Word/PDF/Excel fajlova.

**Cilj:** cena je *podatak u bazi*, a ne tekst na stranici, a osoblje samo upravlja sadržajem.
1. Osoblje u adminu samo unosi tekst i slike za svaki aranžman, bez developera.
2. Osoblje ubaci Excel → vidi šta se promenilo → klikne „Objavi“.
3. Kupac na sajtu izabere aerodrom, termin, broj noći, sobu, uslugu i uzrast dece → odmah vidi tačnu cenu → pošalje rezervacioni zahtev.
4. Osoblje proveri kod hotela → jednim klikom pošalje ponudu i predračun (PDF) sa zamrznutom cenom.

**Osnovni princip: sistem računa, čovek odobrava.** Nijedna cena ne ide uživo bez ljudskog „Objavi“.

**Šta obećavamo klijentu:** merljive stvari (manje sati na cenama, tačna cena na sajtu, brži odgovor kupcu). NE obećavamo „rezervacije 24/7“ ni „udvostručavanje rezultata“ (to je u AI sažetku sastanka, klijent to nije rekao).

---

## Obim

### MVP (Faza 0 + 1) — radimo sada
- **Proizvodi u MVP-u: samo Egipat avionom i Grčka** (autobus / sopstveni prevoz).
- Novi javni sajt: potpuno nov, moderan dizajn, mobilni prvo, veoma brz
- Stranica ponude „na dva skrola“: cena i kalkulator na vrhu, detalji u sklopivim sekcijama
- **Admin za sadržaj:** osoblje samo unosi tekst, program po danima, uključeno/nije uključeno, napomene i galeriju slika za svaki aranžman (vidi „Sadržaj aranžmana“)
- Cenovni motor: aerodrom polaska, deca po uzrastu, treća osoba, doplate, popusti, korekcije cena
- Uvoz Excela za **jednog** dobavljača (egipatski partner): upload → pregled razlika → objava, istorija verzija, stop sale
- Ručni unos cena za Grčku
- Popusti (early booking / first minute) i ručne akcije koje osoblje definiše u adminu
- Ponude „cena na upit“ (npr. Antalija) sa istom formom zahteva, samo bez cene
- Rezervacioni zahtev → zajednički admin inbox → mejlovi kupcu i osoblju
- Ponuda i predračun u PDF-u, sa zamrznutom cenom i rokom važenja koji bira osoblje
- Admin sa ulogama, SEO osnove, politika privatnosti, pristanak za kolačiće, GA4
- Sadržaj kreće od nule: **osoblje unosi sve ponude** kroz admin (nema automatskog prenosa sa WordPressa)

### Kasnije — NE implementirati bez izričitog zahteva
- **Posle MVP-a (van faza):** ture / city break, letnji paketi, skijanje, ostali programi; blog/vesti; nalog kupca; dodela zahteva konkretnom prodavcu; više jezika
- **Faza 2:** elektronsko prihvatanje ugovora (klik + OTP kod), online plaćanje (način NIJE odlučen: virtuelni POS banke / IPS QR / predračun), IPS QR kod na predračunu, fiskalni račun, planovi plaćanja i podsetnici, „kupi odmah“ za sopstveni zakup, brojanje slobodnih mesta (avion, autobus, studiji)
- **Faza 3 (uslovno):** RateHawk API (tek posle potpisanog API ugovora i pismene dozvole za javni prikaz cena sa maržom)
- **Faza 4:** chatbot, zajednički inbox za Meta/Instagram/Viber/WhatsApp poruke, newsletter sa segmentima, portal za subagente

### Nikada
- Scraping B2B portala dobavljača
- Stripe (nije dostupan firmama u Srbiji)
- Objava cena bez ljudskog pregleda
- Da AI/LLM sam upisuje cene
- Unos sirovog HTML-a od strane osoblja
- Izmišljeni podaci kao zamena za prave Excel fajlove i obračune u testovima motora i parsera

---

## Stack

| Deo | Tehnologija | Napomena |
|---|---|---|
| Framework | Next.js 16.4 (najmanje 16.4.1), React 19 (najmanje 19.2.9), TypeScript strict | Node 20.9+; pre koda čitati `node_modules/next/dist/docs/`; minimum proverava CI |
| Stilovi | Tailwind CSS 4 + shadcn/ui | CSS-first konfiguracija (`@theme` u CSS-u), bez `tailwind.config.js`; mobilni prvo |
| Baza | Supabase Postgres | region Frankfurt (eu-central-1) |
| Auth | Supabase Auth + `@supabase/ssr` | samo osoblje u MVP-u |
| Fajlovi | Supabase Storage | bucketi: `images` (javni), `imports` i `documents` (privatni) |
| Editor teksta | Tiptap | ograničeno formatiranje, čuva se kao JSON |
| Redosled slika | dnd-kit | prevlačenje u galeriji |
| Hosting | Vercel | funkcije u regionu `fra1` |
| Mejl | Resend | transakcioni mejlovi |
| Excel | SheetJS 0.20.3 (sa CDN-a) | NE `npm i xlsx` |
| PDF | React-PDF (`@react-pdf/renderer`) | ponuda, predračun |
| Forme | react-hook-form + zod | ista zod šema na klijentu i serveru |
| Testovi | Vitest, fast-check, pgTAP, Playwright | |

Na čekanju: Cloudflare Turnstile (CAPTCHA za formu zahteva i prijavu), ODLUKA-2 u `docs/bezbednost.md`. Ne dodavati dok se ne odobri.

### Nalozi i okruženja
- Nalozi za Vercel, Supabase, Resend i GitHub su za razvoj na **prWeb-u**. Prenos na klijenta (ili ITum) dogovara se kasnije.
- **Pre lansiranja:** Vercel Pro (Hobby plan ne dozvoljava komercijalnu upotrebu) i Supabase Pro (besplatni projekat se pauzira posle neaktivnosti i nema dnevne backup-e).
- Razvoj i testiranje na `*.vercel.app` adresama (preview po grani). Domen `halotravel.rs` prebacuje se tek pri lansiranju. `TODO(halo)`: pristup DNS-u (potreban i za Resend SPF/DKIM).
- Git: privatni GitHub repo. Rad na feature granama + PR + Vercel preview, jer je moguće da se na projekat uključi još jedan developer. `main` je uvek u stanju za deploy.

---

## Komande

```bash
npm run dev              # lokalni razvoj
npm run build            # produkcijski build
npm run lint             # ESLint
npm run typecheck        # tsc --noEmit
npm run test             # Vitest (svi unit testovi)
npm run test:golden      # samo golden testovi cenovnog motora
npm run test:e2e         # Playwright

npx supabase start                         # lokalna baza (Docker)
npx supabase db reset                      # migracije + seed iz nule
npx supabase migration new <ime>           # nova migracija
npx supabase gen types typescript --local > src/lib/supabase/database.types.ts
npx supabase test db                       # pgTAP testovi
```

Skripte `typecheck`, `test`, `test:golden` i `test:e2e` dodaju se u koraku 1 (trenutno postoje samo `dev`, `build`, `start`, `lint`).

Instalacija SheetJS:
```bash
npm i https://cdn.sheetjs.com/xlsx-0.20.3/xlsx-0.20.3.tgz
```

**Definicija „gotovo“ za svaki zadatak:**
`npm run typecheck && npm run lint && npm run test` prolaze. Ako je menjana baza, i `npx supabase db reset && npx supabase test db`, plus regenerisani tipovi.

---

## Struktura projekta

Projekat je pokrenut sa `app/` u korenu. **U koraku 1 sve se prebacuje u `src/`** (`src/app`, `src/lib`…); `tsconfig` alias `@/*` → `./src/*`.

```
src/
  app/
    (public)/                  # javni sajt
      page.tsx                 # početna
      destinacije/[slug]/      # destinacija
      ponude/                  # lista + filteri
      ponuda/[slug]/           # stranica ponude + kalkulator
      zahtev/                  # forma za rezervacioni zahtev, potvrda
      politika-privatnosti/
    (admin)/admin/             # zaštićeno (proxy.ts); sopstveni root layout zbog nonce CSP-a
      ponude/                  # lista aranžmana
        [id]/sadrzaj/          # tekst, program, uključeno/nije, napomene, SEO
        [id]/galerija/         # slike: upload, redosled, naslovna, alt
        [id]/cene/             # ručni unos cena (Grčka), korekcije
        [id]/pregled/          # pregled stranice pre objave
      destinacije/             # destinacije (tekst + naslovna slika)
      zajednicki-tekstovi/     # garancija, opšte napomene, uslovi (jednom za sve)
      popusti/                 # early booking, first minute, akcije
      uvoz/                    # Excel uvoz: upload, diff, objava
      zahtevi/                 # inbox, statusi, putnici, ponuda/predračun
      korisnici/               # uloge (samo admin)
    api/                       # route handlers (kasnije webhook-ovi banke/RateHawka)
  components/
    ui/                        # shadcn
    public/                    # komponente javnog sajta
    admin/                     # komponente admina (editor, galerija, forme)
  lib/
    pricing/                   # CENOVNI MOTOR — čist TypeScript, bez Next/Supabase/React importa
      engine.ts
      types.ts
      money.ts                 # rad sa centima
      dates.ts                 # noćenja, uzrast na dan polaska
    content/                   # sadržaj aranžmana: render Tiptap JSON-a, slug, kopiranje aranžmana
    sources/                   # PriceSource interfejs i implementacije
      price-source.ts
      manual-source.ts
      excel-source.ts
    import/                    # Excel uvoz
      parse.ts
      diff.ts
      profiles/                # mapiranje po dobavljaču (npr. egypt-partner.ts)
    auth/
      require-role.ts          # requireRole(): getClaims + profiles.role/is_active + aal2
      safe-redirect.ts         # ?next= samo relativne putanje unutar /admin
    security/
      safe-href.ts             # linkovi samo http/https/mailto/tel
      file-signature.ts        # provera potpisa slika i Excel fajlova
      rate-limit.ts            # brojač u Postgres-u (HMAC ključevi)
      turnstile.ts             # provera tokena (ODLUKA-2)
    booking/                   # statusi zahteva, dozvoljeni prelazi
    pdf/                       # React-PDF šabloni
    email/                     # Resend šabloni
    validation/                # zod šeme
    supabase/
      server.ts                # server klijent (cookies)
      client.ts                # browser klijent
      admin.ts                 # secret ključ — import 'server-only'
      database.types.ts        # generisano, ne menjati ručno
  config/
    company.ts                 # podaci firme: PIB, MB, licenca, garancija, kontakt, račun
  proxy.ts                     # zaštita /admin
supabase/
  migrations/
  seed.sql
  tests/                       # pgTAP
tests/
  golden/                      # stari obračuni Halo-a (JSON) — ISTINA za cene
  fixtures/excel/              # pravi Excel fajlovi dobavljača (repo je privatan)
  e2e/                         # Playwright
docs/
  odluke.md                    # arhitektonske odluke (kratko, sa datumom)
  otvorena-pitanja.md          # sve što čeka odgovor od Halo-a
  bezbednost.md                # plan bezbednosti; §17 = checklist po koraku Roadmap-a
```

---

## Rečnik domena

Identifikatori u kodu su na engleskom, a UI tekst na srpskom.

| Termin (Halo) | Značenje | U kodu |
|---|---|---|
| Destinacija | npr. Hurgada, Tasos; grupiše ponude | `destination` |
| Aranžman / ponuda | proizvod koji se prodaje; **jedna ponuda = jedan hotel/smeštaj** | `offer` |
| Smeštaj / hotel | objekat | `property` |
| Tip sobe / jedinice | npr. Standard, Sea View, Studio, App 1/4 | `unit_type` |
| Mesto polaska / aerodrom | npr. Niš, Beograd; utiče na cenu | `departure_point` |
| Termin / polazak | datum polaska sa određenog mesta polaska | `departure` |
| Broj noći | 7, 9, 10, 12…; za Egipat iz liste ponuđenih trajanja čartera | `nights` |
| Usluga | NA najam, ND noćenje+doručak, PP polupansion, PA pun pansion, AI all inclusive (`TODO(halo)`: da li postoji i UAI) | `board` (enum) |
| Popunjenost | 1/1, 1/2, 1/3, III osoba, dete sa 1 odraslim | `occupancy` |
| Uzrasne grupe dece | npr. 0–2 beba, 2–6, 2–12, 6–13 | `child_band` |
| Doplata | npr. pogled na more (po osobi/dan), spojena sedišta u avionu, viza | `supplement` |
| Popust | early booking / first minute (npr. 5% do datuma); definiše ga Halo u adminu | `discount` |
| Korekcija cene / akcija | ručna izmena osoblja nad cenom iz Excela; preživljava sledeći uvoz | `price_adjustment` |
| Stop sale | zatvorena prodaja za hotel/datume | `stop_sale` |
| Cenovnik | skup cena jednog dobavljača, ima verzije | `price_list` |
| Update | nova (često delimična) verzija cenovnika | `import_batch` |
| Cena „od“ | najniža cena po osobi u 1/2 sobi za najjeftiniji otvoren termin | `from_price` |
| Cena na upit | ponuda bez kalkulatora; zahtev ide bez cene | `price_mode = 'on_request'` |
| Program putovanja | opis po danima (Dan 1, Dan 2…) | `itinerary_days` |
| Uključeno / nije uključeno | liste stavki u ceni i van cene | `included` / `excluded` |
| Zajednički tekst | garancija, opšte napomene, uslovi — jedan tekst za više ponuda | `content_block` |
| Zakup | kapacitet koji Halo sam zakupljuje (Grčka autobusom, sopstveni studiji) | `allotment` (Faza 2) |
| Rezervacioni zahtev / upit | kupac poslao zahtev sa tačnom cenom (ili „na upit“) | `booking_request` |
| Putnik | ime, prezime, datum rođenja; unosi osoblje posle zahteva | `traveler` |
| Ponuda (dokument) | dokument sa zamrznutom cenom i rokom važenja | `quote` |
| Predračun | dokument za uplatu | `proforma` |
| Avans | prva uplata (npr. 30%) | `deposit` (Faza 2) |
| Subagent | druga agencija koja prodaje Halo programe | `subagent` (Faza 4) |

---

## Arhitektura

### Tri sloja podataka
1. **Sadržaj:** destinacije, aranžmani/hoteli, tipovi soba, slike, kratak opis, program po danima, uključeno/nije uključeno, napomene, zajednički tekstovi, SEO polja.
2. **Cene:** dobavljači, cenovnici sa verzijama, mesta polaska, termini, broj noći, usluge, pravila za decu, doplate, popusti, korekcije cena, stop sale.
3. **Prodaja:** zahtevi, putnici, ponude (snapshot), dokumenti, statusi, dnevnik događaja.

Šema v1 je pretpostavka. Menjaće se kad vidimo prave Excel fajlove; to je očekivano. Šema v1 za cene NE sme da zaključa model „cena po osobi“: Grčka verovatno ima cenu po jedinici po noći (vidi „Proizvodi“).

### Proizvodi u MVP-u

**Egipat (avion + hotel):**
- Izvor cena: Excel egipatskog partnera (jedan dobavljač, jedan profil za uvoz).
- **Fiksni čarteri:** polasci su određenim danima, sa određenih aerodroma; kupac bira jedno od ponuđenih trajanja (npr. 7/10/11 noći), ne slobodne datume.
- Cena zavisi od aerodroma polaska (npr. Niš / Beograd).
- Šta Excel sadrži (netto cena hotela ili gotov paket sa avionom) i formula do cene na sajtu: `TODO(halo)`.
- Mesta u avionu se NE prate u MVP-u; dostupnost proverava osoblje posle zahteva.

**Grčka (autobus / sopstveni prevoz):**
- Izvor cena: ručni unos u adminu.
- Model cene (po jedinici po noći, po osobi, ili smeštaj po jedinici + prevoz po osobi): `TODO(halo)`. Dok ne stigne, ne implementirati motor za Grčku; šema dobija `pricing_basis` tek kad pravilo bude poznato.

**Ponude „na upit“** (npr. Antalija, hoteli koje Halo traži kroz B2B sisteme):
- `price_mode = 'on_request'`: nema kalkulatora, prikazuje se „Cena na upit“, a kupac šalje istu strukturisanu formu zahteva bez cene.

### Izvori cena — `PriceSource`
Cenovni motor NE zna odakle cene dolaze. Svaki izvor implementira isti interfejs:
- `manual-source`: cene koje osoblje unese u admin (Grčka)
- `excel-source`: objavljena verzija uvezenog cenovnika (Egipat)
- `ratehawk-source`: Faza 3 (NE implementirati sada)

Popusti i korekcije cena su odvojeni od izvora: primenjuju se u motoru nad cenom iz bilo kog izvora.

### Korekcije cena i popusti
- Osoblje (`sales`/`admin`) sme da promeni cenu koja je došla iz Excela **samo kao korekciju** (`price_adjustment`): npr. akcija -50 € za hotel X u terminima Y. Cena iz Excela u cenovniku ostaje netaknuta.
- Korekcija preživljava sledeći uvoz. Pregled razlika pri uvozu prikazuje i aktivne korekcije koje se odnose na promenjene redove, da osoblje odluči da li ih zadržava.
- Popuste (early booking / first minute) **definiše Halo u adminu**, ne dolaze iz Excela: procenat ili iznos, rok (do datuma rezervacije), na koje ponude/termine se odnose.
- Da li se popusti i korekcije sabiraju ili važi samo najveći: `TODO(halo)`. Dok se ne odgovori, motor za kombinaciju više popusta vraća `PricingRuleMissing`.
- Svaka korekcija i popust se vide kao zasebna stavka u `lines[]` i beleže ko ih je napravio.

### Tok uvoza Excela
1. Upload direktno iz browsera u privatni bucket `imports`. Vercel funkcije primaju najviše 4,5 MB, pa fajl NE ide kroz server action.
2. Parsiranje u Node runtime-u (route handler / server action), NE u Supabase Edge Function (limit 2 s CPU).
3. `parse` → normalizovani redovi → zod validacija → `diff` prema objavljenoj verziji → nacrt (`price_lists.status = 'draft'`).
4. Admin prikazuje: dodato / promenjeno (stara → nova cena) / uklonjeno / stop sale, i aktivne korekcije na promenjenim redovima.
5. „Objavi“ = jedna Postgres funkcija u jednoj transakciji: nacrt postaje `published`, prethodna verzija `archived`, osvežavaju se keš oznake.
6. Delimični update menja samo hotele koji su u fajlu. Ostali ostaju kako su bili.
7. Svaki uvezeni fajl ostaje u Storage-u i povezan je sa verzijom (dokaz).

### Tok zahteva i statusi (MVP)

| Status (kod) | Prikaz u adminu | Ko menja |
|---|---|---|
| `new` | Novi zahtev | sistem |
| `in_review` | U obradi (provera kod hotela) | prodaja |
| `quote_sent` | Ponuda poslata | prodaja |
| `confirmed` | Potvrđeno | prodaja |
| `cancelled` | Otkazano | prodaja/admin |
| `expired` | Isteklo (prošao rok ponude) | sistem |

- Dozvoljeni prelazi su definisani na jednom mestu (`src/lib/booking/`) I proveravaju se u bazi (Postgres funkcija).
- Svaka promena upisuje red u `booking_events` (ko, kada, iz kog u koji status, napomena).
- Faza 2 dodaje: `contract_accepted`, `deposit_paid`, `paid`.

### Rezervacioni zahtev — šta kupac unosi
- **Obavezno:** ime i prezime nosioca, telefon, mejl, broj odraslih, **uzrast svakog deteta** (redosled je bitan), plus izbor iz kalkulatora (aerodrom, termin, broj noći, soba, usluga, izabrane doplate).
- Opciono: napomena.
- **Imena i datumi rođenja putnika se NE traže u formi.** Osoblje ih unosi u adminu posle provere (tabela `travelers`), pre slanja ponude/predračuna.
- Za ponude „na upit“ ista forma, bez cene; kupac piše željeni period u napomeni ili bira okviran termin.
- Server ponovo računa cenu i čuva je uz zahtev; cena iz forme se ignoriše.

### Obaveštenja
- Svi novi zahtevi idu na **jedan zajednički inbox** (`EMAIL_STAFF_INBOX`); u adminu sva prodaja vidi sve zahteve. Dodela prodavcu je van MVP-a.
- Kupac odmah dobija mejl potvrde sa svim stavkama i brojem zahteva.

### Ponuda i predračun (PDF)
- **Rok važenja ponude bira osoblje pri slanju** (polje sa podrazumevanom vrednošću; `TODO(halo)`: podrazumevani rok).
- Ponuda je snapshot (vidi „Pravila za cene“) i čuva tekst programa iz tog trenutka.
- **Predračun je klasičan PDF:** podaci firme iz `src/config/company.ts`, broj računa, poziv na broj, iznos u EUR sa napomenom o plaćanju u dinarima. IPS QR kod je Faza 2.
- Kupac dobija PDF ponude i predračuna **kao prilog mejla**, ne kao link. Signed URL (kratak rok) koristi samo osoblje u adminu.
- `TODO(halo)`: obrasci ponude i predračuna koje danas koriste.

### Ključne odluke
- Admin je sopstveni, u Next.js + Supabase (bez headless CMS-a).
- Sadržaj (tekst i slike) unosi osoblje Halo-a kroz admin; developer ne kači sadržaj.
- Online plaćanje i e-ugovor NISU u MVP-u.
- Kupac nema nalog u MVP-u (samo mejl + broj zahteva). Login ima samo osoblje.

---

## Sadržaj aranžmana (admin)

Osoblje mora samo, bez developera i bez tehničkog znanja, da napravi i menja kompletnu stranicu aranžmana.

### Šta osoblje unosi za svaki aranžman
- **Osnovno:** naziv, destinacija, hotel i kategorija (zvezdice), tip prevoza (avion / autobus / sopstveni), mesta polaska, trajanje
- **Režim cene:** računa se (kalkulator) ili „cena na upit“
- **Kratak opis:** 2–3 rečenice za vrh stranice (ograničenje broja karaktera, sa brojačem)
- **Program putovanja po danima:** lista dana (Dan 1, Dan 2…), svaki sa naslovom i tekstom; dodavanje, brisanje i redosled prevlačenjem
- **Uključeno u cenu / nije uključeno:** dve liste stavki (dodaj/ukloni/redosled)
- **Napomene i uslovi plaćanja:** tekst iz editora (u MVP-u uslovi plaćanja su tekst; u Fazi 2 postaju pravila)
- **Zajednički tekstovi:** izbor iz biblioteke (garancija putovanja, opšte napomene, Opšti uslovi) — unose se jednom u `zajednicki-tekstovi`, a izmena važi za sve ponude koje ih koriste
- **Oznake:** Last minute, Early booking, Hit, Novo (iz fiksne liste)
- **SEO:** naslov stranice, opis, slug (automatski predlog iz naziva, može ručno)
- **Status:** nacrt / objavljeno; istaknuto na početnoj (da/ne)

### Editor teksta
- Tiptap sa ograničenim formatiranjem: naslov (H2/H3), pasus, podebljano, kurziv, liste, link. Bez boja, fontova, tabela i ugrađenog HTML-a.
- Čuva se kao Tiptap JSON u bazi; na sajtu se renderuje kroz naše komponente (dosledan dizajn). Nikad `dangerouslySetInnerHTML` sa sirovim unosom.
- Lepljenje iz Worda: čisti formatiranje i zadržava samo dozvoljene elemente (osoblje danas radi iz Word/PDF dokumenata).

### Galerija slika
- Upload više slika odjednom (prevuci i pusti ili izbor fajlova), sa indikatorom napretka.
- Upload ide direktno iz browsera u Supabase Storage bucket `images` (javno čitanje, pisanje `editor`/`sales`/`admin`), putanja `offers/<offer_id>/<uuid>.<ext>`.
- Dozvoljeni formati: JPG, PNG, WebP; maksimalno 10 MB po slici; jasna poruka ako je fajl prevelik.
- Redosled prevlačenjem (dnd-kit); jedna slika je **naslovna** (koristi se na karticama, u deljenju na mrežama i kao og:image).
- **Alt tekst** za svaku sliku (obavezan za objavu; predlog = naziv hotela + destinacija).
- Brisanje slike briše i fajl iz Storage-a.
- Prikaz na sajtu isključivo kroz `next/image` (responsive veličine, AVIF/WebP, lazy load; naslovna slika `priority`).
- `TODO(halo)`: da li imaju prava na sve fotografije koje koriste.

### Pregled, objava, kopiranje
- **Pregled:** dugme otvara stranicu tačno kako će izgledati na sajtu, i za nacrt (dostupno samo prijavljenom osoblju).
- **Objava** proverava minimum: naziv, destinacija, kratak opis, bar 1 slika sa alt tekstom, bar jedna cena ili režim „cena na upit“. Ako nešto fali, prikaži listu šta fali.
- Čuvanje objavljenog aranžmana osvežava keš te stranice i liste (`cacheTag`).
- **Kopiraj aranžman:** pravi nov nacrt sa istim tekstom, programom i slikama (za novu sezonu ili sličan hotel), bez cena i bez zahteva.
- Svaka izmena beleži ko i kada ju je napravio (`updated_by`, `updated_at`).

### Statične stranice
O nama, Kontakt, Opšti uslovi, FAQ, Politika privatnosti. Tekstove čekamo od Halo-a (Stefan ih traži na sastanku). Da li ih uređuje osoblje u adminu ili su u kodu odlučuje se kad stignu tekstovi; do tada stranica postoji sa `TODO(halo)` sadržajem. Ne izmišljati pravni tekst.

### Uloge — ko sme šta

| Akcija | `editor` | `sales` | `admin` |
|---|---|---|---|
| Tekst, slike, program, SEO aranžmana | ✓ | ✓ | ✓ |
| Zajednički tekstovi | — | — | ✓ |
| Objava/povlačenje aranžmana | ✓ | ✓ | ✓ |
| Ručni unos cena, korekcije, popusti, uvoz Excela, objava cenovnika | — | ✓ | ✓ |
| Zahtevi, putnici, ponude, predračuni | — | ✓ | ✓ |
| Korisnici i uloge | — | — | ✓ |

`TODO(halo)`: potvrditi ko od osoblja ima koju ulogu.

---

## Pravila za cene — NAJVAŽNIJI DEO

1. **Novac:** uvek ceo broj u centima (`amount_cents`), valuta EUR. Nikad float. Zaokruživanje samo na kraju, po pravilu iz konfiguracije (`TODO(halo)`: pravilo zaokruživanja).
2. **Valuta i plaćanje:** sve cene na sajtu, u ponudi i predračunu su u **EUR**. Kupac **plaća u RSD** po kursu na dan uplate. `TODO(halo)`: koji kurs (srednji NBS ili prodajni banke) i tačna formulacija napomene na predračunu. Sistem u MVP-u ne preračunava u dinare.
3. **Datumi boravka:** kalendarski datumi (`date`), zona Europe/Belgrade. Noćenja = interval `[check_in, check_out)`. Za datume boravka ne koristi `new Date()` direktno, nego helper iz `lib/pricing/dates.ts`.
4. **Deca:** čuvamo uzrast (ili datum rođenja) SVAKOG deteta, ne samo broj dece. Uzrast se računa na dan polaska (podrazumevano; potvrditi po dobavljaču). Redosled dece je bitan (prvo/drugo dete mogu imati različitu cenu).
5. **Ulaz motora:** verzija cenovnika, ponuda, aerodrom polaska, termin, broj noći, tip sobe, usluga, broj odraslih, uzrast svakog deteta, izabrane doplate, datum rezervacije (za early booking). Aktivni popusti i korekcije prosleđuju se kao podatak, motor ih ne čita sam.
6. **Motor je čista funkcija:** `calculatePrice(input): PriceResult` u `src/lib/pricing/`. Bez importa iz `next`, `@supabase/*`, `react`. Deterministička, bez I/O.
7. **Rezultat ima stavke:** `lines[]` (osnovna cena, dete 2–12, doplata, popust, korekcija…) sa opisom pravila koje je primenjeno, plus `total_cents`. Zbir stavki mora biti jednak `total_cents`.
8. **Cena „od“** = najniža cena po osobi u 1/2 sobi, za najjeftiniji otvoren termin (bilo koji aerodrom), sa uračunatim aktivnim popustima. Računa je motor nad objavljenim cenovnikom; osoblje je ne unosi ručno. Na sajtu uvek piše „od … po osobi“.
9. **Server uvek računa.** Cena koja stigne iz forme ili klijenta se ignoriše; server je ponovo izračuna.
10. **Ponuda je snapshot:** čuva ulaz, izlaz, ID verzije cenovnika, primenjene popuste/korekcije, tekst programa i rok važenja. Ponuda se NIKAD ne preračunava posle slanja.
11. **Golden testovi** (`tests/golden/*.json`) su stari obračuni Halo-a i oni su istina. **Ne menjaj očekivane vrednosti.** Ako golden test padne, popravi kod. Ako misliš da je test pogrešan, stani i pitaj.
12. **Nepoznato pravilo = greška, ne pretpostavka.** Ako za kombinaciju ne postoji pravilo (npr. treće dete, više popusta odjednom), motor vraća grešku `PricingRuleMissing`, a sajt kaže „cena na upit“. Ne izmišljaj cene ni popuste.
13. **Bez izmišljenih podataka za motor.** Čekamo prave Excel fajlove i stare obračune od Halo-a. Do tada se smeju raditi samo delovi koji ne zavise od poslovnih pravila: `money.ts`, `dates.ts`, tipovi i fast-check invarijante nad njima.
14. **Prvi golden primer** (sa starog sajta, potvrditi sa Halo-om): Royal Star Beach Resort 4*, Hurgada, polazak 29.10., soba 1/2 → 493 € po osobi. `TODO(halo)`: broj noći, aerodrom i šta tačno ulazi u cenu.

---

## Excel uvoz — pravila

- MVP ima **jedan profil**: egipatski partner, u `src/lib/import/profiles/egypt-partner.ts`. Eksplicitno mapiranje kolona i blokova. Bez generičkog „pametnog“ parsera.
- **Pravi fajlovi još nisu stigli.** Profil se ne piše „naslepo“ po pretpostavljenoj strukturi; čeka 3–5 pravih fajlova (pun cenovnik, delimičan update, stop sale).
- SheetJS spojene ćelije (`!merges`) ne popunjava sam; obradi ih eksplicitno u profilu.
- Podržati i `.xlsx` i stari `.xls`.
- Parser i diff su čiste funkcije sa testovima na pravim fajlovima iz `tests/fixtures/excel/`.
- Fajl koji ne odgovara profilu → jasna greška u adminu (koji list, koji red, šta fali), nikad tiho preskakanje.
- Pravilo „jedna zatvorena noć zatvara ceo termin“ je konfigurabilno po dobavljaču (`TODO(halo)`: potvrditi).
- LLM nikad ne upisuje cene. Eventualno (kasnije) predlaže mapiranje koje čovek potvrdi.
- Pravi Excel fajlovi su poslovni podaci klijenta: samo u privatnom repou, nikad u javnim issue-ima, chatovima ili logovima.

---

## Baza i Supabase

- Sve izmene šeme isključivo kroz `npx supabase migration new`. Nikad ne menjaj već primenjenu migraciju; napravi novu.
- Nikad ne menjaj šemu ručno kroz Studio na produkciji.
- Posle svake migracije: `db reset` → `gen types` → `test db`.
- **RLS je uključen na SVAKOJ tabeli.** Javno (anon) čitanje samo za objavljen sadržaj i aktivne popuste (samo polja koja sajt prikazuje). Pisanje samo prijavljene uloge, po tabeli iz „Uloge — ko sme šta“.
- **Automatskih GRANT-ova nema.** Prva migracija ih ukida (`docs/bezbednost-prilozi.md` C.0), a svaka tabela i funkcija dobija eksplicitan `grant` samo za potrebne operacije i uloge. TRUNCATE, REFERENCES i TRIGGER ne dobija niko.
- **Anon nema pristup tabelama cena** dok se ne odluči ODLUKA-1 (`docs/bezbednost.md`; zavisi od toga da li Excel partnera sadrži netto cene). Kalkulator čita objavljeni cenovnik server-only, kroz `src/lib/pricing/load-published.ts`, i vraća samo rezultat motora. Cena „od“ je kolona na ponudi.
- **Anon nema INSERT politiku nigde.** Rezervacioni zahtev upisuje server kroz `create_booking_request()` (`SECURITY INVOKER`, izvršiva samo za `service_role`; rate limit i ponovne provere su unutar funkcije), posle validacije.
- Uloge u `profiles.role`: `admin`, `sales` (prodaja), `editor` (urednik). Provera kroz SQL helper (npr. `has_role('sales')`).
- **Storage:** `images` (javno čitanje; pisanje `editor`/`sales`/`admin`), `imports` (privatno; `sales`/`admin`), `documents` (privatno; `sales`/`admin`, pristup preko signed URL-a). Storage politike su takođe pokrivene pgTAP testovima.
- Secret ključ (`sb_secret_…`) koristi samo `src/lib/supabase/admin.ts`, koji počinje sa `import 'server-only'`.
- Integritet čuva baza: FK, CHECK, UNIQUE; promena statusa samo kroz funkciju koja proverava prelaz. U Fazi 2 kapaciteti preko exclusion constraint-a (`btree_gist`).
- Region: Frankfurt (eu-central-1); Vercel funkcije u `fra1`.
- **Supabase MCP:** samo lokalna ili dev baza (read-only, ograničen na dev projekat). **Ne povezivati ga sa produkcijom u MVP-u** (lični podaci + prompt injection kroz napomene kupaca, Word tekst i Excel ćelije). Nikad `db reset`, `DELETE`, `DROP` ili `TRUNCATE` na produkciji.

---

## Next.js 16 — pravila

Pre pisanja koda za keširanje, rute, proxy ili server actions, pročitaj odgovarajući vodič u `node_modules/next/dist/docs/`; tamo je merodavna verzija (16.4), ne sećanje.
- Server Components po defaultu; `"use client"` samo za interakciju (kalkulator, forme, filteri, editor, galerija).
- Keširanje je opt-in (`cacheComponents: true`): katalog i stranice ponuda koriste `"use cache"` + `cacheTag(...)` (npr. `offers`, `offer:<id>`, `price-list:<id>`, `discounts`). Posle „Objavi“ (sadržaja, cenovnika ili popusta) osveži oznake (`updateTag` / `revalidateTag(tag, profile)`; drugi argument je obavezan u v16).
- Zaštita `/admin` ide kroz `src/proxy.ts` (ne `middleware.ts`, zastareo u v16). `proxy.ts` samo preusmerava na prijavu (matcher samo `/admin` i prijava) i nikad nije jedina zaštita: admin layout i **svaka** server akcija i funkcija za podatke same zovu `requireRole()`.
- Na serveru identitet uvek kroz `supabase.auth.getClaims()`, nikad `getSession()`.
- CSP: strogi sa nonce-om na `/admin` (iz `proxy.ts`), statički bez nonce-a na javnom sajtu (iz `next.config.ts`, zbog `"use cache"`). Prvo `Report-Only`. Nonce ne radi sa statičkim shell-om, pa `/admin` ima sopstveni root layout sa `instant = false` i `await connection()` (`docs/bezbednost-prilozi.md` B.9).
- `params`, `searchParams`, `cookies()`, `headers()` su async: uvek `await`.
- `next/image` sa `images.remotePatterns` za Supabase Storage domen (ne `images.domains`).
- Server actions: zod validacija ulaza → provera uloge → tipizovan rezultat `{ ok: true, data } | { ok: false, error }`. Nikad ne veruju klijentu.
- Metapodaci kroz `generateMetadata` (naslov, opis i og:image iz SEO polja i naslovne slike), sitemap kroz `app/sitemap.ts`, robots kroz `app/robots.ts`.
- 301 preusmerenja starih adresa (20–50 najposećenijih po Google Search Console-u, uključujući stare blog postove) u `next.config.ts` `redirects`.

---

## UI i dizajn

- Tailwind 4 + shadcn/ui. Potpuno nov, moderan izgled: velike fotografije, čista tipografija, jasna cena i dugme „Rezerviši“.
- **Brend:** Halo ima logo (`TODO(halo)`: logo u vektoru, SVG). Boje i fontovi su slobodni i biraju se uz logo; definišu se kao tokeni u CSS-u (`@theme`).
- **Odobravanje dizajna:** Stefan odobrava izgled pre izrade. Način (Figma mockup ili Vercel preview link) još nije dogovoren (`TODO(itum)`).
- **Mobilni prvo** (360–430 px), zatim tablet i desktop. Kupci najčešće dolaze sa telefona, iz Meta reklama.
- **Stranica ponude „na dva skrola“:**
  - Gore: galerija (naslovna slika + ostale), naslov, oznake, kratak opis, cena (prvo „od … po osobi“, posle izbora tačna), kalkulator (aerodrom → termin → noći → soba → usluga → odrasli i uzrast dece → doplate), „Rezerviši“
  - Dole, u sklopivim sekcijama: Program po danima, Uključeno / Nije uključeno, Uslovi plaćanja, Napomene, zajednički tekstovi (garancija, Opšti uslovi)
- Tabele cena se NE skroluju horizontalno na telefonu → kartice ili kalkulator.
- Valuta je UVEK uz iznos (stari sajt ima gole brojeve). Format: `Intl.NumberFormat('sr-Latn-RS', { style: 'currency', currency: 'EUR' })`; datumi `dd.MM.yyyy.`
- Sav UI tekst je **samo na srpskom, latinica**, sa dijakriticima (č ć š ž đ). Bez i18n biblioteke u MVP-u.
- Bez mega-menija sa 150+ hotela. Navigacija: Destinacije → Ponude, plus pretraga i filteri (destinacija, termin, prevoz, aerodrom).
- **Admin je za netehničko osoblje:** jasne labele na srpskom, pomoćni tekst ispod polja, potvrda pre brisanja, automatsko čuvanje nacrta, poruke o grešci koje kažu šta treba uraditi.
- Performanse (mobilni): LCP < 2,5 s, CLS < 0,1, INP < 200 ms; cilj je zelena ocena u PageSpeed-u. Slike preko `next/image` (AVIF/WebP, lazy), fontovi preko `next/font`.
- Pristupačnost: labele na svim poljima, alt tekst na slikama, vidljiv fokus, kontrast AA.

---

## Bezbednost i lični podaci

- Forma zahteva: zod validacija, honeypot polje, rate limit po IP-u i po mejlu (Turnstile po ODLUKA-2).
- Upload slika: MIME i veličinu proveri na klijentu i kroz podešavanja bucketa (`allowed_mime_types`, `file_size_limit`), a pravi sadržaj kroz proveru potpisa fajla na serveru (Storage politika ne vidi sadržaj). Ime fajla uvek generiši (uuid), nikad ne koristi ime koje je poslao korisnik. Bez SVG-a.
- Lični podaci (imena, uzrast/datumi rođenja dece, telefon, mejl) samo u tabelama sa RLS. **Ne logovati lične podatke.**
- MVP NE prikuplja slike pasoša.
- PDF dokumenti u privatnom bucketu; osoblje pristupa preko signed URL-a sa kratkim rokom, kupac dobija PDF kao prilog mejla.
- Tajne samo u env promenljivim; `.env*` nikad u git.

---

## Pravni minimum (Srbija)

Ovo su smernice za razvoj, ne pravni savet. Pravnik klijenta proverava pre Faze 2.
- Politika privatnosti (stari sajt je nema); GA4 i Meta piksel se učitavaju TEK posle pristanka (opt-in baner).
- Doplate (osiguranje i sl.) nikad unapred štiklirane (Zakon o zaštiti potrošača).
- Pre slanja zahteva prikazati: ukupnu cenu sa stavkama, šta je uključeno, uslove plaćanja, link na program i Opšte uslove, kontakt prodaje.
- Program i cena iz trenutka ponude se čuvaju kao verzija (Zakon o turizmu ne dozvoljava izmenu programa posle prodaje). Snapshot ponude zato čuva i tekst programa, a ne samo cenu.
- Pravilnik o online prodaji (Sl. glasnik RS 97/2024) i upis u E-turistu tiču se Faze 2 (zaključenje ugovora online).
- Podaci firme (naziv, PIB, MB, licenca, garancija putovanja, adresa, broj računa) samo u `src/config/company.ts`, ne prepisani po stranicama. `TODO(halo)`: važeća garancija i adresa (stari sajt ima nedoslednosti).

---

## Testiranje

- **Vitest:**
  - cenovni motor: golden testovi (pravi obračuni Halo-a) + fast-check invarijante (total = zbir stavki, nikad negativno, ista ulazna vrednost → isti rezultat)
  - Excel parser na pravim fixture fajlovima, diff, status mašina, zod šeme
  - sadržaj: render Tiptap JSON-a (dozvoljeni elementi), čišćenje teksta zalepljenog iz Worda, generisanje sluga, kopiranje aranžmana
- **pgTAP (`supabase/tests/`):** RLS po ulozi (anon, editor, sales, admin) za tabele i Storage bucket-e, dozvoljeni i zabranjeni prelazi statusa, objava cenovnika (nova `published`, stara `archived`).
- **Playwright (`tests/e2e/`):**
  - editor: napravi aranžman → unese tekst i program → upload 3 slike → promeni redosled i naslovnu → objavi → stranica se vidi na sajtu
  - kupac: ponuda → kalkulator → zahtev → potvrda
  - kupac: ponuda „na upit“ → zahtev bez cene → potvrda
  - admin: uvoz → diff → objava → cena na sajtu se promenila
  - admin: zahtev → unos putnika → ponuda PDF poslata
- **Lighthouse** za početnu i stranicu ponude na mobilnom pre svakog većeg merge-a.

---

## Radni tok sa Claude Code

- **Jedan korak iz Roadmap-a po sesiji.** Prvo kratak plan (fajlovi, migracije, testovi), pa tek onda kod.
- Za cene, uvoz i statuse: **prvo testovi, pa implementacija.**
- Rad na feature grani (`feat/<korak>-<kratko>`), mali commitovi, Conventional Commits: `feat(pricing): ...`, `feat(content): ...`, `fix(import): ...`, `chore(db): ...`. Commit i push samo kad korisnik traži.
- Ne dodaj nove biblioteke van onih iz Stack tabele bez razloga; za veće (CMS, state management, ORM, i18n) prvo pitaj.
- **Ne izmišljaj** poslovna pravila, cene, tekst uslova ni pravne formulacije. Stavi `TODO(halo): <pitanje>` i dopiši pitanje u `docs/otvorena-pitanja.md`.
- **Ne menjaj bez izričite dozvole:** golden testove, primenjene migracije, `.env*`, fajlove u `tests/fixtures/`, `database.types.ts` ručno.
- Kad doneseš arhitektonsku odluku, upiši je u `docs/odluke.md` (datum, odluka, razlog).
- Posle završenog koraka, štikliraj ga u Roadmap-u ispod.
- Kad treba nešto razjasniti sa korisnikom, postavi pitanja sa ponuđenim odgovorima.

---

## Roadmap i trenutni fokus

Redosled je prilagođen tome da pravi Excel fajlovi, formula cene i stari obračuni još nisu stigli: prvo radimo sve što od njih ne zavisi.

Uz svaki korak ide i bezbednosni checklist iz `docs/bezbednost.md` §17; štiklira se zajedno sa korakom.

### MVP
**A. Temelji**
- [ ] 1. Inicijalizacija: prebacivanje u `src/`, shadcn, lokalni Supabase, Vitest, Playwright, skripte (`typecheck`, `test`, `test:golden`, `test:e2e`), GitHub remote, Vercel preview (`fra1`)
- [ ] 2. Šema v1: sadržaj (destinacije, ponude, hoteli, slike, program, zajednički tekstovi) + osnovne tabele cena i prodaje + Storage bucketi + seed (Royal Star primer) + generisani tipovi
- [ ] 3a. `money.ts`, `dates.ts`, tipovi motora + testovi (ne zavise od poslovnih pravila)

**C. Admin**
- [ ] 7. Prijava, uloge, RLS (tabele + Storage), `proxy.ts`, pgTAP testovi
- [ ] 8. Sadržaj aranžmana:
  - [ ] 8a. Lista aranžmana, osnovna polja, režim cene (računa se / na upit), status nacrt/objavljeno
  - [ ] 8b. Editor teksta (Tiptap), program po danima, uključeno/nije uključeno, napomene
  - [ ] 8c. Galerija: upload više slika, redosled, naslovna, alt tekst, brisanje
  - [ ] 8d. Zajednički tekstovi, SEO polja, pregled pre objave, kopiranje aranžmana

**D. Javni sajt**
- [ ] 10. Dizajn sistem: tokeni (uz logo), header, footer, kartice
- [ ] 11. Katalog + filteri + keširanje i osvežavanje posle objave
- [ ] 12. Stranica ponude (galerija, sadržaj, sklopive sekcije); kalkulator se uključuje kad motor bude spreman, do tada „cena na upit“

**E. Zahtev**
- [ ] 13. Tabele zahteva, putnici, ponude (snapshot), statusi, `booking_events`
- [ ] 14. Forma zahteva (uključujući „na upit“) + Resend mejlovi + zajednički admin inbox + unos putnika
- [ ] 15. PDF ponuda (rok bira osoblje) i predračun (klasičan)

**B. Cene i Excel uvoz — BLOKIRANO dok Halo ne pošalje Excel fajlove, formulu i stare obračune**
- [ ] 3b. Cenovni motor (Egipat) + golden testovi + fast-check; cena „od“
- [ ] 4. Parser + profil egipatskog partnera (pravi fixture fajlovi)
- [ ] 5. Diff između verzija (dodato / promenjeno / uklonjeno / stop sale / korekcije)
- [ ] 6. Tok u bazi: nacrt → objava u transakciji, „Vrati prethodnu verziju“, Storage upload
- [ ] 8e. Ručni unos cena (Grčka), čeka model cene za Grčku
- [ ] 8f. Popusti (early booking / first minute) i korekcije cena u adminu
- [ ] 9. Ekran za uvoz: upload, pregled razlika, „Objavi“
- [ ] 12b. Kalkulator na stranici ponude (isti motor kao u testovima)

**F. Završetak**
- [ ] 16. SEO, 301, politika privatnosti, statične stranice, pristanak za kolačiće, GA4
- [ ] 17. Playwright tok, Lighthouse, uputstvo za osoblje, test sa osobljem (unos aranžmana od nule), prelazak na Vercel Pro i Supabase Pro, domen halotravel.rs, lansiranje

### Faza 2 (ne raditi dok se ne odluči)
- [ ] E-prihvatanje ugovora (klik + OTP)
- [ ] Planovi plaćanja i podsetnici
- [ ] Online plaćanje (način TBD) + potpisan callback + idempotentnost
- [ ] IPS QR na predračunu
- [ ] Fiskalni račun za svaku uplatu
- [ ] „Kupi odmah“ za sopstveni zakup i brojanje mesta (avion, autobus, studiji)

### Faza 3 (uslovno)
- [ ] RateHawk: ugovor → sandbox → `ratehawk-source` → prebook i rezervacija → sertifikacija (14–30 dana nad živim sajtom)

**Trenutni fokus:** korak 1.

---

## Otvorena pitanja — čekamo od Halo-a

Detalji i odgovori se vode u `docs/otvorena-pitanja.md`. Najvažnija:
- Formula od cene dobavljača do cene na sajtu (marža, kurs, avio deo, transfer, zaokruživanje) i šta tačno sadrži Excel partnera
- 3–5 pravih Excel fajlova egipatskog partnera (pun, delimičan update, stop sale)
- 20–50 starih ručnih obračuna (za golden testove)
- Model cene za Grčku (po jedinici, po osobi ili kombinovano)
- Uzrasne granice dece i pravila za treću osobu / dete sa jednim odraslim, po dobavljaču
- Pravilo stop sale-a (zatvara li jedna noć ceo termin?)
- Da li se popusti i korekcije sabiraju
- Kurs za uplatu u RSD i napomena na predračunu
- Podrazumevani rok važenja ponude
- Obrasci ponude, predračuna i ugovora; važeći Opšti uslovi
- Tekstovi statičnih stranica (O nama, Kontakt, FAQ, Politika privatnosti)
- Važeća garancija putovanja, adresa firme i broj računa
- Ko od osoblja ima koju ulogu (editor / prodaja / admin)
- Logo u vektoru; prava na fotografije
- Ime softvera u kome danas prave ugovore i predračune (i da li izdaje fiskalne račune)
- Pristup DNS-u za halotravel.rs i Google Search Console-u
- Sastanak sa Banetom (još nije održan): rok MVP-a, način plaćanja, e-ugovor u MVP-u, održavanje, RateHawk
- Tačno ime B2B sistema za Antaliju („Kartače“) i RateHawk ugovor (Faza 3)

---

## Promenljive okruženja

```bash
NEXT_PUBLIC_SITE_URL=
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY=   # sb_publishable_...
SUPABASE_SECRET_KEY=                    # sb_secret_..., samo src/lib/supabase/admin.ts
RESEND_API_KEY=                         # samo slanje, ograničen na domen
EMAIL_FROM=                             # npr. rezervacije@halotravel.rs (posle DNS verifikacije)
EMAIL_STAFF_INBOX=                      # zajednički inbox za obaveštenja o zahtevima
NEXT_PUBLIC_TURNSTILE_SITE_KEY=         # ODLUKA-2
TURNSTILE_SECRET_KEY=                   # ODLUKA-2
RATE_LIMIT_SECRET=                      # HMAC za IP i mejl u brojaču
NEXT_PUBLIC_GA_ID=                      # učitava se tek posle pristanka
NEXT_PUBLIC_META_PIXEL_ID=              # učitava se tek posle pristanka
```

Preview i produkcija imaju odvojene ključeve; preview nikad ne dobija produkcione.

---

## Bezbednost (detalji: docs/bezbednost.md)

Pre rada na prijavi, ulogama, RLS-u, migracijama, uploadu, Excel uvozu, formi zahteva, mejlovima ili headerima pročitaj `docs/bezbednost.md`.
- Server akcija / route handler: zod (`.strict()`) → `requireRole()` → posao → `audit_log`. `proxy.ts` samo preusmerava (matcher `/admin`).
- Na serveru `getClaims()`, nikad `getSession()`. Uloga i `is_active` se čitaju iz `profiles` pri svakom zahtevu.
- Nova tabela: eksplicitni GRANT + RLS + politike po operaciji i ulozi + pgTAP (privilegije i politike). Nova `SECURITY DEFINER` funkcija: `set search_path = ''`, provera uloge unutar funkcije, `revoke execute ... from public, anon, authenticated`, pa `grant` samo kome treba. View: `with (security_invoker = true)`.
- Anon: samo SELECT objavljenog sadržaja, nikad INSERT. Cene po ODLUKA-1.
- Secret ključ samo u `src/lib/supabase/admin.ts`.
- Bez `dangerouslySetInnerHTML` (izuzetak: JSON-LD komponenta). Linkovi samo `http`/`https`/`mailto`/`tel`.
- Upload: provera potpisa fajla na serveru, uuid imena, bez SVG-a.
- `"use cache"` samo za javne podatke.
- Lični podaci ne idu u logove, URL-ove, analitiku ni AI alate.
- RLS, GRANT-ove, Storage politike i Auth podešavanja ne menjaj bez testa i bez da korisnik pregleda izmenu.
