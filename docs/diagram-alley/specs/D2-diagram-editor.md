---
id: D2
tier: domain
status: draft
version: "0.1"
depends-on: [F1, F2, F4, F5]
tags: [editor, workspace, grid, react-flow, inspector, reverse-sync, undo]
---

# D2 — Diagram Editor

**Version:** 0.1
**Status:** Draft
**Depends on:** F1, F2, F4, F5

---

## Purpose

Define all diagram editor interactions: creating and editing diagrams in the visual grid editor, the inspector panel (properties, validation, spec escape hatch), ASCII panel editing and reverse sync, undo/redo, unsaved change tracking, and auto-save behavior. This spec governs the editor's API surface and state machine.

## Non-Goals

- AI generation and improve flows (→ D1)
- React Flow component architecture and grid coordinate mapping (→ F4 §6)
- Rendering rules and output format (→ F2)
- Version history and restore (→ D3)
- Export file packaging (→ D4)
- Import from external format (→ D5)

---

## Owned Concepts

Create diagram flow; editor interaction contract; node/edge CRUD operations; inspector panel tabs; ASCII reverse sync trigger; detached export enforcement; undo/redo scope; auto-save policy; diagram endpoint full contract.

---

## 1. Create Diagram Flow

### 1.1 Entry Points

| Entry | URL | Initial State |
|-------|-----|---------------|
| New Diagram button (sidebar) | `/workspace/new` | Blank spec for selected type |
| Quick Create (dashboard) | `/workspace/new?type=<type>` | Blank spec for pre-selected type |
| From Template | `/workspace/new?template=<template_id>` | Template spec cloned |
| Open Existing | `/workspace/:diagramId` | Saved spec loaded from API |
| After AI generation (D1) | `/workspace/:diagramId` | AI-generated spec persisted with an initial version snapshot |

### 1.2 New Blank Diagram

When the user opens `/workspace/new`:

1. Frontend renders workspace with a blank spec for the default diagram type (from `user_settings.default_diagram_type`, or `architecture` if unset).
2. A diagram type selector is shown in the prompt panel (can be changed until first save).
3. The diagram is **not persisted** until the user explicitly saves or triggers AI generation.
4. The browser title shows "Untitled – Diagram Alley".
5. If the user navigates away without saving, a browser `beforeunload` prompt is shown.

### 1.3 New From Template

When `template_id` is provided:

1. `GET /api/v1/templates/{template_id}` to load the template spec.
2. A new diagram is created in memory with `spec_json` cloned from the template.
3. `title` is set to `"<template_title> Copy"`.
4. Not persisted until first save.

### 1.4 Open Existing Diagram

1. `GET /api/v1/diagrams/{diagramId}` — full diagram including `spec_json`.
2. `spec_json` is checked for `spec_version`; if migration needed, the backend applies it before responding (F1 §1).
3. React Flow layout is initialized from `spec_json` via `useSpecToFlow` (F4 §6.2).
4. `ascii_cache` is shown immediately in the Preview panel; re-rendered asynchronously if cache is stale (null).

---

## 2. Diagram API Contract (Full)

F5 §10 lists the endpoints. This section defines the request/response contract for each.

### 2.1 `GET /api/v1/diagrams`

Lists diagrams owned by the requesting user. Not project-scoped at the top level (use `GET /api/v1/projects/{id}/diagrams` for project scope).

**Query parameters:**
- `page`, `page_size` (F5 §7)
- `sort_by`: `created_at | updated_at | title` (default `updated_at`)
- `sort_dir`: `asc | desc` (default `desc`)
- `type`: filter by `diagram_type`
- `project_id`: filter by project
- `archived`: `false` (default) | `true`

**Response:** List envelope with diagram summary objects (no `spec_json` — only id, title, diagram_type, updated_at, is_archived).

### 2.2 `POST /api/v1/diagrams`

Creates a blank diagram (non-AI). Used when the user explicitly saves from the workspace for the first time.

```json
{
  "title": "string (required)",
  "diagram_type": "string (required)",
  "project_id": "uuid | null",
  "spec_json": { "...": "full spec" }
}
```

Returns the created diagram object including `ascii_cache` and `mermaid_cache` (rendered synchronously on create).

### 2.3 `GET /api/v1/diagrams/{id}`

Returns the full diagram including `spec_json`, `ascii_cache`, `mermaid_cache`.

### 2.4 `PATCH /api/v1/diagrams/{id}`

Partial update. All fields are optional.

```json
{
  "title": "string",
  "spec_json": { "...": "full spec or null" },
  "project_id": "uuid | null",
  "create_version_snapshot": false,
  "change_summary": "string | null"
}
```

When `spec_json` is provided:
1. Check subscription gate: if the user's subscription is not `trialing` or `active`, or `access_ends_at` has passed, return `403 SUBSCRIPTION_REQUIRED`. Title and `project_id` updates in the same request are still permitted; only `spec_json` is blocked. (D7 §2.3, DEC-022)
2. Validate the new spec (F2 `validate_spec`).
3. If validation errors exist, return `400 SPEC_VALIDATION_FAILED` with error detail.
4. If valid, write `spec_json`, clear `ascii_cache` and `mermaid_cache` (set to NULL), update `updated_at`.
5. Re-render ASCII and Mermaid synchronously; return fresh caches in response.
6. If `create_version_snapshot = true`, create a `diagram_versions` row with `change_summary` or the default summary for the triggering flow. Otherwise, no version snapshot is created.

Returns the updated diagram object.

Updating `project_id` moves the diagram to another project owned by the same user (DEC-028). The service must reject a `project_id` that is missing, archived, or not owned by the current user with `404 PROJECT_NOT_FOUND`.

### 2.5 `DELETE /api/v1/diagrams/{id}`

Archives the diagram (`is_archived = true`). Returns `204 No Content`.

Hard delete is only performed via the account deletion flow and is not exposed as a standalone API operation.

### 2.6 `POST /api/v1/diagrams/{id}/render`

Forces a re-render of ASCII and Mermaid from the current `spec_json`. Useful if the cache is stale or null.

No request body. Returns:
```json
{
  "data": {
    "ascii": "...",
    "mermaid": "...",
    "warnings": []
  }
}
```

Also writes the new cache values to `ascii_cache` and `mermaid_cache` in the database.

---

## 3. Visual Grid Editor Interactions

The grid editor architecture (React Flow, coordinate system, `useSpecToFlow` hook, `nodeTypes`, store wiring) is defined in F4 §6. This section defines the interaction behaviors.

### 3.1 Node Drag

When a node is dragged to a new position:
1. React Flow fires `onNodeDragStop` with new pixel `{x, y}`.
2. `useGridSnap` converts to grid: `col = Math.round((x - 40) / 220)`, `row = Math.round((y - 40) / 120)` (F4 §6.1).
3. `diagramStore.updateNodePosition(nodeId, {col, row})` updates the in-memory spec.
4. Grid editor re-renders immediately (optimistic).
5. A 300ms debounced call to `PATCH /api/v1/diagrams/{id}` is queued with the updated `spec_json`.
6. ASCII re-render is triggered asynchronously (calls `POST /api/v1/diagrams/{id}/render`) and updates the Preview panel.

Nodes snap to the grid at all times (`snapGrid={[220, 120]}` in React Flow). Free-form positioning is not permitted.

### 3.2 Node Selection

Clicking a node:
1. Sets `diagramStore.selectedNodeId`.
2. Inspector panel switches to the **Selection** tab and shows the node's editable properties.

Clicking empty canvas: clears selection. Multi-select (shift+click or drag box) selects multiple nodes but only shows shared fields in the inspector.

### 3.3 Add Node

The inspector or a toolbar action opens an "Add Node" control:
1. User picks a `kind` from the node kind dropdown (20 kinds for architecture; step kinds for flowchart; etc.).
2. User enters a `label`.
3. Frontend assigns a unique `id` (snake_case derived from label; auto-incremented suffix on collision).
4. Node is placed at the next available grid position (first empty `{col, row}` scanning left-to-right, top-to-bottom).
5. `diagramStore` is updated; PATCH debounce fires.

### 3.4 Delete Node

Selecting a node and pressing `Delete` or `Backspace`:
1. Checks if any edges reference this node (`from` or `to`). If yes, those edges are also removed (cascade in-memory).
2. `diagramStore` updated; PATCH debounce fires.
3. Undo entry recorded (§4).

### 3.5 Edge Creation

React Flow's built-in connection drag from a node handle:
1. User drags from a source handle to a target node.
2. Frontend assigns a unique edge `id`.
3. Default style: `solid`.
4. Optional label can be added via the inspector.
5. `diagramStore` updated; PATCH debounce fires.

### 3.6 Edge Deletion

Select edge and press `Delete`:
1. Edge removed from `diagramStore`.
2. PATCH debounce fires.

### 3.7 Group Operations

Groups are React Flow `parentNode` containers (F4 §6.5):
- **Create group:** Select 2+ nodes → "Group" button in inspector → group node created, selected nodes assigned `group_id`.
- **Rename group:** Click group label in React Flow → inline text edit.
- **Dissolve group:** Select group → "Ungroup" → clears `group_id` from all child nodes, removes group entity.
- **Drag group:** Moves all children with it (React Flow handles this via `extent: 'parent'`).

### 3.8 Zoom and Pan

React Flow's built-in zoom (scroll wheel, pinch) and pan (drag on empty canvas). Toolbar buttons: zoom in, zoom out, fit view. No constraints on max zoom in V1.

---

## 4. Undo / Redo

**Scope:** Single-level undo only in V1 (one undo step).

**What is undoable:**
- Node position changes (drag)
- Node addition
- Node deletion
- Edge creation
- Edge deletion
- Group creation / dissolution
- Label edits via the inspector

**What is NOT undoable:**
- AI generation (use version history instead — D3)
- Spec editor (escape hatch) changes
- Saves and auto-saves

**Implementation:**
- `diagramStore` keeps a `previousSpec` snapshot taken immediately before each undoable action.
- `Ctrl+Z` / `Cmd+Z` calls `diagramStore.undo()` which swaps `spec_json` with `previousSpec`.
- `Ctrl+Shift+Z` / `Cmd+Shift+Z` is not implemented in V1 (no redo stack).
- After undo, the PATCH debounce fires with the reverted spec.

---

## 5. Live Persistence and Version Snapshots

### 5.1 Change State Flags

The workspace tracks two separate states (DEC-021):

- `has_unpersisted_changes`: the in-memory `spec_json` differs from the last successful PATCH/POST response.
- `has_unversioned_changes`: the live persisted `spec_json` differs from the latest `diagram_versions` snapshot.

The workspace header shows "Saving..." while `has_unpersisted_changes` is true and a persistence request is in flight. The Save button is enabled while `has_unversioned_changes` is true.

### 5.2 Manual Save / Snapshot

The workspace header has a `[Save]` button. Clicking it:
1. Flushes any pending live-persistence PATCH immediately.
2. Calls `PATCH /api/v1/diagrams/{id}` with `create_version_snapshot = true` and `change_summary = "Manual save"` (or `POST /api/v1/diagrams` for a brand-new diagram).
3. Creates a version snapshot (D3 trigger 1 — explicit save).
4. Clears `has_unpersisted_changes` and `has_unversioned_changes`.

Keyboard shortcut: `Ctrl+S` / `Cmd+S`.

For a brand-new diagram (not yet persisted), Save calls `POST /api/v1/diagrams` and creates the initial manual-save snapshot.

### 5.3 Live Auto-Save

Live auto-save persists the current spec without creating a version snapshot (DEC-021).

Auto-save fires:
- 300ms after interactive editor changes (drag, add/delete node, inspector edit), using the existing PATCH debounce.
- After 30 seconds of inactivity as a fallback if a previous debounce did not complete.
- On browser `beforeunload` if `has_unpersisted_changes`.

Auto-save calls `PATCH /api/v1/diagrams/{id}` with `create_version_snapshot = false`. Snapshots are only created on:
1. Explicit Save (§5.2)
2. Initial AI generation (D1 §3.3)
3. AI modification accepted (D1 §4.2)
4. Version restore (D3)

### 5.4 Navigation Away Guard

If `has_unpersisted_changes` and the user navigates to a different route, the frontend shows a confirmation dialog:
> "Your latest changes are still saving. Save now or discard?"
> [Save] [Discard] [Cancel]

---

## 6. Inspector Panel

The inspector panel (F4 §5.5) has three tabs. This section defines their full behavior.

### 6.1 Selection Tab

Shown when a node or edge is selected in the grid.

**Node fields (editable):**
- `label` — text input; changes fire an immediate `diagramStore.updateNodeLabel(nodeId, label)` and PATCH debounce
- `kind` — dropdown (architecture diagrams only); changing kind re-renders the node component and queues PATCH
- `position.col`, `position.row` — numeric inputs; same flow as drag (§3.1)
- `group_id` — dropdown to assign/remove from group

**Edge fields (editable):**
- `label` — text input
- `style` — dropdown: `solid | dashed | dotted`

**Database diagram specifics:**
- When a table node is selected: show column editor (add/remove/reorder columns, edit name, type, annotations).
- Column changes update `spec_json.tables[n].columns` and queue PATCH.

**Flowchart specifics:**
- When a step is selected: editable `kind` (start, end, process, decision, io) and `label`.

**Project assignment (DEC-028):**
- The diagram action menu includes "Move to project".
- The move dialog lists active projects owned by the user.
- Selecting a project sends `PATCH /api/v1/diagrams/{id}` with only `project_id`.
- Moving a diagram does not create a version snapshot because `spec_json` is unchanged.

**No multi-select editing** in V1 — if multiple nodes are selected, the Selection tab shows "Multiple nodes selected. Deselect to edit."

### 6.2 Validation Tab

Shows live validation state from the last `validate_spec` call:

- **Error list:** red badge count, each error with code and path (e.g. `nodes[1].id`)
- **Warning list:** yellow badge count
- **Auto-fix button:** calls `repair_spec` (F2) on the current spec; if any repairs are applied, shows what was fixed and updates `diagramStore`; PATCH fires after repair
- **Re-validate button:** forces re-validation without repair

Validation runs:
- After every PATCH response
- After every undo
- After every spec editor apply (§6.3)
- Not on every keypress (debounced at 500ms after changes)

### 6.3 Spec Editor Tab (Escape Hatch)

CodeMirror 6 editor with JSON syntax highlighting. Displays the current `spec_json` formatted with 2-space indentation.

Controls:
- **Format** — re-formats JSON in place
- **Validate** — runs `validate_spec` and shows results below the editor
- **Apply** — replaces `diagramStore.spec_json` with the editor content; validation must pass before Apply is enabled. On Apply: grid editor re-renders, ASCII re-render queued.
- **Discard** — resets editor content to current `diagramStore.spec_json`

YAML toggle: the editor can display content as YAML (re-serialized client-side). All edits are deserialized back to JSON before Apply. This is a display layer — storage is always JSON (DEC-011).

---

## 7. ASCII Reverse Sync

This section implements DEC-005 (label changes only in V1).

### 7.1 ASCII Panel Editability

The ASCII preview in the Preview panel is displayed in a `<pre>` that is **not** directly editable by default. A toggle `[Edit ASCII]` switches it to a `<textarea>` with the same monospace content.

### 7.2 Reverse Sync — Label Change Detection

When the user edits the ASCII textarea and clicks `[Sync]` (or focuses away):

1. The backend calls `POST /api/v1/diagrams/{id}/sync-ascii` with `{ "ascii": "<edited ascii text>" }`.
2. The sync service (F2 reverse sync) diffs the new ASCII against the last known render.
3. If only node or edge labels have changed (matched by the `sync_map`): apply label changes to `spec_json`, return updated spec.
4. If structural changes are detected (new nodes, missing nodes, different connections): do NOT update spec; return `{ "synced": false, "reason": "structural_changes_detected" }`.

### 7.3 Detached Export State

When structural ASCII changes are detected (§7.2) or the user directly downloads the ASCII without syncing:

1. `diagrams.ascii_is_detached` is set to `true`.
2. A persistent warning banner appears above the ASCII panel:

   > ⚠️ **This ASCII output has been manually edited and is no longer in sync with your diagram spec.** Changes made here won't affect the structured diagram. [Reset to spec] [Keep detached]

3. The `[Reset to spec]` action: re-renders ASCII from the current spec and clears `ascii_is_detached`.
4. The `[Keep detached]` action: dismisses the banner for this session but the flag remains in the database.

**Note:** `ascii_is_detached` column is in the `diagrams` table (F1 §4.3).

### 7.4 Sync-ASCII Endpoint

**Endpoint:** `POST /api/v1/diagrams/{id}/sync-ascii`

**Request:**
```json
{ "ascii": "string (the edited ASCII text)" }
```

**Response (label sync succeeded):**
```json
{
  "data": {
    "synced": true,
    "labels_changed": [
      { "entity_type": "node", "entity_id": "api_server", "old_label": "API Server", "new_label": "FastAPI Server" }
    ],
    "spec_json": { "...": "updated spec" }
  }
}
```

**Response (structural changes — no sync):**
```json
{
  "data": {
    "synced": false,
    "reason": "structural_changes_detected",
    "spec_json": null
  }
}
```

---

## 8. Diagram Title Editing

The diagram title is editable in the workspace header (inline click-to-edit). Saving the title calls `PATCH /api/v1/diagrams/{id}` with `{ "title": "new title" }`. Title changes do not create version snapshots and do not affect `spec_json`.

---

## Open Questions

None. All D2 decisions are locked.

---

## Acceptance Criteria

- Dragging a node to a new grid position updates `spec_json` via PATCH within 300ms debounce and re-renders ASCII.
- Adding a node via the inspector creates a valid spec entry with a unique snake_case id.
- Deleting a node also removes all edges referencing that node from the spec.
- `Ctrl+Z` reverts the spec to the state before the last undoable action.
- A new diagram is only persisted on explicit Save (or AI generation); navigating away from an unsaved diagram shows a confirmation dialog.
- A valid JSON spec pasted into the Spec Editor and confirmed with Apply updates the grid editor layout.
- Editing a node label in the ASCII panel and clicking Sync updates `spec_json` with the new label.
- Editing a structural element (adding a box) in the ASCII panel and clicking Sync returns `{ "synced": false }` and sets `ascii_is_detached = true`.
- The detached ASCII banner appears when `ascii_is_detached = true` and is cleared by Reset to spec.
- `GET /api/v1/diagrams` with `sort_by=title&sort_dir=asc` returns diagrams alphabetically.
