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
-- Tabel til alle kalorie-registreringer
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

-- Slå Row Level Security til (uden dette kan ingen læse/skrive)
alter table entries enable row level security;

-- Hver bruger må kun røre sine egne rækker
create policy "egne raekker - laes"    on entries for select using (auth.uid() = user_id);
create policy "egne raekker - indsaet" on entries for insert with check (auth.uid() = user_id);
create policy "egne raekker - opdater" on entries for update using (auth.uid() = user_id);
create policy "egne raekker - slet"    on entries for delete using (auth.uid() = user_id);
```

3. Du skulle gerne se **Success. No rows returned**.

**Kontrol:** gå til **Table Editor** → du bør se tabellen `entries`, og øverst
et lille skjold-ikon der viser at RLS er **enabled**.

---

## Del 4 – Aktivér email-login (magic link)

1. Gå til **Authentication** → **Sign In / Providers** (eller **Providers**).
2. Sørg for at **Email** er slået til. Magic link er aktiveret som standard.
   - Du kan slå **Confirm email** til – det er fint; magic-linket bekræfter
     samtidig emailen.
3. Gå til **Authentication** → **URL Configuration**:
   - **Site URL:** sæt til din kommende GitHub Pages-URL (se Del 6). Kender du
     den ikke endnu, så udfyld den her bagefter.
   - **Redirect URLs:** klik **Add URL** og tilføj den samme URL.
   - Til lokal test kan du også tilføje `http://localhost:*`.

> **Vigtigt:** login-linket i mailen virker kun, hvis den URL, appen åbnes fra,
> står på listen over Redirect URLs. Glemmer du dette, får du fejlen
> *"requested path is invalid"* når du klikker på linket.

> **Gratis email-grænse:** Supabase' indbyggede mail-afsender er beregnet til
> lav volumen (nogle få mails i timen). Det er rigeligt til privat brug med få
> brugere. Vil du sende mange, kan du senere koble en SMTP-udbyder på under
> samme menu – men det er ikke nødvendigt for at komme i gang.

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
2. Tryk **⚙️** → skriv din email → **Send login-link**.
3. Åbn mailen på telefonen, tryk linket → du sendes tilbage til appen og er
   logget ind. Status-prikken øverst bliver grøn: *"Synkroniseret …"*.
4. Opret en registrering. Åbn **Table Editor** i Supabase → du bør se rækken i
   `entries` med dit `user_id`.
5. **Tilføj til hjemmeskærm:** tryk Del-knappen i Safari → **Føj til
   hjemmeskærm**. Nu åbner appen i fuldskærm som en rigtig app.

**Test at data overlever:** log ind på en anden enhed (eller en privat
browserfane) med samme email → dine registreringer dukker op. Det bekræfter,
at data ligger i skyen og ikke kun lokalt.

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
| *"requested path is invalid"* efter klik på login-link | App-URL'en er ikke i **Redirect URLs**. Tilføj den præcise URL i Supabase (Del 4). |
| Login-mail kommer ikke | Tjek spam. Supabase' gratis-mail har lav rate – vent lidt og prøv igen. |
| Data vises ikke efter login | Åbn browser-konsollen. Ofte manglende RLS-policies – kør SQL'en i Del 3 igen. |
| *"new row violates row-level security policy"* | `insert`-policyen mangler, eller `user_id` sættes forkert. Genkør Del 3. |
| Login glemmes hele tiden | Normalt hvis du bruger privat/inkognito-fane. I almindelig Safari huskes sessionen længe. |

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
