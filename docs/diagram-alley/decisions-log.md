---
phase: 0
artifact: decisions-log
status: active
ordering: reverse-chronological (newest entry first)
---

# Diagram Alley — Decisions Log

This file is the canonical record of all user-approved decisions. It **overrides** specs and `OUTLINE.md` when there is a conflict. The spec should be updated to match the log, not the other way around.

**Rules:**
- Entries are in reverse-chronological order (newest first).
- Rejected options are recorded to prevent repeat debates.
- Superseded decisions get a new entry; the old entry is not deleted.
- Deferred decisions record what triggers revisiting them.

---

## 2026-05-22 — Phase 10 Mockup Audit Reconciliation

### DEC-030: Visual Editor — File Structure and UI Wireframe Rendering

**Decision:** V1 visually renders `file_structure` diagrams as tree rows and `ui_wireframe` diagrams as nested boxes in the React Flow editor. These diagram types are editable in the visual editor; they are not reduced to read-only ASCII/text previews.

**Source:** Phase 10 mockup audit reconciliation, user decision 2026-05-22.

**Rationale:** This keeps all V1 diagram types visually editable and preserves the product promise of a structured visual workspace across supported diagram types.

**Affects:** F2 (visual renderer behavior), F4 (visual editor UI behavior)

---

### DEC-029: Display Name Fallback — Email Prefix

**Decision:** When `users.name` is null or blank, the display name fallback is the lowercased email prefix before `@`. Avatar initials are derived from the display name when present, otherwise from the email prefix.

**Source:** Phase 10 mockup audit reconciliation, user decision 2026-05-22.

**Rationale:** Email prefix fallback is deterministic, avoids signup friction, and keeps identity surfaces readable.

**Affects:** F1 (user display semantics), F3 (account/profile display), D6 (public share display)

---

### DEC-028: Diagrams Can Move Between Projects in V1

**Decision:** Users can move diagrams between their projects in V1. The move uses the existing `PATCH /api/v1/diagrams/{id}` `project_id` field and appears in the UI as a diagram menu action. No dedicated move endpoint is required.

**Source:** Phase 10 mockup audit reconciliation, user decision 2026-05-22.

**Rationale:** The schema and API already support `project_id` updates, and moving diagrams prevents project assignment mistakes from becoming permanent.

**Affects:** D2 (diagram update behavior and UI action), F4 (project/library UI)

---

### DEC-027: Project Archive UX — Restore, No Hard Delete

**Decision:** V1 supports project archive and restore. Archived projects are hidden from default project lists, can be viewed in an Archived filter/view, and can be restored. V1 does not expose project hard delete as a standalone user action.

**Source:** Phase 10 mockup audit reconciliation, user decision 2026-05-22.

**Rationale:** This matches the soft-delete data model, gives users an undo path, and avoids destructive project deletion scope in V1.

**Affects:** F1 (`projects` lifecycle), F4 (`/projects` UI), F5 (project update/archive API)

---

### DEC-026: V1 Templates — System and User-Created Private Templates

**Decision:** V1 supports both built-in system templates and user-created private templates. User-created templates are owned by the creating user through `templates.created_by`; system templates have `created_by = null`. The `templates.is_public` field remains reserved for a future marketplace and is always `false` in V1.

**Source:** Phase 10 mockup audit reconciliation, user-approved follow-up 2026-05-22.

**Rationale:** F5 already defines user template create/delete endpoints, and the mockups include a "My Templates" area. Recording the decision removes ambiguity without introducing marketplace scope.

**Affects:** F1 (`templates` table), F4 (`/templates` UI), F5 (template endpoints), D2 (new diagram from template)

---

## 2026-05-20 — Audit Reconciliation Decisions

### DEC-025: Stripe Cancellation — Active Until Access Ends

**Decision:** Stripe cancellation uses Stripe's native cancel-at-period-end model. A Pro subscription remains `status = 'active'` while the user retains access. The subscription row records `cancel_at_period_end = true` and `access_ends_at = current_period_end`. When access actually ends, the subscription becomes `status = 'canceled'`.

**Source:** Documentation audit resolution, user decision 2026-05-20.

**Rationale:** This matches Stripe semantics without inventing a separate `canceling` status or treating canceled users as active through side conditions.

**Affects:** F1 (`subscriptions` fields and state machine), F3 (webhook handling and feature gate), D7 (cancellation UX and access rules), A1 (subscription override fields)

---

### DEC-024: Ownership — F1 Owns Tables; F3 Owns Audit Events

**Decision:** F1 owns every database table definition. F3 owns every audit event constant and audit-write convention. Domain specs may define domain behavior and request additions, but they must reference F1/F3 rather than redefining tables or audit events.

**Source:** Documentation audit resolution, user decision 2026-05-20.

**Rationale:** Central table ownership in F1 and central audit-event ownership in F3 best satisfies the "one owner per concept" rule and prevents drift between domain specs and foundations.

**Affects:** SPEC-INDEX, F1, F3, D1, D4, D5, D6, D7, A1

---

### DEC-023: Text Output Modes — Strict ASCII plus UTF-8 Text

**Decision:** Diagram Alley supports two text rendering modes: strict ASCII mode and UTF-8 text mode. Strict ASCII mode uses printable ASCII only. UTF-8 text mode may use Unicode box-drawing/tree characters when they materially improve readability. Specs must not call UTF-8 text output "ASCII output."

**Source:** Documentation audit resolution, user decision 2026-05-20.

**Rationale:** Documentation workflows often need strict ASCII, but file trees and similar structures are significantly more readable with UTF-8 tree glyphs.

**Affects:** glossary, F2, D4, share/export wording

---

### DEC-022: Expired Trial Access — Read, Export, and Share Only

**Decision:** After trial expiry (`status = 'canceled'`), users can view existing saved diagrams, export them, and manage share links. They cannot edit `spec_json`, use AI generation/modification, or create new diagrams until upgrading.

**Source:** Documentation audit resolution, user decision 2026-05-20.

**Rationale:** This preserves access to the user's existing work while keeping continued creation and editing behind an active trial or Pro subscription.

**Affects:** F3 (feature gate), D2 (PATCH/edit gate), D6 (share remains allowed), D7 (trial-expiry UX)

---

### DEC-021: Auto-Save — Persist Edits Without Versioning

**Decision:** Editor changes are auto-saved to the live `diagrams.spec_json` but do not create version snapshots. The Save action means "create a version snapshot" after flushing any pending live persistence.

**Source:** Documentation audit resolution, user decision 2026-05-20.

**Rationale:** This protects user work without flooding version history. Version history remains meaningful because snapshots are deliberate milestones.

**Affects:** F1 (snapshot triggers), D2 (dirty state, auto-save, manual Save), D3 (version snapshot rules)

---

### DEC-020: Initial AI Generation Creates a Version Snapshot

**Decision:** A successful new AI generation creates the persisted diagram row and immediately creates the initial `diagram_versions` snapshot.

**Source:** Documentation audit resolution, user decision 2026-05-20.

**Rationale:** AI-generated diagrams are persisted immediately, so the first accepted generated spec should have a history anchor before later edits occur.

**Affects:** F1 (snapshot triggers), D1 (generation flow), D3 (version snapshot rules)

---

### DEC-019: V1 Exports Include SVG and PNG

**Decision:** V1 includes both SVG and PNG visual exports, in addition to ASCII/text, Markdown, Mermaid, JSON, and YAML. PDF remains deferred to V2.

**Source:** Documentation audit resolution, user decision 2026-05-20.

**Rationale:** PNG is part of the paid V1 value proposition and is useful for wikis, slide decks, and documentation systems that do not embed SVG cleanly.

**Affects:** F0 (render/export dependencies), F2 (visual export renderer), F4 (export controls), F5 (background/export standards), D4 (export format contract), README/SPEC-INDEX

---

### DEC-018: F1 Schema and Migration Contract Frozen for Draft 0.1

**Decision:** The F1 Draft 0.1 JSON schema field names, nesting, required fields, and `spec_version` migration function contract are frozen for current planning. The earlier F1 open question is resolved.

**Source:** Documentation audit resolution, user decision 2026-05-20.

**Rationale:** F1 already defines concrete schemas and the migration contract. Downstream specs are written against those shapes and should no longer treat them as open.

**Affects:** README (open questions), F1 (open questions), DEC-011 and DEC-008 open notes, all domain specs that consume data shapes

---

## 2026-05-20 — Phase 1 Pre-Spec Decisions

### DEC-017: Authentication — JWT via FastAPI Users

**Decision:** Authentication uses **FastAPI Users** with JWT access + refresh tokens. Access tokens are short-lived (15 min). Refresh tokens rotate on use and are stored in the database. No server-side session store required.

**Source:** User decision 2026-05-20.

**Rationale:** FastAPI Users is batteries-included for FastAPI (registration, login, password reset, email verify, JWT lifecycle). Stateless access tokens fit the SPA + REST API architecture. Rotating refresh tokens prevent token theft without session store complexity.

**Affects:** F3 (auth setup, token model, password reset flow, email verification), F5 (auth header convention on every route)

---

### DEC-016: Frontend Hosting — Vercel Free Tier

**Decision:** The built Vite + React frontend is hosted on **Vercel** free tier. Vercel handles deploys from git push, serves the static bundle via global CDN, and generates preview URLs per branch.

**Source:** User decision 2026-05-20.

**Rationale:** Zero-config Vite support, global CDN, branch previews. Industry standard for React/Vite SPA hosting.

**Affects:** F0 (deployment targets, env model, CORS config for FastAPI)

**Constraint:** FastAPI must allow CORS origins from the Vercel production and preview domains. Preview URL patterns must be whitelisted in F0's env model.

---

### DEC-015: FastAPI Backend Hosting — Fly.io Free Tier

**Decision:** The FastAPI backend is hosted on **Fly.io** free tier (3 shared-CPU VMs, no enforced spin-down). FastAPI runs in a Docker container on Fly.io. Secrets are injected as Fly.io app secrets (env vars).

**Source:** User decision 2026-05-20. Extends DEC-014 (Neon Postgres + env var secrets).

**Rationale:** Fly.io free tier provides persistent VMs with ~1s wake time (not 30s+ like Render). Docker-based deploys give full control over the runtime. Pairs with Neon (DEC-014) for Postgres.

**Affects:** F0 (deployment targets, Dockerfile, fly.toml, env vars), F3 (secret injection model)

**Rejected:** Render free tier (30s+ cold start unacceptable for a commercial product); self-hosted VPS (ops overhead for a solo founder).

---

## 2026-05-20 — Open Questions Resolved

### DEC-014: Production Secret Management and Cloud Postgres — Cost-Free

**Decision:** Production Postgres will use **Neon** free tier (serverless Postgres, no activity-based pausing, branching support for dev/staging). Secrets are managed as environment variables injected by the hosting platform — no dedicated secrets manager in V1. A dedicated manager (e.g. AWS Secrets Manager, Doppler) is deferred until scale or compliance requires it.

**Source:** OQ-005, user decision 2026-05-20.

**Rationale:** Zero ongoing cost for a pre-revenue V1. Neon's free tier does not pause on inactivity (unlike Supabase free), which avoids cold-start surprises. Env-var secrets are sufficient when the team is one person with controlled deployment access.

**Affects:** F0 (deployment targets, env model, Postgres provider), F3 (credential storage model)

**Deferred:** Dedicated secrets manager and secret rotation — trigger: compliance requirement or team expansion in V2.

**Rejected:** Railway (has a small free credit cap, not truly free long-term); Render (free Postgres discontinued); AWS RDS + Secrets Manager (cost and ops complexity unjustified at V1 scale).

---

### DEC-013: Visual Grid Editor Library — React Flow

**Decision:** The visual grid editor is built with **React Flow**. No custom SVG rendering or canvas-based approach in V1.

**Source:** OQ-004, user decision 2026-05-20.

**Rationale:** React Flow handles drag, selection, edge routing, zoom, and minimap out of the box. Largest community and best docs among node-edge graph libraries for React. Minimizes custom rendering work before the engine is proven.

**Affects:** F4 (grid editor architecture section), D2 (editor interactions)

**Rejected:** Konva.js (canvas-based, more engineering investment before parity); custom SVG (maximum engineering cost, only justified if React Flow proves inadequate after prototyping).

**Constraint:** React Flow grid coordinates must map to ASCII layout positions deterministically. F4 must define this mapping before D2 implementation begins.

---

### DEC-012: Architecture Node Kind Rendering — All Named Kinds Distinct in V1

**Decision:** All named architecture node kinds get visually distinct rendering in V1. No kinds collapse to a generic labeled box.

**Source:** OQ-003, user decision 2026-05-20.

**Rationale:** User chose maximum fidelity. Full kind coverage from V1 avoids a V2 "visual overhaul" and gives users meaningful kind semantics immediately.

**Affects:** F2 (must enumerate all ~20 node kinds and their distinct rendering rules before implementation)

**Constraint:** F2 must define the complete rendering treatment for every node kind before any renderer work begins. This is a non-trivial scope commitment — the renderer spec must be thorough.

---

### DEC-011: Diagram Spec Primary Format — JSON; YAML as Alternate View

**Decision:** Diagrams are stored as **JSON** in the database (`spec_json` column). The UI can display and edit specs as YAML (re-serialized on the fly). YAML is a presentation layer, not a second storage format. `spec_version` migration logic operates on the JSON representation.

**Source:** OQ-002, user decision 2026-05-20.

**Rationale:** Single canonical storage format simplifies migration, diff, patch, and backup logic. JSON is unambiguous in Python and TypeScript. YAML display is a UX convenience, not an engineering constraint.

**Affects:** F1 (spec schema definition, migration logic, `spec_version` field), all domain specs that reference spec format

**Rejected:** YAML primary storage (more complex migration code paths); dual storage per diagram (two migration paths, more edge cases).

**Resolved by DEC-018:** The exact JSON schema structure and `spec_version` migration path are frozen for F1 Draft 0.1.

---

### DEC-010: Free Trial Limits — BYOK-First (14 Days, Unlimited Diagrams, No Hosted AI)

**Decision:** The free trial is:
- **Duration:** 14 days
- **Saved diagrams:** Unlimited during trial
- **Hosted AI credits:** None — users must connect their own API key (BYOK) or a local model (Ollama/llama.cpp) to use AI generation
- **Export access:** Full (all formats available during trial)
- **Editor access:** Full (visual grid editor, structured editor, JSON/YAML editor)

**Source:** OQ-001, user decision 2026-05-20.

**Rationale:** Zero hosted AI cost per trial user. Filters for users who already have API access or are willing to set up Ollama — the exact audience most likely to pay. Unlimited diagrams lets the trial prove real workflow value rather than artificial limits.

**Affects:** F3 (Stripe product/price config, trial enforcement), D7 (billing domain spec), onboarding flow (must surface BYOK/Ollama setup as step 1 for AI features)

**Rejected:** Hosted AI credits in trial (cost risk before revenue); hard diagram count limits (creates artificial friction before the user can prove value to themselves).

---

## 2026-05-20 — Phase 0 Seeding

### DEC-009: Team Account Forward Compatibility

**Decision:** The `projects` and `diagrams` tables include a nullable `team_id` column from the start. V1 ignores this column entirely. V2 will populate it when team workspaces are introduced.

**Source:** `OUTLINE.md` §13 (Data Model Overview — Team Account Forward Compatibility note)

**Rationale:** Avoids a painful schema migration when V2 team features are added, at zero V1 cost.

**Affects:** F1 (`projects` table, `diagrams` table)

**Consequences:** No team-scoped queries in V1. The column is nullable and unindexed until V2 uses it.

**Deferred:** Full team account model (shared workspaces, role permissions, org-level settings) — trigger: V2 after Paid V1 launch.

---

### DEC-008: Spec Version Field Required on Every Diagram Spec

**Decision:** Every Diagram Spec document must include a `spec_version` field (e.g. `"spec_version": "1.0"`). The backend must check this field on load and either migrate the spec to the current version or return a clear error.

**Source:** `OUTLINE.md` §13 (Data Model Overview — Spec Versioning Strategy)

**Rationale:** Prevents old diagrams from breaking silently as the spec format evolves.

**Affects:** F1 (spec schema definition and migration logic)

**Resolved by DEC-018:** Exact JSON schema format and migration path are frozen for F1 Draft 0.1.

---

### DEC-007: GitHub Integration is a Stretch Goal for Paid V1

**Decision:** GitHub commit/PR workflows are a stretch goal for Paid V1 — included only if the core editor, renderer, version history, and exports are solid before launch; otherwise deferred to V2. Copy/download/export must work first and are the primary V1 delivery path.

**Source:** `OUTLINE.md` §1.1 (Integration Direction), §18.4 (Paid V1 features)

**Rationale:** GitHub integration adds OAuth complexity and scope risk. The core product value (generate → edit → export) does not depend on it.

**Affects:** D8 (write last; depends on D1–D7 and A1 being complete)

**Deferred:** Repo scanning, AI-suggested diagrams from repo structure — trigger: V2.

---

### DEC-006: Mermaid Import Scope — Best-Effort Flowchart and ER Only

**Decision:** V1 Mermaid import supports flowchart and ER diagram types only, at best-effort, with clear warnings for unsupported constructs. Full-fidelity Mermaid import is not promised at launch.

**Source:** `OUTLINE.md` §7.14 (Import and Conversion — Engineering Risk: Mermaid Import)

**Rationale:** Mermaid's syntax is irregular across diagram types. Parsing it into a structured Diagram Alley spec is a non-trivial engineering investment. Scope is limited to prevent silent corruption.

**Affects:** D5 (import spec must document warning behavior and supported constructs)

**Rejected:** Full Mermaid import fidelity at V1 — too much scope risk; silent corruption risk is unacceptable.

**Deferred:** Full Mermaid import across all diagram types — trigger: V2, after the import engineering cost is better understood.

---

### DEC-005: ASCII Reverse Sync Scope — Label Changes Only in V1

**Decision:** ASCII reverse sync in V1 is limited to label changes only. If a user edits a node or edge label in the ASCII output, the app will attempt to sync that change back into the structured spec. All other ASCII edits (adding nodes, removing nodes, changing connections, restructuring layout) produce a Detached Export with a warning. The spec is not updated for non-label ASCII edits.

**Source:** `OUTLINE.md` §1.1 (Closed Technical Decisions — ASCII reverse sync scope), §16.1.1

**Rationale:** General ASCII parsing is a deep engineering rabbit hole. The V1 boundary is firm to avoid scope explosion.

**Affects:** F2 (reverse sync rules and detached export state), D2 (editor behavior)

**Rejected:** Full reverse sync (add/remove nodes, connection changes) — trigger to revisit: V2.

---

### DEC-004: Billing Provider — Stripe

**Decision:** Stripe is the billing provider for V1.

**Source:** `OUTLINE.md` §1.1 (Closed Technical Decisions — Billing provider)

**Rationale:** Industry standard for SaaS subscriptions. Handles trials, metered credits, and subscription management natively.

**Affects:** F3 (Stripe setup, free trial limits, subscription lifecycle), D7 (billing domain spec)

**Resolved by DEC-010:** Free trial limits are 14 days, unlimited saved diagrams, no hosted AI credits, full export/editor access during trial.

---

### DEC-003: Backend Framework — FastAPI (Python)

**Decision:** The backend framework is FastAPI (Python). This decision is closed.

**Source:** `OUTLINE.md` §1.1 (Closed Technical Decisions — Backend framework), §11

**Rationale:** Matches the AI/ML ecosystem, has strong async support, and aligns with the example code throughout the product outline.

**Affects:** F0 (tech stack), all backend implementation

---

### DEC-002: Frontend — Vite + React; Full Stack from the Start

**Decision:**
- The frontend is built with Vite and React.
- The first local test build is a full web app with a backend/API from the start, not a frontend-only prototype.
- Local testing uses a local Postgres database.
- Production uses cloud-hosted Postgres (specific provider to be resolved in F0).

**Source:** `OUTLINE.md` §1.1 (Product Shape)

**Rationale:** Building frontend-only first forces a rework when auth, provider credentials, persistence, billing, and version history are added. Starting with a full-stack shape avoids that.

**Affects:** F0, F4

**Resolved by DEC-014 and DEC-015:** Production Postgres uses Neon free tier; production secrets are platform-injected environment variables on Fly.io/Vercel.

---

### DEC-001: V1 is Single-User SaaS; Team Features Deferred to V2

**Decision:**
- V1 is a commercial web SaaS, single-user account model.
- A free trial is required before paid conversion.
- The recommended pricing model is a recurring Pro subscription.
- Team accounts, shared workspaces, permissions, and realtime collaboration are V2 or later.

**Source:** `OUTLINE.md` §1.1 (Commercial Direction)

**Rationale:** Team features add significant auth, permissions, and data model complexity. Validating the single-user workflow first reduces risk.

**Affects:** F1 (users table, subscriptions table), F3 (auth, billing), D7

---
