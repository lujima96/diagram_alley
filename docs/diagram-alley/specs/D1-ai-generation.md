---
id: D1
tier: domain
status: draft
version: "0.1"
depends-on: [F1, F2, F3, F5]
tags: [ai, generation, providers, byok, ollama, repair, prompt]
---

# D1 — AI Generation

**Version:** 0.1
**Status:** Draft
**Depends on:** F1, F2, F3, F5

---

## Purpose

Define the prompt → structured spec flow, provider routing, BYOK configuration, local model support, repair pass integration, and AI failure handling for both new diagram generation and AI-driven diagram modification.

## Non-Goals

- Rendering rules (→ F2)
- Credential encryption scheme (→ F3 §4)
- Provider CRUD endpoints contract (→ F5 §10, D2)
- Visual diff UI between old and new spec (→ D2)
- Version snapshot creation (→ D3)
- Export after generation (→ D4)

---

## Owned Concepts

Prompt → spec pipeline; system prompt template; provider adapter interface; provider routing; AI generation endpoint contract; AI improve endpoint contract; AI failure handling; retry policy; generation audit trigger points (constants owned by F3).

---

## 1. Provider Model

### 1.1 Supported Provider Types

Three provider types are supported in V1. All are stored in the `model_providers` table (F1 §4.6).

| `provider_type` | Description | Auth |
|-----------------|-------------|------|
| `openai_compatible` | Any OpenAI-compatible REST API (OpenAI, Together.ai, OpenRouter, Groq, Mistral, etc.) | `api_key_encrypted` (BYOK via F3 §4) |
| `ollama` | Local Ollama server (e.g. `http://localhost:11434`) | None (no API key required) |
| `llamacpp` | llama.cpp HTTP server | Optional `api_key_encrypted` |

No hosted AI credits are offered in V1 (DEC-010). Every user must configure at least one provider to use AI generation.

### 1.2 Provider Adapter Interface

The backend defines a common async adapter protocol in `app/providers/base.py`:

```python
class ProviderAdapter(Protocol):
    async def generate(
        self,
        system_prompt: str,
        user_prompt: str,
        max_tokens: int,
        temperature: float,
    ) -> str:
        """Return the raw LLM text response. Raises ProviderError on failure."""
        ...
```

Concrete adapters:
- `app/providers/openai_compatible.py` — uses `httpx` to call `/v1/chat/completions`
- `app/providers/ollama.py` — uses `httpx` to call `/api/chat`
- `app/providers/llamacpp.py` — uses `httpx` to call `/completion` (OpenAI-compatible variant supported too)

The adapter is selected by `provider_type`. All adapters are instantiated in `app/providers/factory.py` with credentials decrypted just-in-time from `api_key_encrypted` (F3 §4.2).

### 1.3 Provider Routing

For each generation request:

1. If the request body includes `provider_id`, use that provider.
2. Otherwise, use the user's default provider (`is_default = true` in `model_providers`).
3. If no provider is configured, return `400 PROVIDER_NOT_CONFIGURED` (see §7.2).

The provider must belong to the requesting user (ownership check in service layer, F3 §2.2).

### 1.4 Generation Parameters

Per-provider defaults are stored in `model_providers`. Per-request overrides are not exposed in V1 (all generation uses the provider's stored `temperature` and `max_tokens`).

| Parameter | Default | Notes |
|-----------|---------|-------|
| `temperature` | `0.2` | Low for deterministic spec output |
| `max_tokens` | `4096` | Override in `model_providers.max_tokens` if set |

---

## 2. Prompt Construction

### 2.1 System Prompt

The system prompt is rendered per-request by `app/services/prompt_builder.py`. It is **not** stored in the database. The prompt:

1. Instructs the LLM to return a valid Diagram Alley JSON spec — never free-form text, markdown, or raw ASCII.
2. Embeds the full JSON schema for the requested `diagram_type` (from F1 §3).
3. States that `spec_version` must be `"1.0"` and `diagram_type` must match the requested type.
4. Prohibits adding commentary outside the JSON object.
5. Provides one short worked example of a correct spec for the diagram type.

System prompt template location: `app/services/prompt_templates/<diagram_type>_system.txt` (one file per diagram type).

### 2.2 User Prompt (Generate)

For new diagram generation, the user prompt is:

```
Generate a <diagram_type> diagram for the following:

<user_prompt_text>
```

`user_prompt_text` is the raw string from the request body. Max length: 2000 characters (validated at API boundary; returns `400 BAD_REQUEST` if exceeded).

### 2.3 User Prompt (Improve)

For AI modification of an existing diagram, the user prompt is:

```
Here is the current diagram spec (JSON):

<current_spec_json>

Apply the following change:

<change_request_text>

Return the complete updated spec JSON. Do not summarize or explain changes — return only the JSON object.
```

`current_spec_json` is the current `diagrams.spec_json` serialized to a compact JSON string.
`change_request_text` is the user's change request. Max length: 1000 characters.

### 2.4 Prompt Length Guard

Before sending to the provider, the service estimates token count (characters / 4 as a conservative heuristic). If the estimated total prompt exceeds `max_tokens - 512` (leaving 512 tokens of headroom for the response), the service:
- For generate: accepts as-is (system prompt + user prompt rarely approach the limit for generation requests).
- For improve: truncates `current_spec_json` to the first 3000 characters and appends `... [truncated for length]`. This is a best-effort path — the repair pass will catch any spec fragments that result.

---

## 3. Generation Flow — New Diagram

**Endpoint:** `POST /api/v1/diagrams/generate` (F5 §10)

### 3.1 Request Body

```json
{
  "diagram_type": "architecture",
  "prompt": "A three-tier web app with React frontend, FastAPI backend, and Postgres database",
  "title": "Three-Tier Web App",
  "provider_id": "uuid | null"
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `diagram_type` | Yes | One of the six V1 types (F1 §2) |
| `prompt` | Yes | Natural language description; max 2000 chars |
| `title` | No | Diagram title; defaults to `"Untitled <type>"` if omitted |
| `provider_id` | No | Override default provider; must belong to requesting user |

### 3.2 Service Steps

```
1. Validate request body (diagram_type, prompt length)
2. Check subscription gate (F3 §5.5 — trialing or active required)
3. Resolve provider (provider_id or user default; fail fast if none configured)
4. Build system prompt for diagram_type
5. Send to provider adapter → raw LLM response string
6. Parse raw response as JSON → DiagramSpec candidate
7. Validate candidate (F2 validate_spec)
8. If validation errors exist → attempt repair pass (F2 repair_spec); max 1 repair attempt
9. Re-validate after repair
10. If still invalid after repair → return 422 GENERATION_INVALID_SPEC (see §7)
11. Create diagrams row; return diagram with rendered outputs
12. Create initial diagram_versions snapshot (F1 §5.3, DEC-020)
13. Emit DIAGRAM_CREATED and DIAGRAM_AI_GENERATED audit events
14. Return 200 with diagram data and generation metadata
```

### 3.3 Persistence on Generate

A new diagram is immediately persisted to the `diagrams` table on successful generation. The user does not need to click Save. The `spec_json` is the validated (and possibly repaired) spec. The response includes the new `diagram_id` so the client can redirect to `/workspace/:diagramId`.

The initial persisted generated spec is also written to `diagram_versions` as the first snapshot (DEC-020). This snapshot is not considered auto-save noise; it is the baseline history anchor for the diagram.

### 3.4 Response Body

```json
{
  "data": {
    "diagram_id": "uuid",
    "title": "Three-Tier Web App",
    "diagram_type": "architecture",
    "spec_json": { "...": "full spec" },
    "ascii_cache": "...rendered ASCII...",
    "mermaid_cache": "...rendered Mermaid...",
    "generation": {
      "provider_id": "uuid",
      "provider_type": "openai_compatible",
      "repairs_applied": [],
      "warnings": []
    }
  }
}
```

`repairs_applied` lists any repair rules applied (from F2 §3). `warnings` lists non-blocking validation warnings.

---

## 4. Improvement Flow — Existing Diagram

**Endpoint:** `POST /api/v1/diagrams/{id}/improve` (F5 §10)

### 4.1 Request Body

```json
{
  "change_request": "Add a Redis cache between the API and the database",
  "provider_id": "uuid | null"
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `change_request` | Yes | Natural language description of change; max 1000 chars |
| `provider_id` | No | Override default provider |

### 4.2 Service Steps

```
1. Validate request; load diagram (ownership check)
2. Check subscription gate
3. Resolve provider
4. Build improve prompt with current spec_json
5. Send to provider → raw LLM response
6. Parse response as JSON → updated DiagramSpec candidate
7. Validate candidate (F2 validate_spec)
8. If validation errors → repair pass (max 1 attempt)
9. Re-validate
10. If still invalid → return 422 GENERATION_INVALID_SPEC
11. Compute spec diff (§4.3)
12. Return proposed spec + diff metadata WITHOUT saving
13. Emit DIAGRAM_AI_IMPROVE_PROPOSED audit event
```

**Note:** The improve flow does NOT auto-save. The updated spec is returned as a proposal. The client shows the diff and the user must explicitly accept. On accept, the client calls `PATCH /api/v1/diagrams/{id}` with the new spec and `create_version_snapshot = true`, which creates a version snapshot (D3).

### 4.3 Spec Diff

The backend computes a simple structural diff between the current spec and the proposed spec:

```json
{
  "nodes_added": ["redis_cache"],
  "nodes_removed": [],
  "nodes_changed": [],
  "edges_added": ["api_to_cache", "cache_to_db"],
  "edges_removed": ["api_to_db"],
  "edges_changed": []
}
```

For diagram types without nodes/edges (database, flowchart, ui_wireframe, file_structure, network), the diff uses the appropriate entity names (tables, steps, components, devices).

### 4.4 Response Body

```json
{
  "data": {
    "proposed_spec": { "...": "full updated spec" },
    "diff": {
      "nodes_added": ["redis_cache"],
      "nodes_removed": [],
      "nodes_changed": [],
      "edges_added": ["api_to_cache", "cache_to_db"],
      "edges_removed": ["api_to_db"],
      "edges_changed": []
    },
    "proposed_ascii": "...rendered ASCII of proposed spec...",
    "generation": {
      "provider_id": "uuid",
      "repairs_applied": [],
      "warnings": []
    }
  }
}
```

---

## 5. Retry Policy

### 5.1 Network Failures

If the provider adapter raises a connection error or timeout:
- Retry once automatically after a 1-second delay.
- If the second attempt fails, return `502 PROVIDER_UNAVAILABLE`.
- No more than 1 automatic retry.

### 5.2 Invalid Spec From LLM

If the LLM returns JSON that fails validation after 1 repair pass:
- Do **not** retry with the same prompt.
- Return `422 GENERATION_INVALID_SPEC` with the validation errors in `detail`.
- The client displays the error and re-enables the prompt textarea.

### 5.3 Malformed JSON From LLM

If the LLM response cannot be parsed as JSON at all:
- Attempt to extract a JSON block from within the response string (scan for first `{` to last `}`).
- If extraction succeeds, continue with the extracted block and treat it as a spec candidate.
- If extraction fails, return `422 GENERATION_PARSE_FAILED`.

### 5.4 Last Valid Spec Preservation

On any failure during improve flow, the existing `diagrams.spec_json` is never modified. The client keeps the existing spec loaded.

---

## 6. Local Model Support

Ollama and llama.cpp adapters follow the same ProviderAdapter protocol. Configuration requirements:

**Ollama:**
- `base_url`: e.g. `http://localhost:11434`
- `model`: e.g. `llama3.2`, `mistral`, `codellama`
- No API key required
- Test connection: `GET {base_url}/api/tags` — must return 200 with model list

**llama.cpp:**
- `base_url`: e.g. `http://localhost:8080`
- `model`: model name (for logging; llama.cpp serves one model at a time)
- Optional `api_key_encrypted` if the server is configured with auth

**Routing (DEC-048):** Routing depends on the `base_url` of the configured provider:

| Provider | `base_url` | Call origin | Reason |
|----------|-----------|-------------|--------|
| OpenAI / Anthropic / OpenAI-compatible | Any (cloud endpoint) | Backend | API key must not reach browser |
| Remote Ollama | Public URL / VPS / Docker | Backend | Backend can reach it; consistent with cloud routing |
| Local Ollama | `127.0.0.1` or `::1` (localhost) | **Browser direct** | Backend cannot reach user's localhost; no API key to protect |

For local Ollama (`base_url` resolves to a loopback address), the frontend calls `{base_url}/api/chat` directly from the browser. Ollama handles CORS for local connections. The backend stores the provider config (base_url, model) but does not proxy the call.

**Security implication:** Local Ollama requires no API key — there is nothing sensitive to protect. The "all API keys are encrypted server-side" guarantee applies to cloud providers only. The product should make this distinction explicit in the provider settings UI.

---

## 7. Error Codes

In addition to standard F5 error codes, AI generation introduces:

| HTTP | Error Code | When |
|------|-----------|------|
| 400 | `PROVIDER_NOT_CONFIGURED` | No provider configured for user |
| 400 | `PROMPT_TOO_LONG` | Prompt exceeds 2000 chars (generate) or 1000 chars (improve) |
| 422 | `GENERATION_PARSE_FAILED` | LLM response is not parseable JSON |
| 422 | `GENERATION_INVALID_SPEC` | Parsed spec fails validation after repair attempt |
| 502 | `PROVIDER_UNAVAILABLE` | Provider connection failed after 1 retry |
| 503 | `PROVIDER_TIMEOUT` | Provider took > 30s (per-request timeout) |

`GENERATION_INVALID_SPEC` always includes `detail.errors` with the validation error list from F2.

---

## 8. Audit Events

F3 §3.2 owns the AI audit event constants used by this flow:

| Event | Trigger | Actor | Entity |
|-------|---------|-------|--------|
| `DIAGRAM_AI_GENERATED` | Successful new diagram generation | user | diagrams |
| `DIAGRAM_AI_IMPROVE_PROPOSED` | Successful improve proposal returned | user | diagrams |

These supplement the `DIAGRAM_CREATED` event from F3 §3.2 (which fires on all diagram creations, including AI-generated ones). `DIAGRAM_AI_GENERATED` carries `detail.provider_type` and `detail.repairs_applied` for analytics.

---

## 9. Provider Test Endpoint

**Endpoint:** `POST /api/v1/providers/{id}/test` (F5 §10)

The test sends a minimal fixed prompt ("Generate a hello world architecture diagram with one node.") and checks that the response is parseable JSON. It does not validate the full spec.

Response:
```json
{
  "data": {
    "success": true,
    "latency_ms": 1243,
    "error": null
  }
}
```

On failure: `"success": false` with `"error": "Connection refused"` or similar. HTTP status is always `200` — the test result is in the body.

---

## 10. Frontend Integration Notes

These are non-normative guidance for D2 and F4 implementers.

- The **Generate button** is disabled until: (a) a provider is configured, and (b) the prompt textarea has content.
- During generation, the entire prompt panel is in a loading state (spinner, textarea disabled).
- On `GENERATION_INVALID_SPEC`, show the error with the validation errors list. Do not clear the prompt — let the user refine it.
- On `PROVIDER_UNAVAILABLE`, show "Provider not reachable. Check your provider settings." with a link to `/settings/providers`.
- On successful generation, redirect the workspace to the new `diagram_id` URL.
- For the improve flow, show the proposed ASCII alongside the current ASCII before the user confirms acceptance.

---

## Open Questions

None. All D1 decisions are locked.

---

## Acceptance Criteria

- `POST /api/v1/diagrams/generate` with a valid prompt and a configured provider returns a persisted diagram with `ascii_cache` populated.
- `POST /api/v1/diagrams/generate` with no provider configured returns `400 PROVIDER_NOT_CONFIGURED`.
- `POST /api/v1/diagrams/generate` by a user with `subscription.status = 'canceled'` returns `403 SUBSCRIPTION_REQUIRED`.
- A generation where the LLM returns invalid JSON triggers the JSON extraction fallback; if extraction fails, `422 GENERATION_PARSE_FAILED` is returned.
- A generation where the spec fails validation triggers one repair pass; if still invalid, `422 GENERATION_INVALID_SPEC` is returned with error detail.
- `POST /api/v1/diagrams/{id}/improve` returns a `proposed_spec` and `diff` without modifying `diagrams.spec_json`.
- A provider connection failure retries once; on second failure returns `502 PROVIDER_UNAVAILABLE`.
- `DIAGRAM_CREATED` and `DIAGRAM_AI_GENERATED` audit events are both written on successful generation.
- `POST /api/v1/providers/{id}/test` returns `{"success": true}` for a correctly configured Ollama provider and `{"success": false}` with an error string when the Ollama server is unreachable.
