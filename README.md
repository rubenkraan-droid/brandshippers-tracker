# Brandshippers · Ads tracker

Eén statische pagina die op Supabase draait. Volgt de funnel van advertentie tot
verkoop: creative → dagcijfers → lead → call → verkoop → termijnbetalingen.

- **Frontend:** `index.html`, één self-contained bestand. Geen build, geen dependencies.
- **Backend:** Supabase project `brandshippers-ads` (`xerpxiypqxseebcdbsav`, eu-central-1)
- **Hosting:** GitHub Pages op `brandshippers.recraparcs.nl`

---

## Waarom de sleutel in de broncode mag staan

`index.html` bevat een Supabase **publishable key**. Die is bedoeld om publiek te
zijn — hij geeft in zijn eentje nul toegang. Alle tabellen staan onder Row Level
Security: je krijgt alleen data te zien als je bent ingelogd én je e-mailadres in
de tabel `access_grants` staat. Een repo die publiek is, is hier dus geen lek.

Wat er **niet** in mag: de `service_role`-sleutel. Die staat nergens in dit project
en moet daar ook nooit terechtkomen.

---

## Eenmalige installatie

### 1. Repo aanmaken en pushen

```bash
git init
git add .
git commit -m "Brandshippers ads tracker"
git branch -M main
git remote add origin https://github.com/rubenkraan-droid/brandshippers-tracker.git
git push -u origin main
```

De repo moet **public** zijn — GitHub Pages op een private repo vereist een
betaald plan.

### 2. GitHub Pages aanzetten

Repo → **Settings** → **Pages**
- Source: `Deploy from a branch`
- Branch: `main`, map `/ (root)`
- Custom domain: `brandshippers.recraparcs.nl`
- Vink **Enforce HTTPS** aan zodra het certificaat is uitgegeven (duurt tot ~15 min)

Het bestand `CNAME` in deze repo zet het domein al goed; GitHub leest hem
automatisch.

### 3. DNS bij je registrar van recraparcs.nl

Voeg één record toe:

| Type  | Naam      | Waarde                        | TTL  |
|-------|-----------|-------------------------------|------|
| CNAME | `brandshippers` | `rubenkraan-droid.github.io` | 3600 |

Let op: de waarde eindigt op `.github.io`, niet op de repo-naam.
Propagatie duurt meestal 5–30 minuten.

### 4. Supabase Auth instellen

Inloggen gaat met **e-mailadres en wachtwoord**, niet met inloglinks.

Supabase dashboard → project `brandshippers-ads`:

**Authentication → URL Configuration**
- Site URL: `https://brandshippers.recraparcs.nl`

**Authentication → Sign In / Providers → Email**
- Email provider: aan
- "Confirm email": uit (anders moet elk nieuw account nog per mail bevestigd worden)
- "Allow new users to sign up": **uit** — accounts maak je zelf aan, niemand meldt zich hier aan

### 5. Accounts aanmaken

Toegang bestaat uit twee losse dingen: een **account** om in te loggen, en een
**grant** die bepaalt wat je mag zien. Je hebt allebei nodig.

**Account:** Supabase → **Authentication** → **Users** → **Add user** →
**Create new user**. Vul het e-mailadres in, kies een wachtwoord en vink
**Auto Confirm User** aan. Geef het wachtwoord door via een kanaal waar je het
daarna weer kunt weghalen — niet per gewone mail.

Wachtwoord vergeten of wijzigen: bij die gebruiker op de drie puntjes →
**Reset password** of **Update user**. Je hoeft daar geen inloglinks voor aan te
zetten.

**Grant:** Supabase → **Table editor** → `access_grants` → rij toevoegen:

| kolom      | waarde                          |
|------------|---------------------------------|
| `email`    | het e-mailadres van die persoon |
| `is_owner` | `false` (alleen jij hoeft true) |
| `can_edit` | `true` om te mogen invoeren, `false` voor alleen lezen |

Daarna kan diegene op de site inloggen met dezelfde e-maillink.

---

## Wijzigingen doorvoeren

`index.html` aanpassen, committen, pushen. GitHub Pages publiceert binnen een
minuut. Geen build-stap.

---

## Twee dingen om in de gaten te houden

**Free-tier projecten pauzeren na 7 dagen inactiviteit.** Werk je de tracker
wekelijks bij, dan gebeurt dat niet. Ligt hij een maand stil, dan zet je hem
handmatig weer aan in het Supabase-dashboard.

**De ingebouwde e-mailservice van Supabase is gelimiteerd** (een handvol mails per
uur). Prima voor twee of drie gebruikers. Ga je met meer mensen werken, koppel dan
een SMTP-provider onder Authentication → Emails.

---

## Datamodel

| Tabel | Waarvoor |
|---|---|
| `angles` | De zeven hoeken uit het advertentieplan (A t/m G) |
| `creatives` | De 23 statics, met copy, hoek, awareness, status en oordeel |
| `test_rounds` | De vier testrondes met hypothese en conclusie |
| `ad_stats_daily` | Dagcijfers per creative uit Meta |
| `leads` | Pseudonieme leads, met investeringsband en status |
| `calls` | Kwalificatie- en salescalls, met uitkomst |
| `sales` | Verkopen met bedrag en aantal termijnen |
| `payments` | De termijnen, automatisch aangemaakt bij een verkoop |
| `story_sequences` | Een story-reeks: het verhaal, de hoek, de CTA en de reach |
| `story_frames` | Per frame: views, exits, weggetikt, linkkliks, reacties, screenshot |
| `other_costs` | Kosten buiten Meta om |
| `access_grants` | Wie mag inloggen en wie mag bewerken |

**Geen persoonsgegevens.** Leads worden alleen via `crm_id` gekoppeld aan
GoHighLevel of Pipedrive. Namen, e-mailadressen en telefoonnummers blijven in het
CRM staan — dat scheelt je een AVG-verplichting op een systeem dat je zelf beheert.

### Views

| View | Waarvoor |
|---|---|
| `v_funnel_totals` | De KPI-rij bovenaan |
| `v_creative_performance` | Per creative: spend tot omzet, met CPL, kosten per gekwalificeerde lead, show rate en ROAS |
| `v_angle_performance` | Hetzelfde geaggregeerd per hoek |
| `v_weekly` | Weekoverzicht |
| `v_story_frames` | Per frame: retentie, afvalrate naar het volgende frame, klikrate |
| `v_story_performance` | Per sequence: doorloop, totale afval, zwakste frame, kliks, leads en omzet |
| `v_open_payments` | Openstaande termijnen, achterstallig gemarkeerd |

---

## Hoe je hem gebruikt

De metric die ertoe doet is **kosten per gekwalificeerde lead**, niet CPL. Een
lead telt als gekwalificeerd bij investeringsruimte van €2.500 of meer — dat veld
vul je vanuit de vraag op je landingspaginaformulier.

Wekelijkse routine:
1. Meta-export openen, kolommen op volgorde zetten, plakken in **Invoer → Dagcijfers**
2. Nieuwe leads invoeren met hun investeringsband
3. Calls en verkopen bijwerken
4. **Hoeken** bekijken: welke boodschap levert gekwalificeerde leads, niet welke levert klikken

### Story sequences

Een sequence is één verhaal over meerdere frames. Je legt per frame de views vast;
het dashboard rekent zelf uit hoeveel procent van de starters er nog kijkt
(retentie) en waar de grootste uitstroom zit.

1. **Stories → Nieuwe sequence**: code, datum, het verhaal in één regel, de hoek
   (dezelfde A t/m G als de advertenties) en de reach
2. **Frames invoeren**: per frame `framenr · views · linkkliks · exits · weggetikt · hook`
3. Klik de sequence aan om de afvalrate per frame te zien

**Screenshots in één keer.** Sleep alle screenshots van een sequence tegelijk op de
filmstrip, of klik "Screenshots kiezen" en selecteer ze allemaal. Ze worden op
volgorde van bestandsnaam aan frame 1, 2, 3… gekoppeld — telefoonscreenshots zijn
op tijd genummerd, dus dat is meteen de goede volgorde. Zijn er meer afbeeldingen
dan frames, dan maakt hij de ontbrekende frames zelf aan; de views vul je daarna
in via "Frames invoeren". Losse screenshot vervangen: klik op dat ene frame.
Die komt in de privé Storage-bucket `story-frames` en is alleen zichtbaar voor wie
is ingelogd. De browser verkleint het beeld eerst naar maximaal 720px breed en
slaat het op als JPEG — een ruwe telefoonscreenshot van ~2 MB wordt zo ~80 kB.
Bij vijf frames per week kost dat ongeveer 20 MB per jaar; de gratis 1 GB is dus
ruim vijftig jaar genoeg.

Waar je op let: het frame met de hoogste afvalrate is waar je verhaal het publiek
kwijtraakt. Zit dat bij frame 1 of 2, dan klopt je opening niet. Zit het vlak voor
de CTA, dan is de opbouw te lang. Doordat sequences dezelfde hoeken gebruiken als
je advertenties, kun je zien of een verhaal dat in stories werkt ook als static
werkt, en andersom.

Leads en verkopen kun je aan een sequence koppelen, zodat naast de kijkcijfers ook
zichtbaar wordt wat een verhaal daadwerkelijk oplevert.

---

Beoordeel geen creative op basis van één week. Bij een ticket van €3.000 met een
callfunnel duurt het twee tot drie weken voordat een oordeel iets waard is.
