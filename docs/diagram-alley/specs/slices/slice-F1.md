---
phase: 11
slice: F1
spec: specs/F1-diagram-spec-and-data-model.md
status: planned
---

# Slice F1 — Diagram Spec and Data Model

## Pre-conditions

- [ ] Slice F0 complete — local Postgres is running, `alembic upgrade head` works against an empty database, FastAPI app starts cleanly.

---

## Build Sequence

1. **SQLAlchemy ORM models** — Create one model class per table in `app/models/`: `User`, `Project`, `Diagram`, `DiagramVersion`, `Template`, `ModelProvider`, `UserSettings`, `DiagramShare`, `Subscription`, `AuditLog`. Match every column, type, nullable constraint, and default from F1 §4. Include the partial unique index on `model_providers (user_id) WHERE is_default = true`. (→ F1 §4)

2. **Initial Alembic migration** — Run `alembic revision --autogenerate -m "initial schema"`. Review the generated file: confirm all ten tables, all foreign keys, all indexes, and the partial unique index are present. Run `alembic upgrade head` against local Postgres and verify with `\dt` in psql. (→ F1 §4, F0 §2.3)

3. **Pydantic spec schemas** — Create `app/schemas/spec/` with one Pydantic model per diagram type matching the JSON shapes in F1 §3: `ArchitectureSpec`, `DatabaseSpec`, `FlowchartSpec`, `FileStructureSpec`, `NetworkSpec` (five types; `ui_wireframe` is V2 per DEC-035). Each must enforce the universal fields (`spec_version`, `diagram_type`, `title`) and type-specific required fields. (→ F1 §1, §3)

4. **`spec_version` migration registry** — Create `app/services/spec_migration.py` with an empty version dispatch table (`{"1.0": None}` — no migrations needed yet) and a `migrate_spec(spec: dict) -> dict` function that chains migrations to the current version. If `spec_version` is unknown, raise a `SpecVersionError` (caught by the API layer as HTTP 422). (→ F1 §1)

5. **Display name helper** — Add a `display_name(user: User) -> str` utility function that returns `user.name` if non-null and non-blank, otherwise returns the lowercased email prefix before `@`. Used in any API response or UI surface that shows a user's name. (→ F1 §4.1, DEC-029)

6. **Pydantic request/response schemas** — Create `app/schemas/` models for the API layer (separate from spec schemas): `DiagramRead`, `DiagramCreate`, `ProjectRead`, `ProjectCreate`, `VersionRead`, `SubscriptionRead`, etc. These wrap ORM rows into API-safe shapes. `DiagramRead` includes `ascii_cache`, `mermaid_cache`, `ascii_is_detached`. (→ F1 §4.3, F5 §6)

7. **Stub CRUD services** — Create `app/services/diagram_service.py`, `project_service.py`, `version_service.py` with stub implementations for create, read, update, and archive. Services must enforce ownership (user_id checks) and the archive-only lifecycle (no hard delete for projects in V1). Wire the `audit_log` write helper stub here; actual event constants come from F3. (→ F1 §5.1, DEC-027)

8. **`subscriptions` row on registration** — Add a post-registration hook (FastAPI Users `on_after_register`) that creates a `subscriptions` row with `plan='free'`, `status='active'` (no `trial_ends_at`). Also creates a `user_settings` row with defaults. No Stripe interaction at registration. (→ F1 §4.9, DEC-034)

9. **Version pruning enforcement** — Add `prune_if_over_limit(diagram_id, db, user: User)` to `version_service.py`: check the user's plan to determine the effective limit (free = 10, pro = 100). If count ≥ limit, delete the oldest row before inserting the new one. Called at write time before every version insert. (→ F1 §4.4, §5.3, DEC-034)

10. **Smoke test: full row creation via API** — Write a test (pytest + asyncpg test client) that registers a user, creates a project, creates a diagram with a V1.0 architecture spec, saves a version, and asserts all rows exist with correct foreign keys and that `subscriptions` has `plan='free'`, `status='active'`. (→ F1 Acceptance Criteria)

---

## Done Criteria

- [ ] `alembic upgrade head` applies all migrations cleanly against a fresh Postgres database with no errors. (→ F1 Acceptance Criteria)
- [ ] All ten tables exist with correct columns, types, nullability, and defaults matching F1 §4. Confirmed via `\dt` and `\d <table>` in psql.
- [ ] A V1.0 architecture spec loaded from `diagrams.spec_json` passes the `spec_version` check and is returned to the API layer without error. (→ F1 Acceptance Criteria)
- [ ] Creating a user, project, diagram, and version via the API produces rows in all four tables with correct foreign keys. (→ F1 Acceptance Criteria)
- [ ] `is_default = true` on `model_providers` is unique per user — a second default insert raises a database integrity error. (→ F1 Acceptance Criteria)
- [ ] A `subscriptions` row with `plan = 'free'`, `status = 'active'`, and no `trial_ends_at` is created automatically on user registration. (→ F1 Acceptance Criteria, DEC-034)
- [ ] A free user creating a 6th private diagram receives `403 DIAGRAM_LIMIT_REACHED`; no diagram row is created. (→ F1 §5.3, DEC-034)
- [ ] `display_name()` returns the email prefix when `users.name` is null. (→ F1 §4.1, DEC-029)
- [ ] `spec_version` unknown value returns HTTP 422 with a clear error message. (→ F1 §1)
- [ ] `prune_if_over_limit` with a free user prunes at 10 versions; with a Pro user prunes at 100. Confirmed by unit test. (→ F1 §4.4, §5.3)
