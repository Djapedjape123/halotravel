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
