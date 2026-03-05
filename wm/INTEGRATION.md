# World Monitor — Data Source Integration Guide

This document catalogs every external data source integrated into World Monitor, how each integration works, the authentication required, and the data normalization performed.

---

## Table of Contents

1. [Integration Architecture Overview](#1-integration-architecture-overview)
2. [Data Source Catalog](#2-data-source-catalog)
3. [RSS / News Feeds](#3-rss--news-feeds)
4. [Geopolitical & Conflict Sources](#4-geopolitical--conflict-sources)
5. [Military & Defense Sources](#5-military--defense-sources)
6. [Maritime & Aviation Sources](#6-maritime--aviation-sources)
7. [Cyber Threat Intelligence Sources](#7-cyber-threat-intelligence-sources)
8. [Natural Events & Climate Sources](#8-natural-events--climate-sources)
9. [Infrastructure & Outage Sources](#9-infrastructure--outage-sources)
10. [Market & Financial Sources](#10-market--financial-sources)
11. [Economic & Trade Sources](#11-economic--trade-sources)
12. [Intelligence & OSINT Sources](#12-intelligence--osint-sources)
13. [AI / LLM Providers](#13-ai--llm-providers)
14. [Proxy & Relay Architecture](#14-proxy--relay-architecture)
15. [Data Normalization Patterns](#15-data-normalization-patterns)
16. [Caching Strategy Per Source](#16-caching-strategy-per-source)
17. [Adding a New Data Source](#17-adding-a-new-data-source)

---

## 1. Integration Architecture Overview

Data sources connect to World Monitor through three ingestion paths:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         DATA SOURCE TYPES                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  REST APIs          RSS/Atom Feeds       WebSocket Streams              │
│  (USGS, ACLED,      (270+ domains,       (AISStream → vessels)         │
│   Cloudflare,       tiered by             Stateful, via relay)          │
│   Yahoo, ...)       authority)                                          │
│                                                                         │
│  Public Datasets    Status Pages          Scrapers                      │
│  (Feodo Tracker,    (33 services:         (FwdStart newsletter,        │
│   C2IntelFeeds,     AWS, Azure,           GitHub trending fallback)     │
│   GDACS, ...)       GitHub, ...)                                        │
│                                                                         │
└────────────┬───────────────┬──────────────────┬────────────────────────┘
             │               │                  │
             ▼               ▼                  ▼
┌───────────────────┐ ┌─────────────────┐ ┌──────────────────┐
│ Vercel Edge       │ │ Railway Relay   │ │ Direct Client    │
│ Functions         │ │ Server          │ │ Fetch            │
│                   │ │                 │ │                  │
│ Sebuf RPC handlers│ │ AIS, OpenSky,  │ │ RSS feeds,       │
│ + standalone      │ │ Telegram, OREF,│ │ YouTube embeds   │
│ endpoints         │ │ Polymarket,    │ │                  │
│                   │ │ market seeding │ │                  │
└────────┬──────────┘ └───────┬─────────┘ └────────┬─────────┘
         │                    │                     │
         ▼                    ▼                     ▼
┌─────────────────────────────────────────────────────────────┐
│                    Proto Normalization Layer                  │
│                                                              │
│  Raw API responses → Protobuf message types → JSON           │
│  Source-specific parsers → Unified domain models             │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    Client State (AppContext)                  │
│                                                              │
│  allNews[], latestMarkets[], intelligenceCache{},           │
│  cyberThreatsCache[], latestPredictions[], ...              │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Data Source Catalog

### At a Glance (40+ Sources)

| Category | Source | Auth | Ingestion Path | Proto Domain |
|----------|--------|------|---------------|--------------|
| News | 270+ RSS feeds | None | Client → rss-proxy → feed | news |
| Earthquakes | USGS | None | Edge → usgs.gov | seismology |
| Wildfires | NASA FIRMS | API key | Edge → firms.modaps | wildfire |
| Climate | Open-Meteo / ERA5 | None | Edge → open-meteo.com | climate |
| Natural disasters | GDACS | None | Edge → gdacs.org | natural |
| Conflict | ACLED | Token | Edge → acleddata.com | conflict |
| Conflict | UCDP | Token | Edge → ucdp.uu.se | conflict |
| Unrest | GDELT | None | Edge → api.gdeltproject.org | unrest |
| Military flights | OpenSky | OAuth2 | Relay → opensky-network.org | military |
| Military enrichment | WingBits | API key | Edge → wingbits.com | military |
| Fleet intel | USNI | None | Edge → news.usni.org | military |
| Vessels | AISStream | API key | Relay WS → aisstream.io | maritime |
| Nav warnings | Various | None | Edge → msi.nga.mil | maritime |
| Airport ops | AviationStack | API key | Edge → aviationstack.com | aviation |
| NOTAMs | ICAO | API key | Relay → icao.int | aviation |
| Cyber threats | Feodo Tracker | None | Edge → feodotracker.abuse.ch | cyber |
| Cyber threats | URLhaus | Auth key | Edge → urlhaus-api.abuse.ch | cyber |
| Cyber threats | C2IntelFeeds | None | Edge → github.com/drb-ra | cyber |
| Cyber threats | AlienVault OTX | API key | Edge → otx.alienvault.com | cyber |
| Cyber threats | AbuseIPDB | API key | Edge → abuseipdb.com | cyber |
| Internet outages | Cloudflare Radar | Token | Edge → api.cloudflare.com | infrastructure |
| Service status | 33 status pages | None | Edge → various | infrastructure |
| Stock markets | Yahoo Finance | None | Proxy → finance.yahoo.com | market |
| Stock markets | Finnhub | API key | Edge → finnhub.io | market |
| Crypto | CoinGecko | None | Edge → api.coingecko.com | market |
| Predictions | Polymarket | None | Relay → polymarket.com | prediction |
| Energy | EIA | API key | Edge → api.eia.gov | economic |
| Economic | FRED | API key | Edge → api.stlouisfed.org | economic |
| Economic | World Bank | None | Edge → api.worldbank.org | economic |
| Economic | BIS | None | Edge → bis.org | economic |
| Trade | WTO | API key | Edge → api.wto.org | trade |
| Displacement | UNHCR | None | Edge → api.unhcr.org | displacement |
| Population | WorldPop | None | Edge → worldpop.org | displacement |
| GPS interference | GPSJam | None | Edge → gpsjam.org | — |
| Telegram OSINT | Telegram MTProto | Session | Relay → telegram.org | — |
| OREF sirens | Israel Home Front | Proxy auth | Relay → oref.org.il | — |
| GeoIP | ipinfo.io | None | Edge → ipinfo.io | cyber |
| PizzINT | Pentagon index | None | Proxy → pizzint.com | intelligence |
| Enrichment | HN/GitHub signals | None | Edge → algolia/github | enrichment |

---

## 3. RSS / News Feeds

### Overview

RSS is the primary news ingestion mechanism, covering 270+ allowed domains across categories.

### Feed Configuration

Feeds are defined in `src/config/feeds.ts` with a tiered authority system:

| Tier | Authority Level | Examples |
|------|----------------|---------|
| **1** | Wire services, government | Reuters, AP, AFP, Bloomberg, White House, Pentagon, UN |
| **2** | Major outlets | BBC, CNN, Guardian, Al Jazeera, CNBC, WSJ |
| **3** | Specialty sources | Defense One, Bellingcat, Oryx OSINT, The War Zone |
| **4** | Aggregators & blogs | Tech blogs, community sources |

### Feed Categories

| Category | Content | Variant |
|----------|---------|---------|
| politics | Global politics, government | full |
| us | US domestic news | full |
| europe | European news | full |
| middleEast | Middle East & North Africa | full |
| tech | Technology, AI, cyber | full, tech |
| ai | AI/ML research & industry | tech |
| finance | Markets, economy | full, finance |
| gov | Government press, policy | full |
| crisis | Crisis, disaster, conflict | full |
| africa | African news | full |
| latam | Latin America | full |
| asia | Asia-Pacific | full |
| energy | Oil, gas, renewables | full, finance |
| positive | Good news, breakthroughs | happy |

### Proxy Architecture

```
Client ──fetch──▶ /api/rss-proxy?url=<encoded-feed-url>
                        │
                        ├── Domain in allowed list? (270+ domains in _rss-allowed-domains.js)
                        │     │
                        │     ├── Yes → Fetch directly with Cache-Control
                        │     │
                        │     └── No → Relay fallback (if WS_RELAY_URL configured)
                        │              └── Railway relay /rss endpoint
                        │
                        └── Parse RSS 2.0 / Atom via DOMParser
                              └── Normalize to NewsItem type
```

### RSS Parsing (`src/services/rss.ts`)

The RSS parser handles:
- **RSS 2.0** and **Atom** feed formats
- `media:content`, `media:thumbnail`, `content:encoded` extensions
- Enclosure extraction (audio/video attachments)
- Date normalization across timezone formats
- Feed cooldown tracking (avoid re-fetching too soon)
- Failure counting with backoff
- **Keyword threat classification**: Headlines scanned for crisis/threat keywords → auto-tagging
- **Geo inference**: Country/region extracted from content for map placement
- **Trending keywords**: `ingestHeadlines()` extracts trending terms across all feeds

### Client-Side Feed Cache

- `persistent-cache.ts` stores parsed feeds in IndexedDB
- TTL: ~30 minutes per feed
- Circuit breaker wraps each feed source

---

## 4. Geopolitical & Conflict Sources

### 4.1 ACLED (Armed Conflict Location & Event Data)

| Property | Value |
|----------|-------|
| **API** | `https://api.acleddata.com/acled/read` |
| **Auth** | `ACLED_ACCESS_TOKEN` (header) |
| **Data** | Armed conflict events: battles, protests, riots, violence, strategic developments |
| **Coverage** | Global, ~300,000 events/year |
| **Handler** | `server/worldmonitor/conflict/v1/list-acled-events.ts` |
| **Proto** | `conflict.v1.ListAcledEventsResponse` |
| **Cache tier** | slow (900s) |
| **Normalization** | ACLED JSON → `AcledEvent` proto (location, date, type, fatalities, actors, notes) |

### 4.2 UCDP (Uppsala Conflict Data Program)

| Property | Value |
|----------|-------|
| **API** | `https://ucdpapi.pcr.uu.se/api/gedevents/` |
| **Auth** | `UCDP_ACCESS_TOKEN` |
| **Data** | Georeferenced armed conflict events (state-based, non-state, one-sided violence) |
| **Handler** | `server/worldmonitor/conflict/v1/list-ucdp-events.ts` |
| **Proto** | `conflict.v1.ListUcdpEventsResponse` |
| **Cache tier** | static (3600s) |

### 4.3 GDELT (Global Database of Events, Language, and Tone)

| Property | Value |
|----------|-------|
| **API** | `https://api.gdeltproject.org/api/v2/geo/geo` |
| **Auth** | None (public) |
| **Data** | Social unrest, protests, demonstrations — geocoded events |
| **Handler** | `server/worldmonitor/unrest/v1/list-unrest-events.ts` |
| **Proto** | `unrest.v1.ListUnrestEventsResponse` |
| **Cache tier** | slow (900s) |
| **Normalization** | GDELT GeoJSON → `SocialUnrestEvent` with country, lat/lon, type mapping |

### 4.4 OREF (Israel Home Front Command)

| Property | Value |
|----------|-------|
| **Source** | `oref.org.il` real-time siren API |
| **Auth** | `OREF_PROXY_AUTH` (proxy credentials for Israel IP requirement) |
| **Ingestion** | Railway relay polling (~10s) → `/oref/alerts` and `/oref/history` |
| **Handler** | `api/oref-alerts.js` (proxies to relay) |
| **Data** | Active rocket/siren alerts with city-level granularity |

---

## 5. Military & Defense Sources

### 5.1 OpenSky Network (Military Flights)

| Property | Value |
|----------|-------|
| **API** | `https://opensky-network.org/api/states/all` |
| **Auth** | `OPENSKY_CLIENT_ID` / `OPENSKY_CLIENT_SECRET` (OAuth2, higher rate limits) |
| **Ingestion** | Railway relay → HTTP proxy → Vercel edge |
| **Handler** | `api/opensky.js` → `server/worldmonitor/military/v1/list-military-flights.ts` |
| **Filtering** | Known military aircraft ICAO24 identifiers, hex ranges, callsign patterns |
| **Proto** | `military.v1.MilitaryFlight` |
| **Cache tier** | no-store (real-time) |

### 5.2 WingBits (Aircraft Enrichment)

| Property | Value |
|----------|-------|
| **API** | WingBits REST API |
| **Auth** | `WINGBITS_API_KEY` |
| **Data** | Aircraft details: owner, operator, type, registration |
| **Handler** | `server/worldmonitor/military/v1/get-aircraft-details.ts`, `get-aircraft-details-batch.ts` |
| **Proto** | `military.v1.GetAircraftDetailsResponse` |

### 5.3 USNI Fleet Tracker

| Property | Value |
|----------|-------|
| **Source** | USNI News fleet tracker page |
| **Auth** | None |
| **Data** | US Navy fleet deployment positions and status |
| **Handler** | `server/worldmonitor/military/v1/get-usni-fleet-report.ts` |
| **Proto** | `military.v1.USNIFleetReport` |
| **Cache tier** | static (3600s) |

---

## 6. Maritime & Aviation Sources

### 6.1 AISStream (Vessel Tracking)

| Property | Value |
|----------|-------|
| **API** | `wss://stream.aisstream.io/v0/stream` |
| **Auth** | `AISSTREAM_API_KEY` |
| **Protocol** | WebSocket (persistent connection) |
| **Ingestion** | Railway relay maintains WebSocket → aggregates vessel state → HTTP snapshot endpoint |
| **Handler** | `api/ais-snapshot.js` (proxies relay `/ais/snapshot`) |
| **Proto** | `maritime.v1.VesselSnapshot` |
| **Cache tier** | no-store |
| **Data** | Live vessel positions, heading, speed, MMSI, vessel type |

### 6.2 AviationStack (Airport Intelligence)

| Property | Value |
|----------|-------|
| **API** | `http://api.aviationstack.com/v1/flights` |
| **Auth** | `AVIATIONSTACK_API` (query param) |
| **Data** | Live flights, airport delays, carrier operations |
| **Handler** | Multiple RPCs under `server/worldmonitor/aviation/v1/` |
| **Proto** | `aviation.v1.*` |
| **Cache tier** | static (delays) / fast (flight status) |

### 6.3 ICAO (NOTAMs)

| Property | Value |
|----------|-------|
| **API** | ICAO NOTAM API |
| **Auth** | `ICAO_API_KEY` |
| **Data** | Notice to Airmen — airport closures, airspace restrictions |
| **Ingestion** | Railway relay `/notam` endpoint |

### 6.4 GPSJam (GPS Interference)

| Property | Value |
|----------|-------|
| **Source** | `gpsjam.org` manifest + H3 hex CSV data |
| **Auth** | None |
| **Data** | GPS interference/jamming zones (H3 hexagon grid) |
| **Handler** | `api/gpsjam.js` — fetches manifest, downloads CSV, caches in Redis |
| **Cache** | Redis, 1-hour TTL |
| **Visualization** | H3 hexagons on Deck.gl map layer |

---

## 7. Cyber Threat Intelligence Sources

All cyber sources are aggregated in `server/worldmonitor/cyber/v1/_shared.ts` and served through the `list-cyber-threats` RPC.

### 7.1 Feodo Tracker (abuse.ch)

| Property | Value |
|----------|-------|
| **URL** | `https://feodotracker.abuse.ch/downloads/ipblocklist_aggressive.json` |
| **Auth** | None (public) |
| **Data** | C2 botnet server IPs (Dridex, Emotet, TrickBot, QakBot) |
| **Format** | JSON array of IP objects |
| **Normalization** | → `CyberThreat` proto with GeoIP enrichment |

### 7.2 URLhaus (abuse.ch)

| Property | Value |
|----------|-------|
| **URL** | `https://urlhaus-api.abuse.ch/v1/urls/recent/` |
| **Auth** | `URLHAUS_AUTH_KEY` |
| **Data** | Malicious URLs (malware distribution, phishing) |
| **Format** | JSON API response |

### 7.3 C2IntelFeeds

| Property | Value |
|----------|-------|
| **URL** | `https://raw.githubusercontent.com/drb-ra/C2IntelFeeds/master/feeds/IPC2s.csv` |
| **Auth** | None (public GitHub) |
| **Data** | Command & Control server IPs from multiple threat feeds |
| **Format** | CSV |

### 7.4 AlienVault OTX

| Property | Value |
|----------|-------|
| **URL** | `https://otx.alienvault.com/api/v1/indicators/` |
| **Auth** | `OTX_API_KEY` |
| **Data** | Threat indicators (IPs, domains, hashes, URLs) |
| **Format** | JSON API |

### 7.5 AbuseIPDB

| Property | Value |
|----------|-------|
| **URL** | `https://api.abuseipdb.com/api/v2/blacklist` |
| **Auth** | `ABUSEIPDB_API_KEY` |
| **Data** | IP reputation blacklist |
| **Format** | JSON API |

### GeoIP Enrichment

All cyber threat IPs are enriched with geolocation using `ipinfo.io` and `freeipapi.com`:
- Results cached in-memory (24h TTL)
- Enables plotting threats on the map by country/city

---

## 8. Natural Events & Climate Sources

### 8.1 USGS Earthquakes

| Property | Value |
|----------|-------|
| **URL** | `https://earthquake.usgs.gov/earthquakes/feed/v1.0/summary/` |
| **Auth** | None (public GeoJSON) |
| **Feeds** | `significant_week.geojson`, `4.5_day.geojson`, `2.5_day.geojson`, `1.0_day.geojson` |
| **Handler** | `server/worldmonitor/seismology/v1/list-earthquakes.ts` |
| **Proto** | `seismology.v1.Earthquake` |
| **Cache tier** | slow (900s) |

### 8.2 NASA FIRMS (Wildfires)

| Property | Value |
|----------|-------|
| **URL** | `https://firms.modaps.eosdis.nasa.gov/api/area/` |
| **Auth** | `NASA_FIRMS_API_KEY` |
| **Data** | Active fire detections from MODIS/VIIRS satellites |
| **Handler** | `server/worldmonitor/wildfire/v1/list-fire-detections.ts` |
| **Proto** | `wildfire.v1.FireDetection` |
| **Cache tier** | static (3600s) |

### 8.3 GDACS (Natural Disasters)

| Property | Value |
|----------|-------|
| **URL** | GDACS RSS/GeoJSON feeds |
| **Auth** | None (public) |
| **Data** | Earthquakes, tsunamis, floods, tropical cyclones, volcanoes |
| **Handler** | `server/worldmonitor/natural/v1/list-natural-events.ts` |
| **Proto** | `natural.v1.NaturalEvent` |
| **Cache tier** | slow (900s) |

### 8.4 Open-Meteo / ERA5 (Climate Anomalies)

| Property | Value |
|----------|-------|
| **URL** | Open-Meteo API (processes Copernicus ERA5 reanalysis) |
| **Auth** | None (public) |
| **Data** | Temperature anomalies, precipitation extremes |
| **Handler** | `server/worldmonitor/climate/v1/list-climate-anomalies.ts` |
| **Proto** | `climate.v1.ClimateAnomaly` |
| **Cache tier** | static (3600s) |

---

## 9. Infrastructure & Outage Sources

### 9.1 Cloudflare Radar (Internet Outages)

| Property | Value |
|----------|-------|
| **URL** | `https://api.cloudflare.com/client/v4/radar/annotations/outages` |
| **Auth** | `CLOUDFLARE_API_TOKEN` (Bearer) |
| **Data** | Internet outages and disruptions by country |
| **Handler** | `server/worldmonitor/infrastructure/v1/list-internet-outages.ts` |
| **Normalization** | Cloudflare annotations → `InternetOutage` with country centroid coordinates |
| **Cache** | Redis (30 min TTL) + CDN (slow tier) |

### 9.2 Service Status Pages (33 Services)

The infrastructure service monitors 33 major tech services by scraping their public status pages:

| Services Monitored |
|-------------------|
| AWS, Azure, GCP, Cloudflare, Fastly, Akamai, Vercel, Netlify, GitHub, GitLab, Bitbucket, Docker Hub, npm, PyPI, Slack, Discord, Zoom, Teams, Twilio, Stripe, PayPal, Shopify, Datadog, PagerDuty, Okta, Auth0, MongoDB Atlas, Supabase, Firebase, Heroku, DigitalOcean, Fly.io, Railway |

**Handler**: `server/worldmonitor/infrastructure/v1/list-service-statuses.ts`
**Proto**: `infrastructure.v1.ServiceStatus`
**Cache tier**: slow (900s)

### 9.3 Temporal Baseline

Tracks historical infrastructure patterns to detect anomalies:
- `get-temporal-baseline`: Returns baseline metrics for a time period
- `record-baseline-snapshot`: Records current state for future comparison

---

## 10. Market & Financial Sources

### 10.1 Yahoo Finance

| Property | Value |
|----------|-------|
| **URL** | `https://query1.finance.yahoo.com/v8/finance/chart/` |
| **Auth** | None |
| **Data** | Stock quotes, indices, commodities, forex |
| **Ingestion** | Vite dev proxy + Railway relay seeding to Redis |
| **Handler** | `server/worldmonitor/market/v1/list-market-quotes.ts` |
| **Proto** | `market.v1.MarketQuote` |
| **Cache tier** | medium (300s) |

### 10.2 Finnhub

| Property | Value |
|----------|-------|
| **URL** | `https://finnhub.io/api/v1/` |
| **Auth** | `FINNHUB_API_KEY` |
| **Data** | Real-time stock quotes, company profiles |

### 10.3 CoinGecko (Crypto)

| Property | Value |
|----------|-------|
| **URL** | CoinGecko public API |
| **Auth** | None (rate-limited) |
| **Data** | Cryptocurrency prices, market caps, volumes |
| **Handler** | `server/worldmonitor/market/v1/list-crypto-quotes.ts` |
| **Proto** | `market.v1.CryptoQuote` |

### 10.4 Polymarket (Prediction Markets)

| Property | Value |
|----------|-------|
| **Source** | Polymarket GAMMA API |
| **Auth** | None |
| **Ingestion** | Railway relay proxy → Vercel edge |
| **Handler** | `api/polymarket.js` |
| **Proto** | `prediction.v1.PredictionMarket` |

---

## 11. Economic & Trade Sources

### 11.1 FRED (Federal Reserve Economic Data)

| Property | Value |
|----------|-------|
| **URL** | `https://api.stlouisfed.org/fred/series/observations` |
| **Auth** | `FRED_API_KEY` |
| **Data** | Economic series: GDP, unemployment, interest rates, CPI, etc. |
| **Handler** | `server/worldmonitor/economic/v1/get-fred-series.ts` |
| **Proto** | `economic.v1.GetFredSeriesResponse` |
| **Cache tier** | static (3600s) |

### 11.2 EIA (Energy Information Administration)

| Property | Value |
|----------|-------|
| **URL** | `https://api.eia.gov/v2/` |
| **Auth** | `EIA_API_KEY` |
| **Data** | Petroleum prices (WTI, Brent), production, inventories, consumption |
| **Handler** | `api/eia/[[...path]].js` (catch-all proxy) |
| **Normalization** | Series IDs → `{ current, previous, date, unit }` |

### 11.3 World Bank

| Property | Value |
|----------|-------|
| **URL** | `https://api.worldbank.org/v2/` |
| **Auth** | None (public) |
| **Data** | Development indicators (GDP per capita, population, inflation, etc.) |
| **Handler** | `server/worldmonitor/economic/v1/list-world-bank-indicators.ts` |

### 11.4 BIS (Bank for International Settlements)

| Property | Value |
|----------|-------|
| **URL** | BIS statistical data API |
| **Auth** | None (public) |
| **Data** | Central bank policy rates, exchange rates, credit statistics |
| **Handlers** | `get-bis-policy-rates.ts`, `get-bis-exchange-rates.ts`, `get-bis-credit.ts` |
| **Cache tier** | daily (86400s) |

### 11.5 WTO (World Trade Organization)

| Property | Value |
|----------|-------|
| **Auth** | `WTO_API_KEY` |
| **Data** | Trade restrictions, tariff data, trade flows |
| **Handler** | `server/worldmonitor/trade/v1/` |

### 11.6 UNHCR (Displacement)

| Property | Value |
|----------|-------|
| **URL** | UNHCR public API |
| **Auth** | None (public, CC BY 4.0) |
| **Data** | Refugee displacement flows, population of concern |
| **Handler** | `server/worldmonitor/displacement/v1/` |

---

## 12. Intelligence & OSINT Sources

### 12.1 GDELT Intelligence Search

| Property | Value |
|----------|-------|
| **URL** | GDELT Doc API v2 |
| **Auth** | None |
| **Data** | Full-text article search, geo-tagged events |
| **Handler** | `server/worldmonitor/intelligence/v1/search-gdelt-documents.ts` |

### 12.2 PizzINT (Pentagon Pizza Index)

| Property | Value |
|----------|-------|
| **Source** | pizzint.com |
| **Auth** | None |
| **Data** | Activity indicator based on pizza delivery patterns near government buildings |
| **Handler** | Proxied via Vite dev server |
| **Proto** | `intelligence.v1.GetPizzintStatusResponse` |

### 12.3 Telegram OSINT Channels

| Property | Value |
|----------|-------|
| **Protocol** | MTProto (Telegram native) |
| **Auth** | `TELEGRAM_API_ID`, `TELEGRAM_API_HASH`, `TELEGRAM_SESSION` (StringSession) |
| **Channels** | Curated list in `data/telegram-channels.json` (buckets: full, tech, finance) |
| **Ingestion** | Railway relay polls channels → `/telegram/feed` endpoint |
| **Handler** | `api/telegram-feed.js` |

### 12.4 Bellingcat & Oryx OSINT

- Integrated via RSS feeds (Tier 3 specialty sources)
- Auto-tagged as OSINT/INTEL sources

### 12.5 Country Intelligence Briefs

`intelligence.v1.GetCountryIntelBrief` aggregates signals per country:
- Critical news count
- Active protests and conflicts
- Military activity (flights, vessels)
- Internet outages and cyber threats
- Natural disasters and climate stress
- GPS jamming and AIS disruptions
- Travel advisories

---

## 13. AI / LLM Providers

### Summarization Fallback Chain

```
1. Ollama (local)     — OLLAMA_API_URL, OLLAMA_MODEL
    │ ↓ fail
2. Groq (cloud)       — GROQ_API_KEY (14,400 req/day free)
    │ ↓ fail
3. OpenRouter (cloud)  — OPENROUTER_API_KEY (50 req/day free)
    │ ↓ fail
4. Browser T5 (ONNX)  — Transformers.js, fully client-side
```

Each provider is called via the `SummarizeArticle` RPC, which handles:
- Provider selection based on availability (`RuntimeFeatureId`)
- Circuit breaker wrapping
- Redis cache check (cross-user deduplication)
- Analytics tracking (`trackLLMUsage`, `trackLLMFailure`)
- Language-aware summarization (21 locales)

---

## 14. Proxy & Relay Architecture

### 14.1 RSS Proxy (`api/rss-proxy.js`)

```
Client request → /api/rss-proxy?url=<encoded>
  │
  ├── Validate URL against allowed domains (270+)
  │     ├── Allowed → fetch directly from feed URL
  │     └── Not allowed → forward to Railway relay /rss
  │
  ├── Parse XML → RSS 2.0 / Atom
  └── Return JSON with Cache-Control headers
```

### 14.2 Relay Endpoints (Railway)

| Endpoint | Purpose | Upstream |
|----------|---------|----------|
| `/rss` | RSS proxy for domains blocked on Vercel | Various |
| `/telegram/feed` | Telegram OSINT channel polling | Telegram MTProto |
| `/oref/alerts` | OREF siren alerts (Israel IP required) | oref.org.il |
| `/oref/history` | 24h siren history | oref.org.il |
| `/polymarket` | Polymarket prediction data | Polymarket GAMMA API |
| `/opensky` | OpenSky flight states | opensky-network.org |
| `/ais/snapshot` | AIS vessel positions | AISStream WebSocket |
| `/notam` | ICAO NOTAMs | ICAO API |
| `/metrics` | Relay health metrics | Internal |

### 14.3 Direct Proxy URLs

For development, `vite.config.ts` configures local proxies:
- `/api/yahoo` → Yahoo Finance
- `/api/earthquake` → USGS
- `/api/fred` → FRED API
- `/api/polymarket` → Polymarket GAMMA
- `/api/pizzint` → PizzINT
- `/api/cloudflare` → Cloudflare Radar

---

## 15. Data Normalization Patterns

### 15.1 Proto-First Design

Every external data source is normalized to a Protobuf message type before reaching the client. This ensures:
- Type safety across the entire pipeline
- Consistent field naming (snake_case in proto, camelCase in TS)
- Backward-compatible schema evolution via proto field numbering
- Auto-generated OpenAPI documentation

### 15.2 Source-Specific Parsers

Each handler in `server/worldmonitor/<domain>/v1/` contains source-specific parsing logic:

```typescript
// Example: ACLED → Proto normalization
function toProtoAcledEvent(raw: AcledRawEvent): AcledEvent {
  return {
    eventId: raw.data_id,
    eventDate: raw.event_date,
    eventType: mapEventType(raw.event_type),
    subEventType: raw.sub_event_type,
    country: raw.country,
    latitude: parseFloat(raw.latitude),
    longitude: parseFloat(raw.longitude),
    fatalities: parseInt(raw.fatalities),
    actor1: raw.actor1,
    actor2: raw.actor2,
    notes: raw.notes,
    source: raw.source,
  };
}
```

### 15.3 Common Normalization Patterns

- **GeoJSON → Proto**: USGS earthquakes, GDACS events
- **CSV → Proto**: C2IntelFeeds, GPSJam H3 hexes
- **XML/RSS → NewsItem**: 270+ feed domains
- **JSON API → Proto**: ACLED, Cloudflare, Yahoo Finance, CoinGecko, BIS
- **HTML scraping → Proto**: FwdStart newsletter, GitHub trending (fallback)

---

## 16. Caching Strategy Per Source

| Source | Client Cache | CDN Cache | Redis Cache |
|--------|-------------|-----------|-------------|
| RSS feeds | IndexedDB, 30 min | rss-proxy headers | — |
| USGS earthquakes | AppContext | slow (900s) | Bootstrap |
| NASA FIRMS | AppContext | static (3600s) | — |
| ACLED/UCDP | AppContext | slow/static | Bootstrap |
| OpenSky flights | AppContext | no-store | — |
| AIS vessels | AppContext | no-store | Relay state |
| Market quotes | AppContext | medium (300s) | Bootstrap |
| Crypto | AppContext | medium (300s) | — |
| Cloudflare outages | AppContext | slow (900s) | 30 min |
| GPS jam | AppContext | — | 1 hour |
| AI summaries | — | — | Cross-user dedup |
| FRED/BIS economic | AppContext | static/daily | — |
| Cyber threats | AppContext | slow (900s) | — |

---

## 17. Adding a New Data Source

### Step-by-Step Guide

1. **Define Proto messages** in `proto/worldmonitor/<domain>/v1/`:
   ```protobuf
   message MyNewDataItem {
     string id = 1;
     double latitude = 2;
     double longitude = 3;
     // ... domain-specific fields
   }

   message ListMyNewDataRequest { /* filters */ }
   message ListMyNewDataResponse { repeated MyNewDataItem items = 1; }
   ```

2. **Add RPC to service** in `proto/worldmonitor/<domain>/v1/service.proto`:
   ```protobuf
   rpc ListMyNewData(ListMyNewDataRequest) returns (ListMyNewDataResponse) {}
   ```

3. **Generate code**: `make generate` (runs buf)

4. **Implement handler** in `server/worldmonitor/<domain>/v1/list-my-new-data.ts`:
   - Fetch from external API
   - Parse and normalize to proto types
   - Return proto-compatible JSON

5. **Register route** in `api/<domain>/v1/[rpc].ts`

6. **Set cache tier** in `server/gateway.ts` → `RPC_CACHE_TIER`

7. **Add env var** (if auth needed) to `.env.example` with registration link

8. **Create client service** in `src/services/<domain>/`:
   - Use generated client from `src/generated/client/`
   - Wrap with circuit breaker

9. **Wire into DataLoaderManager** (`src/app/data-loader.ts`)

10. **Create UI panel** or add to existing panel in `src/components/`

11. **Add map layer** (if geospatial) in `MapContainer` / `DeckGLMap`

12. **Test**: Add data validation test in `tests/` and E2E test in `e2e/`
