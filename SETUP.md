# Kalorietæller – Setup-guide

Denne guide tager dig fra tom mappe til en kørende app på din iPhone, hvor data
gemmes sikkert i skyen og aldrig går tabt.

**Arkitektur i korte træk:**
- **Frontend:** én statisk `index.html` hostet gratis på GitHub Pages.
- **Data:** Supabase (hostet Postgres-database) – kilden til sandhed.
- **Offline:** `localStorage` bruges som cache, så appen virker uden net.
- **Login:** magic-link via email (ingen adgangskode).
- **Adskillelse:** Row Level Security sikrer, at hver bruger kun ser egne data.

Samlet tidsforbrug: ca. 15 minutter. Alt er gratis.

---

## Del 1 – Opret Supabase-projekt

1. Gå til [supabase.com](https://supabase.com) og log ind (fx med GitHub).
2. Klik **New project**.
   - **Name:** fx `kalorietaeller`
   - **Database Password:** vælg en stærk – du får sjældent brug for den, men gem den.
   - **Region:** vælg en tæt på dig, fx *Central EU (Frankfurt)*.
3. Klik **Create new project** og vent ~1 minut, til databasen er klar.

---

## Del 2 – Hent dine nøgler

1. I projektet: gå til **Project Settings** (tandhjul nederst) → **API**
   (kan også hedde **Data API**).
2. Notér to værdier – du skal bruge dem i Del 5:
   - **Project URL** – feltet øverst, ser ud som `https://abcdefgh.supabase.co`.
     Kan du ikke finde det, så kig på browser-adressen inde i projektet
     (`app.supabase.com/project/abcdefgh`) – `abcdefgh` er dit projekt-ref, og
     URL'en er så `https://abcdefgh.supabase.co`. Der er også en **Connect**-knap
     i toppen, der viser URL'en.
   - **`anon` `public` key** – en lang streng der starter med `eyJhbGci...`.
     I den nyere dashboard-udgave ligger den under overskriften
     **"Legacy anon, service_role API keys"** – det er helt fint at bruge; den er
     ikke usikker. (Ser du i stedet en nyere **"Publishable key"** `sb_publishable_...`,
     virker den også – men `anon` under Legacy er den nemmeste.)

> **Er det sikkert at lægge `anon`-nøglen offentligt i koden?**
> Ja. `anon`-nøglen er designet til at ligge i frontend. Den kan ikke læse eller
> ændre data på egen hånd – al adgang styres af Row Level Security-reglerne,
> vi opretter i Del 3. Læg **aldrig** `service_role`-nøglen i frontend – den
> springer sikkerheden over. Vi bruger kun `anon`.

---

## Del 3 – Opret tabel og sikkerhedsregler

1. I Supabase: gå til **SQL Editor** → **New query**.
2. Indsæt hele blokken herunder og klik **Run**.

```sql
-- Tabel til alle kalorie-registreringer.
-- Hver række er én fødevare. Fødevarer samles i et måltid via meal_id
-- (måltidets tidspunkt og type ligger denormaliseret i datetime + meal).
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

-- Giv den indloggede rolle adgang til tabellen (RLS filtrerer stadig rækkerne).
-- Supabase giver normalt disse rettigheder automatisk, men vær eksplicit for
-- en sikkerheds skyld – uden dem fejler alle inserts med
-- "permission denied for table entries" (code 42501).
grant usage on schema public to authenticated;
grant select, insert, update, delete on table entries to authenticated;

-- Slå Row Level Security til (uden dette kan ingen læse/skrive)
alter table entries enable row level security;

-- Hver bruger må kun røre sine egne rækker
create policy "egne raekker - laes"    on entries for select to authenticated using (auth.uid() = user_id);
create policy "egne raekker - indsaet" on entries for insert to authenticated with check (auth.uid() = user_id);
create policy "egne raekker - opdater" on entries for update to authenticated using (auth.uid() = user_id);
create policy "egne raekker - slet"    on entries for delete to authenticated using (auth.uid() = user_id);

-- Slå live-synkronisering (Realtime) til for tabellen, så ændringer på én
-- enhed dukker op på dine andre enheder med det samme. RLS gælder også her,
-- så du får kun dine egne ændringer.
alter publication supabase_realtime add table entries;
```

3. Du skulle gerne se **Success. No rows returned**.

**Kontrol:** gå til **Table Editor** → du bør se tabellen `entries`, og øverst
et lille skjold-ikon der viser at RLS er **enabled**.

> **Har du allerede oprettet tabellen fra en tidligere version?** Kør disse
> to linjer i SQL Editor — dine data bevares. Den første tilføjer måltids-
> kolonnen; den anden slår live-synkronisering til:
>
> ```sql
> alter table entries add column if not exists meal_id text;
> alter publication supabase_realtime add table entries;
> ```
>
> Eksisterende registreringer uden `meal_id` vises automatisk som hver deres
> ét-fødevares måltid. (Kører du `add table` og den allerede er tilføjet, får
> du blot en uskadelig fejl — så er den bare klar i forvejen.)

---

## Del 4 – Aktivér login med email + adgangskode

Appen logger ind med **email og adgangskode** – ikke magic-link. Grunden er
iPhone: en web-app lagt på hjemmeskærmen kører i sit eget lukkede rum og deler
ikke login med Safari. Et magic-link fra Mail åbner altid i Safari (ikke i
hjemmeskærm-appen), så sessionen havner det forkerte sted. En adgangskode tastes
ind *inde i appen* og rammer aldrig det problem. (At vise en kode i mailen i
stedet kræver betalt/ekstern SMTP, som vi undgår.)

1. Gå til **Authentication** → **Sign In / Providers** (eller **Providers**) →
   **Email**.
2. Sørg for at **Email** er slået til.
3. **Slå "Confirm email" FRA.** Det er det vigtigste trin. Er den slået til,
   sender Supabase en bekræftelsesmail med et link – som igen åbner i Safari og
   giver samme problem. Med den slået fra bliver du logget ind med det samme, når
   du opretter kontoen i appen. (Til privat brug med få, betroede brugere er det
   helt fint.)
4. Gå til **Authentication** → **URL Configuration**:
   - **Site URL:** sæt til din kommende GitHub Pages-URL (se Del 6). Kender du
     den ikke endnu, så udfyld den her bagefter.
   - **Redirect URLs:** klik **Add URL** og tilføj den samme URL.
   - Til lokal test kan du også tilføje `http://localhost:*`.

> **Bemærk:** Da vi ikke bruger magic-link, sendes der ingen login-mails, og du
> rammer ikke Supabase' gratis-mailgrænse. Adgangskoden gemmes sikkert (hashet)
> i Supabase Auth – aldrig i din kode eller i localStorage.

---

## Del 5 – Sæt dine nøgler ind i appen

Du kan gøre det på to måder – vælg én.

**Metode A – redigér koden (anbefalet, så nøglerne følger med i repoet):**

Åbn `index.html` og find de to linjer øverst i `<script>`-blokken:

```js
const SUPABASE_URL  = localStorage.getItem('kalorie_sb_url') || 'DIN_SUPABASE_URL';
const SUPABASE_ANON = localStorage.getItem('kalorie_sb_key') || 'DIN_ANON_KEY';
```

Erstat `DIN_SUPABASE_URL` og `DIN_ANON_KEY` med værdierne fra Del 2:

```js
const SUPABASE_URL  = localStorage.getItem('kalorie_sb_url') || 'https://abcdefgh.supabase.co';
const SUPABASE_ANON = localStorage.getItem('kalorie_sb_key') || 'eyJhbGciOi...din-lange-noegle...';
```

**Metode B – indtast i appen (uden at røre koden):**

Åbn appen → tryk **⚙️** → fold **Avanceret** ud → indsæt Project URL og
anon-nøgle → **Gem forbindelse**. Værdierne gemmes i browserens localStorage på
netop den enhed.

---

## Del 6 – Hosting på GitHub Pages

> **Tip om repo-navn:** undgå `æ ø å` i selve GitHub-repoets navn – vi bruger
> `CalorieCounter`. Æøå i URL'en bliver til rodede `%`-koder, som kan drille i
> Supabase' Redirect URLs. Den lokale mappe må gerne hedde `Kalorietæller`.

1. **Opret repo på GitHub** (hvis ikke gjort): på github.com → **New repository**
   → navn `CalorieCounter` → **Create**.

2. **Push din kode.** I projektmappen:

   ```bash
   git add index.html SETUP.md "Kalorietæller - app spec.md"
   git commit -m "Kalorietæller med Supabase-sync"
   git branch -M main
   git remote add origin https://github.com/BashereDK/CalorieCounter.git
   git push -u origin main
   ```

   (Har du allerede et remote, så spring `remote add` over.)

3. **Slå Pages til.** På GitHub: **Settings** → **Pages**:
   - **Source:** *Deploy from a branch*
   - **Branch:** `main`, mappe `/ (root)` → **Save**.
   - Efter ~1 minut vises din URL, fx:
     `https://basheredk.github.io/CalorieCounter/`

4. **Luk cirklen:** kopiér den URL og læg den ind i Supabase under
   **Authentication → URL Configuration** (både Site URL og Redirect URLs,
   jf. Del 4). Uden dette virker login-linket ikke.

---

## Del 7 – Test og tilføj til hjemmeskærmen

1. Åbn din GitHub Pages-URL i **Safari på iPhone**.
2. **Tilføj til hjemmeskærm først:** tryk Del-knappen i Safari → **Føj til
   hjemmeskærm**. Åbn så appen fra det nye ikon – den kører nu i fuldskærm som
   en rigtig app.
3. I appen: tryk **⚙️** → skriv din email + en selvvalgt adgangskode (mindst
   6 tegn) → **Opret konto**. Du bliver logget ind med det samme, og status-
   prikken øverst bliver grøn: *"Synkroniseret …"*.
   - Får du i stedet beskeden om at *"Confirm email"* er slået til, så slå den
     fra i Supabase (Del 4, trin 3) og tryk **Log ind**.
4. Næste gang (og på computeren): skriv samme email + adgangskode → **Log ind**.
5. Opret en registrering. Åbn **Table Editor** i Supabase → du bør se rækken i
   `entries` med dit `user_id`.

**Test at data overlever:** log ind på en anden enhed (eller en privat
browserfane) med samme email + adgangskode → dine registreringer dukker op. Det
bekræfter, at data ligger i skyen og ikke kun lokalt.

---

## Del 8 – Backup

- I appen: **⚙️** → **Eksportér alle data (JSON-backup)** gemmer en fil med alle
  registreringer. God at tage en gang imellem som ekstra sikkerhed.
- Supabase tager også automatiske database-backups (se **Database → Backups**).

---

## Fejlfinding

| Symptom | Årsag / løsning |
|---|---|
| Status står på *"Ikke konfigureret"* | URL/anon-nøgle mangler. Tjek Del 5. |
| *"Email not confirmed"* ved login | *"Confirm email"* er slået til i Supabase. Slå den fra (Del 4, trin 3) og log ind igen. |
| *"Invalid login credentials"* | Forkert email/adgangskode – eller kontoen findes ikke endnu. Første gang skal du bruge **Opret konto**. |
| *"User already registered"* ved Opret konto | Kontoen findes allerede. Brug **Log ind** i stedet. |
| Gammel konto (oprettet med magic-link) har ingen adgangskode | Er du stadig logget ind på én enhed (grøn prik): åbn **⚙️** → **Sæt / skift adgangskode** → gem. Log så ind med den på dine andre enheder. Er du logget helt ud alle steder, kan du nulstille adgangskoden under **Authentication → Users** i Supabase. |
| Data vises ikke efter login | Åbn browser-konsollen. Ofte manglende RLS-policies – kør SQL'en i Del 3 igen. |
| *"new row violates row-level security policy"* | `insert`-policyen mangler, eller `user_id` sættes forkert. Genkør Del 3. |
| Login glemmes hele tiden | Normalt hvis du bruger privat/inkognito-fane. I almindelig Safari/hjemmeskærm-app huskes sessionen længe. |
| Kan ikke logge ind i hjemmeskærm-appen | Log ind *inde i appen* med email + adgangskode (hjemmeskærm-appen deler ikke login med Safari, men adgangskode-login virker fint indefra). |

---

## Sådan hænger det sammen (til senere reference)

```
 iPhone (Safari / hjemmeskærm)
        │
        │  index.html  (GitHub Pages, statisk)
        │     ├─ localStorage  ← offline-cache, kan ryddes af iOS
        │     └─ supabase-js   ← taler med skyen
        ▼
 Supabase
   ├─ Auth (magic link)      → hvem er du?  (auth.uid())
   └─ Postgres: entries      → dine data, beskyttet af RLS
```

Kilden til sandhed er altid `entries`-tabellen i Supabase. localStorage er kun
en hurtig kopi. Derfor mister du ikke data, selv hvis telefonen rydder cachen.
