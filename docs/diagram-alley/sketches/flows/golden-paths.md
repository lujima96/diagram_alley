---
phase: 8
artifact: golden-paths
status: complete
depends-on: [F1, F2, F3, F4, F5, D1, D2, D3, D4, D5, D6, D7, A1, schema-roundup]
---

# Phase 8 — Golden Paths

Golden paths are the minimum end-to-end journeys that must work before implementation can be considered commercially coherent. They cut across specs, flows, mockups, and data contracts.

These are not new product decisions. They are acceptance paths that combine already-owned behavior from F1-F5 and D1-D7.

---

## 1. Golden Path Matrix

| ID | Path | Primary User | Core Value | Launch Criticality |
|----|------|--------------|------------|--------------------|
| GP-01 | Trial signup → provider setup → first diagram | Solo developer | User reaches first useful diagram | Critical |
| GP-02 | Prompt → edit → save → export | Solo developer | Main product promise | Critical |
| GP-03 | Manual edit → validate → export | Technical writer | Non-AI utility | Critical |
| GP-04 | Trial expired → upgrade → diagram unlock | Solo developer | Revenue conversion without data loss | Critical |
| GP-05 | Version restore → re-export | Technical writer | Trust and recoverability | Critical |
| GP-06 | Share link → public view → revoke | Engineering lead | Collaboration without accounts | Important |
| GP-07 | ASCII label sync → detached-state boundary | Solo developer | Clear V1 reverse-sync boundary | Important |
| GP-08 | Admin support lookup → subscription override/audit review | Operator/admin | Support readiness | Important |
| GP-09 | Import → preview warnings → save | Technical writer | Bring existing diagrams into Alley safely | Important |
| GP-10 | Export → Git workflow | Developer | Documentation delivery | Stretch |

---

## 2. GP-01 — Trial Signup to First Diagram

**Goal:** A new user signs up, verifies email, configures a provider, and reaches a usable generated diagram.

**Persona:** Priya, solo developer.

**Start:** Public landing page.

**End:** Saved diagram exists with initial version snapshot and rendered ASCII.

### Required Steps

1. User opens `/` and starts trial.
2. User registers at `/register`.
3. Backend creates `users` row and `subscriptions` row with `plan='trial'`, `status='trialing'`.
4. User verifies email through `/verify-email`.
5. User lands on `/dashboard`.
6. User opens `/settings/providers`.
7. User creates a BYOK or local provider and marks it default.
8. User starts a new diagram at `/workspace/new`.
9. User submits a prompt through `POST /api/v1/diagrams/generate`.
10. Backend validates, repairs if needed, persists the diagram, renders outputs, creates initial version snapshot.
11. User sees visual grid and ASCII output.

### Acceptance Criteria

- Registration creates exactly one active trial subscription row for the user.
- Unverified users cannot access authenticated app routes until verification, if F3 enforces verification.
- Provider setup can succeed without hosted AI credits.
- First successful generation creates both `diagrams` and `diagram_versions` rows.
- Route redirects to `/workspace/:diagramId`.
- ASCII tab renders without manual refresh.

### Owners

F1, F3, F4, F5, D1, D3, D7.

---

## 3. GP-02 — Prompt to Export

**Goal:** A user generates a diagram, makes a small visual edit, saves a version, and exports documentation-ready output.

**Persona:** Priya.

**Start:** Authenticated dashboard with configured provider.

**End:** User has copied or downloaded a valid export.

### Required Steps

1. User opens `/workspace/new`.
2. User selects diagram type.
3. User enters prompt and clicks Generate.
4. Backend returns persisted diagram, spec, ASCII, Mermaid where supported, and warnings if any.
5. User edits title and a node label in the visual editor.
6. Auto-save persists live `spec_json` without creating a version snapshot.
7. User clicks Save.
8. Backend creates a version snapshot.
9. User opens Export modal.
10. User exports Strict ASCII, Mermaid, SVG, PNG, JSON, or YAML according to D4 availability.

### Acceptance Criteria

- Generation is blocked with `SUBSCRIPTION_REQUIRED` if subscription status is not allowed.
- Invalid generated specs either repair successfully or return a clear error.
- Visual edits update `spec_json` and invalidate render caches.
- Save creates a version snapshot and is not confused with auto-save.
- Export does not mutate `spec_json`.
- Unsupported export formats return `EXPORT_FORMAT_UNSUPPORTED` with a clear UI state.

### Owners

D1, D2, D3, D4, F2, F3, F5.

---

## 4. GP-03 — Manual Edit Without AI

**Goal:** A technical writer edits an existing diagram without AI and exports it.

**Persona:** Marcus, technical writer.

**Start:** Existing saved database diagram.

**End:** Updated diagram is saved and exported.

### Required Steps

1. User opens `/workspace/:diagramId`.
2. User selects an existing table/node.
3. User adds a table or node through the inspector.
4. User edits fields/columns/relationships.
5. User drags the new item on the grid.
6. Validation updates in the inspector.
7. User saves a named version snapshot.
8. User exports ASCII and Mermaid.

### Acceptance Criteria

- Manual editing works with no configured AI provider.
- Inspector changes update the canonical `spec_json`.
- Validation errors block invalid spec persistence where owner specs require blocking.
- Undo/redo remains session-scoped and does not create version rows.
- Export uses the current saved spec.

### Owners

D2, D3, D4, F1, F2, F4.

---

## 5. GP-04 — Trial Expiry to Upgrade Unlock

**Goal:** An expired trial user can still access existing work, upgrade, and immediately regain editing.

**Persona:** Priya.

**Start:** Trial has expired: `plan='trial'`, `status='canceled'`.

**End:** User is Pro active and can edit existing diagrams.

### Required Steps

1. User opens dashboard after trial expiry.
2. Existing diagrams remain visible.
3. Workspace opens in read-only/upgrade state.
4. User can export and manage share links.
5. User clicks upgrade entry point.
6. Backend creates Stripe Checkout session.
7. Stripe completes checkout and webhook sets `plan='pro'`, `status='active'`.
8. User returns to app and editor unlocks.

### Acceptance Criteria

- Expired users cannot patch `spec_json`, use AI, restore versions, or create new diagrams.
- Expired users can view, export, and manage share links.
- UI overlay does not replace API enforcement.
- Webhook-driven subscription update unlocks access without data migration.
- No diagrams or versions are lost during expiry.

### Owners

F1, F3, F5, D2, D4, D6, D7.

---

## 6. GP-05 — Version Restore to Re-Export

**Goal:** A user restores a previous version after an unwanted change, saves a new version, and exports the restored diagram.

**Persona:** Marcus.

**Start:** Diagram has at least two version snapshots.

**End:** Restored diagram is live, version history is append-only, and export uses restored state.

### Required Steps

1. User opens version history drawer.
2. User previews a previous version.
3. User clicks Restore and confirms.
4. Backend creates pre-restore snapshot.
5. Backend restores selected version's `spec_json`.
6. Backend clears render caches and `ascii_is_detached`.
7. Backend creates post-restore snapshot.
8. Response returns `pre_restore_version_id` and `restored_version_id` (DEC-032).
9. User optionally saves a named follow-up snapshot.
10. User exports restored diagram.

### Acceptance Criteria

- Restore never deletes version history.
- Restored snapshots migrate to current `spec_version` before apply.
- Invalid restored specs return `SPEC_VALIDATION_FAILED`.
- Trial-expired users cannot restore.
- Export after restore reflects restored live `spec_json`.

### Owners

D3, D4, F1, F2, F3.

---

## 7. GP-06 — Share Link Lifecycle

**Goal:** A user shares a live read-only diagram with non-users and later revokes access.

**Persona:** Rafael, engineering lead.

**Start:** Saved diagram with active subscription or trial.

**End:** Share link is revoked and no longer reveals diagram data.

### Required Steps

1. User opens Share settings from workspace.
2. User creates share link.
3. Backend creates `diagram_shares` row and token.
4. Viewer opens `/share/:shareToken` without auth.
5. Viewer sees read-only ASCII/visual/Mermaid tabs.
6. User revokes link.
7. Revoked link returns share-not-found view.

### Acceptance Criteria

- Public share route does not expose `spec_json`.
- Share view has no editing controls.
- Share link reflects current live diagram state, not a fixed version snapshot.
- Revoked and invalid tokens do not reveal which case occurred.
- Expired trial users can manage share links per DEC-022.

### Owners

F1, F3, F4, F5, D6.

---

## 8. GP-07 — ASCII Reverse Sync Boundary

**Goal:** A user can sync label-only ASCII edits, but structural ASCII edits produce detached export state.

**Persona:** Priya.

**Start:** Existing architecture diagram with rendered text output.

**End:** Label edits sync; structural edits are safely detached and reversible.

### Required Steps

1. User enters ASCII edit mode.
2. User changes only a node/edge label.
3. Backend detects label-only diff and updates `spec_json`.
4. Visual editor and text output re-render.
5. User edits ASCII structurally by adding/removing a box or edge.
6. Backend rejects reverse sync and sets `ascii_is_detached=true`.
7. UI shows detached warning and Reset to spec option.
8. User resets to spec or exports detached text.

### Acceptance Criteria

- Label-only sync changes canonical `spec_json`.
- Structural text edits never mutate canonical `spec_json`.
- Detached state persists via `diagrams.ascii_is_detached`.
- Reset clears detached state and re-renders from spec.
- Export clearly indicates when detached ASCII is being exported.

### Owners

F1, F2, D2, D4, DEC-005, DEC-023.

---

## 9. GP-08 — Admin Support Lookup

**Goal:** A superuser can investigate account state, subscription status, and audit history for support.

**Persona:** Product/admin operator.

**Start:** Superuser is authenticated.

**End:** Admin can identify user status and apply an allowed support override.

### Required Steps

1. Superuser opens `/admin`.
2. Superuser searches users in `/admin/users`.
3. Superuser opens `/admin/users/:userId`.
4. Admin page shows profile, subscription, diagrams summary, and user audit events.
5. Superuser applies an allowed subscription override if needed.
6. Audit log records the admin action.
7. Superuser reviews `/admin/audit-log`.

### Acceptance Criteria

- Non-superusers receive `403 FORBIDDEN` for admin routes and endpoints.
- Admin pages never bypass F3 audit conventions.
- Subscription overrides respect A1 and DEC-025 fields.
- Audit log filters support support-debug workflows.

### Owners

A1, F1, F3, F5.

---

## 10. GP-09 — Import to Preview to Save

**Goal:** A user imports an external Mermaid/JSON/YAML diagram, reviews warnings, and saves a clean Diagram Alley diagram without corrupting existing work.

**Persona:** Marcus, technical writer.

**Start:** User has Mermaid, JSON, or YAML diagram content.

**End:** A new saved diagram exists, or import fails without writing partial data.

### Required Steps

1. User opens Import from sidebar, library, or workspace drag-and-drop.
2. User selects `input_format` and provides content.
3. User clicks Preview.
4. Backend calls `POST /api/v1/diagrams/import` with `preview_only=true`.
5. Backend converts, validates, repairs if possible, renders ASCII preview, and returns warnings.
6. User reviews converted preview and warning list.
7. User clicks Import & Save.
8. Backend creates a new `diagrams` row and render caches in a transaction.
9. Backend emits `DIAGRAM_IMPORTED`.
10. User lands in `/workspace/:diagramId`.

### Acceptance Criteria

- Preview mode never creates a `diagrams` row.
- Import always creates a new diagram and never overwrites an existing diagram.
- Unsupported Mermaid types return `IMPORT_UNSUPPORTED_FORMAT`.
- Parse failures return `IMPORT_PARSE_FAILED`.
- Validation failures after repair return `SPEC_VALIDATION_FAILED` and write no partial rows.
- Warnings are visible in the import UI but are not persisted on the diagram row (DEC-031).
- `_export_meta` from D4 JSON exports is stripped before validation.

### Owners

D5, F1, F2, F3, F5, DEC-006, DEC-031.

---

## 11. GP-10 — Export to Git Workflow (Stretch)

**Goal:** A developer gets Diagram Alley output into a repository workflow.

**Persona:** Solo developer.

**Start:** Saved diagram.

**End:** Diagram output is committed manually or, if D8 ships, through GitHub integration.

### V1 Required Path

1. User exports Markdown/ASCII/Mermaid/SVG/PNG/JSON/YAML.
2. User places the file in a repo manually.
3. User commits through their normal Git workflow.

### Stretch Path

If D8 is implemented in Paid V1:

1. User connects GitHub.
2. User selects repository and branch.
3. App commits diagram spec and selected exports.
4. App opens or updates a pull request.

### Acceptance Criteria

- Copy/download/export works independently of GitHub.
- GitHub commit/PR workflow remains optional stretch scope.
- Exported files include metadata only when requested by D4.

### Owners

D4, D8, DEC-007.

---

## 12. Cross-Path Launch Gates

These must hold across all critical golden paths:

- Auth and subscription gates are enforced by API, not only UI.
- `spec_json` remains the canonical source for diagram state.
- Auto-save and version snapshots remain distinct.
- Exports never mutate diagram state.
- Imports never overwrite existing diagram state.
- Public share never exposes raw `spec_json`.
- Trial-expired users retain view/export/share management access only.
- Every write that owner specs require to be audited emits the correct audit event.
- Mobile scope remains public share viewing only for V1.

---

## 13. Phase 8 Status

Phase 8 is complete. No new decisions were discovered while drafting this artifact.
