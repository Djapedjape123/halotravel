# Arhitektonske i projektne odluke

Format: datum — odluka — razlog.

## 20.10.2026.
- **MVP proizvodi su samo Egipat avionom i Grčka.** Ture, letnji paketi i ostalo idu posle MVP-a. Razlog: tu je najviše ručnog rada; manji obim do lansiranja.
- **Jedna ponuda = jedan hotel.** Destinacija grupiše ponude. Razlog: jednostavna stranica „na dva skrola“, jasna cena „od“, dobar SEO po hotelu.
- **Aerodrom polaska je deo cene** (`departure_point`). Razlog: cena se razlikuje po aerodromu (Niš / Beograd).
- **Egipat = fiksni čarteri sa ponuđenim trajanjima.** Razlog: tako rade letovi; kupac ne bira slobodne datume.
- **Cene u EUR, uplata u RSD;** sistem u MVP-u ne preračunava u dinare. Razlog: tako Halo radi danas; kurs je otvoreno pitanje.
- **Samo jedan Excel profil (egipatski partner)** u MVP-u; Grčka ide kroz ručni unos.
- **Ručna izmena cene iz Excela = korekcija (`price_adjustment`), ne prepisivanje cene.** Razlog: korekcija preživljava sledeći uvoz, a Excel cena ostaje dokaz.
- **Popuste (early booking / first minute) definiše Halo u adminu.** Ne dolaze iz Excela.
- **Cena „od“ = najniža cena po osobi u 1/2 sobi za najjeftiniji otvoren termin**, računa je motor.
- **Bez praćenja mesta u avionu u MVP-u;** dostupnost potvrđuje osoblje.
- **Zahtev traži kontakt i uzrast dece, ne imena putnika;** imena unosi osoblje kasnije. Razlog: kraća forma, veća konverzija, manje ličnih podataka.
- **Rok važenja ponude bira osoblje pri slanju.**
- **Kupac nema nalog u MVP-u.**
- **Predračun je klasičan PDF;** IPS QR je Faza 2.
- **Jedan zajednički inbox za zahteve;** bez dodele prodavcu.
- **Ponude „na upit“ koriste istu formu zahteva, bez cene.**
- **Sajt je samo na srpskom, latinica.**
- **Bez bloga u MVP-u;** stari postovi se rešavaju kroz 301.
- **Logo postoji, boje i fontovi su slobodni.**
- **Osoblje unosi sav sadržaj;** developer pravi admin i obučava.
- **Nalozi (Vercel, Supabase, Resend, GitHub) su na prWeb-u** za razvoj; pre lansiranja Vercel Pro i Supabase Pro. Razlog: Hobby ne dozvoljava komercijalnu upotrebu, besplatni Supabase pauzira projekat.
- **Privatni GitHub, feature grane + PR + Vercel preview.** Razlog: moguće je da se uključi još jedan developer.
- **Razvoj na `*.vercel.app`;** halotravel.rs tek pri lansiranju.
- **Kod se prebacuje iz `app/` u `src/`** u koraku 1.
- **Bez izmišljenih podataka za cenovni motor i parser;** čekamo prave Excel fajlove i obračune. Roadmap je preuređen tako da se prvo radi sve što od njih ne zavisi.
- **CLAUDE.md uvozi `AGENTS.md`** (`@AGENTS.md`), koji generiše Next 16 sa uputstvom za tu verziju.

## 22.10.2026.
- **`docs/` je u gitu.** Razlog: `CLAUDE.md` se oslanja na `docs/`, a drugi developer bez njega ne vidi odluke, pitanja i plan bezbednosti. Repo je privatan.
- **Predlozi iz `docs/bezbednost.md` §18 primenjeni na `CLAUDE.md`**, a zabrane iz Priloga B.7 dodate su u `.claude/settings.json`. Razlog: dokumenti su se protivili (MCP na produkciji, anon čitanje cenovnika, imena env promenljivih).
- **ODLUKA-1 (privremeno): anon nema pristup tabelama cena;** kalkulator čita cenovnik server-only. Razlog: ako Excel partnera sadrži netto cene, javno čitanje otkriva maržu. Konačno posle odgovora Halo-a o sadržaju Excela.
- **Supabase MCP se ne povezuje sa produkcijom u MVP-u.** Razlog: lični podaci i prompt injection kroz unos kupaca i fajlove dobavljača.
- **Kupac dobija PDF ponude i predračuna kao prilog mejla;** signed URL samo za osoblje. Razlog: kratak link ističe pre otvaranja, a dug je javni link do ličnih podataka.
- **Projektni Claude Code skilovi se prave tek kad stignu odgovori klijenta** i žive u `.claude/skills/` u repou. Razlog: skil treba da opiše pravi obrazac, a ne pretpostavljeni.
- **`docs/bezbednost.md` podeljen:** prilozi B–D (skice koda, SQL, env) u `docs/bezbednost-prilozi.md`; obrisani primenjena sekcija 18 i Prilog A. Razlog: CLAUDE.md traži čitanje celog fajla pre rada na bezbednosnim delovima, a prilozi su bili ~40%% sadržaja i zastarevaju čim se napiše pravi kod. Dodato: monitoring i alarmi, isključivanje `pg_graphql`/Realtime, zaštita od zip bomb-a, MFA enrollment na `aal1` stranici, bez enumeracije naloga, obavezan pouzdan IP header za rate limit.

## 24.10.2026.
- **Nalazi spoljnog pregleda plana bezbednosti uneti su u `docs/bezbednost.md` (verzija 2) i `docs/bezbednost-prilozi.md`.** Razlog: ispravke na papiru su besplatne dok nema koda. Ključne tvrdnje su proverene: Supabase changelog za GRANT-ove, Next.js bezbednosno izdanje od 14.10.2026 i lokalni CSP vodič.
- **Bez automatskih GRANT-ova od prve migracije:** `alter default privileges … revoke`, pa eksplicitni `grant` po tabeli i funkciji. Razlog: Supabase to ionako uvodi (novi projekti od 30.05.2026, postojeći od 30.10.2026), a time važi i zlatno pravilo 4 („sve je zabranjeno dok se izričito ne dozvoli“).
- **`/admin` ima sopstveni root layout** (`src/app/(admin)/admin/layout.tsx`) sa `instant = false` i `await connection()`. Razlog: nonce CSP ne radi sa statičkim shell-om koji pravi Cache Components; javni sajt ostaje statički, sa CSP-om iz `next.config.ts`.
- **Opoziv sesije kroz bazu:** `private.session_is_valid()` u restrictive politici i u `has_role()`, a offboarding briše redove iz `auth.sessions`. Razlog: `getClaims()` ne vidi opoziv, a access token važi do 1 h. `auth.admin.signOut()` ne prima ID korisnika.
- **`create_booking_request` je `SECURITY INVOKER`, izvršiv samo za `service_role`,** sa rate limitom i ponovnim proverama unutar funkcije. Razlog: direktan grant za `authenticated` je preživljavao `revoke … from public, anon`, a `private.hit_rate_limit` se ne može zvati kroz `rpc()`.
- **`set_user_role` zaključava promene uloga advisory lock-om** i aktivira nalog samo uz potvrđen TOTP. Razlog: dva admina su istovremeno mogla da ostave sistem bez admina.
- **Brisanje slika ide kroz Storage DELETE/SELECT politike za osoblje,** ne preko secret ključa. Razlog: RLS ostaje jedina provera, kao i za upload.
- **`audit_log` i `booking_events` su append-only i kroz trigere;** brisanje ličnih podataka na zahtev znači anonimizaciju. Razlog: `service_role` i vlasnik tabele zaobilaze RLS.
- **Minimalna verzija `next` 16.4.1 / `react` 19.2.9, sa proverom u CI-ju.** Razlog: kritični RCE u `use cache`, a mi koristimo Cache Components.
