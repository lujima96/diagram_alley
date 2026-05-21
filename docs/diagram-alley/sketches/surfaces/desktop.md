---
phase: 3
artifact: surface — desktop
status: complete
depends-on: [F4, F3, D6]
---

# Surface: Desktop

V1 desktop surface definition. Read F4 for the authoritative route map and workspace layout. This file defines device assumptions, the navigation pattern, each route's purpose and access rules, the offline/network-loss behavior, and degraded-state handling.

---

## Device Assumptions

| Property | Value |
|----------|-------|
| Minimum supported width | 1024px |
| Optimal width | 1280px – 1920px |
| Input | Keyboard + pointer (mouse or trackpad) |
| Connection | Online required for AI generation, saving, and export jobs |
| Browser support | Latest Chrome, Firefox, Safari, Edge |
| Local storage | Used for in-memory access token and unsaved draft recovery only |

Tablet (768–1023px): editor is functional but not optimized. Diagram library and settings pages work without layout breakage. The workspace panel resizes to a single-column layout; the ASCII panel and inspector become collapsible drawers.

---

## Navigation Pattern

The app uses a persistent left sidebar plus a top header bar.

### Sidebar

Always visible on desktop. Collapsed to icon-only at 1024px–1279px; full labels at ≥1280px.

| Item | Route | Icon label | Auth |
|------|-------|------------|------|
| Dashboard | `/dashboard` | Home | Required |
| Library | `/library` | Library | Required |
| Templates | `/templates` | Templates | Required |
| Projects | `/projects` | Projects | Required |
| Settings | `/settings/account` (first child) | Settings | Required |
| Admin | `/admin` | Admin | Required + superuser only |

The New Diagram action is a primary button in the sidebar header, not a nav item.

### Header Bar

Persistent across all authenticated routes. Contains:

- App logo / wordmark (links to `/dashboard`)
- Current diagram title (editable inline, on workspace routes only)
- Save button (workspace routes only — creates a version snapshot per D3/DEC-021)
- Export button (workspace routes only — opens export modal per D4)
- Account menu (avatar → account settings, billing, log out)
- Trial/plan status chip (visible during active trial — links to `/settings/billing`)

On public routes (`/`, `/login`, `/register`, `/share/:shareToken`), the sidebar is hidden and the header is a minimal marketing nav.

---

## Route Reference

Matches the authoritative route map in F4 §2. This table adds context for surface planning.

### Public Routes

| Route | Page | Purpose | Notes |
|-------|------|---------|-------|
| `/` | Landing | Marketing and conversion | Links to `/register`; shows product demo |
| `/login` | Login | Authenticate | Redirects to `/dashboard` if already authed |
| `/register` | Register | Create account | Starts free trial on success (DEC-010) |
| `/verify-email` | Email verify | One-time verification landing | Shown after registration |
| `/reset-password` | Reset password | Password recovery form | Token from email link |
| `/share/:shareToken` | Share view | Read-only diagram viewing | Public; no auth required (D6) |

### Authenticated Routes — Navigation

| Route | Page | Primary persona | Key actions |
|-------|------|-----------------|-------------|
| `/dashboard` | Dashboard | P1, P2 | Quick create, recent diagrams, recent projects, provider status chip |
| `/projects` | Projects list | P2, P6 | Create project, search, open project |
| `/projects/:projectId` | Project detail | P2, P6 | Diagrams in project, add diagram, rename/archive project |
| `/library` | Library | P1, P2 | Search/filter all diagrams, open, duplicate, delete |
| `/templates` | Templates | P2 | Browse templates by category, preview, use template |

### Authenticated Routes — Workspace

| Route | Page | Primary persona | Key actions |
|-------|------|-----------------|-------------|
| `/workspace/new` | Workspace (new) | P1, P2 | New unsaved diagram; prompt panel; visual grid editor; export; Save to persist |
| `/workspace/:diagramId` | Workspace (saved) | P1, P2, P6 | Open saved diagram; AI modify; manual edit; version history drawer; export |

### Authenticated Routes — Settings

| Route | Page | Auth | Key actions |
|-------|------|------|-------------|
| `/settings/account` | Account | Required | Profile, email, password change, delete account |
| `/settings/billing` | Billing | Required | Plan status, trial countdown, upgrade to Pro, manage subscription (D7) |
| `/settings/providers` | Providers | Required | Add/edit/delete BYOK and local model providers (D1, F3) |

### Admin Routes

| Route | Page | Auth | Key actions |
|-------|------|------|-------------|
| `/admin` | Admin dashboard | Required + superuser | User count, recent audit events, system health |
| `/admin/users` | User list | Required + superuser | Search users, filter by plan, view detail |
| `/admin/users/:userId` | User detail | Required + superuser | Profile, subscription override, audit log for user |
| `/admin/audit-log` | Audit log | Required + superuser | Full audit event log, filter by event type and user |

### Error Routes

| Route | Page | Notes |
|-------|------|-------|
| `*` | 404 | Public; offers link back to dashboard or landing |

---

## Workspace Layout

The workspace is the most complex surface. Full architecture is owned by F4. Summary for surface planning:

```
+------------------+------------------------------------------+------------------+
| Sidebar          | Prompt / AI Panel                        | Preview Panel    |
| (nav)            |                                          | (ASCII tab)      |
|                  | [Prompt textarea]                        | [Mermaid tab]    |
|                  | Diagram type / Provider selector         | [SVG/PNG tab]    |
|                  | [Generate] [Improve]                     | [JSON/YAML tab]  |
+------------------+------------------------------------------+                  |
|                  | Visual Grid Editor (React Flow — DEC-013)|                  |
|                  | (drag nodes, edit edges, grid layout)    | Copy / Download  |
|                  |                                          |                  |
+------------------+------------------------------------------+------------------+
| Inspector panel (selected node/edge properties, validation errors, spec escape) |
+---------------------------------------------------------------------------------+
```

The inspector panel collapses when nothing is selected. The preview panel can be minimized to maximize editor space.

---

## Panel Modes

The workspace supports multiple working modes that determine which panels are foregrounded. Exact behavior is owned by D2.

| Mode | Foregrounded panels |
|------|---------------------|
| Prompt Mode | Prompt/AI panel; preview panel shows live output |
| Visual Grid Edit Mode | Visual grid editor; inspector panel |
| Structured Form Mode | Inspector panel (expanded form view); preview panel |
| JSON/YAML Spec Mode | JSON/YAML editor occupies center; preview panel |
| ASCII Edit Mode | ASCII panel (editable); reverse sync boundary warning visible |
| Export Mode | Export modal (overlays workspace) |

---

## Offline and Network-Loss Behavior

V1 requires an active connection for AI generation, save operations, and export jobs. The following behaviors apply when the network is unavailable or a request fails:

| Scenario | Behavior |
|----------|----------|
| Network lost while editing | Auto-save to live `spec_json` continues to queue; unsaved changes are held in memory and persisted when reconnection occurs. User sees a "connection lost — changes are held" banner. |
| AI generation request fails | Show clear error with retry button; preserve the prompt input; keep the last valid diagram state (DEC per D1 AI failure handling). |
| Save fails (network error) | Toast error; unsaved changes remain in memory; retry on next manual Save or reconnect. |
| Export job fails | Show error; allow retry; offer the copy-to-clipboard fallback for ASCII/Mermaid/JSON formats. |
| Session token expired | Silent token refresh attempt (F3 §1.1). If refresh fails, redirect to `/login` with a "Your session expired" message. |

Draft recovery: if the browser is closed with unsaved changes, the in-memory draft is lost. V1 does not use localStorage or IndexedDB for draft persistence — that is a V2 concern. Users are encouraged to Save (create a version snapshot) frequently.

Stretch: In memory draft

---

## Accessibility Notes (V1 Scope)

- All interactive controls must be keyboard-focusable and have visible focus styles.
- The visual grid editor (React Flow) handles its own keyboard interaction for node selection and basic navigation.
- ASCII output panels must be screen-reader readable as plain text.
- Color is never the sole indicator of state (validation errors use icons + text, not color alone).
- Full WCAG AA compliance audit is deferred to V2 but the above constraints are required for V1.

---

## Empty States

Each main page has a defined empty state for new users:

| Page | Empty state |
|------|-------------|
| Dashboard | "Create your first diagram" call-to-action card; quick-start diagram type buttons |
| Library | "No diagrams yet" with New Diagram button |
| Projects | "No projects yet" with Create Project button |
| Templates | Template gallery always has content (system templates ship with the product) |
| Workspace (new) | Prompt panel is focused; example prompts are displayed in a suggestions list |

---

## Trial and Plan UX

Per DEC-010: free trial is 14 days, unlimited diagrams, no hosted AI, full editor and export access.

- A trial countdown chip is shown in the header bar for the last 5 days of the trial.
- When the trial expires (DEC-022), the user can view, export, and manage share links but cannot edit `spec_json`, use AI, or create new diagrams.
- An upgrade banner replaces the prompt panel on workspace pages after trial expiry.
- The billing page (`/settings/billing`) is always accessible.
