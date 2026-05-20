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

`users`, `projects`, `diagrams`, `diagram_versions`, `templates`, `model_providers`, `user_settings`, `subscriptions` tables; all Diagram Spec JSON schemas; `spec_version` migration contract; diagram lifecycle states.

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

Six types are defined in V1. Each has its own JSON schema (§3). `diagram_type` values are snake_case strings:

| Value | Display name |
|-------|-------------|
| `architecture` | Architecture Diagram |
| `database` | Database Schema |
| `flowchart` | Flowchart |
| `ui_wireframe` | UI Wireframe |
| `file_structure` | File Structure |
| `network` | Network Diagram |

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

### 3.4 UI Wireframe

```json
{
  "spec_version": "1.0",
  "diagram_type": "ui_wireframe",
  "title": "string (required)",
  "root": {
    "id": "string (required, unique within spec)",
    "kind": "page | header | sidebar | main | footer | card | table | form | button | input | text | container | nav",
    "label": "string (required)",
    "layout": "column | row | null",
    "children": [ "...recursive component objects..." ]
  }
}
```

**Component tree:** The `root` is always `kind: page`. Children can be nested to any depth but the renderer only supports meaningful depth ≤ 4 in V1. Deeper nesting is accepted without error but may not render readably.

---

### 3.5 File Structure

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

### 3.6 Network Diagram

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

---

### 4.3 `diagrams`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | UUID | PK | Unique diagram id |
| `project_id` | UUID | FK → projects.id, NOT NULL | Parent project |
| `team_id` | UUID | NULLABLE | V2 forward compat (DEC-009) |
| `title` | TEXT | NOT NULL, max 255 | Diagram title |
| `diagram_type` | TEXT | NOT NULL | One of six type values (§2) |
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
**Retention:** No automatic pruning in V1. Soft limit of 100 versions per diagram (enforced at write time; oldest version is pruned if over limit).
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
| `plan` | TEXT | NOT NULL | `trial \| pro` |
| `status` | TEXT | NOT NULL | `trialing \| active \| past_due \| canceled \| incomplete` |
| `stripe_customer_id` | TEXT | NULLABLE | Stripe customer object ID |
| `stripe_subscription_id` | TEXT | NULLABLE | Stripe subscription object ID |
| `trial_ends_at` | TIMESTAMPTZ | NULLABLE | Trial expiry timestamp |
| `current_period_start` | TIMESTAMPTZ | NULLABLE | Billing period start |
| `current_period_end` | TIMESTAMPTZ | NULLABLE | Billing period end |
| `canceled_at` | TIMESTAMPTZ | NULLABLE | Cancellation timestamp |
| `created_at` | TIMESTAMPTZ | NOT NULL, default now() | — |
| `updated_at` | TIMESTAMPTZ | NOT NULL, default now() | — |

**Trial logic:** `status = 'trialing'` and `trial_ends_at` is set 14 days from registration (DEC-010). On expiry, a Stripe webhook fires and updates `status` to `canceled` unless the user has upgraded.

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

A diagram in `created` state only exists in the browser — it has not been persisted. The API creates a persisted diagram on first explicit save or on first AI generation (auto-save).

### 5.2 Subscription States

```
trialing → active       (user enters payment details and subscribes)
trialing → canceled     (trial expires without payment)
active   → past_due     (payment fails; Stripe retries)
past_due → active       (payment retry succeeds)
past_due → canceled     (all retries exhausted)
active   → canceled     (user cancels)
canceled → active       (user resubscribes)
```

### 5.3 Version Snapshot Triggers

A new `diagram_versions` row is created when:
1. User clicks "Save" explicitly.
2. User triggers an AI modification and accepts the result.
3. User restores a previous version (creates a new version with the restored spec).

A version is **not** created on every keystroke, every render call, or on auto-save. Auto-save (D2 §5.3) fires after 30 seconds of inactivity and persists the spec via PATCH without creating a version row.

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

None. All field names in §3 have been verified consistent with F2 validation rules (all F2 path references match F1 schema fields with no ambiguity). F1 field names are frozen.

---

## Acceptance Criteria

- `alembic upgrade head` applies all migrations cleanly against a fresh Postgres database.
- A V1.0 architecture spec loaded from `diagrams.spec_json` passes `spec_version` check and reaches the renderer without error.
- Creating a user, project, diagram, and version via the API produces rows in all four tables with correct foreign keys.
- `is_default = true` on `model_providers` is unique per user (partial index enforced; second default insert raises a database error).
- A `subscriptions` row with `status = 'trialing'` and `trial_ends_at = now() + 14 days` is created automatically on user registration.
