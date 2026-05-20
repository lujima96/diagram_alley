---
id: F4
tier: foundation
status: draft
version: "0.1"
depends-on: [F0]
tags: [ui, surface, workspace, grid-editor, navigation, react-flow]
---

# F4 — UI Surface and Workspace Architecture

**Version:** 0.1
**Status:** Draft
**Depends on:** F0

---

## Purpose

Define the desktop-first surface map, the workspace layout, the React Flow grid editor architecture (including the grid coordinate ↔ ASCII position mapping), navigation patterns, component conventions, and responsive rules. This spec must be read before any domain UI work begins.

## Non-Goals

- Editor interactions (drag, edit, inspector) (→ D2)
- AI prompt panel behavior (→ D1)
- Export controls (→ D4)
- Individual mockup files (→ Phase 6)

---

## Owned Concepts

Route map; workspace panel layout; React Flow grid editor contract; grid coordinate → pixel mapping; navigation pattern; component naming conventions; responsive breakpoints; mobile scope.

---

## 1. Surface Scope

V1 is a **desktop-first** web application. Mobile is read-only share viewing only (share links — D6).

| Surface | Scope | Notes |
|---------|-------|-------|
| Desktop (≥1024px) | Full product | All creation, editing, export, settings |
| Tablet (768–1023px) | Degraded | Editor is usable but not optimized; diagram library and settings work |
| Mobile (<768px) | Share view only | Authenticated users see a "use desktop" banner; public share links show view-only rendering |

---

## 2. Route Map

All routes are client-side (React Router v6). The backend is API-only.

| Route | Page | Auth | Description |
|-------|------|------|-------------|
| `/` | Landing | Public | Marketing page |
| `/login` | Login | Public (redirect if authed) | Login form |
| `/register` | Register | Public (redirect if authed) | Registration form |
| `/verify-email` | Email verify | Public | Verification landing page |
| `/reset-password` | Reset password | Public | Password reset form |
| `/dashboard` | Dashboard | Required | Recent diagrams, quick create |
| `/projects` | Projects | Required | Project list |
| `/projects/:projectId` | Project detail | Required | Diagrams in a project |
| `/workspace/new` | Workspace | Required | New diagram (unsaved) |
| `/workspace/:diagramId` | Workspace | Required | Open saved diagram |
| `/library` | Library | Required | All diagrams, search, filter |
| `/templates` | Templates | Required | Template gallery |
| `/settings/account` | Account | Required | Profile, password, delete account |
| `/settings/billing` | Billing | Required | Plan, subscription, upgrade |
| `/settings/providers` | Providers | Required | Model provider config |
| `/share/:shareToken` | Share view | Public | View-only shared diagram |
| `/admin` | Admin Dashboard | Required + superuser | User stats, recent events |
| `/admin/users` | Admin Users | Required + superuser | User list, search, actions |
| `/admin/users/:userId` | Admin User Detail | Required + superuser | User profile, subscription, audit log |
| `/admin/audit-log` | Admin Audit Log | Required + superuser | Full audit event log |
| `*` | 404 | Public | Not found |

---

## 3. App Shell

The app shell is present on all authenticated routes. Public routes (landing, auth, share view) use minimal or no shell.

### 3.1 Shell Structure

```
+------------------------------------------------------------------+
| Header bar (64px)                                                |
| [Logo] [Project breadcrumb]              [User menu] [Upgrade]  |
+------------------------------------------------------------------+
| Sidebar (240px fixed) | Main content area (flex)                |
| Navigation links      |                                          |
| Provider status       |                                          |
+------------------------------------------------------------------+
```

### 3.2 Sidebar Navigation

```
Diagram Alley
─────────────
+ New Diagram         ← primary CTA button
─────────────
Dashboard
Library
Templates
Projects
─────────────
Settings
  Account
  Billing
  Providers
─────────────
[Provider status]     ← indicator dot (green/grey)
```

The sidebar collapses to icons-only on viewport widths 768–1023px.

### 3.3 Header Bar

| Section | Contents |
|---------|---------|
| Left | Logo, project breadcrumb (when in workspace: `Project Name / Diagram Name`) |
| Right | Trial days remaining badge (if trialing), Upgrade button (if not Pro), User avatar + menu |

User avatar menu: Profile, Settings, Logout.

---

## 4. Page Layouts

### 4.1 Dashboard (`/dashboard`)

```
+------------------------------------------------------------------+
| Quick Create                                                     |
| [Architecture] [Database] [Flowchart] [UI Wireframe] [+]        |
+------------------------------------------------------------------+
| Recent Diagrams                              [View all →]        |
| [Card] [Card] [Card] [Card]                                      |
+------------------------------------------------------------------+
| Recent Projects                              [View all →]        |
| [Card] [Card] [Card]                                             |
+------------------------------------------------------------------+
| Featured Templates                           [Browse all →]      |
| [Card] [Card] [Card]                                             |
+------------------------------------------------------------------+
```

Empty state (new user): large centered prompt with quick-create options and a "Start from a template" link.

### 4.2 Library (`/library`)

```
+------------------------------------------------------------------+
| [Search] [Type filter] [Sort]                                    |
+------------------------------------------------------------------+
| [Diagram card] [Diagram card] [Diagram card]                     |
| [Diagram card] [Diagram card] [Diagram card]                     |
+------------------------------------------------------------------+
```

List view toggle available. Pagination: 24 per page.

### 4.3 Templates (`/templates`)

```
+------------------------------------------------------------------+
| [All] [Architecture] [Database] [Flowchart] [UI] [More]         |
+------------------------------------------------------------------+
| [Template card] [Template card] [Template card]                  |
| [Template card] [Template card] [Template card]                  |
+------------------------------------------------------------------+
```

---

## 5. Workspace Layout

The workspace is the most complex page. It must be specced before any domain UI work.

### 5.1 Panel Arrangement

```
+------------------------------------------------------------------+
| Header: [Logo] [Project / Diagram name] [Save] [Export] [...]   |
+------------------------------------------------------------------+
| Prompt panel  | Visual Grid Editor (React Flow)  | Preview panel |
| (300px)       | (flex, fills remaining width)    | (320px)       |
|               |                                   |               |
|               |                                   | [ASCII]       |
|               |                                   | [Mermaid]     |
|               |                                   | [Visual]      |
|               |                                   | [JSON/YAML]   |
|               |                                   |               |
|               |                                   | [Copy] [DL]   |
+------------------------------------------------------------------+
| Inspector panel (full width, collapsible, 200px default height)  |
| Node/edge properties | Validation warnings | Spec escape hatch   |
+------------------------------------------------------------------+
```

Panel resizing: the side panels have drag handles. Minimum widths: Prompt 240px, Preview 260px, Editor 400px.

Panels can be collapsed individually. Collapsed state is persisted in `localStorage` per user.

### 5.2 Prompt Panel

Contents:
- Diagram type selector (dropdown; current type shown)
- Prompt textarea (resizable, auto-expand, max 10 rows)
- Model provider selector (dropdown; shows configured providers)
- `[Generate]` button (primary, disabled when no provider configured)
- `[Improve existing]` button (secondary, only shown when a diagram is loaded)
- "No provider configured" → link to `/settings/providers`

States:
- **Empty:** placeholder text, disabled Generate button
- **Generating:** spinner on Generate button, textarea disabled
- **Error:** red banner with error message, textarea re-enabled
- **Success:** automatically closes prompt (collapses) and focuses the grid editor

### 5.3 Visual Grid Editor (React Flow)

See §6 for the full contract. Summary:
- Fills the center panel
- Renders nodes, edges, groups from the Diagram Spec
- Supports drag, select, zoom, pan
- Node drag → updates `spec_json` positions → triggers ASCII re-render
- Toolbar: zoom in/out, fit view, toggle grid lines, undo/redo (V1: single-level undo)

### 5.4 Preview Panel

Tab bar at top:
- `ASCII` (default)
- `Mermaid`
- `Visual` (SVG render — same as grid editor but read-only view for export)
- `JSON` / `YAML` (toggle)

Contents per tab:
- **ASCII:** monospace `<pre>` block, horizontal scroll, line/char count, Copy button, Download button
- **Mermaid:** monospace block, Copy button, Download button; "Not available for this diagram type" for wireframe/file structure
- **Visual:** React Flow in read-only mode (no drag), Export SVG button, Export PNG button
- **JSON/YAML:** CodeMirror read-only (editable in inspector panel escape hatch)

### 5.5 Inspector Panel

Tabs:
- **Selection:** When a node or edge is selected in the React Flow editor, shows its editable properties (label, kind for nodes; label, style for edges)
- **Validation:** Live validation status: error list, warning list, auto-fix button
- **Spec editor:** CodeMirror 6 with JSON syntax highlighting; format button, validate button, apply changes button. This is the "escape hatch" for power users.

---

## 6. React Flow Grid Editor Architecture

This section defines the contract between the Diagram Spec, the React Flow component, and the ASCII renderer. It must be stable before D2 implementation begins.

### 6.1 Grid Coordinate System

Each node in the Diagram Spec has `position: { col: integer, row: integer }`. These are zero-indexed logical grid cells.

The React Flow canvas uses pixel coordinates. The mapping (from F2 §6.2):

```
x = col * CELL_WIDTH  + CELL_PADDING    // CELL_WIDTH=220, CELL_PADDING=40
y = row * CELL_HEIGHT + CELL_PADDING    // CELL_HEIGHT=120, CELL_PADDING=40
```

Reverse (drag → grid):
```
col = Math.round((x - CELL_PADDING) / CELL_WIDTH)
row = Math.round((y - CELL_PADDING) / CELL_HEIGHT)
```

Snapping is enforced with React Flow's `snapToGrid` prop.

### 6.2 Spec → React Flow Node Transform

The `useSpecToFlow` hook converts a Diagram Spec into React Flow's `nodes[]` and `edges[]` arrays:

```typescript
function useSpecToFlow(spec: DiagramSpec): { nodes: Node[]; edges: Edge[] }
```

Each Diagram Spec node becomes a React Flow node:
```typescript
{
  id: node.id,
  type: node.kind,           // maps to a custom node component
  position: { x, y },        // from grid formula
  data: { label: node.label, kind: node.kind, groupId: node.group_id }
}
```

Each Diagram Spec edge becomes a React Flow edge:
```typescript
{
  id: edge.id,
  source: edge.from,
  target: edge.to,
  label: edge.label ?? undefined,
  type: 'smoothstep',
  style: { strokeDasharray: edge.style === 'dashed' ? '5,5' : undefined }
}
```

### 6.3 React Flow Node → Spec Round-Trip

When the user drags a node, the `onNodeDragStop` callback:
1. Receives the new `position: { x, y }` from React Flow
2. Converts to grid: `col = Math.round((x - 40) / 220)`, `row = Math.round((y - 40) / 120)`
3. Updates the Zustand `diagramStore` with the new `position` on the corresponding node
4. Triggers a debounced `spec_json` patch to the backend (300ms debounce)
5. Triggers an ASCII re-render (async, non-blocking)

### 6.4 Custom Node Components

Each node kind maps to a custom React Flow node component registered in the `nodeTypes` prop:

```typescript
const nodeTypes = {
  user_client: UserClientNode,
  frontend: FrontendNode,
  backend: BackendNode,
  service: ServiceNode,
  gateway: GatewayNode,
  database: DatabaseNode,
  // ... all 20 kinds
  generic: GenericNode,
}
```

Each custom node component renders:
- A styled `div` with the kind-specific border style (from F2 §4.3)
- The node label (bold)
- The kind annotation line (lighter weight)
- A selection handle ring when selected
- React Flow `Handle` components for edge connections (top, bottom, left, right)

### 6.5 Groups in React Flow

Diagram Spec `groups` map to React Flow's `parentNode` pattern:
- A group node is rendered as a React Flow node with `type: 'group'`
- Child nodes have `parentNode: group.id` and `extent: 'parent'`
- Group drag moves all children together

### 6.6 State Management

The workspace uses three Zustand stores:

| Store | Responsibility |
|-------|---------------|
| `diagramStore` | Current diagram spec, unsaved changes flag, selected node/edge id, ASCII cache |
| `workspaceUiStore` | Panel visibility, panel sizes, active preview tab, inspector tab |
| `providerStore` | Configured providers list, selected provider for current generation |

The `diagramStore` is the single source of truth for the spec. React Flow's internal state is derived from it via `useSpecToFlow`. The spec is the authority — never persist React Flow's internal node positions as primary state.

---

## 7. Component Naming Conventions

All UI components are in `frontend/src/components/`. Naming:

| Pattern | Example | Description |
|---------|---------|-------------|
| `<Domain><Component>` | `DiagramCard` | Reusable component in a domain |
| `<Page>Page` | `DashboardPage` | Top-level route component |
| `<Name>Panel` | `PromptPanel` | A workspace panel |
| `<Name>Modal` | `ExportModal` | A modal dialog |
| `<Name>Drawer` | `VersionDrawer` | A slide-in drawer |
| `<Kind>Node` | `DatabaseNode` | A React Flow custom node |

Component files are co-located with their styles (Tailwind classes inline — no separate CSS files except global).

---

## 8. Loading, Empty, and Error States

Every page and data-loading component must define all three states:

| State | Requirement |
|-------|------------|
| Loading | Skeleton loader or spinner; never a blank white screen |
| Empty | Illustrated empty state with a primary action (e.g., "Create your first diagram") |
| Error | Error message with a retry button or navigation path out |

The workspace additionally defines:
- **Generating:** spinner on the prompt panel, editor locked
- **Validation error:** red banner in inspector, render blocked
- **Detached ASCII:** persistent warning banner (see F2 §7.3)
- **No provider:** informational banner linking to provider settings

---

## 9. Responsive Rules

| Viewport | Behavior |
|----------|---------|
| ≥1280px (large desktop) | Full 3-panel workspace; sidebar expanded |
| 1024–1279px (desktop) | 3-panel workspace; sidebar may be narrower |
| 768–1023px (tablet) | Sidebar collapses to icons; workspace panels stack (no prompt panel, grid editor + preview stacked) |
| <768px (mobile) | Authenticated users: "Please use a desktop browser" banner + library read access. Share view: read-only diagram display |

---

## Open Questions

None. All F4 decisions are locked.

---

## Acceptance Criteria

- The workspace renders with three panels (Prompt, Grid Editor, Preview) visible simultaneously on a 1280px viewport.
- A node dragged to column 2, row 1 in React Flow snaps to `x = 480, y = 160` and updates `spec_json.nodes[n].position = { col: 2, row: 1 }`.
- The `diagramStore` spec is the single source of truth — refreshing the page and reloading from the backend produces identical React Flow layout.
- All 20 node kind custom components render without errors in the grid editor.
- The inspector Spec Editor accepts a valid JSON spec edit and updates the grid editor and ASCII preview.
- On mobile (<768px), authenticated workspace routes display the "use desktop" banner instead of the editor.
- The public share route renders a view-only diagram on mobile without auth.
