# World Monitor + Hackerdogs: Integration Review & Fork Strategy

This document reviews how the **external** [World Monitor](https://github.com/koala73/worldmonitor) project (koala73/worldmonitor) could use **Hackerdogs API and MCP** as data sources, and how to build a **Hackerdogs-specific frontend** by forking World Monitor. The dashboard is **use-case-centric**: end users operate on **use cases** (e.g. “External domain inventory”, “Daily threat brief”); each use case has a **panel** that surfaces continuous information; **schedules and conversations** work behind the scenes.

**Note:** There is no `worldmonitor` folder in the hackerdogs-core repository. World Monitor is a separate open-source project (AGPL v3, ~30k stars) hosted at [worldmonitor.app](https://worldmonitor.app) with variants: Tech Monitor, Finance Monitor, Happy Monitor.

---

## 1. World Monitor at a Glance

| Aspect | Details |
|--------|---------|
| **Stack** | TypeScript, Vite, React-style SPA; `api/`, `server/`, `convex/`, `proto/`, `shared/`; Tauri for desktop; PWA |
| **Data sources** | GDELT, ACLED, UCDP, RSS (170+ feeds), ADS-B, AIS, NASA FIRMS, Cloudflare Radar, Polymarket, Yahoo Finance, etc. |
| **Backend** | 22 typed proto services, Redis cache, Convex, optional Node sidecar (60+ API handlers) |
| **UI** | Resizable panels, dual map engine (3D globe + deck.gl flat), 45+ map layers, CII country risk heatmap, AI briefs, live video |
| **Variants** | Single codebase; switch via header (World / Tech / Finance / Happy) |

World Monitor already has a **Tech Monitor** variant focused on startups, AI/ML, cloud, and **cybersecurity** — the natural place to plug in Hackerdogs as an additional intelligence source.

---

## 2. Use-Case-Centric Dashboard Model

### 2.1 Principle: Users Operate on Use Cases, Not Schedules

- **End user sees:** Use cases (e.g. “External domain inventory”, “Daily threat brief”, “Attack path report”). Each use case has a **panel** that shows continuous, up-to-date information (last run, latest deliverables, evidence links).
- **Behind the scenes:** Each use case is implemented by one or more **schedules** and **conversations**. The dashboard:
  - Lists use cases from a **central repository** (remote JSON, editable in settings).
  - For each use case, displays the **latest conversation** (or a chosen run) and its deliverables/evidence.
  - Allows “Run now” and “Schedule” (daily/weekly) at the **use-case** level; the app translates these into `schedule_now` / `create_schedule` / `run_schedule_now` and tracks which conversation belongs to which use case.

So: **schedules and conversations are implementation details**; the **use case** is the first-class entity in the UI.

### 2.2 Central Repository of Use Cases and Prompts

A **single JSON file** defines all use cases and their step-by-step execution (prompts and optional schedule bindings). This file is:

- **Loaded remotely** (URL configured in dashboard settings, e.g. `https://app.hackerdogs.ai/use-cases/registry.json` or a CDN).
- **Editable in settings** (admin can paste a different URL or edit a copy for their tenant).
- **Schema:** Each use case has an id, name, description, and an ordered list of **steps**. Each step can be a prompt template (for `schedule_now` or for creating a schedule) and optional schedule_type/start_at for recurring use cases.

**Example JSON schema (use case registry):**

```json
{
  "version": "1.0",
  "use_cases": [
    {
      "id": "external-domain-inventory",
      "name": "External Domain Inventory",
      "description": "Root domains and brand variants with ownership signals.",
      "stage": "Stage 1",
      "steps": [
        {
          "order": 1,
          "prompt_template": "Gather known root domains for {{target}} and expand using brand, product, and geo variations. Verify ownership signals (registrant patterns, DNS control). Produce a Root Domain Catalog with confidence and business mapping.",
          "run_on_schedule": false
        }
      ],
      "default_schedule_type": "once",
      "panel_icon": "globe"
    },
    {
      "id": "daily-threat-brief",
      "name": "Daily Threat Brief",
      "description": "Top cybersecurity incidents and CVEs in the last 24 hours.",
      "stage": "Stage 3",
      "steps": [
        {
          "order": 1,
          "prompt_template": "Generate a daily cybersecurity threat brief: top incidents in the last 24 hours, new CVEs affecting our tech stack, and relevant threat actor activity. Format as an executive summary.",
          "run_on_schedule": true,
          "schedule_type": "daily"
        }
      ],
      "default_schedule_type": "daily",
      "panel_icon": "briefing"
    },
    {
      "id": "subdomain-discovery",
      "name": "Subdomain Discovery",
      "description": "Passive DNS and cert-based subdomain discovery for a target.",
      "stage": "Stage 1",
      "steps": [
        {
          "order": 1,
          "prompt_template": "Run passive DNS enumeration and certificate transparency mining for {{target}}. Discover subdomains and categorize prod/stage/dev and high-risk (admin, vpn, sso). Produce a Subdomain Inventory with tags and risk category.",
          "run_on_schedule": false
        }
      ],
      "default_schedule_type": "once",
      "panel_icon": "search"
    }
  ]
}
```

- **Panel behavior:** For each use case, the dashboard shows one panel. The panel displays:
  - Title, description, “Run now”, “Schedule” (if supported).
  - **Latest run:** conversation_id, last run time, and a link to open the full conversation (deliverables, evidence).
  - Optional: list of recent conversations for this use case (from `list_conversations` with search or tags if available).
- **Execution:** When the user clicks “Run now” for a use case that has multiple steps, the client can run step 1 via `schedule_now`, then (after completion) run step 2 with `conversation_id` from step 1, etc. For single-step use cases, one `schedule_now` suffices. Recurring use cases create a `create_schedule` with the prompt template and desired `schedule_type`/`start_at`.

---

## 3. ATEM Use Cases (Extracted from ATEM Paper)

The following use cases are extracted from [ATEM Paper.md](ATEM%20Paper.md). They can populate the **use case registry** and drive panels in the Hackerdogs World Monitor fork.

### Stage 1: External Attack Surface Management (OSINT 360)

| Use case | Description | Deliverable |
|----------|-------------|-------------|
| Define scope and ownership rules | Business entities, owned vs related vs untrusted, confidence rules | Asset Ownership Policy + confidence scoring model |
| Domain inventory | Root domains + brand/product/geo variations, typo-squat candidates, ownership verification | Root Domain Catalog |
| Subdomain discovery | Passive DNS, brute/permutation, cloud-generated names, categorization (prod/stage/dev, high-risk) | Subdomain Inventory with tags + risk category |
| Certificate & TLS intelligence | Pull certs, SANs, issuers, expiry; use cert names to find new subdomains | Certificate Graph + “new asset via certs” feed |
| Resolve and map to infrastructure | Resolve A/AAAA/CNAME, map to CDN/cloud/ASN, identify clusters | External Infrastructure Map |
| Service discovery | Exposed services, ports, protocols; fingerprint tech stack; exposure records | Exposure Inventory |
| Web & app surface mapping | Crawl apps, endpoints, API schemas, leaked secrets in JS, identity entry points | Application Surface Map + Identity Entry Points |
| External identity & reputation | Login portals, email patterns, brand impersonation, SPF/DKIM/DMARC | External Identity & Brand Risk summary |
| External leak & breach adjacency | Credential dumps, exposed buckets, paste/code mentions; correlate to assets | Leak-to-Asset Correlation Report |
| Threat-Exposure Queue | Convert findings to work items; score by exploitability, exposure, criticality; fix paths | Ranked Exposure Backlog + remediation playbooks |
| Continuous exposure management | Track work items, validate fixes, detect drift, executive brief + practitioner queue | Management (not just discovery) |
| AI-driven external steps (1–20) | Entity/brand modeling, root domain expansion, passive DNS, cert mining, resolution, cloud edge, service discovery, tech fingerprinting, API/secret discovery, identity surface, email validation, impersonation, leak/dark web monitoring, risk scoring, work item generation, remediation validation, drift monitoring | Full external ASM maturity |

### Stage 2: Internal Attack Surface Management (Threat 360)

| Use case | Description | Deliverable |
|----------|-------------|-------------|
| Normalize identity and asset sources | Ingest asset lists; unified asset identity; unified identity model; business criticality tags | Single Source of Truth Graph (assets + identities) |
| Internal attack surface inventory | Endpoints, servers, VMs, containers; shadow assets; map apps to runtime | Internal Asset Graph + shadow asset list |
| Internal exposure heatmap | Admin tools, internal dashboards, lateral “highways”, ingress points | Internal exposure heatmap + ingress catalog |
| Vulnerability and weakness mapping | Vulns + exploitability + configuration weaknesses | Unified Weakness Backlog |
| Privilege and identity path modeling | Privilege graph, high-risk patterns, identities → crown jewels | Privilege Paths + Crown Jewel Access Map |
| Attack path modeling | Crown jewels; compute paths (ingress → foothold → lateral → jewel); path breakers | Attack Path Report + Top Path Breakers |
| Closure and control validation | Verify fixes, monitor drift | Verified Remediation Ledger + Drift Alerts |
| AI-driven internal steps (1–17) | Asset aggregation, unified resolution, cloud/K8s inventory, vuln ingestion, misconfig, identity graph, privilege escalation, internal exposure, secrets risk, crown jewel tagging, attack path modeling, path-breaker optimization, lateral simulation, remediation validation, identity hygiene, drift detection | Full Threat 360 maturity |

### Stage 3: Autonomous Maturity

| Use case | Description | Deliverable |
|----------|-------------|-------------|
| Always-on discovery and enrichment | Continuous external + internal sync; automatic enrichment | “What changed since yesterday?” with proof |
| Autonomous risk triage and work creation | Turn signals into work items; auto-route to owners | Live, ranked execution queue |
| Autonomous verification | Re-test after fixes; validate config; recompute paths; close only when validated | “Closed with evidence” ledger |
| Autonomous drift detection | Detect re-opened ports, new subdomains, new roles; baseline policies; meaningful deltas only | Drift alerts + recommended fix |
| Autonomous threat-driven campaigns | Themed campaigns (e.g. ransomware readiness); measure outcomes | Campaign briefs + measurable reduction |
| Autonomous reporting | Board/Exec, CISO, practitioners, partners/MSSPs | Weekly intelligence briefs + daily ops queues |
| Full autonomous operation (1–14) | Continuous sync, graph updates, risk re-scoring, work creation, remediation suggestions, validation loop, drift guardrails, campaign automation, executive brief generation, predictive modeling, noise suppression, self-healing playbooks, AI audit trail, confidence calibration | Autonomous control |

### Customer Maturity Levels (Levels 0–5)

- **Level 0 — Reactive & Blind:** No complete inventory; no identity graph; siloed vulns; no attack path visibility. *Use case: Establish ground truth.*
- **Level 1 — External Visibility (Surface Awareness):** Verified domain/subdomain inventory, infrastructure map, service exposure, web/API map, identity portals, typosquat/impersonation, secrets/leak monitoring, ranked backlog, re-testing, drift detection. *Use cases: All Stage 1 use cases.*
- **Level 2 — Internal Exposure Control (Threat 360):** Unified asset/identity graph, cloud/container visibility, vuln ingestion, config risk, privilege paths, crown jewels, attack path modeling, path breakers, remediation validation, identity hygiene, drift. *Use cases: All Stage 2 use cases.*
- **Level 3 — Risk-Based Execution:** Risk scoring (E/R/I/path criticality), ticket routing, ownership, SLA, campaign-based reduction, verified ledger, regression detection. *Use cases: Risk and triage use cases.*
- **Level 4 — Autonomous Exposure Reduction:** Continuous discovery, graph updates, risk re-scoring, work creation, fix validation, policy enforcement, predictive modeling, noise suppression, self-healing, executive briefs, audit logs, confidence calibration. *Use cases: All Stage 3 autonomous use cases.*
- **Level 5 — Strategic Exposure Governance:** Adversary simulation, breach likelihood modeling, insurance-grade scoring, business unit attribution, exposure benchmarking. *Use cases: Governance and strategic reporting.*

These ATEM use cases can be turned into **registry entries** with one or more prompt steps and optional `schedule_type` for recurring panels.

---

## 4. How Hackerdogs API & MCP Support the Dashboard

Hackerdogs exposes **scheduled prompts**, **conversations**, **briefings**, and **chat completions**. The dashboard uses them to back use-case panels and “Run now” / “Schedule” actions.

**Base URLs & authentication**

| API | Base path | Purpose |
|-----|-----------|---------|
| Scheduled Prompts | `https://app.hackerdogs.ai/scheduled-prompts-api` | Schedules, schedule-now, run-now, LLM options, schedule types |
| Conversations | `https://app.hackerdogs.ai/conversation-api` | List/get/delete conversations (feed, full packet, execution_complete) |
| Briefings | `https://app.hackerdogs.ai/briefings-api` | List, search, and find similar intelligence briefings |
| Chat Completions | `https://app.hackerdogs.ai/v1` | Streaming/non-streaming chat with tools |

All endpoints require an API key: `X-API-Key: hd-...` or `Authorization: Bearer hd-...`. The MCP server is at `https://app.hackerdogs.ai/mcp/` with the same Bearer token in headers.

### 4.1 Scheduled Prompts API (REST) and MCP

| Capability | REST API | MCP | Use in use-case panel |
|------------|----------|-----|------------------------|
| List schedules | `GET /scheduled-prompts-api/schedules` | `list_schedules` | List schedules that back a use case (e.g. filter by search or by convention). |
| Get one schedule | `GET /scheduled-prompts-api/schedules/{id}` | `get_schedule` | Show next run, prompt snippet, linked `conversation_id`. |
| Create schedule | `POST /scheduled-prompts-api/schedules` | `create_schedule` | Create recurring run for a use case (e.g. daily threat brief). |
| Update schedule | `PATCH /scheduled-prompts-api/schedules/{id}` | `update_schedule` | Edit prompt, `schedule_type`, `start_at`, `conversation_id`, etc. |
| Delete schedule | `DELETE /scheduled-prompts-api/schedules/{id}` | `delete_schedule` | Remove schedule for a use case. |
| Schedule and run now | `POST /scheduled-prompts-api/schedule-now` | `schedule_now` | “Run now” for a use case (prompt from registry; optional `conversation_id` to continue). |
| Run existing schedule now | `POST /scheduled-prompts-api/schedules/{id}/run-now` | `run_schedule_now` | “Run again” for an existing schedule. |
| LLM options | `GET /scheduled-prompts-api/llm-options` | `get_llm_options` | Populate model picker when configuring run. |
| Schedule types | `GET /scheduled-prompts-api/schedule-types` | `get_schedule_types` | Validate `schedule_type` (once/daily/weekly) for plan. |

**Request body (relevant fields):**

- **POST /schedule-now:** `prompt` (or `llm_prompt_template`), `schedule_type`, `llm_config`, `system_prompt`, `conversation_id`, `tool_config`, `use_agentic_mode`.
- **POST /schedules:** `llm_prompt_template`, `schedule_type`, `start_at` (ISO 8601), `llm_config`, `system_prompt`, `conversation_id`, `tool_config`, `use_agentic_mode`.
- **PATCH /schedules/{id}:** Any of the above; `conversation_id` is updatable via REST.

All require **API key** (`X-API-Key` or `Authorization: Bearer hd-...`).

### 4.2 Conversation API (REST) and MCP

| Capability | REST API | MCP | Use in use-case panel |
|------------|----------|-----|------------------------|
| List conversations | `GET /conversation-api/conversations` | `list_conversations` | Show recent runs for a use case (search by session name or content; optional `from_date`/`to_date`). |
| Get conversation | `GET /conversation-api/conversations/{id}` | `get_conversation` | Fetch deliverables, evidence, files, and **execution_complete** for the panel. |
| Delete conversation | `DELETE /conversation-api/conversations/{id}` | `delete_conversation` | Remove a conversation from the UI. |

**GET /conversations/{id}** response includes: `conversation_id`, `user_id`, `tenant_id`, `files`, `deliverable_groups`, `evidence_items`, `evidence_files`, and **`execution_complete`** (boolean). Use `execution_complete` to know when the run has finished (see stability note below).

**GET /conversations** (list) query params: `limit` (1–250, default 250), `offset`, `search` (ILIKE session name and message content), `from_date`, `to_date` (ISO 8601). Returns conversation_id, session_name, message_count, latest_message_time, and up to 5 message previews per conversation.

**Execution complete (stability):** A conversation returns 200 as soon as the first file exists; deliverables and evidence may still be writing. Whether execution is complete is determined by the same check as the Intelligence Briefings expander in HDChatStreaming: the presence of the txtanalytics completed marker in storage (`chat/{user_id}/{conversation_id}/analytics/{message_id}/completed.txt`). The Conversation API and MCP both expose this as `execution_complete` in the GET conversation response. Clients should poll until `execution_complete` is true before treating the run as complete (rather than relying only on stable deliverable/evidence counts). The Python clients (`schedule_now_and_wait`) can use this field for consistent behavior with the UI.

### 4.3 Briefings API (REST) and MCP

| Capability | REST API | MCP | Use in use-case panel |
|------------|----------|-----|------------------------|
| List briefings | `GET /briefings-api/briefings` | `list_briefings` | Show latest intelligence briefs (e.g. in a “Briefings” panel). |
| Search briefings | `GET /briefings-api/briefings/search` | `search_briefings` | Keyword, vector, or hybrid search; filter by category/classification. |
| Similar briefings | `GET /briefings-api/briefings/similar` | `get_similar_briefings` | Find briefings similar to a query, message_id, or brief_id. |

**Query params:** For search: `query` (required), `query_type` (keyword_search | vector_search | hybrid_search), `limit`, `offset`, `alpha` (hybrid), `category`, `classification`. For similar: at least one of `query`, `message_id`, `brief_id`; plus `query_type`, `limit`, `offset`, `alpha`.

### 4.5 Polling for completion and conversation chaining

- **Polling:** After `schedule_now` or `run_schedule_now`, poll **GET /conversation-api/conversations/{id}** (or MCP `get_conversation`) until the response has **`execution_complete: true`**. Do not rely only on HTTP 200 or on stable file/deliverable counts; the `execution_complete` field is the authoritative signal (same as the Intelligence Briefings expander in the Hackerdogs UI).
- **Conversation chaining:** To run multi-step use cases (e.g. recon → deep scan → report), pass the **`conversation_id`** from the first run into the next **POST /schedule-now** (or MCP `schedule_now`) as `conversation_id`. Each step enriches the same conversation; then call **GET /conversations/{id}** once `execution_complete` is true to retrieve the full packet.

### 4.6 Chat Completions API (REST only)

- **POST /v1/chat/completions:** Streaming or non-streaming chat with optional tools. Use for a live “Ask Hackerdogs” panel or ad-hoc analysis; not use-case steps per se.
- **GET /v1/models:** List models for the account.

---

## 5. Mapping Use Cases to Panels and Backing APIs

| Panel (use case) | Backing APIs | Notes |
|------------------|--------------|--------|
| Use case: “Daily Threat Brief” | Create one schedule (daily) from registry prompt; `list_schedules` + `get_schedule`; `get_conversation` for latest run | Panel shows last run time, link to conversation deliverables. |
| Use case: “External Domain Inventory” | `schedule_now` with prompt from registry (template with `{{target}}`); poll then `get_conversation` | Single or multi-step; optional “Continue” with same `conversation_id`. |
| Use case: “Subdomain Discovery” | Same pattern; prompt from registry | Can chain with “Certificate & TLS” in same conversation. |
| Recent conversations | `list_conversations` with optional `search`/date range | Show per use case if session_name or tags identify use case. |
| Briefings | `list_briefings`, `search_briefings`, `get_similar_briefings` | Separate panel or tab for intelligence briefs. |
| Live ask | `POST /v1/chat/completions` (streaming) | One panel for free-form chat with tools. |

---

## 6. Building the Hackerdogs Frontend (Fork Strategy)

### 6.1 Fork and Variant

1. Fork [koala73/worldmonitor](https://github.com/koala73/worldmonitor) (e.g. `hackerdogs/hackerdogs-monitor`).
2. Add a **“Hackerdogs”** variant; optionally make it the default. Reuse existing variant switcher and config.

### 6.2 Use Case Registry and Panels

1. **Load use case registry:** Fetch JSON from a configurable URL (settings). Parse `use_cases` and optional `version`.
2. **Render one panel per use case:** Panel shows name, description, “Run now”, “Schedule” (if `default_schedule_type` not `once`), and **latest run** (conversation_id, time, link to conversation view).
3. **Run now:** From registry step(s), build prompt (substitute `{{target}}` etc.); call `schedule_now` (and optionally chain steps with `conversation_id`). Poll for completion; then call `get_conversation` and show deliverables/evidence in the panel or a slide-out.
4. **Schedule:** Call `get_schedule_types`, then `create_schedule` with prompt and `schedule_type`/`start_at`. Optionally attach a “use case id” in session name or metadata so `list_conversations` can filter later.
5. **Settings:** Allow editing the **registry URL** and (if supported) an override JSON blob. Store Hackerdogs API key and base URL as today.

### 6.3 Data Layer: Hackerdogs Client

Implement a small REST client (or use existing Python clients from hackerdogs-core for a backend-for-frontend). Endpoints:

- Schedules: list, get, create, PATCH, delete, schedule-now, run-now.
- Conversations: list, get, delete.
- Briefings: list, search, similar.
- Optional: Chat completions (streaming) for “Ask Hackerdogs” panel.

MCP is for AI assistants (Cursor, Claude); the browser dashboard uses REST.

### 6.4 Reuse and Simplify

- Keep map, resizable panels, theme, Cmd+K. Add “Run Hackerdogs use case: &lt;name&gt;” to command palette.
- For Hackerdogs variant, reduce or hide non-Hackerdogs data sources; keep map for future geo layer (e.g. targets/findings from conversations).
- Licensing: AGPL v3; keep “Based on World Monitor” attribution.

### 6.5 Minimal First Milestone

1. Fork → add `hackerdogs` variant.
2. Implement Hackerdogs REST client (schedules, schedule-now, list/get conversation).
3. Load a **static use case registry** (embedded or single URL); render one panel per use case.
4. “Run now” from panel → `schedule_now` → poll → show conversation deliverables in panel.
5. Settings: API key, base URL, registry URL.

Then: recurring schedule creation from panel, briefings panel, list_conversations in panel, multi-step execution from registry.

---

## 7. Summary

| Question | Answer |
|----------|--------|
| **Where is worldmonitor?** | External project [koala73/worldmonitor](https://github.com/koala73/worldmonitor); not in hackerdogs-core. |
| **What do users see?** | **Use cases** (e.g. ATEM stages); each use case has a **panel** with continuous info; schedules/conversations are behind the scenes. |
| **Where are use cases defined?** | **Central JSON registry** (remote URL, editable in settings) with steps (prompt templates, optional schedule_type). |
| **How do panels get data?** | Schedules API (list, get, create, schedule-now, run-now), Conversation API (list, get), Briefings API (list, search, similar). |
| **Latest API surface?** | Schedules (list, get, create, PATCH, delete, schedule-now, run-now, llm-options, schedule-types). Conversations (list with search/date, get with **execution_complete**, delete). Briefings (list, search, similar). PATCH schedule supports `conversation_id`. GET conversation returns **execution_complete**; poll until true for completion. |

**Error handling (typical HTTP codes):** 401 invalid API key; 402 insufficient credits; 403 schedule type not allowed for plan; 404 conversation not found (e.g. still running); 422 validation error; 503 service unavailable. Handle gracefully so the dashboard can degrade when Hackerdogs is unavailable.

---

## References

- **[API_MCP_USER_GUIDE.md](../api/API_MCP_USER_GUIDE.md)** — Full REST and MCP reference: endpoints, request/response shapes, polling with `execution_complete`, briefings, and sample flows.
- **Conversation API:** [CONVERSATIONS_API.md](../conversation/CONVERSATIONS_API.md). **Briefings API:** [BRIEFINGS_API.md](../conversation/BRIEFINGS_API.md).
