---
phase: 11
slice: D6
spec: specs/D6-sharing-publishing.md
status: planned
---

# Slice D6 — Sharing and Publishing

## Pre-conditions

- [ ] Slice F1 complete — `diagram_shares` table exists.
- [ ] Slice F3 complete — audit event helper available; public route list confirmed (no auth on `GET /api/v1/share/{token}`).
- [ ] Slice F5 complete — response envelope, error codes, endpoint stubs registered.

Note: D6 can be built independently of D2 as long as diagrams exist in the database. It does not depend on the editor.

---

## Build Sequence

1. **Share CRUD endpoints** — Implement `POST /api/v1/diagrams/{id}/share` (create), `DELETE /api/v1/diagrams/{id}/share` (revoke), and `GET /api/v1/diagrams/{id}/share` (status). Create generates a 32-char URL-safe token via `secrets.token_urlsafe(24)`. Creating a link while one is active revokes the old one atomically in the same transaction. Revoke sets `revoked_at=now()`. Status returns the active link or `null`. Emit `SHARE_CREATED` and `SHARE_REVOKED` audit events. (→ D6 §2.1–2.3, F3 §3.2)

2. **Public share access endpoint** — Implement `GET /api/v1/share/{token}` (no auth). Look up `diagram_shares` by token; if not found or `revoked_at IS NOT NULL`, return `404 SHARE_NOT_FOUND`. Load the diagram (spec_json, ascii_cache, mermaid_cache). Increment `access_count` and update `last_accessed_at` asynchronously (FastAPI `BackgroundTasks` — non-blocking). Return view-only response: title, diagram_type, ascii_cache, mermaid_cache, shared_by_name (display name helper from F1 §4.1 / DEC-029), created_at, updated_at. **Do not include `spec_json` in the public response.** Emit `SHARE_ACCESSED` without PII. (→ D6 §2.4, DEC-022, DEC-029)

3. **Share view UI** — Implement `<ShareViewPage>` at `/share/:shareToken`. Public route — no app shell. Calls `GET /api/v1/share/{token}`. Layout per D6 §3.1: logo, diagram title + "by <name>", Sign Up CTA button linking to `/register`, three tabs (ASCII, Mermaid, Visual). ASCII and Mermaid tabs are read-only `<pre>` blocks with Copy buttons. Visual tab is React Flow with `nodesDraggable=false`, `nodesConnectable=false`, `elementsSelectable=false`. Error states: share not found → "This share link has expired or been revoked." (→ D6 §3, GP-06)

4. **Mobile behavior for share view** — On mobile (<768px), show only the ASCII tab by default. Tab switching available. Hide the Visual tab React Flow on mobile and show "Open on desktop to view the visual diagram" message. (→ D6 §3.4)

5. **Embed mode** — Detect `?embed=1` query parameter in `<ShareViewPage>`. When present: hide header, footer, and Sign Up CTA; render only the tab content; default tab is ASCII unless `?tab=visual` or `?tab=mermaid` is set. No additional server changes needed — the share endpoint is already used. (→ D6 §4)

6. **Share controls in workspace** — Add a Share button/section to the workspace (accessible from the diagram action menu or a share icon in the header). Opens a popover showing: current share status (active link with copy URL button, or "Not shared yet"), Create Link button, Revoke Link button. Uses `GET/POST/DELETE /api/v1/diagrams/{id}/share`. Expired trial users can manage share links per DEC-022. (→ D6 §2, DEC-022, GP-06)

---

## Done Criteria

- [ ] `POST /api/v1/diagrams/{id}/share` returns a unique 32-character share token. (→ D6 Acceptance Criteria)
- [ ] `GET /api/v1/share/{token}` without auth returns the diagram's ASCII and Mermaid renders. (→ D6 Acceptance Criteria)
- [ ] `spec_json` is not present in the public share response. (→ D6 Acceptance Criteria, GP-06)
- [ ] Accessing a revoked share token returns `404 SHARE_NOT_FOUND`. (→ D6 Acceptance Criteria)
- [ ] Creating a second share link for the same diagram revokes the first — old token returns 404. (→ D6 Acceptance Criteria)
- [ ] `DELETE /api/v1/diagrams/{id}/share` with no active link returns `404 SHARE_NOT_FOUND`. (→ D6 Acceptance Criteria)
- [ ] `access_count` increments by 1 on each public share access. (→ D6 Acceptance Criteria)
- [ ] `?embed=1` renders the share view without header, footer, or Sign Up CTA. (→ D6 Acceptance Criteria)
- [ ] `SHARE_CREATED` and `SHARE_REVOKED` audit events are written on the respective actions. (→ D6 Acceptance Criteria)
- [ ] An expired trial user can still create and revoke share links. (→ DEC-022, GP-04)
