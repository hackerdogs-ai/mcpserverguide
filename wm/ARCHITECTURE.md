# World Monitor — Technical Architecture

> **Version:** 2.5.24 · **License:** AGPL-3.0-only

World Monitor is a real-time global intelligence dashboard that aggregates 45+ data layers — conflicts, military activity, markets, cyber threats, natural disasters, infrastructure status, and more — into a single map-centric interface with AI-powered summarization.

---

## Table of Contents

1. [High-Level Overview](#1-high-level-overview)
2. [Platform Variants](#2-platform-variants)
3. [Repository Layout](#3-repository-layout)
4. [Tech Stack Summary](#4-tech-stack-summary)
5. [Frontend Architecture](#5-frontend-architecture)
6. [Backend Architecture](#6-backend-architecture)
7. [API Layer — Sebuf RPC](#7-api-layer--sebuf-rpc)
8. [Railway Relay Server](#8-railway-relay-server)
9. [Data Flow Pipeline](#9-data-flow-pipeline)
10. [AI / ML Pipeline](#10-ai--ml-pipeline)
11. [Caching Architecture](#11-caching-architecture)
12. [Authentication & Security](#12-authentication--security)
13. [Deployment Topology](#13-deployment-topology)
14. [Desktop Application (Tauri)](#14-desktop-application-tauri)
15. [Proto / Code Generation](#15-proto--code-generation)
16. [Testing Strategy](#16-testing-strategy)
17. [Internationalization](#17-internationalization)
18. [Configuration & Environment](#18-configuration--environment)

---

## 1. High-Level Overview

```
┌──────────────────────────────────────────────────────────────────────────┐
│                              CLIENTS                                     │
│                                                                          │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                  │
│   │  Web (SPA)   │  │ Desktop App  │  │     PWA      │                  │
│   │  Vite+Preact │  │   (Tauri 2)  │  │  (Offline)   │                  │
│   └──────┬───────┘  └──────┬───────┘  └──────┬───────┘                  │
│          │                 │                 │                            │
└──────────┼─────────────────┼─────────────────┼───────────────────────────┘
           │                 │                 │
           ▼                 ▼                 ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                         VERCEL EDGE FUNCTIONS                            │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐     │
│  │              Sebuf Gateway (server/gateway.ts)                  │     │
│  │  CORS → API Key → Rate Limit → Route → Handler → Cache Headers │     │
│  └─────────────────────────────────────────────────────────────────┘     │
│                                                                          │
│  ┌─────────────┐ ┌──────────┐ ┌───────────┐ ┌───────────────────┐      │
│  │ 19 Domain   │ │Bootstrap │ │ RSS Proxy │ │ Standalone Edges  │      │
│  │ RPC Svcs    │ │ (batch)  │ │ (270+     │ │ (geo, telegram,   │      │
│  │ (100+ RPCs) │ │          │ │  domains) │ │  OREF, gpsjam...) │      │
│  └──────┬──────┘ └────┬─────┘ └─────┬─────┘ └────────┬──────────┘      │
│         │              │             │                 │                  │
└─────────┼──────────────┼─────────────┼─────────────────┼─────────────────┘
          │              │             │                 │
          ▼              ▼             ▼                 ▼
┌──────────────────┐ ┌──────────────────┐ ┌───────────────────────────────┐
│  Upstash Redis   │ │  Railway Relay   │ │    External Data Sources      │
│  (cache, rate    │ │  (AIS, OpenSky,  │ │    (40+ APIs, feeds, WS)     │
│   limiting)      │ │   Telegram,      │ │                               │
│                  │ │   OREF, RSS,     │ │    USGS · NASA · ACLED ·     │
│                  │ │   Polymarket,    │ │    Cloudflare · Yahoo ·      │
│                  │ │   seeding)       │ │    GDELT · AISStream · ...   │
│                  │ │                  │ │                               │
└──────────────────┘ └──────────────────┘ └───────────────────────────────┘
                                           │
                     ┌─────────────────────┘
                     ▼
               ┌──────────────┐
               │  Convex DB   │
               │ (registrations│
               │   only)       │
               └──────────────┘
```

---

## 2. Platform Variants

World Monitor ships four UI variants from a single codebase, controlled by `VITE_VARIANT`:

| Variant | Domain | Focus |
|---------|--------|-------|
| **full** | worldmonitor.app | All 45+ layers — geopolitics, military, markets, cyber, climate, infrastructure |
| **tech** | tech.worldmonitor.app | Technology sector — AI research, GitHub trending, HN, tech events, service status |
| **finance** | — | Financial markets — Gulf economies, ETF flows, trade policy, commodities, stablecoins |
| **happy** | — | Positive global news — species recovery, renewable energy, breakthroughs, giving |

Each variant selects its own panel set, feed categories, map layers, and theme via `src/config/variant.ts` and conditional rendering in `PanelLayoutManager`.

---

## 3. Repository Layout

```
worldmonitor/
├── api/                    # Vercel serverless/edge functions
│   ├── <domain>/v1/        # Sebuf RPC handlers (19 domains)
│   ├── bootstrap.js        # Batch data hydration
│   ├── rss-proxy.js        # RSS feed proxy (270+ domains)
│   ├── telegram-feed.js    # Telegram OSINT feed
│   ├── oref-alerts.js      # Israel OREF siren alerts
│   ├── opensky.js          # OpenSky flight data
│   ├── ais-snapshot.js     # AIS vessel snapshot
│   ├── polymarket.js       # Prediction markets
│   ├── gpsjam.js           # GPS interference data
│   ├── eia/                # EIA energy data proxy
│   ├── enrichment/         # Company & signal enrichment
│   ├── youtube/            # Live detection & embed
│   ├── _api-key.js         # API key validation (shared)
│   ├── _cors.js            # CORS headers (shared)
│   └── _rate-limit.js      # Rate limiting (shared)
├── convex/                 # Convex backend (schema + mutations)
├── data/                   # Static data (telegram channels, gamma irradiators, ...)
├── docs/                   # OpenAPI specs (22 services, YAML + JSON)
├── e2e/                    # Playwright E2E tests
├── proto/                  # Protobuf definitions (134 .proto files)
├── public/                 # Static assets, map styles
├── scripts/                # Build, relay, data seed scripts
│   ├── ais-relay.cjs       # Railway relay server
│   └── telegram/           # Telegram session auth
├── server/                 # Proto-generated server handlers
│   ├── gateway.ts          # Shared gateway middleware
│   ├── router.ts           # Request router
│   └── worldmonitor/       # 19 domain handler directories
├── src/                    # Main application code
│   ├── app/                # App bootstrap, layout, data loading, refresh
│   ├── components/         # 74+ UI components (panels, modals, maps, controls)
│   ├── config/             # Feeds, variants, ML config, feature flags
│   ├── generated/          # Proto-generated TS client/server code
│   ├── services/           # 20 service modules (market, conflict, cyber, ...)
│   ├── styles/             # CSS (main, panels, RTL, happy theme, ...)
│   ├── types/              # TypeScript type definitions
│   ├── utils/              # Shared utilities (proxy, cache, circuit breaker, ...)
│   └── workers/            # Web Workers (ML inference, vector DB, analysis)
├── src-tauri/              # Tauri 2 desktop app configuration
├── tests/                  # Unit tests (data validation, feeds)
├── index.html              # Main SPA entry point
├── settings.html           # Settings window entry
├── live-channels.html      # Live channels manager entry
├── package.json            # Dependencies & scripts
├── vite.config.ts          # Vite build config (variants, proxies, chunks)
├── vercel.json             # Vercel deployment config (CORS, CSP, caching)
├── Makefile                # Buf/proto tooling
└── playwright.config.ts    # E2E test config
```

---

## 4. Tech Stack Summary

| Layer | Technology |
|-------|-----------|
| **Frontend Framework** | Preact (JSX in limited areas), vanilla TypeScript with custom DOM helpers |
| **Build** | Vite 6, multi-entry (main, settings, liveChannels) |
| **Maps** | Deck.gl + MapLibre GL (2D/3D), globe.gl (3D globe), D3 (SVG mobile fallback) |
| **Clustering** | Supercluster, H3 hex grid |
| **AI/ML (browser)** | Transformers.js (ONNX), ONNX Runtime Web — embeddings, summarization, NER, sentiment |
| **AI/ML (cloud)** | Ollama (local) → Groq → OpenRouter — LLM summarization chain |
| **Data Transport** | Protocol Buffers (buf/sebuf), REST, WebSocket, RSS/Atom |
| **API Layer** | Vercel Edge Functions, Sebuf RPC gateway (19 typed services, 100+ RPCs) |
| **Long-Running Server** | Railway relay (Node.js) — AIS, OpenSky, Telegram, OREF, seeding |
| **Cache** | Upstash Redis (server), IndexedDB (client), localStorage (settings) |
| **Database** | Convex (email registrations only) |
| **Desktop** | Tauri 2 (macOS, Windows, Linux) — keychain secrets, native updates |
| **PWA** | Service Worker, offline map tiles |
| **i18n** | i18next, 21 locales including RTL (Arabic, Hebrew, Farsi, Urdu) |
| **Testing** | Playwright (E2E + visual), tsx --test (unit/data) |
| **Monitoring** | Sentry (errors), Vercel Analytics (web vitals) |
| **CI/CD** | GitHub Actions (typecheck, lint, desktop build, test, seed) |
| **Deployment** | Vercel (web), Railway (relay), GitHub Releases (desktop) |

---

## 5. Frontend Architecture

### 5.1 Application Bootstrap

```
index.html
  └─ src/main.ts
       ├── Theme init (inline script for FOUC prevention)
       ├── i18next init (21 locales)
       ├── Sentry init
       ├── Runtime config patching
       ├── Route: ?settings=1 → settings-window.ts
       ├── Route: ?live-channels=1 → live-channels-window.ts
       └── Default: new App('app').init()
              ├── AppContext (global state container)
              ├── PanelLayoutManager → header + map + panels grid
              ├── DataLoaderManager → fetchBootstrapData + loadAllData
              ├── EventHandlerManager → user interactions
              ├── SearchManager → full-text + semantic search
              ├── CountryIntelManager → deep-dive intelligence
              ├── RefreshScheduler → periodic data refresh
              └── DesktopUpdater → Tauri auto-updates
```

### 5.2 State Management

There is no Redux, Zustand, or React Context. The application uses a centralized `AppContext` interface passed through managers:

- **`AppContext`** (`src/app/app-context.ts`): Holds all runtime state — panels, news, markets, predictions, clusters, intelligence cache, cyber threats, monitors, map layers, time range, UI references.
- **Persistence**: `localStorage` via `settings-persistence.ts` for panel order, spans, layers, theme, AI settings.
- **Events**: Custom DOM events (`ai-flow-changed`, `stream-quality-changed`, `runtime-config-changed`) for cross-component communication.

### 5.3 Component System

Components are class-based with a `Panel` base class providing:
- Collapsible/expandable behavior
- Drag-and-drop reordering
- Column span (1×, 2×, 3×) configuration
- Resize handling
- Virtual scrolling via `VirtualList`

DOM construction uses `h()` helper from `dom-utils.ts` (not JSX). Preact is used sparingly for `VerificationChecklist`.

### 5.4 Map Architecture

Three map backends, selected by device and user preference:

| Backend | Library | Use Case |
|---------|---------|----------|
| **DeckGLMap** | Deck.gl + MapLibre GL | Primary desktop (WebGL 2D/3D) |
| **GlobeMap** | globe.gl (Three.js) | 3D globe view |
| **Map** | D3 + SVG | Mobile fallback |

`MapContainer` (`src/components/MapContainer.ts`) selects the backend and unifies the interface. Layers include:
- Clustered events (Supercluster)
- Military flights and vessels
- Satellite fire detections
- GPS interference (H3 hexagons)
- Internet outages (country shading)
- Earthquake circles
- Conflict/unrest markers
- Navigation warnings (maritime)

### 5.5 Panel Layout

The dashboard uses a CSS Grid layout with two zones:
- **Side panels** (`#panelsGrid`): Vertical scrollable panel grid
- **Bottom panels** (`#mapBottomGrid`): Wide-screen secondary area

`PanelLayoutManager` (`src/app/panel-layout.ts`) handles:
- Variant-specific panel selection
- Drag-and-drop panel reordering (persisted)
- Region-based content (Global, Americas, MENA, EU, Asia, LATAM, Africa, Oceania)
- TV mode (auto-scrolling for wall displays)
- Mobile responsive layout with bottom sheet navigation

---

## 6. Backend Architecture

### 6.1 Vercel Edge Functions

All server-side code runs as Vercel Edge Functions (V8 isolates) for minimal cold-start latency. Two categories:

**Sebuf RPC handlers** — Type-safe Protobuf services under `api/<domain>/v1/[rpc].ts`:
Each handler is routed through `server/gateway.ts` which applies the middleware pipeline.

**Standalone edge functions** — Direct HTTP handlers for non-RPC endpoints:
`bootstrap.js`, `rss-proxy.js`, `telegram-feed.js`, `opensky.js`, `ais-snapshot.js`, `polymarket.js`, `gpsjam.js`, `geo.js`, `eia/`, `register-interest.js`, `cache-purge.js`, `seed-health.js`, `fwdstart.js`, `download.js`, `version.js`, `story.js`, `og-story.js`, `youtube/`, `enrichment/`.

### 6.2 Gateway Middleware Pipeline

```
Request
  │
  ├─ 1. Origin check → 403 if disallowed
  ├─ 2. CORS headers (worldmonitor.app, Vercel previews, localhost, Tauri)
  ├─ 3. OPTIONS preflight → 204
  ├─ 4. API key validation (desktop: required; trusted origin: optional; other: required)
  ├─ 5. Rate limiting (Upstash Redis, sliding window: 600 req/60s)
  ├─ 6. Route match (POST → GET compat for legacy clients)
  ├─ 7. Handler execution
  └─ 8. Cache-tier headers (fast/medium/slow/static/daily/no-store)
```

### 6.3 Cache Tiers

Every RPC has an assigned cache tier controlling Vercel CDN edge caching:

| Tier | `s-maxage` | `stale-while-revalidate` | Use Case |
|------|-----------|-------------------------|----------|
| **fast** | 120s | 30s | Real-time data (flight status) |
| **medium** | 300s | 60s | Market quotes, crypto, commodities |
| **slow** | 900s | 120s | Earthquakes, outages, conflict events |
| **static** | 3600s | 300s | Historical data, research, intel briefs |
| **daily** | 86400s | 3600s | BIS rates, displacement, trade data |
| **no-store** | — | — | Live vessel tracking, aircraft tracking |

### 6.4 Domain Services (19 Sebuf RPC Domains)

| Domain | RPCs | Data Sources |
|--------|------|-------------|
| **seismology** | `list-earthquakes` | USGS GeoJSON |
| **wildfire** | `list-fire-detections` | NASA FIRMS |
| **climate** | `list-climate-anomalies` | Open-Meteo, ERA5 |
| **natural** | `list-natural-events` | GDACS |
| **conflict** | `list-acled-events`, `list-ucdp-events`, `list-iran-events`, `get-humanitarian-summary` | ACLED, UCDP |
| **unrest** | `list-unrest-events` | ACLED, GDELT |
| **military** | `list-military-flights`, `get-theater-posture`, `get-aircraft-details[_batch]`, `get-wingbits-status`, `get-usni-fleet-report`, `list-military-bases` | OpenSky relay, WingBits, USNI |
| **maritime** | `get-vessel-snapshot`, `list-navigational-warnings` | AISStream relay |
| **aviation** | `list-airport-flights`, `list-airport-delays`, `get-airport-ops-summary`, `get-carrier-ops`, `track-aircraft`, `search-flight-prices`, `list-aviation-news`, `get-flight-status` | AviationStack, ICAO, Travelpayouts |
| **cyber** | `list-cyber-threats` | Feodo Tracker, URLhaus, C2IntelFeeds, AlienVault OTX, AbuseIPDB |
| **infrastructure** | `list-internet-outages`, `list-service-statuses`, `get-temporal-baseline`, `record-baseline-snapshot`, `get-cable-health` | Cloudflare Radar, 33 status pages |
| **intelligence** | `get-risk-scores`, `get-pizzint-status`, `classify-event`, `get-country-intel-brief`, `search-gdelt-documents`, `deduct-situation` | GDELT, PizzINT, LLM classification |
| **market** | `list-market-quotes`, `list-crypto-quotes`, `list-commodity-quotes`, `get-sector-summary`, `list-stablecoin-markets`, `list-etf-flows`, `get-country-stock-index`, `list-gulf-quotes` | Yahoo Finance, Finnhub, CoinGecko |
| **economic** | `get-fred-series`, `list-world-bank-indicators`, `get-energy-prices`, `get-macro-signals`, `get-energy-capacity`, `get-bis-policy-rates`, `get-bis-exchange-rates`, `get-bis-credit` | FRED, World Bank, EIA, BIS |
| **prediction** | `list-prediction-markets` | Polymarket relay |
| **news** | `list-feed-digest`, `summarize-article`, `summarize-article-cache` | RSS feeds, LLM providers |
| **research** | `list-arxiv-papers`, `list-trending-repos`, `list-hackernews-items`, `list-tech-events` | arXiv, GitHub, HN Algolia |
| **trade** | `get-trade-restrictions`, `get-tariff-trends`, `get-trade-flows`, `get-trade-barriers` | WTO, custom data |
| **supply-chain** | `get-shipping-rates`, `get-chokepoint-status`, `get-critical-minerals` | FRED, custom data |
| **displacement** | `get-displacement-summary`, `get-population-exposure` | UNHCR, WorldPop |
| **giving** | `get-giving-summary` | Custom data |
| **positive-events** | `list-positive-geo-events` | Custom data |

---

## 7. API Layer — Sebuf RPC

### 7.1 Protocol Buffers

The project uses **134 `.proto` files** under `proto/worldmonitor/` defining typed request/response messages for all 19 domains. The `buf` toolchain (`Makefile`) generates:

- **TypeScript client** (`src/generated/client/`) — used by the frontend
- **TypeScript server** (`src/generated/server/`) — used by edge function handlers
- **OpenAPI specs** (`docs/`) — YAML + JSON for each service

### 7.2 Sebuf Wire Format

Sebuf is a lightweight JSON-over-HTTP RPC framework. Requests use POST with JSON bodies containing proto-serialized fields. Responses return JSON with proto-compatible field names (snake_case ↔ camelCase auto-conversion).

### 7.3 Domain Isolation

Each domain is a separate Vercel Edge Function bundle (`api/<domain>/v1/[rpc].ts`), meaning Vercel only packages the code for that one domain per function. This cuts cold-start cost by ~20× compared to a monolithic handler.

---

## 8. Railway Relay Server

`scripts/ais-relay.cjs` is a long-running Node.js server deployed on Railway that handles workloads incompatible with serverless (stateful connections, polling, WebSocket):

| Responsibility | Protocol | Source |
|---------------|----------|--------|
| AIS vessel tracking | WebSocket → HTTP snapshot | AISStream.io |
| OpenSky flight data | HTTP proxy | opensky-network.org |
| Telegram OSINT feed | MTProto polling | Telegram channels |
| OREF siren alerts | HTTP polling | Israel Home Front |
| Polymarket data | HTTP proxy | Polymarket API |
| RSS proxy (blocked domains) | HTTP proxy | Various RSS feeds |
| Market/commodity seeding | HTTP → Redis | Yahoo Finance, etc. |
| ICAO NOTAM | HTTP proxy | ICAO |

**Authentication**: `RELAY_SHARED_SECRET` via `x-relay-key` header. Vercel edge functions authenticate to the relay using this shared secret.

**Communication**: Vercel → Railway via `WS_RELAY_URL` (HTTPS). The relay exposes HTTP endpoints that Vercel edge functions call.

---

## 9. Data Flow Pipeline

### 9.1 Bootstrap Flow (Initial Page Load)

```
Browser loads index.html
  │
  ├── Critical CSS (inlined) + theme detection
  ├── i18n + Sentry init
  │
  └── App.init()
        │
        ├── fetchBootstrapData('/api/bootstrap?tier=fast')
        │     └── Redis → [earthquakes, outages, markets, crypto, military, GPS jam]
        │
        ├── fetchBootstrapData('/api/bootstrap?tier=slow')
        │     └── Redis → [conflict, trade, climate, displacement, intel briefs]
        │
        └── DataLoaderManager.loadAllData()
              ├── fetchCategoryFeeds() → 100+ RSS feeds via /api/rss-proxy
              ├── fetchMultipleStocks() → /api/market/v1/list-market-quotes
              ├── fetchCrypto() → /api/market/v1/list-crypto-quotes
              ├── fetchPredictions() → Polymarket via relay
              ├── fetchMilitaryFlights() → /api/military/v1/*
              ├── fetchInternetOutages() → /api/infrastructure/v1/*
              ├── fetchConflictData() → /api/conflict/v1/*
              └── ... (20+ parallel data fetches)
```

### 9.2 Refresh Cycle

`RefreshScheduler` runs periodic data refreshes at configured intervals:
- **Markets**: ~60s
- **News/RSS**: ~300s
- **Geopolitical layers**: ~900s
- **Static/historical**: ~3600s

### 9.3 Real-Time Data

| Source | Mechanism | Refresh |
|--------|-----------|---------|
| AIS vessels | Relay WebSocket → HTTP snapshot | On-demand |
| OREF sirens | Relay polling → Edge function | ~10s polling |
| Telegram OSINT | Relay MTProto → Edge function | ~30s polling |
| Markets (seeded) | Relay → Redis → Bootstrap | ~60s |

---

## 10. AI / ML Pipeline

### 10.1 Summarization Chain

The AI pipeline uses a fallback chain for article summarization:

```
Ollama (local LLM, if configured)
  │ fail/unavailable
  ▼
Groq (cloud, 14,400 req/day free tier)
  │ fail/rate-limited
  ▼
OpenRouter (cloud, 50 req/day free tier)
  │ fail/rate-limited
  ▼
Browser T5 (ONNX Runtime Web, fully client-side)
```

The `SummarizeArticle` RPC (`server/.../news/v1/summarize-article.ts`) handles Ollama/Groq/OpenRouter. Redis caching deduplicates identical summarization requests across users.

### 10.2 Browser ML Worker

`src/workers/ml.worker.ts` runs in a Web Worker using Transformers.js (ONNX):

| Capability | Model | Use Case |
|-----------|-------|----------|
| **Embeddings** | all-MiniLM-L6-v2 | Headline similarity, semantic search |
| **Summarization** | T5-small | Fallback article summarization |
| **Sentiment** | distilbert-sentiment | News sentiment classification |
| **NER** | bert-base-NER | Entity extraction from headlines |
| **Semantic Clustering** | — | DBSCAN on embeddings for story grouping |

### 10.3 Headline Memory (RAG)

When enabled, the system implements client-side RAG:

1. **Ingest**: New headlines are embedded via the ML worker
2. **Store**: Vectors stored in IndexedDB (`worldmonitor_vector_store`, max 5,000 vectors)
3. **Search**: Cosine similarity search for semantic recall
4. **Context**: Relevant past headlines inform the summarization prompt

### 10.4 AI Flow Settings

Users control the AI pipeline via `AiFlowSettings`:
- `browserModel`: Enable/disable in-browser ONNX inference
- `cloudLlm`: Enable/disable cloud LLM providers
- `mapNewsFlash`: Show breaking news flashes on the map
- `headlineMemory`: Enable/disable RAG headline memory
- `badgeAnimation`: Animated intelligence badges

---

## 11. Caching Architecture

### 11.1 Multi-Layer Cache

```
┌─────────────────────────────────────────┐
│            Client (Browser)             │
│                                         │
│  localStorage ──── Panel settings,      │
│                    theme, AI flow,      │
│                    panel order          │
│                                         │
│  IndexedDB ─────── RSS feed cache,     │
│                    vector store (RAG),  │
│                    circuit breaker      │
│                    state, API cache     │
│                                         │
│  In-memory ─────── AppContext state,    │
│                    intelligence cache   │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│            Vercel CDN Edge              │
│                                         │
│  Cache-Control headers per cache tier   │
│  (fast: 120s, medium: 300s, slow: 900s,│
│   static: 3600s, daily: 86400s)        │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│            Upstash Redis                │
│                                         │
│  Bootstrap data ── 30-45 min TTL       │
│  GPS jam cache ─── 1 hour TTL          │
│  Internet outages ─ 30 min TTL         │
│  AI summary cache ─ cross-user dedup   │
│  Rate limit state ─ sliding window     │
│  Seed health ───── domain freshness    │
│  AIS relay state ── vessel positions   │
└─────────────────────────────────────────┘
```

### 11.2 Circuit Breaker

`src/utils/circuit-breaker.ts` wraps external API calls with:
- Configurable failure threshold
- Exponential backoff
- Optional persistent cache fallback (stale data on failure)
- Per-source TTL management

---

## 12. Authentication & Security

### 12.1 API Key Validation

`api/_api-key.js` implements tiered access:

| Client | Key Required | Notes |
|--------|-------------|-------|
| Desktop (Tauri) | Yes | Key stored in OS keychain |
| Trusted browser origins | No | worldmonitor.app, Vercel previews, localhost |
| Other origins | Yes | Prevents unauthorized API usage |

Valid keys are stored in `WORLDMONITOR_VALID_KEYS` (comma-separated, format: `wm_<hex>`).

### 12.2 CORS

`api/_cors.js` allows:
- `*.worldmonitor.app`
- Vercel preview URLs
- `localhost:*`
- `tauri://localhost` (desktop app)

### 12.3 Rate Limiting

Upstash Redis sliding window: **600 requests / 60 seconds** per IP. IP extracted from `x-real-ip`, `cf-connecting-ip`, or `x-forwarded-for`.

### 12.4 Relay Authentication

Railway relay uses `RELAY_SHARED_SECRET` via `x-relay-key` header. Emergency override via `ALLOW_UNAUTHENTICATED_RELAY` (disabled in production).

### 12.5 Content Security

- CSP headers in `vercel.json`
- `DOMPurify` for HTML sanitization
- `safeHtml()` utility for DOM construction

---

## 13. Deployment Topology

```
┌──────────────────────┐    ┌──────────────────────┐
│     GitHub Repo      │    │    GitHub Actions     │
│  (source of truth)   │───▶│  (CI/CD pipeline)    │
│                      │    │                       │
└──────────────────────┘    └───────┬───────────────┘
                                    │
                   ┌────────────────┼────────────────┐
                   ▼                ▼                ▼
          ┌────────────────┐ ┌────────────┐ ┌──────────────────┐
          │  Vercel (web)  │ │  Railway   │ │ GitHub Releases  │
          │                │ │  (relay)   │ │ (desktop builds) │
          │  Edge functions│ │            │ │                  │
          │  CDN + static  │ │  ais-relay │ │  macOS (dmg)     │
          │                │ │  .cjs      │ │  Windows (nsis)  │
          │  worldmonitor  │ │            │ │  Linux (appimage)│
          │  .app          │ │            │ │                  │
          └────────────────┘ └────────────┘ └──────────────────┘
                   │                │
                   ▼                ▼
          ┌────────────────┐ ┌────────────┐
          │ Upstash Redis  │ │  Convex    │
          │ (managed)      │ │  (managed) │
          └────────────────┘ └────────────┘
```

### Build Variants

- `npm run build` — full variant
- `npm run build:tech` — tech variant
- `npm run build:finance` — finance variant
- `npm run build:happy` — happy variant
- `npm run build:desktop` — Tauri desktop (all platforms)

---

## 14. Desktop Application (Tauri)

The desktop app wraps the web SPA using Tauri 2:

- **Platforms**: macOS (DMG), Windows (NSIS), Linux (AppImage)
- **Variant configs**: `tauri.conf.json`, `tauri.tech.conf.json`, `tauri.finance.conf.json`
- **Secrets**: API keys stored in OS keychain (not localStorage)
- **Updates**: Auto-update from GitHub Releases
- **Cloud fallback**: Desktop clients use `WORLDMONITOR_VALID_KEYS` for authenticated API access

---

## 15. Proto / Code Generation

### 15.1 Buf Configuration

- `proto/buf.yaml`: Linting (STANDARD + COMMENTS), breaking change detection
- `proto/buf.gen.yaml`: Plugins for TS client, TS server, OpenAPI YAML/JSON

### 15.2 Generated Code

```
src/generated/
├── client/worldmonitor/    # TypeScript RPC clients (browser)
│   ├── seismology/v1/
│   ├── conflict/v1/
│   ├── ... (19 domains)
│   └── service_client.ts
└── server/worldmonitor/    # TypeScript server stubs (edge)
    ├── seismology/v1/
    ├── conflict/v1/
    ├── ... (19 domains)
    └── service_server.ts

docs/
├── worldmonitor.seismology.v1.yaml
├── worldmonitor.seismology.v1.json
├── ... (22 services × 2 formats)
```

### 15.3 Workflow

```
Edit .proto files
  → make generate (buf generate)
    → TS clients + servers + OpenAPI specs regenerated
      → Vite bundles updated clients into SPA
      → Vercel bundles updated servers into edge functions
```

---

## 16. Testing Strategy

| Type | Tool | Location | Command |
|------|------|----------|---------|
| **E2E** | Playwright | `e2e/` | `npm run test:e2e` |
| **Visual regression** | Playwright | `e2e/` | `npm run test:e2e:visual` |
| **Data validation** | tsx --test | `tests/` | `npm run test:data` |
| **Feed validation** | tsx --test | `tests/` | `npm run test:feeds` |
| **Typecheck** | tsc | — | `npx tsc --noEmit` |
| **Lint** | markdownlint | docs/ | `npm run lint:md` |

E2E tests use Chromium with SwiftShader for WebGL support, running against the dev server at `127.0.0.1:4173`.

---

## 17. Internationalization

- **Library**: i18next
- **Locales**: 21 languages — en, es, fr, de, it, pt, nl, sv, ja, ko, zh, ar, he, fa, ur, hi, tr, pl, uk, ru, id
- **RTL support**: Arabic, Hebrew, Farsi, Urdu — with dedicated `rtl-overrides.css`
- **Translation**: Locale JSON files, lazy-loaded per language
- **Build**: Manual chunks per locale in Vite config for efficient code splitting

---

## 18. Configuration & Environment

All environment variables are optional — features degrade gracefully when keys are missing.

| Category | Variables | Purpose |
|----------|-----------|---------|
| **AI** | `GROQ_API_KEY`, `OPENROUTER_API_KEY` | Cloud LLM summarization |
| **Cache** | `UPSTASH_REDIS_REST_URL`, `UPSTASH_REDIS_REST_TOKEN` | Cross-user cache + rate limiting |
| **Markets** | `FINNHUB_API_KEY` | Stock quotes |
| **Energy** | `EIA_API_KEY` | Oil/gas prices, production |
| **Economic** | `FRED_API_KEY` | Federal Reserve data |
| **Aviation** | `AVIATIONSTACK_API`, `ICAO_API_KEY`, `WINGBITS_API_KEY` | Flight data, NOTAMs |
| **Conflict** | `ACLED_ACCESS_TOKEN`, `UCDP_ACCESS_TOKEN` | Armed conflict events |
| **Outages** | `CLOUDFLARE_API_TOKEN` | Internet outage detection |
| **Fires** | `NASA_FIRMS_API_KEY` | Satellite fire detection |
| **Cyber** | `OTX_API_KEY`, `ABUSEIPDB_API_KEY`, `URLHAUS_AUTH_KEY` | Threat intelligence |
| **Relay** | `AISSTREAM_API_KEY`, `OPENSKY_CLIENT_ID/SECRET`, `TELEGRAM_*`, `RELAY_SHARED_SECRET` | Real-time feeds |
| **Site** | `VITE_VARIANT`, `VITE_WS_API_URL`, `VITE_SENTRY_DSN` | App configuration |
| **Desktop** | `WORLDMONITOR_VALID_KEYS` | API key validation for Tauri |
| **DB** | `CONVEX_URL` | Email registration storage |

See `.env.example` for the complete list with registration links for each service.
