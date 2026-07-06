Kalorietæller – App-spec til Claude Code

Formål

En simpel iPhone-webapp (PWA, "Add to Home Screen") til at registrere kalorieindtag manuelt, uden App Store. Køres som en statisk HTML/JS-side hostet på for eksempel GitHub Pages.

Datafelter pr. registrering

Måltid – valgliste: Morgen, Middag, Aften, Mellemmåltid, Snack
Følgende kolonner knyttes til det valgte måltid


Dato/tid – sættes automatisk ved oprettelse (ikke redigerbar)

Fødevare – fritekst (fx "mælk", "ost")

Kalorier pr. 100 g – tal, indtastes manuelt

Gram – tal, indtastes manuelt

Kalorier (beregnet) – kalorier_pr_100g * gram / 100, skrivebeskyttet/afledt felt


Visning

Total kalorier i dag og i alt, øverst

Liste over registreringer, grupperet pr. dag (nyeste øverst), med dagstotal pr. gruppe

Hver registrering viser: måltid (farvet mærkat), fødevare, klokkeslæt, gram, kcal/100g, samlet kcal

Mulighed for at slette en registrering

Lagring og synkronisering

Lokalt: data caches i browserens localStorage, så appen virker offline og loader hurtigt
Bemærk: localStorage er KUN en cache — iOS Safari kan slette den efter ~7 dages inaktivitet. Den permanente kopi lever i skyen.
Cloud (kilde til sandhed): Supabase (hostet Postgres) med login → data ligger server-side og går ikke tabt

Login: magic-link via email (Supabase Auth signInWithOtp) — ingen adgangskode, session huskes længe via refresh-token
Tabel: entries med kolonner id, user_id, datetime, food, meal, kcal_100, grams, kcal
Row Level Security (RLS): hver bruger kan kun se/redigere rækker hvor user_id = auth.uid()
Debounced upsert (ca. 800 ms efter ændring) + fuld pull ved login (fletter lokale-kun registreringer op i skyen)
Eksport til JSON-fil som ekstra manuel backup (tandhjul-ikon)


Vigtigt: hver bruger har sin egen private log (ikke delt) — to personer bruger samme app-URL, men logger ind med hver deres email, og får dermed adskilte data (håndhævet af RLS)

Krav til opsætning (uden for koden)

Hostes på GitHub Pages som statisk side (fil skal hedde index.html)
Kræver et gratis Supabase-projekt med:

En tabel entries (se SQL nedenfor) med Row Level Security slået til
Email-login (magic link) aktiveret under Authentication → Providers
GitHub Pages-URL'en tilføjet under Authentication → URL Configuration (Site URL + Redirect URLs)


Supabase Project URL + anon (public) key indsættes i koden (KONFIGURATION øverst i index.html) eller via tandhjul-ikonet.
Bemærk: anon-nøglen er designet til at ligge offentligt i frontend — sikkerheden kommer fra RLS, ikke fra at skjule nøglen. Ingen private secrets i koden.

SQL til at oprette tabel + policies (kør i Supabase → SQL Editor):

  create table entries (
    id text primary key,
    user_id uuid not null default auth.uid() references auth.users(id) on delete cascade,
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