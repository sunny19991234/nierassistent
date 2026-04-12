# Product Requirement Document — Sunny's NierRobot

**Versie:** 1.5  
**Datum:** april 2026  
**Status:** Actief  
**Platform:** GitHub Pages PWA  
**Repository:** dezelfde repo als `index.html`

---

## Inhoudsopgave

1. [Visie & doelstelling](#1-visie--doelstelling)
2. [Gebruikers & context](#2-gebruikers--context)
3. [Technische architectuur](#3-technische-architectuur)
4. [Functionaliteitsoverzicht](#4-functionaliteitsoverzicht)
5. [AI-kernlogica & system prompt](#5-ai-kernlogica--system-prompt)
6. [Data-architectuur & sync](#6-data-architectuur--sync)
7. [Non-functionele eisen](#7-non-functionele-eisen)
8. [Changelog](#8-changelog)
9. [Roadmap & openstaande items](#9-roadmap--openstaande-items)

---

## 1. Visie & doelstelling

Sunny's NierRobot is een persoonlijke medische kennisassistent, specifiek gebouwd voor iemand die een niertransplantatie heeft ondergaan. De tool biedt gefundeerde medische informatie op maat, laboratoriumwaarden bijhouden, symptomen loggen en vragen voorbereiden voor ziekenhuisafspraken — alles in één interface die zowel op telefoon als laptop werkt.

### Kernwaarden

- **Wetenschappelijk gefundeerd** — elke feitelijke claim voorzien van bewijsniveau (KDIGO/ERBP, expert opinion, beperkt bewijs, onzeker)
- **Privacy-first** — API-sleutel lokaal opgeslagen per apparaat, gesprekken nooit gesynchroniseerd naar de server
- **Fase-bewust** — het systeem berekent automatisch hoeveel dagen post-transplantatie het is en past urgentiedrempels en tacrolimus-streefspiegels hierop aan
- **Proactief** — signaleert alarmwaarden, medicijninteracties en urgente situaties zonder dat de gebruiker er specifiek naar vraagt

### Buiten scope

- Diagnostische conclusies — dat blijft altijd bij het behandelend team van Amsterdam UMC
- Meerdere patiënten — dit is een persoonlijk instrument voor één gebruiker
- Native app-distributie via App Store of Play Store

---

## 2. Gebruikers & context

### Primaire gebruiker

| Veld | Waarde |
|---|---|
| Naam | Sunny |
| Transplantatiedatum | 26 maart 2026 |
| Transplantatiecentrum | Amsterdam UMC |
| Donor | Levende donor — moeder |
| Voorafgaand | Geen dialyse |
| Huidaandoening | Warmte urticaria (cholinergisch/hitte-geïnduceerd) |
| Medicatie | Tacrolimus + mycofenolaatmofetil + prednison (instelbaar via Profiel-tab) |
| Apparaten | Android (homescherm PWA) + laptop (browser) |

### Gebruikssituaties

- **Direct na een labmeting** — waarden invoeren en context krijgen over wat ze betekenen
- **Voorafgaand aan polikliniek** — vragen opstellen en klachten samenvatten in de Notities-tab
- **Bij onzekerheid over symptomen of medicijnen** — gerichte vraag stellen aan de NierRobot
- **Op reizen of buiten kantooruren** — terugkijken op eerder gegeven informatie via de gesprekslog

---

## 3. Technische architectuur

### Deployment & distributie

- Eén zelfstandig `index.html`-bestand, gehost op GitHub Pages
- Geen buildstap, geen externe frameworks, vanilla JavaScript
- PWA: installeerbaar via Android "Toevoegen aan startscherm", iOS-compatibel via meta-tags
- **Deploystrategie:** Sunny uploadt gewijzigde `index.html` via GitHub.com UI → live binnen ~60 seconden. Data in Supabase en localStorage blijft onaangetast.

### Externe services

| Service | Doel | Sleutelopslag |
|---|---|---|
| Anthropic Claude API | Streaming chat-responses | Lokaal (localStorage per apparaat, nooit op server) |
| Supabase (EU-regio) | Data-sync lab, agenda, profiel | Publieke anon-key in broncode |
| Google Fonts | Crimson Pro + DM Sans | n.v.t. |

### Model & API-parameters

| Parameter | Waarde |
|---|---|
| Model | `claude-sonnet-4-20250514` |
| Max tokens | 2500 |
| Streaming | Server-sent events (SSE), directe browser-fetch |
| Context truncatie | Laatste 20 berichten per API-call |
| Anthropic-versie header | `2023-06-01` |

### Aandachtspunten voor aanpassingen

- Nooit Python-injecties gebruiken bij aanpassingen — backtick-escaping brak eerder de JavaScript. Altijd volledige schone herschrijving als er veel wijzigt.
- De backtick-regex in de markdown-parser is altijd aanwezig en altijd geldig — dit is geen fout.
- API-sleutel blijft lokaal per apparaat, nooit in Supabase opslaan.
- Gesprekken (chat log) worden alleen lokaal opgeslagen, niet gesynchroniseerd naar Supabase.

---

## 4. Functionaliteitsoverzicht

### Tabbladen

| Tab | Functie |
|---|---|
| **Chat** | Streaming medische Q&A met Claude, markdown-rendering, gespreksgeheugen (max 30 gesprekken in localStorage) |
| **Lab** | 6 meetwaarden invoeren met datumkiezer, rode alertbadges bij alarmwaarden, CSV-export, max 200 entries |
| **Grafiek** | Canvas-lijngrafiek per metric (laatste 15 metingen), referentielijnen, delta t.o.v. vorige meting met kleurcodering; bloeddruk toont twee aparte lijnen |
| **Notities** | Doktersvragen (afvinken, wissen), klachtenlog met urgentieniveaus Mild / Matig / Ernstig |
| **Log** | Historische gesprekken inzien, herladen en verwijderen; preview van eerste vraag |
| **Profiel** | Naam, geboortedatum, transplantatiedatum, ziekenhuis, donortype, donorrelatie, medicatielijst (met dosis, eenheid en innametijden), bijzonderheden, API-sleutel, verbindingstest, uitloggen |

### Labdagboek — meetwaarden

| Veld | Eenheid | Alarmlogica |
|---|---|---|
| Datum meting | — | Datumkiezer, standaard vandaag, aanpasbaar naar het verleden |
| Creatinine | µmol/L | Rood bij >120% van persoonlijk minimum |
| eGFR | ml/min/1.73m² | — |
| Tacrolimus | ng/mL | Rood bij <8 of >15 ng/mL |
| Bloeddruk | sys/dia mmHg | — |
| Gewicht | kg | — |
| Temperatuur | °C | Rood bij ≥37,8 °C |

**Datumopslag:** metingen worden opgeslagen als `YYYY-MM-DD`-string (niet als timestamp), zodat de datum altijd correct wordt weergegeven ongeacht tijdzone. Bij opslaan naar Supabase wordt `T12:00:00.000Z` toegevoegd om UTC-interpretatie te voorkomen.

### Trendgrafiek — referentielijnen per metric

| Metric | Referentielijn | Alarmgrens |
|---|---|---|
| Creatinine | 110 µmol/L (bovengrens normaal) | — |
| eGFR | 60 ml/min (grens CKD3) | <30 |
| Tacrolimus | 10 + 8 ng/mL (streefspiegels) | >15 of <6 |
| Bloeddruk | 130 mmHg (sys) + 80 mmHg (dia) | sys >160 of dia >100 |
| Gewicht | — | — |
| Temperatuur | 37,8 °C | >37,8 |

**Bloeddruk grafiek:** toont systolisch (oranje, doorgetrokken lijn) en diastolisch (blauw, gestippelde lijn) als twee aparte lijnen op één canvas, met eigen referentielijnen en legenda. Alarmkleuring rood bij systolisch ≥160 of diastolisch ≥100.

### Medicatielijst — gegevensstructuur

Elk medicijn heeft de volgende velden:

| Veld | Type | Beschrijving |
|---|---|---|
| `name` | string | Naam van het medicijn |
| `dose` | string | Hoeveelheid (bijv. `5`, `0.5`) |
| `unit` | string | Eenheid (bijv. `mg`, `mcg`, `ml`) |
| `times` | string[] | Innametijden in HH:MM-formaat (bijv. `["08:00", "20:00"]`) |

Eén medicijn kan meerdere innametijden hebben. Tijden worden gesorteerd weergegeven als blauwe badges onder de medicijnnaam. Bestaande entries zonder `unit`/`times` worden zonder fout weergegeven (terugwaartse compatibiliteit). De medicatielijst wordt gesynchroniseerd naar Supabase via het `profile`-object (veld `meds`).

---

## 5. AI-kernlogica & system prompt

### Dynamisch gegenereerde context (bij elke API-call)

De system prompt wordt live samengesteld en bevat:

- Huidige datum in Nederlands formaat
- Automatisch berekende post-transplantatiedag (op basis van `profile.txdate`, default 26 maart 2026)
- **Fasebepaling:**
  - Dag 0–30: vroege fase — tacrolimus streef 10–15 ng/mL
  - Dag 31–90: intermediaire fase — streef daalt richting 8–12 ng/mL
  - Dag 91–365: late vroege fase — onderhoud, streef 5–10 ng/mL
  - Dag >365: chronische fase
- Profieldata: naam, transplantatiedatum, donor, ziekenhuis, medicatielijst (inclusief dosis, eenheid en innametijden), bijzonderheden
- Laatste 5 labmetingen als trendcontext voor het model

### Vaste patiëntcontext (hardcoded als fallback)

- Huidaandoening: warmte urticaria (cholinergisch) — wordt proactief benoemd bij relevante vragen
- Geen dialyse voorafgaand aan transplantatie

### Verplichte gedragsregels voor het model

- **Bewijsniveau** vermelden bij elke feitelijke claim:
  - `Sterke richtlijnconsensus (KDIGO/ERBP):` — meerdere RCTs of brede internationale consensus
  - `Richtlijnaanbeveling op basis van expert opinion:` — in richtlijnen maar beperkt bewijs
  - `Beperkt bewijs / centrumafhankelijk:` — praktijken die per ziekenhuis verschillen
  - `Onzeker / actief debat:` — literatuur is verdeeld
- **Kennisgrens** augustus 2025 vermelden bij snel-evoluerende onderwerpen
- **Proactieve interactiesignalering:** grapefruit/pompelmoes (CYP3A4-remming), sint-janskruid (CYP3A4-inductie), NSAID's (nefrotoxiciteit), rauwe/ongepasteuriseerde producten (infectierisico)
- **Urgentiedrempels** — bij deze signalen direct doorverwijzen naar Amsterdam UMC of SEH:
  - Koorts ≥37,8 °C in eerste 3 maanden
  - Creatinine stijging >20–25% t.o.v. laagste waarde
  - Pijn in transplantaatregio
  - Urineproductie daling >30% op één dag
- Antwoorden in lopende tekst, geen standaard-disclaimers, opsommingstekens alleen indien noodzakelijk
- Nooit diagnostische conclusies voor Sunny persoonlijk — altijd doorverwijzen naar Amsterdam UMC

---

## 6. Data-architectuur & sync

### Supabase-tabellen

| Tabel | Kolommen (relevant) | RLS |
|---|---|---|
| `lab_entries` | user_id, entry_date, creatinine, egfr, tacrolimus, bp, weight, temp | user_id = eigen ID |
| `agenda_vragen` | user_id, local_id, text, entry_date, done | user_id = eigen ID |
| `agenda_klachten` | user_id, local_id, text, entry_date, urgency | user_id = eigen ID |
| `profile` | user_id, notes, meds, updated_at | user_id = eigen ID |

### Lokale opslag (localStorage)

| Sleutel | Inhoud | Gesynchroniseerd naar Supabase |
|---|---|---|
| `nk_api_key` | Anthropic API-sleutel | **Nooit** |
| `nk_profile` | Volledig profiel incl. profielvelden en medicatielijst | Gedeeltelijk (notes + meds) |
| `nk_lab` | Max 200 labmetingen (datum als YYYY-MM-DD string) | Ja |
| `nk_agenda` | Vragen + klachten | Ja |
| `nk_convs` | Max 30 gesprekken | **Nooit** |
| `nierrobot_auth` | Supabase sessie-token | n.v.t. |

### Sync-architectuur — Web Lock fix

De kernoplossing voor het "eeuwig hangende sync"-probleem na browser-inactiviteit:

- `autoRefreshToken: false` in Supabase client — voorkomt aanvragen van Web Locks
- Alle DB-calls via directe `fetch()` naar Supabase REST API met handmatig token (`dbFetch()`)
- Token wordt proactief vernieuwd elke 45 minuten via `setInterval`
- Sessie wordt gelezen uit `localStorage` via `readSession()` — geen `getSession()` aanroep
- Sync-guard: `syncInProgress` boolean met 15 seconden hard-timeout als failsafe
- Realtime-sync via Supabase Postgres Changes (websocket) op tabellen lab_entries, agenda_vragen, agenda_klachten, profile

---

## 7. Non-functionele eisen

| Categorie | Eis |
|---|---|
| Privacy | API-sleutel nooit op server opgeslagen. Gesprekken alleen lokaal. Supabase op EU-servers. |
| Beveiliging | Row Level Security actief op alle tabellen. Sessie-token verloopt na 1 uur, automatische refresh via fetch(). |
| Beschikbaarheid | Offline leesbaar via localStorage (chat en lab). Schrijven naar Supabase vereist netwerkverbinding. |
| Performance | Eerste render <2 seconden op 4G. Canvas-grafiek client-side berekend, geen externe charting library. |
| Compatibiliteit | iOS Safari 15+, Android Chrome 90+, desktop Chrome / Firefox / Safari. |
| Onderhoudbaarheid | Eén bestand, geen buildstap. Deploy via GitHub UI zonder technische kennis vereist. |
| Medisch | Geen diagnostische uitspraken voor individuele gebruiker. Bewijsniveau altijd vermeld. Altijd doorverwijzing bij urgentiesignalen. |

---

## 8. Changelog

### v1.5 — april 2026
**Medicatielijst uitgebreid, labdatum aanpasbaar, bloeddrukgrafiek dubbele lijn**

- **Medicatielijst:** elk medicijn heeft nu afzonderlijke velden voor dosis (getal), eenheid (bijv. mg, mcg) en innametijden. Meerdere tijden per medicijn mogelijk, weergegeven als gesorteerde blauwe badges. Terugwaartse compatibiliteit met bestaande entries zonder deze velden gegarandeerd. Medicatietijden worden ook doorgegeven aan de system prompt van de AI.
- **Lab — datumkiezer:** nieuw datumveld boven de meetvelden, standaard ingesteld op vandaag. Aanpasbaar naar een eerdere datum zodat metingen van een ziekenhuisbezoek achteraf correct geregistreerd kunnen worden. Datum opgeslagen als `YYYY-MM-DD`-string; reset automatisch naar vandaag na opslaan.
- **Bloeddrukgrafiek:** systolisch en diastolisch worden nu als twee afzonderlijke lijnen getoond op één canvas. Systolisch: oranje doorgetrokken lijn. Diastolisch: blauwe gestippelde lijn. Eigen referentielijnen voor 130 (sys) en 80 (dia) mmHg. Legenda toegevoegd. Alarmkleuring bij sys ≥160 of dia ≥100.

### v1.4 — april 2026
**Profiel uitgebreid & dynamische system prompt**
- Profielpagina herbouwd: naam, geboortedatum, transplantatiedatum, ziekenhuis, donortype, donorrelatie, bijzonderheden
- Chirurgische statusvelden verwijderd
- System prompt trekt naam, ziekenhuis, transplantatiedatum en donorinfo dynamisch uit profiel i.p.v. hardcoded waarden
- "Agenda"-tab en -panel hernoemd naar "Notities"
- Nier-SVG icoon toegevoegd aan header

### v1.3 — maart 2026
**Web Lock bug fix — sessie-architectuur herschreven**
- Kernprobleem opgelost: Supabase `autoRefreshToken: true` veroorzaakte ghost lock na browser-inactiviteit, waardoor sync eeuwig hing
- Fix: `autoRefreshToken: false`, alle `getSession()`-aanroepen vervangen door `ensureSession()`, token refresh via directe `fetch()`
- `syncInProgress` hard-timeout van 15 seconden als failsafe toegevoegd
- Sign-out knop hernoemd naar `doSignOut()`, realtime channel teardown vóór signOut

### v1.2 — maart 2026
**Streaming chat & markdown-renderer**
- Claude API streaming via server-sent events (SSE) geïmplementeerd
- Eigen markdown-parser: koppen H1–H3, bold, italic, lijsten, codeblokken, blockquote, HR
- Typing-indicator animatie (drie pulsererende stippen)
- Context-venster truncatie op 20 berichten

### v1.1 — maart 2026
**Supabase sync & Realtime**
- Lab-entries, agenda-items en profiel gesynchroniseerd via Supabase REST API
- Row Level Security ingesteld op alle tabellen
- Realtime Postgres Changes voor multi-apparaat live updates
- CSV-export labdagboek
- Proactieve token-refresh timer (45 minuten interval)

### v1.0 — maart 2026
**Initiële release**
- Eerste werkende versie: Chat (niet-streaming), labdagboek, canvas-trendgrafiek, notities/agenda, gesprekslog, profielpagina
- Supabase auth (email + wachtwoord), persistSession via localStorage
- PWA-meta-tags voor Android homescherm installatie
- Zes meetwaarden: creatinine, eGFR, tacrolimus, bloeddruk, gewicht, temperatuur

---

## 9. Roadmap & openstaande items

### Mogelijke uitbreidingen

| Prioriteit | Feature | Beschrijving |
|---|---|---|
| Hoog | Urgentie-detectie in chat | Bij alarmwoorden (koorts, pijn, weinig plassen) een prominente waarschuwingskaart tonen boven het chatantwoord |
| Hoog | Tacrolimus-alert fase-afhankelijk | Alarmgrenzen in lab-UI dynamisch aanpassen op basis van post-transplantatiedag in plaats van vaste waarden |
| Middel | CSV-import labwaarden | Historische data inladen vanuit extern exportbestand |
| Middel | Lab-context knop in chat | Knop om recente meetwaarden direct als chatbericht te sturen naar de NierRobot |
| Laag | Gesprekslog naar Supabase | Cross-apparaat toegang tot gespreksgeschiedenis |
| Laag | Pushnotificaties | Service Worker-notificatie bij gemiste meting (optioneel, opt-in) |

### Bekende beperkingen

- Gesprekken worden niet gesynchroniseerd — bij wisselen van apparaat begint de chat opnieuw
- Tacrolimus-alarmgrenzen in de lab-UI zijn vaste waarden (8–15 ng/mL), niet fase-afhankelijk
- Offline: labwaarden en notities leesbaar, maar geen schrijven naar Supabase zonder netwerk
- Kennisgrens AI-model: augustus 2025

---

*Dit document wordt bijgehouden samen met `index.html` in dezelfde GitHub-repository. Bij elke goedgekeurde feature-aanpassing wordt de changelog bijgewerkt en worden relevante secties herzien.*
