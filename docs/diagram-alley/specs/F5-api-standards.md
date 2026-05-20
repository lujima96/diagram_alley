---
id: F5
tier: foundation
status: draft
version: "0.1"
depends-on: [F0, F3]
tags: [api, rest, conventions, pagination, errors, background-jobs]
---

# F5 — API and Integration Standards

**Version:** 0.1
**Status:** Draft
**Depends on:** F0, F3

---

## Purpose

Define the REST API conventions, auth header requirements, pagination contract, error code taxonomy, background job standards, and rate limiting policy. Domain specs reference these contracts — they do not redefine them. Any new API operation must conform to this spec.

## Non-Goals

- Database table definitions (→ F1)
- Auth implementation (→ F3)
- Domain-specific endpoints (→ D1–D8)
- UI patterns (→ F4)

---

## Owned Concepts

URL naming convention; HTTP method usage; auth header; request/response envelope; pagination; error shape; error codes; background task contract; rate limit policy.

---

## 1. Base URL

| Environment | Base URL |
|-------------|----------|
| Local | `http://localhost:8000` |
| Production | `https://api.diagram-alley.fly.dev` (or custom domain) |

All API routes are prefixed `/api/v1/`. The frontend reads `VITE_API_BASE_URL` and appends `/api/v1` internally.

---

## 2. URL Naming Convention

- **Plural nouns** for collections: `/diagrams`, `/projects`, `/providers`
- **Kebab-case** for multi-word resource segments: `/diagram-versions`
- **Resource ID in path** for specific items: `/diagrams/{diagram_id}`
- **Action verbs as sub-resources** for non-CRUD operations: `/diagrams/{id}/render`, `/diagrams/{id}/export`, `/diagrams/{id}/versions/{version_id}/restore`
- **No trailing slashes**

```
Collection:         GET    /api/v1/diagrams
Create:             POST   /api/v1/diagrams
Read one:           GET    /api/v1/diagrams/{diagram_id}
Update:             PATCH  /api/v1/diagrams/{diagram_id}
Delete/Archive:     DELETE /api/v1/diagrams/{diagram_id}
Action on resource: POST   /api/v1/diagrams/{diagram_id}/render
```

**PUT vs PATCH:** Use `PATCH` for partial updates (the default for spec updates, title changes, etc.). Use `PUT` only when the entire resource body is always replaced (e.g., replacing a full spec_json). In practice, V1 uses PATCH everywhere for consistency.

---

## 3. HTTP Method Usage

| Method | Usage |
|--------|-------|
| `GET` | Read-only; never modifies state |
| `POST` | Create or trigger an action (generate, render, export, restore) |
| `PATCH` | Partial update |
| `DELETE` | Archive or delete |

---

## 4. Authentication Header

Every non-public route requires:

```
Authorization: Bearer <access_token>
```

The `access_token` is a JWT issued by FastAPI Users (F3 §1). If the token is absent, malformed, or expired, the API returns `401 UNAUTHORIZED`.

---

## 5. Request Format

All request bodies are JSON. Content-Type: `application/json`.

Path parameters are UUID strings where applicable. UUIDs are lowercase, hyphenated: `3fa85f64-5717-4562-b3fc-2c963f66afa6`.

---

## 6. Response Format

### 6.1 Success Envelope

All successful responses return a JSON object. No top-level array responses — even collections are wrapped:

```json
{
  "data": { ... },        // or "data": [ ... ] for lists
  "meta": {               // present on list responses only
    "total": 42,
    "page": 1,
    "page_size": 24,
    "has_next": true
  }
}
```

Single-resource responses:
```json
{ "data": { "id": "...", "title": "...", ... } }
```

List responses:
```json
{ "data": [ {...}, {...} ], "meta": { "total": 42, "page": 1, "page_size": 24, "has_next": true } }
```

Action responses (render, export, generate):
```json
{ "data": { "ascii": "...", "mermaid": "...", "warnings": [], "repairs_applied": [] } }
```

### 6.2 Empty Success

204 No Content for actions that produce no body (e.g., archive, delete).

---

## 7. Pagination

**Method:** Page-based (not cursor-based in V1). Cursor pagination is a V2 upgrade if needed.

**Query parameters:**
- `page` (integer, default 1, min 1)
- `page_size` (integer, default 24, max 100)

**Sort parameters:**
- `sort_by` (string, field name; allowed values are per-endpoint)
- `sort_dir` (`asc` | `desc`, default `desc`)

**Filter parameters:** defined per endpoint.

**Response `meta`:**
```json
{
  "total": 42,
  "page": 1,
  "page_size": 24,
  "has_next": true,
  "has_prev": false
}
```

---

## 8. Error Shape

All errors return a JSON body:

```json
{
  "error": {
    "code": "DIAGRAM_NOT_FOUND",
    "message": "Diagram with id '...' not found or you do not have access.",
    "detail": null
  }
}
```

`detail` is an optional field for structured error context (e.g., validation errors):

```json
{
  "error": {
    "code": "SPEC_VALIDATION_FAILED",
    "message": "The diagram spec has validation errors.",
    "detail": {
      "errors": [
        { "severity": "error", "code": "DUPLICATE_NODE_ID", "message": "...", "path": "nodes[1].id" }
      ]
    }
  }
}
```

---

## 9. Error Code Taxonomy

### 9.1 HTTP Status → Error Code Mapping

| HTTP | Error Code | When |
|------|-----------|------|
| 400 | `BAD_REQUEST` | Malformed JSON or missing required field |
| 400 | `SPEC_VALIDATION_FAILED` | Diagram spec fails validation (detail included) |
| 400 | `IMPORT_PARSE_FAILED` | Import input cannot be parsed |
| 400 | `IMPORT_UNSUPPORTED_FORMAT` | Import format or Mermaid subtype not supported in V1 |
| 400 | `EXPORT_FORMAT_UNSUPPORTED` | Requested format is not supported for this diagram type |
| 400 | `ALREADY_SUBSCRIBED` | User is already on an active Pro subscription |
| 400 | `PROVIDER_NOT_CONFIGURED` | No provider configured for this user |
| 400 | `PROMPT_TOO_LONG` | Prompt or change request exceeds length limit |
| 401 | `UNAUTHORIZED` | Missing or expired auth token |
| 401 | `TOKEN_EXPIRED` | Access token specifically expired (frontend should refresh) |
| 403 | `FORBIDDEN` | Authenticated but not authorized (wrong owner) |
| 403 | `SUBSCRIPTION_REQUIRED` | Feature requires active subscription |
| 403 | `EMAIL_VERIFICATION_REQUIRED` | Action requires verified email |
| 404 | `DIAGRAM_NOT_FOUND` | — |
| 404 | `PROJECT_NOT_FOUND` | — |
| 404 | `VERSION_NOT_FOUND` | — |
| 404 | `SHARE_NOT_FOUND` | Share token does not exist or has been revoked |
| 404 | `PROVIDER_NOT_FOUND` | — |
| 404 | `TEMPLATE_NOT_FOUND` | — |
| 409 | `CONFLICT` | Duplicate resource (e.g., two default providers) |
| 422 | `UNPROCESSABLE_ENTITY` | Request body passes JSON parse but fails Pydantic validation |
| 422 | `GENERATION_PARSE_FAILED` | LLM response is not parseable as JSON after extraction fallback |
| 422 | `GENERATION_INVALID_SPEC` | LLM spec fails validation after repair attempt |
| 429 | `RATE_LIMITED` | Rate limit exceeded |
| 500 | `INTERNAL_ERROR` | Unhandled server error (never expose stack trace in production) |
| 502 | `PROVIDER_UNAVAILABLE` | AI provider connection failed after 1 retry |
| 503 | `PROVIDER_TIMEOUT` | Provider response took >30s (per-request timeout) |
| 503 | `SERVICE_UNAVAILABLE` | Maintenance mode |

### 9.2 Error Handling in the Frontend

The frontend API client (`src/api/client.ts`) handles:
- `401 TOKEN_EXPIRED`: silent refresh attempt, retry original request once
- `403 SUBSCRIPTION_REQUIRED`: redirect to `/settings/billing` with an upgrade banner
- `502 PROVIDER_UNAVAILABLE`: show provider error in the prompt panel, do not crash
- All others: show an error toast or inline error message appropriate to context

---

## 10. API Endpoint Index

Domain specs define the full contract for each endpoint. This table is the canonical list of all V1 operations.

### Auth (FastAPI Users — no `/api/v1/` prefix)
| Method | Path | Description |
|--------|------|-------------|
| POST | `/auth/register` | Register |
| POST | `/auth/login` | Login, get tokens |
| POST | `/auth/refresh` | Refresh access token |
| POST | `/auth/logout` | Invalidate refresh token |
| POST | `/auth/forgot-password` | Send reset email |
| POST | `/auth/reset-password` | Apply reset |
| POST | `/auth/verify` | Verify email |

### Diagrams (→ D2 for full contract)
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/diagrams` | List user's diagrams |
| POST | `/api/v1/diagrams` | Create diagram |
| GET | `/api/v1/diagrams/{id}` | Get diagram |
| PATCH | `/api/v1/diagrams/{id}` | Update diagram |
| DELETE | `/api/v1/diagrams/{id}` | Archive diagram |
| POST | `/api/v1/diagrams/{id}/render` | Re-render ASCII + Mermaid |
| POST | `/api/v1/diagrams/{id}/sync-ascii` | Sync ASCII label edits back to spec |
| POST | `/api/v1/diagrams/{id}/export` | Export in a format |

### AI Generation (→ D1 for full contract)
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/diagrams/generate` | Generate new diagram from prompt |
| POST | `/api/v1/diagrams/{id}/improve` | AI-modify existing diagram |

### Versions (→ D3 for full contract)
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/diagrams/{id}/versions` | List versions |
| GET | `/api/v1/diagrams/{id}/versions/{vid}` | Get a version |
| PATCH | `/api/v1/diagrams/{id}/versions/{vid}` | Update version change_summary |
| POST | `/api/v1/diagrams/{id}/versions/{vid}/restore` | Restore a version |

### Projects
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/projects` | List projects |
| POST | `/api/v1/projects` | Create project |
| GET | `/api/v1/projects/{id}` | Get project |
| PATCH | `/api/v1/projects/{id}` | Update project |
| DELETE | `/api/v1/projects/{id}` | Archive project |
| GET | `/api/v1/projects/{id}/diagrams` | List project's diagrams |

### Templates
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/templates` | List templates (system + user's) |
| POST | `/api/v1/templates` | Create user template |
| GET | `/api/v1/templates/{id}` | Get template |
| POST | `/api/v1/templates/{id}/use` | Create diagram from template |
| DELETE | `/api/v1/templates/{id}` | Delete user template (not system) |

### Providers
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/providers` | List user's providers |
| POST | `/api/v1/providers` | Add provider |
| GET | `/api/v1/providers/{id}` | Get provider |
| PATCH | `/api/v1/providers/{id}` | Update provider |
| DELETE | `/api/v1/providers/{id}` | Delete provider |
| POST | `/api/v1/providers/{id}/test` | Test provider connection |

### Import (→ D5)
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/diagrams/import` | Import and convert external format |

### Sharing (→ D6)
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/diagrams/{id}/share` | Get active share link status |
| POST | `/api/v1/diagrams/{id}/share` | Create share link |
| DELETE | `/api/v1/diagrams/{id}/share` | Revoke share link |
| GET | `/api/v1/share/{token}` | Public: get shared diagram (no auth) |

### Billing (→ D7)
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/billing/subscription` | Get subscription status |
| POST | `/api/v1/billing/checkout-session` | Create Stripe checkout session |
| POST | `/api/v1/billing/portal-session` | Create Stripe billing portal session |
| POST | `/webhooks/stripe` | Stripe webhook (no auth, signature-verified) |

### Admin (→ A1)
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/admin/users` | List users |
| GET | `/api/v1/admin/users/export` | Export users as CSV |
| GET | `/api/v1/admin/users/{id}` | Get user |
| PATCH | `/api/v1/admin/users/{id}` | Update user |
| PATCH | `/api/v1/admin/users/{id}/subscription` | Override user subscription |
| GET | `/api/v1/admin/audit-log` | List audit events |
| GET | `/api/v1/admin/audit-log/{id}` | Get audit event |

### Health
| Method | Path | Description |
|--------|------|-------------|
| GET | `/health` | Health check (no auth, no `/api/v1/` prefix) |

---

## 11. Background Tasks

V1 uses FastAPI's built-in `BackgroundTasks` for async work that doesn't need to block the response.

**Pattern:**
```python
@router.post("/diagrams/{id}/export")
async def export_diagram(id: UUID, background_tasks: BackgroundTasks, ...):
    task_id = uuid4()
    background_tasks.add_task(run_export, task_id, id, ...)
    return {"data": {"task_id": str(task_id), "status": "queued"}}
```

**When to use BackgroundTasks:**
- Email sending (always async)
- Future long-running exports if they exceed the V1 synchronous budget

**Limitations:** FastAPI BackgroundTasks run in-process. If the server restarts mid-task, the task is lost. This is acceptable for V1 (email fails silently, user can retry export). A proper queue (Celery) is V2 if needed.

**Task status polling:** Not implemented in V1 for exports. V1 exports, including SVG and PNG (DEC-019), are synchronous. PNG is generated from the deterministic SVG export path rather than from a queued headless-browser screenshot workflow.

---

## 12. Rate Limiting

V1 rate limits are enforced at the FastAPI middleware layer using `slowapi`.

| Route group | Limit | Rationale |
|-------------|-------|-----------|
| `POST /auth/login` | 10 req/min per IP | Brute-force protection |
| `POST /auth/register` | 5 req/min per IP | Registration spam |
| `POST /auth/forgot-password` | 5 req/min per IP | Email abuse |
| `POST /diagrams/generate` | 20 req/min per user | AI abuse protection |
| `POST /diagrams/{id}/improve` | 20 req/min per user | — |
| All other routes | 300 req/min per user | General API |

Rate limit exceeded → 429 `RATE_LIMITED` with `Retry-After` header.

---

## 13. CORS

See F0 §6. The `ALLOWED_ORIGINS` env var drives CORSMiddleware config. All API routes must accept CORS pre-flight (`OPTIONS`) requests from allowed origins.

---

## Open Questions

None. All F5 decisions are locked.

---

## Acceptance Criteria

- `GET /api/v1/diagrams` without a token returns `401 {"error": {"code": "UNAUTHORIZED", ...}}`.
- `GET /api/v1/diagrams/{id}` for a diagram owned by a different user returns `403 {"error": {"code": "FORBIDDEN", ...}}`.
- `GET /api/v1/diagrams?page=2&page_size=10` returns at most 10 results with correct `meta.page = 2`.
- `POST /api/v1/diagrams/generate` from a user with `status = 'canceled'` returns `403 SUBSCRIPTION_REQUIRED`.
- `POST /auth/login` attempted 11 times in 1 minute from the same IP returns `429 RATE_LIMITED` on the 11th attempt.
- A request with an expired access token returns `401 TOKEN_EXPIRED` (not the generic `UNAUTHORIZED`).
- `GET /health` returns `200 {"status": "ok"}` without any auth token.
