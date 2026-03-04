# Hackerdogs API & MCP Server User Guide

This guide covers two ways to interact with Hackerdogs programmatically:

1. **MCP Server** -- Connect from Claude, Cursor, or any MCP-compatible client. The AI calls Hackerdogs tools directly.
2. **REST API** -- Call the Scheduled Prompts, Conversation, and Chat Completions endpoints over HTTP with an API key.

Both methods require a Hackerdogs API key.

---

## Prerequisites: Creating an API Key

1. Log in to Hackerdogs at [https://app.hackerdogs.ai](https://app.hackerdogs.ai).
2. Navigate to **API Keys** (sidebar or User Settings).
3. Click **Create API Key**, give it a name, and copy the key (format: `hd-...`).
4. Store it securely -- it is shown only once.

---

# Section 1: Using the Hackerdogs MCP Server

The MCP (Model Context Protocol) server lets AI assistants call Hackerdogs tools -- schedule prompts, retrieve conversations, manage schedules -- through a standardized protocol.

**MCP Server URL:** `https://app.hackerdogs.ai/mcp/`

## MCP Tools Reference

### Scheduled Prompts Tools

| Tool | Parameters | Purpose |
|------|-----------|---------|
| `get_llm_options` | `enabled_only` (bool, default `true`) | List LLM provider/model pairs available for scheduling. Call first when creating a schedule to discover valid models. Returns `items` (list of `{provider, model}`) and `default`. |
| `get_schedule_types` | *(none)* | List schedule types allowed by the user's subscription plan (e.g. `once`, `daily`, `weekly`). **Always call before creating/updating a schedule** -- the API returns 403 if the type is not allowed. |
| `list_schedules` | `search` (str, optional), `status_filter` (`all`/`active`/`inactive`), `order_by` (`next_run_asc`/`next_run_desc`/`prompt_asc`/`prompt_desc`), `limit` (1-500), `offset` (int) | List the user's scheduled prompts with optional filters. Returns `items`, `total`, `limit`, `offset`. |
| `get_schedule` | `schedule_id` (str, required) | Fetch full details of a single schedule (prompt text, LLM config, next run, conversation_id). |
| `create_schedule` | `llm_prompt_template` (str, required), `schedule_type` (str, default `"once"`), `start_at` (ISO 8601, required), `llm_config` (object, optional), `system_prompt` (str, optional), `tool_config` (object, optional), `use_agentic_mode` (bool, optional), `conversation_id` (str, optional) | Create a schedule for a future or recurring run. Returns `id` and `next_run_at`. Pass `conversation_id` to continue in an existing conversation. |
| `schedule_now` | `prompt` (str, required), `llm_config` (object, optional), `schedule_type` (str, default `"once"`), `system_prompt` (str, optional), `conversation_id` (str, optional) | Create a schedule and run it immediately. Returns `schedule_id`, `task_id`, `conversation_id`, `message`. The run executes in the background. Pass `conversation_id` to continue in an existing conversation. |
| `run_schedule_now` | `schedule_id` (str, required) | Trigger an existing schedule to run immediately. Returns `task_id`, `conversation_id`. |
| `update_schedule` | `schedule_id` (str, required), plus any of: `llm_prompt_template`, `schedule_type`, `start_at`, `llm_config`, `system_prompt`, `tool_config`, `use_agentic_mode` | Partial update of an existing schedule. Send only the fields to change. |
| `delete_schedule` | `schedule_id` (str, required) | Permanently delete a scheduled prompt. |

### Conversation Tools

| Tool | Parameters | Purpose |
|------|-----------|---------|
| `get_conversation` | `conversation_id` (str, required) | Fetch the full conversation packet from storage: files, `deliverable_groups` (docs/reports with full URLs), `evidence_items` (tool output links with full URLs). Call after `schedule_now` or `run_schedule_now` completes. |
| `delete_conversation` | `conversation_id` (str, required) | Delete a conversation and all stored files. |

## Installing in Claude Desktop

Add to your `claude_desktop_config.json` (typically `~/Library/Application Support/Claude/claude_desktop_config.json` on macOS):

```json
{
  "mcpServers": {
    "hackerdogs": {
      "url": "https://app.hackerdogs.ai/mcp/",
      "headers": {
        "Authorization": "Bearer hd-YOUR_API_KEY_HERE"
      }
    }
  }
}
```

Restart Claude Desktop. The Hackerdogs tools will appear in the tools panel.

## Installing in Cursor

Add to `.cursor/mcp.json` in your project root (or global settings):

```json
{
  "mcpServers": {
    "hackerdogs": {
      "url": "https://app.hackerdogs.ai/mcp/",
      "headers": {
        "Authorization": "Bearer hd-YOUR_API_KEY_HERE"
      }
    }
  }
}
```

Restart Cursor. The tools will appear as MCP tools available to the AI.

## Installing via Hackerdogs Tools Catalog

1. Log in to Hackerdogs at [https://app.hackerdogs.ai](https://app.hackerdogs.ai).
2. Go to **API Keys** and create a key (if you haven't already). Copy the key.
3. Go to **Tools Catalog** (sidebar).
4. Find the **Hackerdogs MCP Server** entry.
5. Paste your API key and click **Install**.
6. The config snippet is generated for you -- copy it into Claude or Cursor as shown above.

## Sample Prompts & Use Cases

### Basic: Attack Surface Analysis

> Run a comprehensive attack surface analysis on hackerdogs.ai. Identify all exposed services, open ports, subdomains, and potential entry points. Create a risk report.

The AI will call `schedule_now` with this prompt. Hackerdogs runs the analysis using its OSINT tools and returns a conversation_id. The AI then calls `get_conversation` to retrieve the full report with deliverables.

### Basic: User Enumeration

> Enumerate all publicly discoverable users, emails, and social profiles associated with the domain example.com. Check for credential leaks and breached accounts.

### Scheduled: Daily Threat Monitoring

> "Schedule a daily check: find today's top 10 cybersecurity incidents and breaches. Correlate with our infrastructure and create a risk report."

The AI calls `get_schedule_types` to confirm `daily` is allowed, then `create_schedule` with `schedule_type="daily"` and `start_at` set to tomorrow at 8:00 AM UTC.

### Advanced: Continuing an Existing Conversation

When you want to build on a previous analysis instead of starting fresh, pass the `conversation_id` from a prior run:

> "Using the conversation from our last attack surface scan (conversation_id: 01KJVJ4FHWEV43MR1A9YKCG3NB), now run a deeper analysis on the 3 highest-risk findings. Add port scanning and vulnerability assessment."

The AI calls:
```
schedule_now(
  prompt="Deeper analysis on the 3 highest-risk findings...",
  conversation_id="01KJVJ4FHWEV43MR1A9YKCG3NB"
)
```

The new run continues in the same conversation thread, with access to the prior context.

### Advanced: Multi-Step Recursive Analysis

This pattern chains multiple Hackerdogs runs, where each step feeds the next:

**Step 1 -- Initial reconnaissance:**
> "Run an initial reconnaissance on target.com: subdomains, DNS records, WHOIS, tech stack detection."

The AI calls `schedule_now`, waits for completion, calls `get_conversation` to get results, and extracts the conversation_id (e.g., `01KJX...`).

**Step 2 -- Deep dive using step 1's conversation:**
> "Continue the investigation in conversation 01KJX... -- for each subdomain found, run port scanning and service fingerprinting. Focus on anything that looks like a staging or dev environment."

The AI calls `schedule_now` with the same `conversation_id`, building on the previous findings.

**Step 3 -- Vulnerability assessment:**
> "Still in conversation 01KJX... -- take the exposed services from step 2 and check for known CVEs, misconfigurations, and default credentials. Generate a prioritized vulnerability report."

Each step enriches the same conversation, creating a comprehensive investigation thread.

### Advanced: Creating a New Conversation for Comparison

> "Run the same attack surface analysis as conversation 01KJX... but against staging.target.com. I want a separate conversation so I can compare them side by side."

Here, the AI omits `conversation_id` so Hackerdogs creates a fresh one. The user ends up with two independent conversation threads to compare.

---

# Section 2: Using the Hackerdogs REST API

The REST API provides direct HTTP access to three API groups:

- **Scheduled Prompts API** (`/scheduled-prompts-api`) -- Create, manage, and run scheduled prompts.
- **Conversation API** (`/conversation-api`) -- Retrieve or delete conversation results.
- **Chat Completions API** (`/v1`) -- OpenAI-compatible streaming chat endpoint.

**Base URL:** `https://app.hackerdogs.ai`
**Authentication:** `X-API-Key: hd-YOUR_KEY` header (all endpoints), or `Authorization: Bearer hd-YOUR_KEY` (Chat Completions).

## 2.1 Scheduled Prompts API

Base path: `/scheduled-prompts-api`

### Endpoints

| Method | Path | Description | Status |
|--------|------|-------------|--------|
| GET | `/llm-options` | List available LLM provider/model pairs | 200 |
| GET | `/schedule-types` | List schedule types allowed by plan | 200 |
| GET | `/schedules` | List scheduled prompts (with filters) | 200 |
| GET | `/schedules/{id}` | Get one schedule by ID | 200 |
| POST | `/schedules` | Create a schedule (future/recurring) | 201 |
| PATCH | `/schedules/{id}` | Partial update of a schedule | 200 |
| DELETE | `/schedules/{id}` | Delete a schedule | 200 |
| POST | `/schedule-now` | Create + run immediately | 202 |
| POST | `/schedules/{id}/run-now` | Run existing schedule now | 202 |

### POST /schedule-now (Create + Run Immediately)

The most common endpoint. Creates a schedule and runs it in one call.

**Request:**

```bash
curl -X POST "https://app.hackerdogs.ai/scheduled-prompts-api/schedule-now" \
  -H "X-API-Key: hd-YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "Find today'\''s top 10 cybersecurity attacks and breaches. Summarize the findings.",
    "schedule_type": "once"
  }'
```

**Response (202):**

```json
{
  "schedule_id": "01KJVJ4FHFTEH1F1YZGHEB6C4W",
  "task_id": "01KJVJ4FHWEV43MR1A9YKCG3N9",
  "conversation_id": "01KJVJ4FHWEV43MR1A9YKCG3NB",
  "message": "Schedule triggered; conversation will appear in Chat."
}
```

The run executes in the background. Use the `conversation_id` to retrieve results.

**Optional fields:**

| Field | Type | Description |
|-------|------|-------------|
| `prompt` | string (required) | The prompt to execute. |
| `llm_config` | object | `{"provider": "openai", "model": "gpt-4o-mini"}`. If omitted, uses your default LLM from User Settings. |
| `schedule_type` | string | Default `"once"`. Must be allowed by your plan. |
| `system_prompt` | string | Custom system prompt for this run. |
| `conversation_id` | string | Continue in an existing conversation instead of creating a new one. |
| `tool_config` | object | Tool configuration for agentic mode. |
| `use_agentic_mode` | bool | Enable/disable tools. Defaults to your User Settings value. |

### POST /schedule-now with conversation_id (Continue Existing Conversation)

```bash
# Step 1: Initial run (new conversation)
RESULT=$(curl -s -X POST "https://app.hackerdogs.ai/scheduled-prompts-api/schedule-now" \
  -H "X-API-Key: hd-YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"prompt": "Reconnaissance on target.com: subdomains, DNS, WHOIS."}')

CONV_ID=$(echo "$RESULT" | python3 -c "import sys,json; print(json.load(sys.stdin)['conversation_id'])")
echo "Conversation ID: $CONV_ID"

# Step 2: Continue in the same conversation
curl -X POST "https://app.hackerdogs.ai/scheduled-prompts-api/schedule-now" \
  -H "X-API-Key: hd-YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d "{
    \"prompt\": \"Now run port scanning on the subdomains found. Focus on staging environments.\",
    \"conversation_id\": \"$CONV_ID\"
  }"
```

### POST /schedules (Create for Future/Recurring)

```bash
curl -X POST "https://app.hackerdogs.ai/scheduled-prompts-api/schedules" \
  -H "X-API-Key: hd-YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "llm_prompt_template": "Daily threat intel: top cybersecurity incidents in the last 24 hours.",
    "schedule_type": "daily",
    "start_at": "2026-03-05T08:00:00Z"
  }'
```

**Response (201):**

```json
{
  "id": "01KJXYZ...",
  "next_run_at": "2026-03-05T08:00:00+00:00"
}
```

### GET /schedules (List)

```bash
curl "https://app.hackerdogs.ai/scheduled-prompts-api/schedules?status_filter=active&limit=10" \
  -H "X-API-Key: hd-YOUR_KEY"
```

### POST /schedules/{id}/run-now (Trigger Existing)

```bash
curl -X POST "https://app.hackerdogs.ai/scheduled-prompts-api/schedules/01KJXYZ.../run-now" \
  -H "X-API-Key: hd-YOUR_KEY"
```

### Error Codes

| Code | Meaning |
|------|---------|
| 401 | Missing or invalid API key |
| 402 | Insufficient credits or tenant credit limit reached |
| 403 | Plan does not support the requested `schedule_type` (upgrade required) |
| 404 | Schedule or endpoint not found |
| 422 | Validation error (missing prompt, invalid schedule_type, etc.) |
| 503 | Service unavailable (backend, DB, or credit check) |

---

## 2.2 Conversation API

Base path: `/conversation-api`

Conversations are the output of scheduled prompt runs. Each run produces files, deliverables (reports/documents), and evidence items (tool outputs) stored in the conversation.

### Endpoints

| Method | Path | Description | Status |
|--------|------|-------------|--------|
| GET | `/conversations/{conversation_id}` | Get full conversation packet | 200 |
| DELETE | `/conversations/{conversation_id}` | Delete conversation + stored files | 200 |

### GET /conversations/{conversation_id}

Retrieve the complete conversation output after a run completes.

```bash
curl "https://app.hackerdogs.ai/conversation-api/conversations/01KJVJ4FHWEV43MR1A9YKCG3NB" \
  -H "X-API-Key: hd-YOUR_KEY"
```

**Response (200):**

```json
{
  "conversation_id": "01KJVJ4FHWEV43MR1A9YKCG3NB",
  "user_id": "...",
  "tenant_id": "...",
  "files": [
    {"path": "chat/user_id/conv_id/content.json", "content": {...}},
    {"path": "chat/user_id/conv_id/report.md", "content": "..."}
  ],
  "deliverable_groups": [
    {
      "title": "Reports",
      "deliverables": [
        {"display_name": "Risk Report", "url": "https://...signed-url..."}
      ]
    }
  ],
  "evidence_items": [
    {"tool_name": "nmap_scan", "url": "https://...signed-url..."}
  ],
  "evidence_files": [
    {"blob_name": "...", "url": "https://...signed-url..."}
  ]
}
```

**Fields:**

| Field | Description |
|-------|-------------|
| `files` | All objects stored for this conversation (path + content). |
| `deliverable_groups` | Grouped documents/reports with presigned download URLs. Same structure as webhooks and email notifications. |
| `evidence_items` | Tool output references extracted from content files, with presigned URLs. |
| `evidence_files` | All tool/evidence blobs with direct download URLs. |

### DELETE /conversations/{conversation_id}

```bash
curl -X DELETE "https://app.hackerdogs.ai/conversation-api/conversations/01KJVJ4FHWEV43MR1A9YKCG3NB" \
  -H "X-API-Key: hd-YOUR_KEY"
```

**Response (200):**

```json
{
  "success": true,
  "conversation_id": "01KJVJ4FHWEV43MR1A9YKCG3NB",
  "deleted_count": 15,
  "storage_deleted_count": 8
}
```

### Full Workflow: Schedule + Wait + Get Conversation

```bash
# 1. Schedule and run
RESULT=$(curl -s -X POST "https://app.hackerdogs.ai/scheduled-prompts-api/schedule-now" \
  -H "X-API-Key: hd-YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"prompt": "Enumerate subdomains for example.com and check for takeover vulnerabilities."}')

echo "$RESULT"
CONV_ID=$(echo "$RESULT" | python3 -c "import sys,json; print(json.load(sys.stdin)['conversation_id'])")

# 2. Wait for completion (poll until ready)
echo "Waiting for conversation $CONV_ID..."
for i in $(seq 1 30); do
  sleep 10
  HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" \
    "https://app.hackerdogs.ai/conversation-api/conversations/$CONV_ID" \
    -H "X-API-Key: hd-YOUR_KEY")
  echo "  Poll $i: HTTP $HTTP_CODE"
  if [ "$HTTP_CODE" = "200" ]; then
    break
  fi
done

# 3. Get the full packet
curl -s "https://app.hackerdogs.ai/conversation-api/conversations/$CONV_ID" \
  -H "X-API-Key: hd-YOUR_KEY" | python3 -m json.tool
```

---

## 2.3 Chat Completions API (OpenAI-compatible)

Base path: `/v1`

Drop-in replacement for the OpenAI Chat Completions API. Compatible with Open WebUI, LangChain, and any OpenAI SDK client. Supports streaming (SSE) and non-streaming modes.

### Endpoints

| Method | Path | Description |
|--------|------|-------------|
| POST | `/v1/chat/completions` | Chat completion (stream or non-stream) |
| GET | `/v1/models` | List available models |

### Authentication

Use `Authorization: Bearer hd-YOUR_KEY` (preferred) or `X-API-Key: hd-YOUR_KEY`.

### POST /v1/chat/completions (Streaming)

```bash
curl -X POST "https://app.hackerdogs.ai/v1/chat/completions" \
  -H "Authorization: Bearer hd-YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [
      {"role": "system", "content": "You are a cybersecurity analyst."},
      {"role": "user", "content": "Analyze the attack surface of example.com"}
    ],
    "stream": true
  }'
```

Streams Server-Sent Events (SSE) in OpenAI format:

```
data: {"id":"chatcmpl-01KJX...","object":"chat.completion.chunk","model":"gpt-4o-mini","choices":[{"index":0,"delta":{"role":"assistant","content":"Based on"},"finish_reason":null}]}

data: {"id":"chatcmpl-01KJX...","object":"chat.completion.chunk","model":"gpt-4o-mini","choices":[{"index":0,"delta":{"content":" my analysis"},"finish_reason":null}]}

...

data: {"id":"chatcmpl-01KJX...","object":"chat.completion.chunk","model":"gpt-4o-mini","choices":[{"index":0,"delta":{},"finish_reason":"stop"}],"usage":{"prompt_tokens":25,"completion_tokens":150,"total_tokens":175}}

data: [DONE]
```

### POST /v1/chat/completions (Non-streaming)

```bash
curl -X POST "https://app.hackerdogs.ai/v1/chat/completions" \
  -H "Authorization: Bearer hd-YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [
      {"role": "user", "content": "What are the OWASP Top 10 for 2025?"}
    ],
    "stream": false
  }'
```

**Response:**

```json
{
  "id": "chatcmpl-01KJX...",
  "object": "chat.completion",
  "model": "gpt-4o-mini",
  "created": 1709510400,
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "The OWASP Top 10 for 2025..."
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 12,
    "completion_tokens": 200,
    "total_tokens": 212
  }
}
```

### Request Body

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `messages` | array (required) | -- | Array of `{role, content}` objects. Roles: `system`, `user`, `assistant`, `tool`. |
| `stream` | bool | `true` | Stream SSE chunks or return single JSON response. |
| `model` | string | user's default | Override the model (must be available in your account). |
| `temperature` | float | user's default | Sampling temperature. |
| `max_tokens` | int | user's default | Maximum tokens to generate. |
| `tools` | array/bool | `true` (all tools) | Pass `false` or `[]` to disable tools. Pass `null` or omit for all tools. |

### GET /v1/models

```bash
curl "https://app.hackerdogs.ai/v1/models" \
  -H "Authorization: Bearer hd-YOUR_KEY"
```

**Response:**

```json
{
  "object": "list",
  "data": [
    {"id": "openai/gpt-4o-mini", "object": "model", "created": 0, "owned_by": "hackerdogs"},
    {"id": "anthropic/claude-sonnet-4-20250514", "object": "model", "created": 0, "owned_by": "hackerdogs"}
  ]
}
```

### Using with OpenAI Python SDK

```python
from openai import OpenAI

client = OpenAI(
    api_key="hd-YOUR_KEY",
    base_url="https://app.hackerdogs.ai/v1",
)

response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[
        {"role": "system", "content": "You are a cybersecurity analyst with access to OSINT tools."},
        {"role": "user", "content": "Run a subdomain enumeration on example.com"},
    ],
    stream=True,
)

for chunk in response:
    if chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end="")
```

### Using with Open WebUI

In Open WebUI settings, add a new connection:

- **API Base URL:** `https://app.hackerdogs.ai/v1`
- **API Key:** `hd-YOUR_KEY`

Models will auto-populate from the `/v1/models` endpoint.

---

## Python Client Libraries

Hackerdogs ships Python clients for programmatic use:

### SchedulePromptNowClient

```python
from api_clients.sch.schedule_prompt_now_client import SchedulePromptNowClient

client = SchedulePromptNowClient(
    api_key="hd-YOUR_KEY",
    base_url="https://app.hackerdogs.ai",
)

# Schedule and run now
result = client.schedule_now(prompt="Analyze attack surface of example.com")
print(result["conversation_id"])

# Schedule and run now, continuing an existing conversation
result = client.schedule_now(
    prompt="Now scan the top 3 subdomains for vulnerabilities.",
    conversation_id="01KJVJ4FHWEV43MR1A9YKCG3NB",
)

# Schedule and wait for full results
result = client.schedule_now_and_wait(
    prompt="Full OSINT report on target.com",
    wait_secs=300,
    poll_interval_sec=10,
)
print(result["conversation"])  # Full packet with deliverables and evidence
```

### ConversationClient

```python
from api_clients.conversation.conversation_client import ConversationClient

client = ConversationClient(
    api_key="hd-YOUR_KEY",
    base_url="https://app.hackerdogs.ai",
)

# Get conversation
conv = client.get_conversation("01KJVJ4FHWEV43MR1A9YKCG3NB")
print(f"Files: {len(conv['files'])}")
print(f"Deliverables: {len(conv['deliverable_groups'])}")
print(f"Evidence: {len(conv['evidence_items'])}")

# Delete conversation
client.delete_conversation("01KJVJ4FHWEV43MR1A9YKCG3NB")
```

---

## Use Case Examples

### Automated Daily Threat Brief

```bash
# Create a daily schedule at 8 AM UTC
curl -X POST "https://app.hackerdogs.ai/scheduled-prompts-api/schedules" \
  -H "X-API-Key: hd-YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "llm_prompt_template": "Generate a daily cybersecurity threat brief: top incidents in the last 24 hours, new CVEs affecting our tech stack (React, Python, PostgreSQL, nginx), and any relevant threat actor activity. Format as an executive summary.",
    "schedule_type": "daily",
    "start_at": "2026-03-05T08:00:00Z"
  }'
```

### Incident Response: Multi-Step Investigation

```bash
# Step 1: Initial triage
R1=$(curl -s -X POST "https://app.hackerdogs.ai/scheduled-prompts-api/schedule-now" \
  -H "X-API-Key: hd-YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"prompt": "Investigate IP 203.0.113.42: reverse DNS, geolocation, ASN, reputation checks, and any associated threat intelligence."}')
CONV=$(echo "$R1" | python3 -c "import sys,json; print(json.load(sys.stdin)['conversation_id'])")

# Step 2: Deeper investigation in same conversation
curl -s -X POST "https://app.hackerdogs.ai/scheduled-prompts-api/schedule-now" \
  -H "X-API-Key: hd-YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d "{
    \"prompt\": \"Based on the findings, check if this IP has been involved in any botnets, C2 infrastructure, or scanning campaigns. Cross-reference with recent CVE exploits.\",
    \"conversation_id\": \"$CONV\"
  }"

# Step 3: Generate incident report
curl -s -X POST "https://app.hackerdogs.ai/scheduled-prompts-api/schedule-now" \
  -H "X-API-Key: hd-YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d "{
    \"prompt\": \"Compile all findings into a formal incident report with timeline, IOCs, risk assessment, and recommended mitigations.\",
    \"conversation_id\": \"$CONV\"
  }"
```

### Competitive Intelligence

```bash
curl -X POST "https://app.hackerdogs.ai/scheduled-prompts-api/schedule-now" \
  -H "X-API-Key: hd-YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "Perform OSINT on competitor.com: tech stack, infrastructure, recent changes, employee LinkedIn profiles in engineering roles, GitHub repos, and any public security disclosures. Create a competitive intelligence brief."
  }'
```
