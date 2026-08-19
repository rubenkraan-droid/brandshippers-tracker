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

### 4. Supabase Auth op het nieuwe domein zetten

Zonder deze stap komen de inloglinks op de verkeerde URL uit.

Supabase dashboard → project `brandshippers-ads` → **Authentication** → **URL Configuration**
- **Site URL:** `https://brandshippers.recraparcs.nl`
- **Redirect URLs:** voeg toe `https://brandshippers.recraparcs.nl/**`

### 5. Jezelf en anderen toegang geven

Je eigen adres staat er al in. Iemand toevoegen:

Supabase → **Table editor** → `access_grants` → rij toevoegen:

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

Beoordeel geen creative op basis van één week. Bij een ticket van €3.000 met een
callfunnel duurt het twee tot drie weken voordat een oordeel iets waard is.
