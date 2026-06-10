# CRIMINAL TRACKER — Colorado Intelligence System
**Public crime pattern intelligence. Offender registry mapping. AI-powered field analysis.**

A browser-based law enforcement intelligence tool for Colorado, built on publicly available crime data from the Colorado Bureau of Investigation (CBI) sex offender registry and FBI National Incident-Based Reporting System (NIBRS). Criminal Tracker maps where crime concentrates, not who committed it — giving law enforcement, commissioners, and community leaders a clear geographic picture of offense patterns across the state with focus on the San Luis Valley region.

> ⚠ **Current version uses sample data for demonstration purposes.** Architecture is production-ready. Real CBI registry data is obtained via formal records request and drops in as a single variable swap.

---

## What It Does

Criminal Tracker ingests publicly available crime data, applies an intelligence scoring model to geographic grid cells, and renders the results as an interactive map with ranked hot zones, county-level choropleth overlays, and an AI field analyst that interprets the data and answers tactical questions in real time.

Three crime categories are tracked simultaneously:
- **Sex Offenders** — geocoded from CBI Adult Sex Offender Registry
- **Drug Offenses** — FBI NIBRS arrest and incident data by county
- **Violent Offenses** — FBI NIBRS assault, robbery, and related data by county

---

## How It Works

### Architecture

Criminal Tracker is a single-file HTML application — no build step, no backend, no database. Open it in any modern browser and it runs. All computation is client-side.

```
criminal-tracker.html
├── CSS         — Dark tactical UI, CSS variables, blue/amber/red palette
├── Leaflet     — Map rendering (OSM/ESRI tiles, GeoJSON, heatmap)
├── JavaScript
│   ├── SAMPLE_OFFENDERS       — Seed dataset (replace with real CBI data)
│   ├── COUNTY_CRIME_DATA      — FBI NIBRS 2022 county-level counts, all 64 CO counties
│   ├── runScan()              — Filter pipeline, triggers all render functions
│   ├── buildGridCells()       — 0.25° grid aggregation + four-factor scoring
│   ├── renderMap()            — Heat layer + grid zone rectangles
│   ├── renderIntelBrief()     — Tactical summary with signal-strength recommendation
│   ├── renderZoneList()       — Ranked zone sidebar cards
│   ├── loadCountyLayer()      — Lazy-fetch Colorado county GeoJSON choropleth
│   ├── countyStyle()          — Crime-density color scaling per county
│   ├── buildFIPSMap()         — FIPS → county name lookup for all 64 CO counties
│   ├── setBaseLayer()         — MAP/SAT tile swap
│   ├── ScanGrid (L.GridLayer) — Dynamic canvas grid, moves with map zoom/pan
│   └── Field Intelligence AI  — Multi-provider BYOK chat with live session context
└── External dependencies (CDN)
    ├── leaflet@1.9.4
    ├── leaflet.heat@0.2.0
    ├── IBM Plex Mono / IBM Plex Sans / Bebas Neue (Google Fonts)
    └── OpenStreetMap + ESRI World Imagery tiles
```

### Data Flow

```
User clicks RUN SCAN
        │
        ▼
Filter pipeline
  → Crime type filter (sex / drug / violent — any combination)
  → Region filter (SLV counties only vs all Colorado)
  → Year filter (2020 / 2021 / 2022 / all years)
  → Severity filter (all / felony / SVP only)
        │
        ▼
buildGridCells()
  → Bin records into 0.25° grid cells (~13–17 miles per side in Colorado)
  → Aggregate by crime type, count SVP flags, note recency
  → Score each cell (four-factor model)
  → Sort by score descending
        │
        ▼
renderMap()          →  Heat layer + grid zone rectangles
renderStats()        →  Signal Summary panel counts
renderIntelBrief()   →  Tactical brief with recommendation
renderZoneList()     →  Ranked hot zone sidebar

[Optional] COUNTIES button
  → Fetch Colorado county GeoJSON (Plotly open dataset, FIPS-indexed)
  → Map FIPS codes to county names via buildFIPSMap()
  → Apply countyStyle() choropleth shading based on crime density per 100k
  → SLV focus counties highlighted with blue border
  → Click any county for breakdown popup

[Optional] FIELD AI button
  → Select provider (Groq / Claude / OpenAI / Grok)
  → Enter API key (session only, never stored)
  → Chat with full live session context injected automatically
```

---

## Scoring Model

Every 0.25° grid cell is scored 0–100 based on four factors:

| Factor | Weight | What It Measures |
|---|---|---|
| Density | 40 pts | Log-scaled record count per cell — more records, higher signal |
| Severity | 30 pts | Weighted by offense type — sex offenses highest, violent next, drug lower |
| SVP Bonus | 20 pts | Sexually Violent Predators (SVP) add significant risk weight |
| Recency | 10 pts | More recent records score higher — 2022 = 10pts, 2021 = 6pts, older = 3pts |

### Score Interpretation

| Score | Classification | Recommended Action |
|---|---|---|
| ≥ 75 | CRITICAL | Priority patrol, monitoring, parole check-ins |
| 50–74 | ELEVATED | Increased surveillance, community awareness |
| < 50 | MODERATE | Standard monitoring protocols |

Score breakdowns are shown in every zone popup — each factor displayed individually so analysts can see exactly why a zone scored high.

---

## Features

### Map Layers
- **Heat Map** — Density-weighted intensity map. Color gradient: blue (low presence) → amber (moderate) → orange (significant) → red (critical). SVP-flagged records burn hotter.
- **Grid Zones** — 0.25° scored rectangles, color-coded by score tier. Click any zone for full score breakdown popup.
- **Both** — Heat underneath grid for maximum information density.

### County Choropleth
- All 64 Colorado counties shaded by crime density per 100,000 population
- Color scales from dark (low density) to red (high density) based on active crime type filters
- SLV focus counties (Alamosa, Conejos, Costilla, Rio Grande, Saguache, Mineral) highlighted with blue border
- Click any county for offense counts, population, and per-100k rate
- Loaded on demand — click COUNTIES button

### Base Layers (MAP | SAT)
- **MAP** — OpenStreetMap with dark tactical filter
- **SAT** — ESRI World Imagery satellite view — true terrain colors, no filter. Same imagery source as CPW hunting atlas.

### Crime Type Toggles
Three independent toggles — SEX OFFENDERS (blue), DRUG OFFENSES (amber), VIOLENT OFFENSES (red). Any combination can be active simultaneously. County choropleth updates to reflect active types.

### Filters
- **Region** — SLV Counties only, or all Colorado
- **Data Year** — 2020, 2021, 2022, or all years combined
- **Severity** — All offenses, felony only, or SVP/high-risk only

### Intel Brief
After each scan, a tactical summary generates automatically:
- Top zone score, hot zone count, SVP count, lead county, lead coordinates
- Signal-strength recommendation — CRITICAL / ELEVATED / MODERATE — with specific tactical language

### Hot Zone Ranking
- Top 20 scoring zones listed in sidebar
- Click any zone card to fly the map to that location
- Each card shows: score, record count, breakdown by crime type, SVP count

### ⬡ Field Intelligence AI
An embedded AI analyst with full context of the current session — active filters, zone scores, record counts, top-ranked zones with coordinates.

**Supported providers:**

| Provider | Model | Cost | Get Key |
|---|---|---|---|
| **Groq** | Llama 3.3 70B | **Free tier** | [console.groq.com](https://console.groq.com) |
| Claude | claude-haiku-4-5 | Paid | [console.anthropic.com](https://console.anthropic.com) |
| OpenAI | GPT-5.4 Mini | Paid | [platform.openai.com](https://platform.openai.com/api-keys) |
| Grok (xAI) | grok-4.3 | Paid | [console.x.ai](https://console.x.ai) |

> **Free path:** Create a Groq account (email only, no credit card), generate an API key, paste it in. Under two minutes.

API keys are stored in browser `sessionStorage` only — cleared when the tab closes. Never transmitted anywhere except directly to the provider you select.

The AI analyst knows:
- The four-factor scoring model and how to interpret each tier
- Colorado crime patterns and SLV regional context
- Law enforcement patrol and resource allocation strategy
- Sex offender registry interpretation and recidivism context
- Data limitations — always honest about what the data can and cannot show

**Quick-prompt buttons** cover the most common intelligence questions. Full multi-turn conversation supported.

### Collapsible UI
- **Sidebar** — collapses to icon rail via `‹/›` tab, map expands to fill
- **Header controls** — collapse via `⊟/⊞` button, AI button always visible
- **Mobile responsive** — sidebar starts collapsed on mobile, AI drawer goes full-width, 44px minimum touch targets

---

## Data Sources

| Source | Data | License |
|---|---|---|
| CBI Adult Sex Offender Registry | Registered offender addresses, offense type, SVP status | Public record — Colorado Revised Statute 16-22-111 |
| FBI NIBRS / Crime Data Explorer | County-level offense counts by type, 2022 | Public domain — U.S. Department of Justice |
| Plotly Open Dataset | Colorado county GeoJSON boundaries (FIPS-indexed) | MIT |
| OpenStreetMap | Street base map tiles | ODbL |
| ESRI World Imagery | Satellite base map tiles | Esri, Maxar, Earthstar Geographics |

### Getting Real CBI Data

The MVP uses sample data. To load real registry data:

1. Submit the Adult Registered Sex Offender Request Form to the Colorado Bureau of Investigation with the required fee
2. CBI returns a structured list with name, DOB, address, offense, conviction date, and SVP status
3. Geocode addresses to lat/lng using a geocoding API (Google Maps, Nominatim, or Census Bureau Geocoder — free)
4. Replace the `SAMPLE_OFFENDERS` array in the HTML with the real geocoded dataset
5. Remove the `⚠ SAMPLE DATA` warning labels

**No other code changes required.** The architecture is built for this swap.

---

## Running Locally

No installation required.

```bash
git clone https://github.com/jkh2/criminal-tracker.git
cd criminal-tracker
open index.html   # macOS
# or
start index.html  # Windows
# or drag the file into any modern browser
```

Internet connection required for map tiles, county GeoJSON, and AI provider API calls.

---

## Limitations & Honest Notes

**Current version uses sample data.** Records are fictional, geocoded to realistic SLV locations. They demonstrate the architecture, not actual crime patterns. See *Getting Real CBI Data* above.

**Registry data is addresses, not crime locations.** The CBI registry shows where offenders are registered to live — not where offenses occurred. These are different things and should be communicated clearly to any audience.

**County crime data is aggregate counts.** FBI NIBRS county data gives total offense counts per year — not GPS incident locations. The choropleth shows relative density, not specific crime scenes.

**Rural areas may be underrepresented.** Smaller agencies have lower NIBRS participation rates. Sparse data in a county may reflect reporting gaps, not absence of crime.

**Not a real-time system.** Data represents the 2022 FBI NIBRS reporting year. The county choropleth does not update automatically.

**Not legal advice.** Criminal Tracker is an intelligence visualization tool. It does not constitute legal advice, does not guarantee accuracy of public records data, and should not be used as the sole basis for any law enforcement action.

**Best experienced on desktop.** Criminal Tracker works on mobile but is data-dense — a larger screen gives you the full map, panel, and AI drawer without compromise. On phone, start with the panel collapsed and use landscape orientation for best results.

---

## Roadmap

**Planned:**
- [ ] Real CBI sex offender registry data (formal records request — architecture ready)
- [ ] Flock Safety ALPR camera integration — vehicle plate read heat map as a fourth data layer. A November 2025 Washington state court ruling established that Flock ALPR data held by government agencies constitutes a public record subject to public records requests. Colorado CORA applicability is unsettled pending state-level legal precedent. Integration paths: (1) CORA public records request to local agencies, (2) direct agency data export for law enforcement customers, (3) Flock Safety API partnership for approved vendors.
- [ ] Live FBI Crime Data Explorer API integration (BYOK api.data.gov key) for fresher state-level trend data
- [ ] Drug trafficking corridor overlay (US-160, US-285, I-25 pressure scoring)
- [ ] Parole and probation check-in compliance layer (law enforcement version)
- [ ] Time-of-day and day-of-week incident pattern analysis
- [ ] Multi-agency patrol resource allocation recommendations
- [ ] Incident upload — law enforcement can upload their own CAD/RMS export for street-level precision
- [ ] Colorado CORA public records request generator — auto-generate a properly formatted records request for any Colorado agency's Flock ALPR data
- [ ] Offline mode with cached dataset
- [ ] PWA / mobile-optimized layout

**AI Integration (BYOK — already shipped in v1.0):**
- [x] Multi-provider AI Field Intelligence (Groq free tier, Claude, OpenAI, Grok)
- [x] Live session context injection — AI sees current filters, zone scores, record counts
- [x] Tactical recommendation engine in Intel Brief panel
- [ ] AI-generated CORA records request drafting
- [ ] AI patrol route suggestions based on zone rankings
- [ ] Natural language filter control — "show me SVP offenders in Alamosa from the last two years"

---

## Legal & Ethical Notes

Criminal Tracker uses only publicly available data. No private records, no unauthorized data access, no PII beyond what is public record under Colorado law.

The heat map displays density patterns — no individual names, addresses, or identifying information is shown to end users. This is an intentional design decision that protects civil liberties while preserving analytical utility.

Law enforcement agencies considering deployment should consult with their legal counsel regarding applicable state and federal regulations governing the display and use of registry data in operational contexts.

---

## License

Copyright © 2026 James Keith Harwood II / Sentinel AI Systems
Contact: jameskharwood2@gmail.com
GitHub: github.com/jkh2

Licensed under the **Sentinel Source Available License v1.0** (see LICENSE).

**You are free to:**
- Use this software for personal, non-commercial, or internal law enforcement evaluation
- Study and modify the code for personal use
- Share the unmodified source with attribution

**You may not:**
- Sell, sublicense, or commercially distribute this software or any derivative
- Incorporate this software into a paid product or service without a commercial license
- Remove or alter copyright notices or license terms

For commercial licensing and law enforcement deployment inquiries: jameskharwood2@gmail.com

---

## About

Built by James Keith Harwood II and Claude Sentinel (Anthropic) under the SIDLF framework — a research initiative exploring symbiotic human-AI partnership.

Criminal Tracker is part of the Sentinel AI Systems field intelligence portfolio, alongside ELK SCOUT (wildlife intelligence) and other Colorado-focused tools.

*"The first step in solving a problem is knowing where it is."*
