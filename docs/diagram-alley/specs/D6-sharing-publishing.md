---
id: D6
tier: domain
status: draft
version: "0.1"
depends-on: [F1, F3, F5]
tags: [sharing, public, share-link, embed, view-only]
---

# D6 — Sharing and Publishing

**Version:** 0.1
**Status:** Draft
**Depends on:** F1, F3, F5

---

## Purpose

Define public share links, embed support, and view-only access for shared diagrams. This spec covers the share token model, the public endpoint contract, and the share view UI.

## Non-Goals

- Team workspaces or shared editing (V2)
- GitHub publishing (→ D8)
- Export download from share view (share view is display-only; download is a future feature)

---

## Owned Concepts

Share token model; share link creation/revocation; public share endpoint; embed iframe contract; share view UI; share audit trigger points (constants owned by F3).

---

## 1. Share Model

### 1.1 Share Storage

The `diagram_shares` table is owned by F1 §4.8 (DEC-024). D6 owns the share-link lifecycle and token rules.

**One active share link per diagram** in V1. Creating a new link while one exists revokes the old one first (atomically).

**Share link URL format:** `https://diagram-alley.vercel.app/share/<share_token>` or the configured production frontend domain (frontend route `/share/:shareToken` → F4 §2). The public API remains `GET /api/v1/share/{token}`.

### 1.2 Share Token Generation

- 32 characters of URL-safe base64 (using `secrets.token_urlsafe(24)` in Python)
- Unique enforced by DB index
- Not derived from diagram content — opaque, unguessable

---

## 2. Share Endpoints

### 2.1 Create Share Link

**`POST /api/v1/diagrams/{id}/share`** (requires auth)

No request body.

Service steps:
```
1. Load diagram (ownership check)
2. If an active share exists → revoke it (set revoked_at = now())
3. Generate share_token (32 chars url-safe base64)
4. Insert diagram_shares row
5. Return share link
```

Response:
```json
{
  "data": {
    "share_token": "abcd1234...",
    "share_url": "https://diagram-alley.vercel.app/share/abcd1234...",
    "created_at": "2026-05-20T14:32:00Z"
  }
}
```

### 2.2 Revoke Share Link

**`DELETE /api/v1/diagrams/{id}/share`** (requires auth)

Sets `revoked_at = now()` on the active share link. After revocation, the share URL returns `404 SHARE_NOT_FOUND`.

Returns `204 No Content`.

If no active share link exists, returns `404 SHARE_NOT_FOUND` (idempotent intent — the client can treat this as success).

### 2.3 Get Share Status

**`GET /api/v1/diagrams/{id}/share`** (requires auth)

Returns the current active share link if one exists, or `null`.

```json
{
  "data": {
    "share_token": "abcd1234...",
    "share_url": "https://diagram-alley.vercel.app/share/abcd1234...",
    "created_at": "2026-05-20T14:32:00Z",
    "access_count": 7,
    "last_accessed_at": "2026-05-20T16:00:00Z"
  }
}
```

Or:
```json
{ "data": null }
```

Endpoint is listed in F5 §10.

### 2.4 Public Share Access

**`GET /api/v1/share/{token}`** (no auth required — F3 §1.4)

Service steps:
```
1. Look up diagram_shares by share_token
2. If not found or revoked_at IS NOT NULL → 404 SHARE_NOT_FOUND
3. Load diagram (spec_json, ascii_cache, mermaid_cache)
4. Increment access_count, update last_accessed_at (async, non-blocking)
5. Return view-only diagram data
```

Response:
```json
{
  "data": {
    "diagram_id": "uuid",
    "title": "Three-Tier Web App",
    "diagram_type": "architecture",
    "ascii_cache": "...rendered ASCII...",
    "mermaid_cache": "...rendered Mermaid...",
    "shared_by_name": "Paul",
    "created_at": "2026-05-20T10:00:00Z",
    "updated_at": "2026-05-20T14:00:00Z"
  }
}
```

`shared_by_name`: the display name of the diagram owner. Uses `users.name` when present; otherwise uses the lowercased email prefix before `@` (DEC-029). Used in the share view header.

`spec_json` is **not** included in the public response — the share view only exposes rendered output, not the raw spec.

---

## 3. Share View (UI)

**Route:** `/share/:shareToken` (F4 §2)

The share view is public (no auth required). It has a minimal shell — no sidebar, no header with navigation.

### 3.1 Layout

```
+------------------------------------------------------------------+
| Diagram Alley logo   |   "Three-Tier Web App" by Paul   [Sign up]|
+------------------------------------------------------------------+
|  [ASCII]  [Mermaid]  [Visual]                                    |
+------------------------------------------------------------------+
|                                                                  |
|  [Selected tab content — read-only render]                       |
|                                                                  |
+------------------------------------------------------------------+
| Shared via Diagram Alley · Create your own →                    |
+------------------------------------------------------------------+
```

### 3.2 Tab Contents

- **ASCII tab:** Read-only `<pre>` block, monospace font, Copy button
- **Mermaid tab:** Read-only monospace block, Copy button; "Not available" for ui_wireframe/file_structure
- **Visual tab:** React Flow in read-only mode (`nodesDraggable=false`, `nodesConnectable=false`, `elementsSelectable=false`)

### 3.3 Sign Up CTA

A non-intrusive "Sign up free" button is shown in the share view header. It links to `/register`. The share view does not gate access behind sign-up.

### 3.4 Mobile Behavior

On mobile (<768px), the share view renders only the ASCII tab by default (most compact). Tab switching is available. The React Flow visual tab is hidden on mobile (shows a message: "Open on desktop to view the visual diagram").

### 3.5 Error States

- **Share not found / revoked:** Full-page message: "This share link has expired or been revoked."
- **Diagram not renderable:** Shows the error with a message: "This diagram could not be rendered."

---

## 4. Embed Support

The share URL can be embedded in an iframe. The share view accepts an `?embed=1` query parameter:
- Hides the header and footer entirely
- Removes the sign-up CTA
- Keeps only the tab content (ASCII by default; can be overridden with `?embed=1&tab=visual`)

No additional iframe sandbox attributes are required from the embedding side — the share view uses only standard web APIs.

Embed URL example: `https://diagram-alley.vercel.app/share/abcd1234...?embed=1&tab=ascii`

---

## 5. Audit Events

F3 §3.2 owns the share audit event constants: `SHARE_CREATED`, `SHARE_REVOKED`, and `SHARE_ACCESSED`.

`SHARE_ACCESSED` does not record IP or user identity (anonymous access — no PII logged). It records only `share_token` and timestamp in `detail`.

---

## Open Questions

None. All D6 decisions are locked.

---

## Acceptance Criteria

- `POST /api/v1/diagrams/{id}/share` returns a unique 32-character share token.
- Accessing the share URL `GET /api/v1/share/{token}` without auth returns the diagram's ASCII and Mermaid renders.
- Accessing a revoked share token returns `404 SHARE_NOT_FOUND`.
- Creating a second share link for the same diagram revokes the first (old token returns 404).
- `DELETE /api/v1/diagrams/{id}/share` with no active link returns `404 SHARE_NOT_FOUND`.
- `access_count` increments by 1 on each `GET /api/v1/share/{token}` access.
- `spec_json` is not present in the public share response.
- The embed URL with `?embed=1` renders without header, footer, or sign-up CTA.
- `SHARE_CREATED` and `SHARE_REVOKED` audit events are written on the respective actions.
