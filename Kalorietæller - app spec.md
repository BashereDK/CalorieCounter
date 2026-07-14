Kalorietæller – App-spec til Claude Code

Formål

En simpel iPhone-webapp (PWA, "Add to Home Screen") til at registrere kalorieindtag manuelt, uden App Store. Køres som en statisk HTML/JS-side hostet på for eksempel GitHub Pages.

Datamodel

Et måltid er en container for flere fødevare-registreringer, så man kun vælger
måltidstype én gang og derefter kan tilføje fx havregryn, mælk og sukker under
samme måltid.

Måltid (container)
Måltidstype – valgliste: Morgen, Middag, Aften, Mellemmåltid, Snack
Dato/tid – sættes automatisk ved oprettelse af måltidet (ikke redigerbar)

Fødevare (flere pr. måltid)
Fødevare – fritekst (fx "mælk", "ost"), kan også udfyldes ved stregkode-skanning
Kalorier pr. 100 g – tal, indtastes manuelt eller hentes ved stregkode-skanning
Gram – tal, indtastes manuelt
Kalorier (beregnet) – kalorier_pr_100g * gram / 100, skrivebeskyttet/afledt felt

Stregkode-skanning: knappen "📷 Skan stregkode" i fødevare-modalen åbner kameraet
og aflæser stregkoden. Navn + kcal/100g udfyldes automatisk. Kilde-rækkefølge:
1) Supabase products-tabel (delt cache, hurtig/offline), 2) Open Food Facts API
(gratis, ingen nøgle) – og resultatet gemmes derefter i products til næste gang,
3) manuel indtastning hvis varen ikke findes. iOS Safari mangler det native
BarcodeDetector-API, så afkodningen sker med ZXing (self-hostet i repo'et, ikke
CDN). Kamerabilleder behandles lokalt og forlader ikke enheden.

Vægt (én registrering pr. dag)
Dato – sættes automatisk til dagen i dag (datoen er nøglen, ikke et løbenummer)
Kg – tal, indtastes manuelt. Registrerer man igen samme dag, opdateres dagens vægt
i stedet for at oprette en ny linje.

Adgang: kun brugere der er logget ind kan oprette måltider, tilføje fødevarer og registrere vægt.


Visning

Appen har to faner nederst: Kalorier og Vægt.

Kalorier-fanen:

Total kalorier i dag og i alt, øverst

Liste over måltider som kort, grupperet pr. dag (nyeste øverst), med dagstotal pr. gruppe

Hvert måltidskort viser: måltidstype (farvet mærkat), klokkeslæt, måltidets totale kcal, og
en liste af de fødevarer der indgår (fødevare, gram, kcal/100g, kcal pr. fødevare)

Knap på hvert kort til at tilføje flere fødevarer til måltidet

Mulighed for at slette en enkelt fødevare eller hele måltidet

Vægt-fanen:

Seneste vægt vist stort, med dato

Graf over udviklingen: hver registrering er et datapunkt, og ovenpå tegnes en linje
med det glidende gennemsnit (trailing, 7 registreringer). Dag-til-dag-udsving i vægt
er for en stor del vand — trend-linjen er signalet. X-aksen følger datoen, så en pause
i registreringerne også ses som et hul. Grafen er ren inline SVG (intet bibliotek) og
vises først ved mindst to registreringer.

Historik med alle registreringer (nyeste øverst), som kan slettes enkeltvis

Lagring og synkronisering

Lokalt: data caches i browserens localStorage, så appen virker offline og loader hurtigt
Bemærk: localStorage er KUN en cache — iOS Safari kan slette den efter ~7 dages inaktivitet. Den permanente kopi lever i skyen.
Cloud (kilde til sandhed): Supabase (hostet Postgres) med login → data ligger server-side og går ikke tabt

Login: email + adgangskode via Supabase Auth (signUp opretter konto, signInWithPassword logger ind) — session huskes længe via refresh-token.
Glemt adgangskode: resetPasswordForEmail sender et nulstillingslink tilbage til appen. Åbnes appen via linket (event PASSWORD_RECOVERY), beder den straks om en ny adgangskode. Nødvendigt, fordi "Sæt / skift adgangskode" kræver en aktiv session — uden nulstilling er en bruger uden adgangskode låst ude for evigt. Adgangskode bruges frem for magic-link/kode, fordi et link fra Mail på iOS åbner i Safari og ikke i hjemmeskærm-appen (som har sit eget login-rum), og fordi kode-i-mail kræver custom SMTP. Adgangskode-login sker helt inde i appen. Kræver at "Confirm email" er slået fra i Supabase, så signUp logger ind med det samme.
Tabel: entries med kolonner id, user_id, meal_id, datetime, food, meal, kcal_100, grams, kcal
(meal_id grupperer fødevarer i samme måltid; datetime + meal er måltidets tid/type, denormaliseret på hver fødevare. Tomme måltider uden fødevarer lever kun lokalt indtil første fødevare tilføjes.)
Tabel: weights med kolonner user_id, date, kg — primærnøgle (user_id, date), så en upsert af
samme dag fra to enheder rammer den samme række og aldrig giver dubletter. Vægt synkroniseres
ved login/app-start (ikke via Realtime; den ændrer sig sjældent).
Row Level Security (RLS): hver bruger kan kun se/redigere rækker hvor user_id = auth.uid()
Debounced upsert (ca. 800 ms efter ændring) + fuld pull ved login (fletter lokale-kun registreringer op i skyen)
Eksport til JSON-fil som ekstra manuel backup (tandhjul-ikon)


Vigtigt: hver bruger har sin egen private log (ikke delt) — to personer bruger samme app-URL, men logger ind med hver deres email, og får dermed adskilte data (håndhævet af RLS)

Krav til opsætning (uden for koden)

Hostes på GitHub Pages som statisk side (fil skal hedde index.html)
Kræver et gratis Supabase-projekt med:

En tabel entries (se SQL nedenfor) med Row Level Security slået til
En tabel weights til vægt-sporing – privat pr. bruger, RLS slået til (uden den gemmes vægt kun lokalt)
En tabel products (valgfri) til stregkode-cache – delt katalog: alle indloggede kan læse/bidrage, RLS slået til
Email-provider aktiveret under Authentication → Providers, og "Confirm email" slået fra (så signUp logger ind uden email-bekræftelse)
GitHub Pages-URL'en tilføjet under Authentication → URL Configuration (Site URL + Redirect URLs)


Supabase Project URL + anon (public) key indsættes i koden (KONFIGURATION øverst i index.html) eller via tandhjul-ikonet.
Bemærk: anon-nøglen er designet til at ligge offentligt i frontend — sikkerheden kommer fra RLS, ikke fra at skjule nøglen. Ingen private secrets i koden.

SQL til at oprette tabel + policies (kør i Supabase → SQL Editor):

  create table entries (
    id text primary key,
    user_id uuid not null default auth.uid() references auth.users(id) on delete cascade,
    meal_id text,
    datetime timestamptz not null,
    food text not null,
    meal text,
    kcal_100 numeric not null,
    grams numeric not null,
    kcal numeric not null,
    created_at timestamptz not null default now()
  );

  alter table entries enable row level security;

  create policy "egne rækker – læs"   on entries for select using (auth.uid() = user_id);
  create policy "egne rækker – indsæt" on entries for insert with check (auth.uid() = user_id);
  create policy "egne rækker – opdater" on entries for update using (auth.uid() = user_id);
  create policy "egne rækker – slet"    on entries for delete using (auth.uid() = user_id);

Design

Mobilførst, mørkt tema, native iPhone-look
PWA-meta-tags så den kan tilføjes til hjemmeskærmen og køre i fuldskærm (apple-mobile-web-app-capable)
Ingen backend-server — ren client-side HTML/CSS/JS