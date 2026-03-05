# World Monitor — UI Architecture Guide

This document describes the complete UI architecture of World Monitor: how the interface is structured, how components communicate, how state flows, how the map system works, and how to build new panels and features.

---

## Table of Contents

1. [UI Architecture Overview](#1-ui-architecture-overview)
2. [Application Bootstrap Flow](#2-application-bootstrap-flow)
3. [Layout System](#3-layout-system)
4. [Component Architecture](#4-component-architecture)
5. [Panel System](#5-panel-system)
6. [Map System](#6-map-system)
7. [State Management](#7-state-management)
8. [Data Fetching & Loading](#8-data-fetching--loading)
9. [Styling Architecture](#9-styling-architecture)
10. [Variant System](#10-variant-system)
11. [Navigation & Routing](#11-navigation--routing)
12. [Country Deep Dive](#12-country-deep-dive)
13. [Modals & Overlays](#13-modals--overlays)
14. [Search System](#14-search-system)
15. [AI Integration in UI](#15-ai-integration-in-ui)
16. [Responsive Design](#16-responsive-design)
17. [Accessibility & i18n](#17-accessibility--i18n)
18. [Performance Patterns](#18-performance-patterns)
19. [Component Catalog](#19-component-catalog)
20. [Building New Panels](#20-building-new-panels)

---

## 1. UI Architecture Overview

World Monitor uses a **panel-based dashboard architecture** built on vanilla TypeScript with minimal framework usage. The UI is map-centric — a large interactive map fills the primary viewport, with information panels arranged in a configurable grid alongside it.

```
┌──────────────────────────────────────────────────────────────────────┐
│  HEADER                                                              │
│  [☰] [Variant Switcher] [Logo] [Region] [Search] [Theme] [⛶] [⚙]  │
├───────────────────────────────────┬──────────────────────────────────┤
│                                   │                                  │
│         MAP SECTION               │        PANELS GRID               │
│                                   │                                  │
│   ┌───────────────────────────┐   │   ┌──────────┐ ┌──────────┐    │
│   │                           │   │   │ NewsPanel │ │ Market   │    │
│   │    DeckGL / Globe / SVG   │   │   │          │ │ Panel    │    │
│   │                           │   │   ├──────────┤ ├──────────┤    │
│   │    Layers:                │   │   │ Military │ │ Intel    │    │
│   │    • Clusters             │   │   │ Panel    │ │ Panel    │    │
│   │    • Military flights     │   │   ├──────────┤ ├──────────┤    │
│   │    • Vessels              │   │   │ Conflict │ │ Cyber    │    │
│   │    • Fires                │   │   │ Panel    │ │ Panel    │    │
│   │    • Outages              │   │   └──────────┘ └──────────┘    │
│   │    • GPS interference     │   │                                  │
│   │    • Earthquakes          │   │                                  │
│   │                           │   │                                  │
│   └───────────────────────────┘   │                                  │
│                                   │                                  │
│   ┌─────────────────────────────────────────────────────────┐       │
│   │  MAP BOTTOM GRID (ultra-wide only)                       │       │
│   │  [Panel A] [Panel B] [Panel C]                           │       │
│   └─────────────────────────────────────────────────────────┘       │
│                                                                      │
├──────────────────────────────────────────────────────────────────────┤
│  STATUS BAR · BREAKING NEWS BANNER · PLAYBACK CONTROLS               │
└──────────────────────────────────────────────────────────────────────┘
```

### Key Design Principles

1. **Map-first**: The map is the primary interface element; panels provide detail and context
2. **Panel composability**: Each data domain is a self-contained panel with consistent behavior
3. **Variant-adaptive**: The same codebase renders four different product variants
4. **Progressive enhancement**: Features degrade gracefully when data sources are unavailable
5. **Performance-first**: Virtual scrolling, lazy loading, code splitting, web workers for ML

---

## 2. Application Bootstrap Flow

### Entry Points

World Monitor has three HTML entry points, each loading a different module:

| Entry | File | Module | Purpose |
|-------|------|--------|---------|
| Main dashboard | `index.html` | `src/main.ts` → `App` | Primary intelligence dashboard |
| Settings | `settings.html` | `src/settings-window.ts` | User preferences window |
| Live channels | `live-channels.html` | `src/live-channels-window.ts` | YouTube/livestream manager |

### Main Bootstrap Sequence (`src/main.ts`)

```
1. Inline theme detection (in <head> for FOUC prevention)
2. Load src/main.ts
3. Initialize i18next (21 locales, lazy-loaded)
4. Initialize Sentry error reporting
5. Apply runtime config patches
6. Route check:
   ├── ?settings=1    → mount settings window
   ├── ?live-channels=1 → mount live channels window
   └── default        → new App('app').init()
```

### App.init() Sequence (`src/App.ts`)

```
1. Create AppContext (global state)
2. Initialize managers:
   ├── PanelLayoutManager   → renders header + map + panels grid
   ├── DataLoaderManager    → fetches bootstrap + all data
   ├── EventHandlerManager  → wires up user interactions
   ├── SearchManager        → full-text + semantic search
   ├── CountryIntelManager  → country deep dive intelligence
   ├── RefreshScheduler     → periodic data refresh cycles
   └── DesktopUpdater       → Tauri auto-update (desktop only)
3. Parse URL state (?view, ?zoom, ?layers, ?country, ?expanded)
4. Apply initial state (map position, active layers, expanded panels)
5. Mark initialLoadComplete
```

---

## 3. Layout System

### 3.1 Overall Layout Structure

The layout is managed by `PanelLayoutManager` (`src/app/panel-layout.ts`):

```html
<div id="app">
  <header>
    <div class="header-left">
      <button class="hamburger" />        <!-- Mobile menu -->
      <div class="variant-switcher" />     <!-- World/Tech/Finance/Good News -->
      <div class="logo" />
    </div>
    <div class="header-center">
      <div class="region-selector" />      <!-- Global/Americas/MENA/EU/Asia/... -->
    </div>
    <div class="header-right">
      <button class="search-trigger" />    <!-- Search modal -->
      <button class="theme-toggle" />      <!-- Light/dark -->
      <button class="fullscreen" />        <!-- Fullscreen toggle -->
      <button class="tv-mode" />           <!-- TV mode (happy variant) -->
      <div id="unified-settings-mount" />  <!-- Settings gear -->
    </div>
  </header>

  <main class="main-content">
    <section id="mapSection">
      <div id="mapContainer" />            <!-- Map backend mount point -->
      <div id="mapResizeHandle" />         <!-- Drag to resize map vs panels -->
    </section>

    <section id="panelsGrid" />            <!-- Primary panel grid -->
    <section id="mapBottomGrid" />         <!-- Ultra-wide bottom panels -->
  </main>

  <aside id="country-deep-dive-panel" />   <!-- Country intel slide-in -->

  <!-- Floating elements -->
  <div id="signal-modal" />                <!-- Signal detail modal -->
  <div id="search-modal" />                <!-- Search overlay -->
  <div id="breaking-banner" />             <!-- Breaking news strip -->
  <div id="status-panel" />                <!-- Connection/health status -->
  <div id="playback-control" />            <!-- Time playback bar -->
</div>
```

### 3.2 Grid Layout

The panels grid uses CSS Grid with configurable column spans:

```css
.panels-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(360px, 1fr));
  gap: 12px;
  padding: 12px;
}

/* Panel span variants */
.panel[data-span="1"] { grid-column: span 1; }
.panel[data-span="2"] { grid-column: span 2; }
.panel[data-span="3"] { grid-column: span 3; }
```

### 3.3 Zone Management

`ensureCorrectZones()` dynamically assigns panels to zones based on viewport width:

| Width | Layout |
|-------|--------|
| < 768px | Mobile: stacked panels, SVG map |
| 768–1599px | Standard: side-by-side map + panels |
| ≥ 1600px | Ultra-wide: map + side panels + bottom grid |

### 3.4 Map/Panel Resize

A drag handle (`#mapResizeHandle`) between the map and panels grid lets users resize the map-to-panel ratio. The ratio is persisted in `localStorage`.

---

## 4. Component Architecture

### 4.1 DOM Helpers (No Framework)

The UI is built using a custom DOM helper library (`src/utils/dom-utils.ts`):

```typescript
// h() creates elements with typed attributes and children
const card = h('div', { className: 'card', onclick: handler },
  h('h3', {}, 'Title'),
  h('p', {}, 'Content'),
);

// fragment() creates DocumentFragments
const frag = fragment(child1, child2, child3);

// replaceChildren() efficiently updates DOM
replaceChildren(container, newChild1, newChild2);

// safeHtml() sanitizes HTML strings via DOMPurify
element.innerHTML = safeHtml(untrustedHtml);
```

### 4.2 Component Lifecycle

Components are class-based with a common lifecycle:

```typescript
class MyComponent {
  private el: HTMLElement;

  constructor(container: HTMLElement, ctx: AppContext) {
    this.el = h('div', { className: 'my-component' });
    container.appendChild(this.el);
  }

  update(data: MyData): void {
    // Re-render with new data
    replaceChildren(this.el, this.renderContent(data));
  }

  destroy(): void {
    this.el.remove();
    // Clean up listeners, timers, etc.
  }
}
```

### 4.3 Preact Usage

Preact is used minimally — only for `VerificationChecklist` and a few interactive form components. The rest of the UI is vanilla DOM.

```typescript
import { h, Component, render } from 'preact';

class VerificationChecklist extends Component<Props, State> {
  render() {
    return h('div', { className: 'checklist' }, /* ... */);
  }
}

// Mount into DOM
render(h(VerificationChecklist, props), container);
```

---

## 5. Panel System

### 5.1 Panel Base Class

All dashboard panels extend `Panel` (`src/components/Panel.ts`):

```typescript
class Panel {
  protected container: HTMLElement;
  protected header: HTMLElement;
  protected body: HTMLElement;
  protected ctx: AppContext;

  // Capabilities provided by base class:
  collapse(): void;           // Toggle collapsed state
  expand(): void;
  setSpan(n: 1 | 2 | 3): void;  // Column span
  enableDrag(): void;         // Drag-and-drop reordering
  showLoading(): void;        // Loading skeleton
  showError(msg: string): void;
  showEmpty(msg: string): void;
}
```

### 5.2 Panel Anatomy

Every panel follows a consistent structure:

```
┌─────────────────────────────────────────┐
│  HEADER                                  │
│  [icon] [Title]        [▼] [⚙] [↔] [×] │
│                                          │
│  ─ collapse toggle ─ settings ─ span ─   │
├─────────────────────────────────────────┤
│                                          │
│  BODY                                    │
│                                          │
│  • Content rendered by subclass          │
│  • VirtualList for long lists            │
│  • Loading skeleton / error / empty      │
│                                          │
│                                          │
└─────────────────────────────────────────┘
```

### 5.3 Panel Categories by Variant

#### Full Variant Panels

| Panel | Component | Content |
|-------|-----------|---------|
| **Live News** | `LiveNewsPanel` | Real-time RSS feed aggregation |
| **Breaking Banner** | `BreakingNewsBanner` | Critical breaking news strip |
| **Market** | `MarketPanel` | Stock indices, tickers |
| **Heatmap** | `HeatmapPanel` | Market sector heatmap |
| **Commodities** | `CommoditiesPanel` | Oil, gold, commodities |
| **Crypto** | `CryptoPanel` | Cryptocurrency quotes |
| **Prediction** | `PredictionPanel` | Polymarket prediction markets |
| **GDELT Intel** | `GdeltIntelPanel` | GDELT intelligence events |
| **CII** | `CIIPanel` | Critical Infrastructure Index |
| **Strategic Risk** | `StrategicRiskPanel` | Country risk scoring |
| **Strategic Posture** | `StrategicPosturePanel` | Military theater posture |
| **Deduction** | `DeductionPanel` | AI-powered situation deduction |
| **Cascade** | `CascadePanel` | Cascading risk analysis |
| **Satellite Fires** | `SatelliteFiresPanel` | NASA FIRMS fire detections |
| **UCDP Events** | `UcdpEventsPanel` | Uppsala conflict events |
| **Displacement** | `DisplacementPanel` | Refugee displacement |
| **Climate** | `ClimateAnomalyPanel` | Climate anomalies |
| **Population** | `PopulationExposurePanel` | Population at risk |
| **OREF Sirens** | `OrefSirensPanel` | Israel siren alerts |
| **Telegram Intel** | `TelegramIntelPanel` | Telegram OSINT channels |
| **Service Status** | `ServiceStatusPanel` | 33 service health checks |
| **Economic** | `EconomicPanel` | FRED/BIS economic data |
| **Macro Signals** | `MacroSignalsPanel` | Economic macro indicators |
| **ETF Flows** | `ETFFlowsPanel` | ETF money flows |
| **Stablecoin** | `StablecoinPanel` | Stablecoin market caps |
| **Monitor** | `MonitorPanel` | Custom keyword monitors |
| **World Clock** | `WorldClockPanel` | Multi-timezone clock |

#### Tech Variant Panels

| Panel | Content |
|-------|---------|
| `LiveNewsPanel` | Tech news feeds |
| `InsightsPanel` | AI-generated insights |
| `TechEventsPanel` | Tech conferences, launches |
| `TechReadinessPanel` | Technology readiness levels |
| `TechHubsPanel` | Global tech hub activity |
| `ServiceStatusPanel` | Service status monitoring |
| `RuntimeConfigPanel` | Runtime feature toggles |

#### Finance Variant Panels

| Panel | Content |
|-------|---------|
| `GulfEconomiesPanel` | Gulf state economies |
| `InvestmentsPanel` | Investment tracking |
| `TradePolicyPanel` | Trade policy tracker |
| `SupplyChainPanel` | Supply chain disruptions |
| `SecurityAdvisoriesPanel` | Security advisories |

#### Happy Variant Panels

| Panel | Content |
|-------|---------|
| `CountersPanel` | Global positive metrics |
| `ProgressChartsPanel` | Progress area charts (D3) |
| `BreakthroughsTickerPanel` | Scientific breakthroughs |
| `HeroSpotlightPanel` | Featured positive stories |
| `GoodThingsDigestPanel` | Curated good news digest |
| `SpeciesComebackPanel` | Species recovery tracker |
| `RenewableEnergyPanel` | Renewable energy growth |
| `PositiveNewsFeedPanel` | Positive RSS feeds |

### 5.4 Panel Drag-and-Drop

Users can reorder panels by dragging their headers. The order is persisted:

```
User drags panel header
  → PanelLayoutManager handles dragstart/dragover/drop
    → Recompute panel order
      → Save to localStorage (ctx.PANEL_ORDER_KEY)
        → Re-render panels grid
```

### 5.5 Panel Settings Persistence

Panel configuration is stored in `localStorage`:

| Key | Data |
|-----|------|
| `wm-panel-order` / `wm-panel-order-bottom` | Panel ordering for each zone |
| `wm-panel-spans` | Column span per panel (1, 2, or 3) |
| `wm-panel-settings` | Per-panel config (collapsed state, filters) |

---

## 6. Map System

### 6.1 Map Backend Selection

`MapContainer` (`src/components/MapContainer.ts`) selects the map backend:

```
MapContainer.init()
  │
  ├── Mobile device?
  │     └── Yes → Map (D3 SVG)
  │
  ├── User preference = globe?
  │     └── Yes → GlobeMap (globe.gl / Three.js)
  │
  └── Default → DeckGLMap (Deck.gl + MapLibre GL)
```

### 6.2 DeckGLMap (Primary)

The primary map uses Deck.gl layers over a MapLibre GL base map:

```
DeckGLMap
  │
  ├── MapLibre GL base map
  │     └── Vector tiles (map styles in public/)
  │
  └── Deck.gl layer stack (bottom to top):
        ├── GeoJsonLayer      → Country borders, regions
        ├── H3HexagonLayer    → GPS interference zones
        ├── ScatterplotLayer   → Earthquakes (sized by magnitude)
        ├── IconLayer          → Conflict events, protests
        ├── IconLayer          → Military flights
        ├── IconLayer          → Vessels
        ├── IconLayer          → Fire detections
        ├── ScatterplotLayer   → Internet outages
        ├── ArcLayer           → Trade flows, displacement routes
        ├── TextLayer          → Labels, annotations
        └── SuperclusterLayer  → Clustered events
```

### 6.3 Map Layers (Toggleable)

Users can toggle layers via the map layer control. Layer state is persisted in `localStorage` and URL parameters.

| Layer Group | Layers |
|-------------|--------|
| **Events** | Earthquakes, wildfires, natural disasters, conflicts, protests |
| **Military** | Military flights, military bases, naval vessels |
| **Infrastructure** | Internet outages, service disruptions, GPS interference |
| **Maritime** | Commercial vessels, navigational warnings |
| **Intelligence** | OREF sirens, Telegram alerts, GDELT events |
| **Economic** | Trade flows, displacement routes |
| **Climate** | Temperature anomalies, extreme weather |

### 6.4 Clustering

Events are clustered using Supercluster to prevent visual overload:

```typescript
const cluster = new Supercluster({
  radius: 40,        // Pixel radius for clustering
  maxZoom: 14,       // Max zoom for clustering
  minZoom: 0,
});

cluster.load(geoJsonFeatures);
const clusters = cluster.getClusters(bbox, zoom);
```

Clusters expand on click and show aggregate counts with breakdown by event type.

### 6.5 Map Interactions

| Interaction | DeckGL | Globe | SVG |
|------------|--------|-------|-----|
| Pan/zoom | ✓ | ✓ | ✓ |
| 3D pitch/rotation | ✓ (configurable) | ✓ | ✗ |
| Click event | ✓ (picking) | ✓ | ✓ |
| Hover tooltip | ✓ | ✓ | ✓ |
| Cluster expansion | ✓ | ✓ | ✓ |
| Country selection | ✓ | ✓ | ✓ |
| Time filter | ✓ | ✓ | ✓ |

### 6.6 Map Popups

`MapPopup` displays detail cards on feature click:

```
┌─────────────────────────────────┐
│  [icon] Event Title              │
│  Source: Reuters · 2h ago        │
│  Location: Kyiv, Ukraine         │
│                                  │
│  Brief description or summary... │
│                                  │
│  [Open] [Country Intel] [Share]  │
└─────────────────────────────────┘
```

---

## 7. State Management

### 7.1 AppContext

All runtime state lives in `AppContext` (`src/app/app-context.ts`):

```typescript
interface AppContext {
  // UI references
  map: MapContainer | null;
  panels: Record<string, Panel>;
  newsPanels: Record<string, NewsPanel>;
  signalModal: SignalModal | null;
  statusPanel: StatusPanel | null;
  searchModal: SearchModal | null;
  breakingBanner: BreakingNewsBanner | null;
  playbackControl: PlaybackControl | null;
  unifiedSettings: UnifiedSettings | null;

  // Data state
  allNews: NewsItem[];
  newsByCategory: Record<string, NewsItem[]>;
  latestMarkets: MarketData[];
  latestPredictions: PredictionMarket[];
  latestClusters: ClusteredEvent[];
  intelligenceCache: IntelligenceCache;
  cyberThreatsCache: CyberThreat[] | null;

  // User state
  mapLayers: MapLayers;
  panelSettings: Record<string, PanelConfig>;
  disabledSources: Set<string>;
  currentTimeRange: TimeRange;
  monitors: Monitor[];
  seenGeoAlerts: Set<string>;

  // Flags
  isMobile: boolean;
  isDesktopApp: boolean;
  isPlaybackMode: boolean;
  isIdle: boolean;
  initialLoadComplete: boolean;
  resolvedLocation: 'global' | 'america' | 'mena' | 'eu' | 'asia' | 'latam' | 'africa' | 'oceania';
}
```

### 7.2 Data Flow Pattern

```
External Source
  → DataLoaderManager fetches via service module
    → Service normalizes data to typed arrays
      → DataLoaderManager updates AppContext fields
        → Panels read from AppContext and re-render
```

There is no reactive binding system — panels are imperatively updated when data changes.

### 7.3 Persistence Layer

| Storage | Data | Module |
|---------|------|--------|
| `localStorage` | Panel order, spans, theme, AI settings, map layers, monitors | `settings-persistence.ts` |
| `IndexedDB` | RSS feed cache, vector store (RAG), API response cache | `persistent-cache.ts` |
| URL params | Map view, zoom, layers, country, expanded panel | `urlState.ts` |

### 7.4 Cross-Component Communication

Components communicate through:

1. **Direct method calls**: Managers hold references to panels and call `update()` directly
2. **Custom DOM events**: `window.dispatchEvent(new CustomEvent('ai-flow-changed', { detail }))` for settings changes
3. **AppContext mutations**: Shared state modified by managers, read by panels
4. **Callback registration**: `subscribeAiFlowChange(cb)` pattern for settings

---

## 8. Data Fetching & Loading

### 8.1 DataLoaderManager

`src/app/data-loader.ts` orchestrates all data fetching:

```typescript
class DataLoaderManager {
  async loadAllData(): Promise<void> {
    await Promise.allSettled([
      this.loadNewsFeedsByCategory(),
      this.loadMarketData(),
      this.loadMilitaryData(),
      this.loadConflictData(),
      this.loadInfrastructureData(),
      this.loadCyberThreats(),
      this.loadPredictions(),
      // ... 20+ parallel fetch groups
    ]);
  }
}
```

### 8.2 Bootstrap Data

For fast initial paint, the bootstrap endpoint returns pre-cached data from Redis:

```
/api/bootstrap?tier=fast  → earthquakes, outages, markets, crypto, military, GPS jam
/api/bootstrap?tier=slow  → conflict, trade, climate, displacement, intel briefs
```

This avoids cold-starting 20+ individual edge functions on first load.

### 8.3 Service Modules

Each data domain has a service module under `src/services/`:

```
src/services/
├── aviation/index.ts       # AviationStack, ICAO
├── climate/index.ts        # Open-Meteo
├── conflict/index.ts       # ACLED, UCDP
├── cyber/index.ts          # Feodo, URLhaus, OTX, AbuseIPDB
├── displacement/index.ts   # UNHCR, WorldPop
├── economic/index.ts       # FRED, World Bank, BIS
├── giving/index.ts         # Giving data
├── infrastructure/index.ts # Cloudflare, status pages
├── intelligence/index.ts   # GDELT, risk scores, PizzINT
├── market/index.ts         # Yahoo, Finnhub, CoinGecko
├── maritime/index.ts       # AISStream
├── military/index.ts       # OpenSky, WingBits, USNI
├── news/index.ts           # RSS, feed digest
├── prediction/index.ts     # Polymarket
├── research/index.ts       # arXiv, GitHub, HN
├── supply-chain/index.ts   # Shipping rates, chokepoints
├── trade/index.ts          # WTO, tariffs
├── unrest/index.ts         # GDELT unrest
└── wildfires/index.ts      # NASA FIRMS
```

### 8.4 Generated RPC Clients

Each service uses auto-generated TypeScript clients from Protobuf:

```typescript
import { MarketServiceClient } from '@/generated/client/worldmonitor/market/v1/service_client';

const client = new MarketServiceClient('', { fetch: globalThis.fetch });
const response = await client.listMarketQuotes({ symbols: ['SPY', 'QQQ', 'DIA'] });
```

### 8.5 Refresh Scheduler

`RefreshScheduler` (`src/app/refresh-scheduler.ts`) runs periodic data refreshes:

```typescript
const REFRESH_INTERVALS = {
  markets:       60_000,   // 1 min
  news:          300_000,  // 5 min
  military:      120_000,  // 2 min
  infrastructure: 300_000, // 5 min
  conflict:      900_000,  // 15 min
  intelligence:  600_000,  // 10 min
  // ...
};
```

The scheduler respects idle detection — refresh rates slow when the user is inactive.

---

## 9. Styling Architecture

### 9.1 CSS Organization

```
src/styles/
├── main.css              # Primary styles (~13k lines)
├── base-layer.css        # CSS layers for specificity control
├── panels.css            # Panel-specific styles
├── country-deep-dive.css # Country intel panel
├── happy-theme.css       # Happy variant theme
├── rtl-overrides.css     # Right-to-left language overrides
├── settings-window.css   # Settings window styles
└── (critical CSS)        # Inlined in index.html <head>
```

### 9.2 Theme System

Two themes (`light` and `dark`) controlled via `data-theme` attribute on `<html>`:

```css
[data-theme="dark"] {
  --bg-primary: #0a0a0f;
  --bg-secondary: #141420;
  --text-primary: #e0e0e8;
  --accent: #4a9eff;
  /* ... 50+ CSS custom properties */
}

[data-theme="light"] {
  --bg-primary: #ffffff;
  --bg-secondary: #f5f5f7;
  --text-primary: #1a1a2e;
  --accent: #2563eb;
  /* ... */
}
```

Theme is detected from system preference and persisted in `localStorage`. An inline `<script>` in `<head>` applies the theme before CSS loads to prevent FOUC.

### 9.3 Variant Themes

Variant-specific styles are applied via `data-variant` attribute:

```css
[data-variant="happy"] {
  --bg-primary: #fefce8;
  --accent: #22c55e;
  font-family: 'Nunito', sans-serif;
}
```

### 9.4 No Tailwind / No CSS-in-JS

The project uses plain CSS with BEM-like class naming:

```css
.panel { }
.panel__header { }
.panel__body { }
.panel--collapsed { }
.panel--loading { }
```

### 9.5 Responsive Breakpoints

```css
/* Mobile */
@media (max-width: 767px) { }

/* Tablet */
@media (min-width: 768px) and (max-width: 1199px) { }

/* Desktop */
@media (min-width: 1200px) { }

/* Ultra-wide */
@media (min-width: 1600px) { }

/* 4K */
@media (min-width: 2560px) { }
```

---

## 10. Variant System

### 10.1 Configuration

`src/config/variant.ts` exports `SITE_VARIANT` from `import.meta.env.VITE_VARIANT`:

```typescript
export const SITE_VARIANT: 'full' | 'tech' | 'finance' | 'happy' = import.meta.env.VITE_VARIANT || 'full';
```

### 10.2 Variant-Specific Behavior

| Aspect | Full | Tech | Finance | Happy |
|--------|------|------|---------|-------|
| **Panels** | All 30+ | Tech-focused 8 | Finance-focused 8 | Positive 8 |
| **Feeds** | All categories | tech, ai | finance, energy | positive |
| **Map layers** | All | Tech-relevant | Finance-relevant | Positive events |
| **Theme** | Dark default | Dark default | Dark default | Light/warm, Nunito font |
| **Header** | Standard | Standard | Standard | Festive, TV mode |
| **Special features** | — | — | — | Confetti (canvas-confetti) |

### 10.3 Build Variants

Vite config uses `VITE_VARIANT` to:
- Set feature flags for conditional imports
- Select feed categories
- Configure panel visibility
- Apply variant-specific CSS

```bash
VITE_VARIANT=full npm run build      # Full variant
VITE_VARIANT=tech npm run build      # Tech variant
VITE_VARIANT=finance npm run build   # Finance variant
VITE_VARIANT=happy npm run build     # Happy variant
```

---

## 11. Navigation & Routing

### 11.1 URL State

The URL encodes the current view state (`src/utils/urlState.ts`):

| Parameter | Purpose | Example |
|-----------|---------|---------|
| `view` | Map view type | `flat`, `globe` |
| `zoom` | Map zoom level | `4` |
| `lat` / `lon` | Map center | `48.8566,2.3522` |
| `timeRange` | Time filter | `24h`, `7d`, `30d` |
| `layers` | Active map layers | `earthquakes,military,fires` |
| `country` / `c` | Selected country | `US`, `UA` |
| `expanded` | Expanded panel | `market` |
| `t` | Country intel type | `brief`, `story` |

### 11.2 Region Selector

The header region selector switches between geo-focused views:

| Region | Content Focus |
|--------|--------------|
| Global | Worldwide events |
| Americas | US, Canada, LATAM |
| MENA | Middle East & North Africa |
| EU | Europe |
| Asia | Asia-Pacific |
| LATAM | Latin America |
| Africa | Africa |
| Oceania | Australia, Pacific |

Region selection filters:
- RSS feed sources by region relevance
- Map initial viewport and zoom
- Panel content emphasis
- Saved via `resolvedLocation` in AppContext

### 11.3 Deep Links

Direct links to country intelligence:
- `/?c=UA` → Open country deep dive for Ukraine
- `/?c=UA&t=story` → Open country story for Ukraine
- `/api/story?c=UA` → Server-rendered story (for crawlers/social sharing)

---

## 12. Country Deep Dive

The country deep dive panel (`CountryDeepDivePanel`, `CountryBriefPanel`, `CountryTimeline`) provides comprehensive country-level intelligence.

### 12.1 Signal Aggregation

`CountryBriefSignals` aggregates all data sources for a country:

```typescript
interface CountryBriefSignals {
  criticalNews: number;
  protests: number;
  militaryFlights: number;
  militaryVessels: number;
  outages: number;
  aisDisruptions: number;
  satelliteFires: number;
  temporalAnomalies: number;
  cyberThreats: number;
  earthquakes: number;
  displacementOutflow: number;
  climateStress: number;
  conflictEvents: number;
  activeStrikes: number;
  orefSirens: number;
  aviationDisruptions: number;
  travelAdvisories: number;
  gpsJammingHexes: number;
  isTier1: boolean;
}
```

### 12.2 Country Intel Flow

```
User clicks country on map or selects from search
  → CountryIntelManager.openCountry(code)
    → Fetch /api/intelligence/v1/get-country-intel-brief
    → Aggregate signals from AppContext caches
    → Render CountryBriefPanel with:
      ├── Risk score and trend
      ├── Signal summary (traffic-light indicators)
      ├── Recent news filtered by country
      ├── Active conflicts/protests
      ├── Military activity
      ├── Infrastructure status
      ├── Economic indicators
      └── Timeline of events
```

---

## 13. Modals & Overlays

### 13.1 SignalModal

Full-screen detail view for any intelligence signal:
- Article content with AI summary
- Source verification badges
- Related signals
- Share/export options

### 13.2 SearchModal

Global search across all data:
- Full-text search across news, events, countries
- Semantic search using ML embeddings (when Headline Memory is enabled)
- Results grouped by category

### 13.3 Breaking News Banner

A sliding banner at the top for critical breaking events:
- Auto-detected from high-tier sources
- Keyword matching for crisis terms
- Configurable via AI flow settings (`mapNewsFlash`)

---

## 14. Search System

`SearchManager` (`src/app/search-manager.ts`) provides:

1. **Full-text search**: Filters `allNews`, markets, events by keyword
2. **Semantic search**: When Headline Memory is enabled, uses ML embeddings for meaning-based search
3. **Country search**: Auto-complete for country names
4. **Results ranking**: Combines text relevance, source tier, and recency

---

## 15. AI Integration in UI

### 15.1 Article Summarization

When a user expands a news article, the UI requests an AI summary:

```
User clicks "Summarize" on article
  → summarizeArticle(headlines, geoContext, lang)
    → Try cloud providers (Ollama → Groq → OpenRouter)
      → Fall back to browser T5 (Web Worker)
        → Display summary with provider badge
```

### 15.2 Headline Memory (RAG)

When enabled (`headlineMemory: true`), the system builds a local knowledge base:
- New headlines are embedded in the ML worker
- Vectors stored in IndexedDB (5,000 max)
- Semantic search recalls related past headlines
- Context injected into summarization prompts

### 15.3 Intelligence Badges

`IntelligenceGapBadge` shows animated indicators when new intelligence is detected:
- New cyber threats
- New conflict events
- Infrastructure anomalies
- Military activity changes

### 15.4 Situation Deduction

`DeductionPanel` uses AI to analyze the current intelligence picture:
- Aggregates signals from multiple domains
- Sends to `intelligence.v1.DeductSituation` RPC
- Renders structured assessment with confidence levels

---

## 16. Responsive Design

### 16.1 Mobile Layout

```
┌────────────────────┐
│  HEADER (compact)  │
│  [☰] [Logo] [🔍]  │
├────────────────────┤
│                    │
│    MAP (SVG/D3)    │
│    Full-width      │
│                    │
├────────────────────┤
│                    │
│  PANELS (stacked)  │
│  Single column     │
│  Swipe navigation  │
│                    │
├────────────────────┤
│  MOBILE NAV BAR    │
│  [Map][News][Intel]│
└────────────────────┘
```

### 16.2 Device Detection

```typescript
const isMobile = window.matchMedia('(max-width: 767px)').matches
                 || /Android|iPhone|iPad/i.test(navigator.userAgent);
```

Mobile-specific adaptations:
- SVG map instead of WebGL (better battery, broader support)
- Simplified panel rendering (fewer DOM nodes)
- Bottom sheet navigation instead of sidebar
- Touch-optimized interactions (larger tap targets)
- Reduced refresh frequency

---

## 17. Accessibility & i18n

### 17.1 Internationalization

- **21 languages** supported via i18next
- Locale files lazy-loaded as Vite chunks
- RTL languages (Arabic, Hebrew, Farsi, Urdu) with `rtl-overrides.css`
- Date/number formatting via `Intl` APIs
- AI summarization respects current language

### 17.2 RTL Support

```css
[dir="rtl"] .panels-grid { direction: rtl; }
[dir="rtl"] .panel__header { flex-direction: row-reverse; }
[dir="rtl"] .map-controls { left: 12px; right: auto; }
```

---

## 18. Performance Patterns

### 18.1 Code Splitting

Vite config defines manual chunks for large libraries:

```typescript
manualChunks: {
  'maplibre': ['maplibre-gl'],
  'deck-stack': ['@deck.gl/core', '@deck.gl/layers', '@deck.gl/geo-layers'],
  'transformers': ['@xenova/transformers'],
  'onnxruntime': ['onnxruntime-web'],
  // + locale chunks
}
```

### 18.2 Virtual Scrolling

Long lists (news feeds, event lists) use `VirtualList`:
- Only renders visible items + buffer
- Recycles DOM elements on scroll
- Handles variable item heights

### 18.3 Web Workers

Heavy computation runs off the main thread:

| Worker | Task |
|--------|------|
| `ml.worker.ts` | ONNX inference (embeddings, summarization, NER, sentiment) |
| `vector-db.ts` | IndexedDB vector operations (cosine similarity search) |
| `analysis.worker.ts` | Data analysis and aggregation |

### 18.4 Lazy Loading

- Map backends loaded on demand (globe.gl only when selected)
- ML models downloaded on first use
- Locale files loaded per language
- Panel content loaded when panel is first expanded

### 18.5 Critical CSS

The `<head>` of `index.html` includes inlined critical CSS for:
- Theme application (prevent FOUC)
- Layout skeleton
- Loading indicators

---

## 19. Component Catalog

### Core Layout

| Component | File | Purpose |
|-----------|------|---------|
| `App` | `src/App.ts` | Root application controller |
| `PanelLayoutManager` | `src/app/panel-layout.ts` | Layout orchestration |
| `Panel` | `src/components/Panel.ts` | Base panel class |
| `VirtualList` | `src/components/VirtualList.ts` | Virtual scrolling |
| `UnifiedSettings` | `src/components/UnifiedSettings.ts` | Settings UI |

### Map

| Component | File | Purpose |
|-----------|------|---------|
| `MapContainer` | `src/components/MapContainer.ts` | Map backend selector |
| `DeckGLMap` | `src/components/DeckGLMap.ts` | Deck.gl + MapLibre |
| `GlobeMap` | `src/components/GlobeMap.ts` | globe.gl 3D globe |
| `Map` | `src/components/Map.ts` | D3 SVG fallback |
| `MapPopup` | `src/components/MapPopup.ts` | Feature detail popup |

### Modals

| Component | File | Purpose |
|-----------|------|---------|
| `SignalModal` | `src/components/SignalModal.ts` | Article detail view |
| `SearchModal` | `src/components/SearchModal.ts` | Global search |
| `StoryModal` | `src/components/StoryModal.ts` | Country story |
| `MobileWarningModal` | `src/components/MobileWarningModal.ts` | Mobile experience warning |
| `CountryIntelModal` | `src/components/CountryIntelModal.ts` | Country selection |

### Controls

| Component | File | Purpose |
|-----------|------|---------|
| `PlaybackControl` | `src/components/PlaybackControl.ts` | Time playback bar |
| `StatusPanel` | `src/components/StatusPanel.ts` | Connection/health |
| `DownloadBanner` | `src/components/DownloadBanner.ts` | Desktop download prompt |
| `BreakingNewsBanner` | `src/components/BreakingNewsBanner.ts` | Breaking news strip |
| `PizzIntIndicator` | `src/components/PizzIntIndicator.ts` | Pentagon activity |

---

## 20. Building New Panels

### Step-by-Step Guide

1. **Create the panel component**:

```typescript
// src/components/MyNewPanel.ts
import { Panel } from './Panel';
import { h, replaceChildren } from '@/utils/dom-utils';
import type { AppContext } from '@/app/app-context';

export class MyNewPanel extends Panel {
  constructor(container: HTMLElement, ctx: AppContext) {
    super(container, ctx, {
      id: 'my-new-panel',
      title: 'My New Panel',
      icon: '📊',
      defaultSpan: 1,
      collapsible: true,
    });
  }

  update(data: MyDataType[]): void {
    const content = data.map(item =>
      h('div', { className: 'item' },
        h('span', { className: 'item-title' }, item.title),
        h('span', { className: 'item-value' }, item.value),
      )
    );
    replaceChildren(this.body, ...content);
  }
}
```

2. **Add styles** in `src/styles/panels.css`:

```css
#my-new-panel .item {
  display: flex;
  justify-content: space-between;
  padding: 8px 0;
  border-bottom: 1px solid var(--border);
}
```

3. **Register in PanelLayoutManager** (`src/app/panel-layout.ts`):
   - Add to the variant's panel list
   - Set default position in panel order

4. **Wire data loading** in `DataLoaderManager` (`src/app/data-loader.ts`):
   - Add fetch call in `loadAllData()`
   - Update AppContext with fetched data
   - Call `panel.update(data)` after fetch

5. **Add to refresh cycle** in `RefreshScheduler`:
   - Set refresh interval based on data freshness needs

6. **Add map layer** (if geospatial):
   - Add layer in `DeckGLMap.ts` layer stack
   - Add toggle in map layer control
   - Add to `MapLayers` type

7. **Test**:
   - Visual regression test in `e2e/`
   - Data validation test in `tests/`
