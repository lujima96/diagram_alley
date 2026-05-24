---
id: F1
tier: foundation
status: draft
version: "0.1"
depends-on: [F0]
tags: [data-model, spec-schema, state-machines, lifecycle]
---

# F1 — Diagram Spec and Data Model

**Version:** 0.1
**Status:** Draft
**Depends on:** F0

---

## Purpose

Define the canonical JSON schema for every diagram type, the `spec_version` migration contract, and the complete database schema. Every domain spec that references a model, table, field, or diagram spec field must cite this document — it does not redefine them.

## Non-Goals

- Rendering rules (→ F2)
- Auth and permissions (→ F3)
- API naming conventions (→ F5)
- Domain-specific workflows (→ D1–D8)

---

## Owned Concepts

`users`, `projects`, `diagrams`, `diagram_versions`, `templates`, `model_providers`, `user_settings`, `diagram_shares`, `subscriptions`, `audit_log` tables; all Diagram Spec JSON schemas; `spec_version` migration contract; diagram lifecycle states.

---

## 1. Diagram Spec — Universal Fields

Every Diagram Spec document, regardless of type, must include these fields:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `spec_version` | string | Yes | Schema version, e.g. `"1.0"`. Checked on every load. |
| `diagram_type` | string enum | Yes | One of the six V1 types (see §2). |
| `title` | string | Yes | Human-readable diagram title. Max 255 chars. |

The backend checks `spec_version` on load via a version-dispatch table. If the stored version is older than current, a migration function is applied before the spec is returned to the client. If the version is unknown, the API returns HTTP 422 with a clear error.

**Current version:** `"1.0"` (V1 initial release)

**Migration contract:**
- Migration functions are registered in `app/services/spec_migration.py`.
- Each function signature: `migrate_<from>_to_<to>(spec: dict) -> dict`
- Migrations are chained. A spec at version 0.9 passes through `0.9 → 1.0` then any further steps.
- Migrations must be backwards-compatible in output (they never remove fields — they add or rename).
- After migration, the stored `spec_json` is updated in the database and `spec_version` is written to the current version.

---

## 2. Diagram Types

Five types are defined in V1 (DEC-035). Each has its own JSON schema (§3). `diagram_type` values are snake_case strings:

| Value | Display name |
|-------|-------------|
| `architecture` | Architecture Diagram |
| `database` | Database Schema |
| `flowchart` | Flowchart |
| `file_structure` | File Structure |
| `network` | Network Diagram |

`ui_wireframe` is deferred to V2 (DEC-035).

---

## 3. Diagram Spec JSON Schemas

### 3.1 Architecture Diagram

```json
{
  "spec_version": "1.0",
  "diagram_type": "architecture",
  "title": "string (required, max 255)",
  "direction": "top_down | left_right",
  "nodes": [
    {
      "id": "string (required, unique within spec, snake_case)",
      "label": "string (required, max 100)",
      "kind": "see node kind enum §3.1.1",
      "group_id": "string | null (references a group id)",
      "position": { "col": 0, "row": 0 }
    }
  ],
  "edges": [
    {
      "id": "string (required, unique within spec)",
      "from": "string (node id, required)",
      "to": "string (node id, required)",
      "label": "string | null",
      "style": "solid | dashed | dotted"
    }
  ],
  "groups": [
    {
      "id": "string (required, unique within spec)",
      "label": "string (required)",
      "node_ids": ["string"]
    }
  ]
}
```

**Direction default:** `"top_down"` if omitted.
**Position:** `col` and `row` are zero-indexed integers. Position is optional — if absent, the ASCII renderer assigns positions automatically.
**Edge style default:** `"solid"` if omitted.

#### 3.1.1 Architecture Node Kind Enum

All named kinds get distinct rendering in V1 (DEC-012). F2 defines the rendering rule for each.

```
user_client
frontend
backend
service
gateway
database
cache
queue
worker
external_api
auth
storage
cdn
message_broker
job_scheduler
search_index
file_blob_storage
third_party_service
internal_service
generic
```

---

### 3.2 Database Schema Diagram

```json
{
  "spec_version": "1.0",
  "diagram_type": "database",
  "title": "string (required)",
  "tables": [
    {
      "id": "string (required, unique, snake_case)",
      "label": "string (required, the table name as displayed)",
      "columns": [
        {
          "name": "string (required)",
          "type": "string (e.g. UUID, TEXT, INTEGER, DECIMAL, DATE, BOOLEAN, JSONB)",
          "primary_key": false,
          "foreign_key": false,
          "nullable": true,
          "unique": false,
          "annotation": "string | null (e.g. PK, FK, INDEX — displayed in ASCII)"
        }
      ],
      "position": { "col": 0, "row": 0 }
    }
  ],
  "relationships": [
    {
      "id": "string (required, unique)",
      "from_table": "string (table id)",
      "from_column": "string (column name)",
      "to_table": "string (table id)",
      "to_column": "string (column name)",
      "type": "one_to_one | one_to_many | many_to_many",
      "label": "string | null"
    }
  ]
}
```

---

### 3.3 Flowchart

```json
{
  "spec_version": "1.0",
  "diagram_type": "flowchart",
  "title": "string (required)",
  "direction": "top_down | left_right",
  "steps": [
    {
      "id": "string (required, unique)",
      "kind": "start | end | process | decision | io",
      "label": "string (required)",
      "position": { "col": 0, "row": 0 }
    }
  ],
  "transitions": [
    {
      "id": "string (required, unique)",
      "from": "string (step id)",
      "to": "string (step id)",
      "label": "string | null (e.g. Yes, No, True)"
    }
  ]
}
```

**Step kind rendering:** `start`/`end` → rounded box; `decision` → diamond; `process` → rectangle; `io` → parallelogram.

---

### 3.4 File Structure

```json
{
  "spec_version": "1.0",
  "diagram_type": "file_structure",
  "title": "string (required)",
  "root": {
    "id": "string (required, unique)",
    "kind": "folder | file",
    "label": "string (required, the path segment name)",
    "annotation": "string | null (shown after label in output)",
    "children": [ "...recursive (folders only have children; files have none)..." ]
  }
}
```

---

### 3.5 Network Diagram

```json
{
  "spec_version": "1.0",
  "diagram_type": "network",
  "title": "string (required)",
  "zones": [
    {
      "id": "string (required, unique)",
      "label": "string (required)",
      "device_ids": ["string"]
    }
  ],
  "devices": [
    {
      "id": "string (required, unique)",
      "kind": "router | firewall | switch | server | client_device | camera | access_point | printer | generic",
      "label": "string (required)",
      "zone_id": "string | null",
      "position": { "col": 0, "row": 0 }
    }
  ],
  "connections": [
    {
      "id": "string (required, unique)",
      "from": "string (device id)",
      "to": "string (device id)",
      "label": "string | null (e.g. Gigabit, VLAN 10)",
      "protocol": "string | null"
    }
  ]
}
```

---

## 4. Database Tables

### 4.1 `users`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | UUID | PK, default gen_random_uuid() | Unique user id |
| `email` | TEXT | NOT NULL, UNIQUE | Login email; lowercased on write |
| `hashed_password` | TEXT | NOT NULL | bcrypt hash via FastAPI Users |
| `is_active` | BOOLEAN | NOT NULL, default true | FastAPI Users field |
| `is_verified` | BOOLEAN | NOT NULL, default false | Email verification state |
| `is_superuser` | BOOLEAN | NOT NULL, default false | Admin flag |
| `name` | TEXT | NULLABLE | Display name |
| `created_at` | TIMESTAMPTZ | NOT NULL, default now() | Account creation time |
| `updated_at` | TIMESTAMPTZ | NOT NULL, default now() | Last profile update |

**Display fallback (DEC-029):** When `name` is null or blank, UI and API display helpers use the lowercased email prefix before `@` as the display name. Avatar initials are derived from `name` when present, otherwise from the email prefix.

**Soft delete:** Users are not hard-deleted. Set `is_active = false` and clear PII on account deletion request (GDPR compliance path).

---

### 4.2 `projects`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | UUID | PK | Unique project id |
| `user_id` | UUID | FK → users.id, NOT NULL | Owner |
| `team_id` | UUID | NULLABLE | V2 forward compat (DEC-009); no FK in V1 |
| `name` | TEXT | NOT NULL, max 255 | Project name |
| `description` | TEXT | NULLABLE | Optional notes |
| `is_archived` | BOOLEAN | NOT NULL, default false | Archived projects are hidden but not deleted |
| `created_at` | TIMESTAMPTZ | NOT NULL, default now() | — |
| `updated_at` | TIMESTAMPTZ | NOT NULL, default now() | — |

**Index:** `(user_id, is_archived)` for dashboard queries.
**Archive UX (DEC-027):** Archived projects are hidden from default project lists, appear in an Archived filter/view, and can be restored by setting `is_archived = false`. V1 does not expose project hard delete as a standalone user action.

---

### 4.3 `diagrams`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | UUID | PK | Unique diagram id |
| `project_id` | UUID | FK → projects.id, NOT NULL | Parent project |
| `team_id` | UUID | NULLABLE | V2 forward compat (DEC-009) |
| `title` | TEXT | NOT NULL, max 255 | Diagram title |
| `diagram_type` | TEXT | NOT NULL | One of five type values (§2) |
| `spec_json` | JSONB | NOT NULL | The canonical Diagram Spec |
| `spec_version` | TEXT | NOT NULL | e.g. `"1.0"` |
| `ascii_cache` | TEXT | NULLABLE | Cached ASCII render (invalidated on spec change) |
| `mermaid_cache` | TEXT | NULLABLE | Cached Mermaid render |
| `ascii_is_detached` | BOOLEAN | NOT NULL, default false | True when ASCII has been manually edited and is no longer in sync with spec_json (F2 §7.3, D2 §7.3) |
| `is_template` | BOOLEAN | NOT NULL, default false | True if this diagram was converted to a template |
| `is_archived` | BOOLEAN | NOT NULL, default false | — |
| `created_at` | TIMESTAMPTZ | NOT NULL, default now() | — |
| `updated_at` | TIMESTAMPTZ | NOT NULL, default now() | — |

**Index:** `(project_id, is_archived, diagram_type)` for library queries.
**Cache invalidation:** `ascii_cache` and `mermaid_cache` are set to NULL whenever `spec_json` is written. The next render call rehydrates the cache.

---

### 4.4 `diagram_versions`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | UUID | PK | Unique version id |
| `diagram_id` | UUID | FK → diagrams.id, NOT NULL | Parent diagram |
| `spec_json` | JSONB | NOT NULL | Snapshot of spec at save time |
| `spec_version` | TEXT | NOT NULL | Schema version at time of snapshot |
| `change_summary` | TEXT | NULLABLE | Human or AI description of what changed |
| `created_by` | UUID | FK → users.id, NOT NULL | Actor who saved |
| `created_at` | TIMESTAMPTZ | NOT NULL, default now() | Version timestamp |

**Index:** `(diagram_id, created_at DESC)` for version list queries.
**Retention (DEC-040):** Free users: versions older than 14 days are pruned automatically (rolling window). Pro users: unlimited retention. Hard limit for both plans: 500 versions per diagram (oldest pruned on write if exceeded).
**Immutability:** Versions are never updated or deleted by normal flows. A restore operation creates a new version with the restored spec; it does not overwrite history.

---

### 4.5 `templates`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | UUID | PK | — |
| `title` | TEXT | NOT NULL, max 255 | Template name |
| `diagram_type` | TEXT | NOT NULL | — |
| `description` | TEXT | NULLABLE | Short explanation of what the template depicts |
| `spec_json` | JSONB | NOT NULL | Starting spec |
| `spec_version` | TEXT | NOT NULL | — |
| `is_system` | BOOLEAN | NOT NULL, default false | True = built-in; false = user-created |
| `created_by` | UUID | FK → users.id, NULLABLE | Null for system templates |
| `is_public` | BOOLEAN | NOT NULL, default false | V2 marketplace — always false in V1 |
| `created_at` | TIMESTAMPTZ | NOT NULL, default now() | — |
| `updated_at` | TIMESTAMPTZ | NOT NULL, default now() | — |

**V1 system templates (`is_system=true`):** The following developer-specific templates ship with V1 to give users immediate value. All are available on both Free and Pro plans:

| Title | `diagram_type` | Description |
|-------|---------------|-------------|
| React + FastAPI + PostgreSQL | `architecture` | Three-tier web app: React frontend, FastAPI service, PostgreSQL database |
| Microservice Architecture | `architecture` | Multiple services with API gateway, message queue, and shared database |
| Authentication Flow | `flowchart` | User login, JWT issuance, token refresh, and logout steps |
| CI/CD Pipeline | `flowchart` | Commit → build → test → staging → production deployment |
| API Request Lifecycle | `flowchart` | HTTP request through middleware, auth, route handler, and response |
| Database ERD (Starter) | `database` | Users, sessions, and audit_log tables with relationships |
| Monorepo File Structure | `file_structure` | Frontend, backend, and shared packages in a monorepo layout |
| Docker Compose App | `architecture` | App container, database container, reverse proxy, and volumes |
| Cloud Deployment | `network` | Load balancer, app servers, managed database, and CDN |

**Pro-only advanced templates:** High-fidelity starting points for complex systems (multi-region deployment, event-driven architecture, full SaaS stack). Added in V1.1+ as the template library grows.

---

### 4.6 `model_providers`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | UUID | PK | — |
| `user_id` | UUID | FK → users.id, NOT NULL | Owner |
| `provider_type` | TEXT | NOT NULL | `openai_compatible \| ollama \| llamacpp` |
| `name` | TEXT | NOT NULL, max 100 | Display name (e.g. "My Ollama") |
| `base_url` | TEXT | NOT NULL | Provider endpoint URL |
| `model` | TEXT | NOT NULL | Model name or ID |
| `api_key_encrypted` | TEXT | NULLABLE | Encrypted BYOK key (see F3 §4 for encryption scheme) |
| `is_default` | BOOLEAN | NOT NULL, default false | One default per user (enforced in service layer) |
| `store_credentials` | BOOLEAN | NOT NULL, default true | User opt-out for credential persistence |
| `temperature` | FLOAT | NULLABLE, default 0.2 | Generation temperature |
| `max_tokens` | INTEGER | NULLABLE | Provider max token override |
| `created_at` | TIMESTAMPTZ | NOT NULL, default now() | — |
| `updated_at` | TIMESTAMPTZ | NOT NULL, default now() | — |

**Constraint:** Only one `is_default = true` per `user_id`. Enforced by a partial unique index: `CREATE UNIQUE INDEX ON model_providers (user_id) WHERE is_default = true`.

---

### 4.7 `user_settings`

One row per user. Created with defaults on user registration.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | UUID | PK | — |
| `user_id` | UUID | FK → users.id, NOT NULL, UNIQUE | One row per user |
| `default_spec_format` | TEXT | NOT NULL, default `'json'` | `json \| yaml` (display preference only; storage is always JSON per DEC-011) |
| `default_export_format` | TEXT | NOT NULL, default `'ascii'` | Preferred export format |
| `default_diagram_type` | TEXT | NULLABLE | Default selection in New Diagram flow |
| `created_at` | TIMESTAMPTZ | NOT NULL, default now() | — |
| `updated_at` | TIMESTAMPTZ | NOT NULL, default now() | — |

---

### 4.8 `diagram_shares`

Stores public share links for diagrams. One active link per diagram enforced at write time (D6 §1.1).

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | UUID | PK | — |
| `diagram_id` | UUID | FK → diagrams.id, NOT NULL | Shared diagram |
| `share_token` | TEXT | NOT NULL, UNIQUE | URL-safe random token, 32 chars |
| `created_by` | UUID | FK → users.id, NOT NULL | User who created the link |
| `created_at` | TIMESTAMPTZ | NOT NULL, default now() | — |
| `revoked_at` | TIMESTAMPTZ | NULLABLE | Set on revocation; NULL = active |
| `access_count` | INTEGER | NOT NULL, default 0 | Access count |
| `last_accessed_at` | TIMESTAMPTZ | NULLABLE | Last access timestamp |

**Index:** `(share_token)` for public lookup; `(diagram_id, revoked_at)` for share status queries.

---

### 4.9 `subscriptions`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | UUID | PK | — |
| `user_id` | UUID | FK → users.id, NOT NULL, UNIQUE | One active subscription per user |
| `plan` | TEXT | NOT NULL | `free \| pro` |
| `status` | TEXT | NOT NULL | `active \| past_due \| canceled \| incomplete` |
| `stripe_customer_id` | TEXT | NULLABLE | Stripe customer object ID |
| `stripe_subscription_id` | TEXT | NULLABLE | Stripe subscription object ID |
| `current_period_start` | TIMESTAMPTZ | NULLABLE | Billing period start (null for free tier) |
| `current_period_end` | TIMESTAMPTZ | NULLABLE | Billing period end |
| `cancel_at_period_end` | BOOLEAN | NOT NULL, default false | True when Pro access remains active until `access_ends_at` after cancellation |
| `access_ends_at` | TIMESTAMPTZ | NULLABLE | Time paid/trial access ends; for cancel-at-period-end this equals `current_period_end` |
| `canceled_at` | TIMESTAMPTZ | NULLABLE | Cancellation timestamp |
| `created_at` | TIMESTAMPTZ | NOT NULL, default now() | — |
| `updated_at` | TIMESTAMPTZ | NOT NULL, default now() | — |

**Free tier logic (DEC-034):** New users start with `plan='free', status='active'`. No Stripe interaction required. Free tier is permanent and has no expiry. Feature limits (5 private diagrams, 14-day version history, Mermaid/PNG/SVG exports only) are enforced by the feature gate in F3 §5.5.

**Upgrade logic:** When a user subscribes to Pro via Stripe checkout, `plan` updates to `'pro'` and `status` updates to `'active'` via webhook.

**Cancellation and downgrade logic (DEC-025, DEC-034):** Pro cancellation at period end keeps `status = 'active'`, sets `cancel_at_period_end = true`, and sets `access_ends_at = current_period_end`. When `access_ends_at` is reached, `plan` reverts to `'free'` and `status` returns to `'active'`. Diagrams created during Pro that exceed the free tier private limit become read-only; they are not deleted.

---

### 4.10 `audit_log`

Stores append-only audit events. F3 owns the event constants and write convention; F1 owns this table shape (DEC-024).

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | UUID | PK | — |
| `user_id` | UUID | FK → users.id, NULLABLE | Actor; null for system or anonymous events |
| `event` | TEXT | NOT NULL | Event constant owned by F3 |
| `entity_type` | TEXT | NOT NULL | Table/entity name, e.g. `diagrams` |
| `entity_id` | UUID | NULLABLE | Affected row ID when applicable |
| `detail` | JSONB | NOT NULL, default `{}` | Event-specific context |
| `ip_address` | TEXT | NULLABLE | Request IP when logged |
| `created_at` | TIMESTAMPTZ | NOT NULL, default now() | Event timestamp |

**Index:** `(created_at DESC)` for admin audit list; `(user_id, created_at DESC)` for user support views; `(entity_type, entity_id)` for entity history lookup.

**Immutability:** Audit log rows are append-only. No update or delete operations are exposed in V1.

---

## 5. Lifecycle States

### 5.1 Diagram States

```
created (spec_json set, not yet saved to library)
    ↓
saved (stored in diagrams table, visible in library)
    ↓
archived (is_archived = true, hidden from library but not deleted)
    ↓
deleted (hard delete, performed only by user via account deletion flow)
```

Project lifecycle:

```
active (visible in default project lists)
    ↓
archived (is_archived = true, hidden by default, visible in Archived filter/view)
    ↓
active (restored by setting is_archived = false)
```

Projects have no standalone hard-delete transition in V1 (DEC-027).

A diagram in `created` state only exists in the browser — it has not been persisted. The API creates a persisted diagram on first explicit save, successful AI generation, or import.

### 5.2 Subscription States

Free tier is the entry state; there is no time-limited trial (DEC-034).

```
free     → pro          (user completes Stripe checkout; plan='pro', status='active')
pro      → past_due     (payment fails; Stripe retries)
past_due → pro          (payment retry succeeds)
past_due → free         (all retries exhausted; plan reverts to 'free', status='active')
pro      → pro          (user cancels at period end; cancel_at_period_end=true, status stays 'active')
pro      → free         (access_ends_at reached; plan reverts to 'free', status='active')
free     → pro          (canceled user resubscribes)
```

Note: `status='canceled'` is only a transient Stripe webhook state during Pro-to-free transition; once reverted the row settles to `plan='free', status='active'`.

### 5.3 Plan Feature Gate (DEC-034)

Enforcement logic lives in F3 §5.3. The table below defines the limits that F3 gates against:

| Feature | Free (`plan='free'`) | Pro (`plan='pro'`) |
|---------|----------------------|--------------------|
| Private diagrams | 5 (hard cap) | Unlimited |
| Public diagrams | Unlimited | Unlimited |
| Basic exports (Mermaid, PNG, SVG) | Yes | Yes |
| ASCII/Markdown copy (with attribution) | Yes | Yes (clean, no attribution) |
| ASCII/Markdown file download | No | Yes |
| Blueprint exports (JSON, YAML) | No | Yes |
| Documentation Bundle (ZIP) | No | Yes |
| GitHub commit | No | Yes |
| BYOK AI generation | Yes | Yes |
| Version history | 14-day rolling window | Unlimited |
| Public share links | Yes (Diagram Alley branded) | Yes (unbranded) |
| Advanced templates | No | Yes |

When a free user reaches 5 private diagrams, diagram creation is blocked at the API layer and a `USER_FREE_TIER_LIMIT_REACHED` audit event is emitted. Existing diagrams are never deleted.

---

### 5.4 Version Snapshot Triggers

A new `diagram_versions` row is created when:
1. User clicks "Save" explicitly.
2. A new AI-generated diagram is accepted and persisted.
3. User triggers an AI modification and accepts the result.
4. User restores a previous version (creates pre-restore and restored snapshots).

A version is **not** created on every keystroke, every render call, or on live auto-save. Auto-save (D2 §5.3) persists the spec via PATCH without creating a version row (DEC-021). Free tier users have a 14-day rolling window; versions older than 14 days are pruned (DEC-040). Pro users have unlimited retention up to the 500-version hard limit.

---

## 6. Migration Checklist

When this spec changes version (e.g., 1.0 → 1.1):

- [ ] Write a migration function in `app/services/spec_migration.py`
- [ ] Register the function in the version dispatch table
- [ ] Write a migration test with a real V1.0 fixture
- [ ] Update the `spec_version` default in all schema definitions
- [ ] Run `alembic revision` if the DB schema changed
- [ ] Update SPEC-INDEX.md status

---

## Open Questions

None. All field names in §3 have been verified consistent with F2 validation rules (all F2 path references match F1 schema fields with no ambiguity). F1 field names and the migration function contract are frozen for Draft 0.1 by DEC-018.

---

## Acceptance Criteria

- `alembic upgrade head` applies all migrations cleanly against a fresh Postgres database.
- A V1.0 architecture spec loaded from `diagrams.spec_json` passes `spec_version` check and reaches the renderer without error.
- Creating a user, project, diagram, and version via the API produces rows in all four tables with correct foreign keys.
- `is_default = true` on `model_providers` is unique per user (partial index enforced; second default insert raises a database error).
- A `subscriptions` row with `plan = 'free'` and `status = 'active'` is created automatically on user registration (no `trial_ends_at`).
- A free user who creates a 6th private diagram receives a `403 DIAGRAM_LIMIT_REACHED` error; no diagram row is created.
