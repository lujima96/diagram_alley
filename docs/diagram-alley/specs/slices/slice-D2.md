---
phase: 11
slice: D2
spec: specs/D2-diagram-editor.md
status: planned
---

# Slice D2 — Diagram Editor

## Pre-conditions

- [ ] Slice F1 complete — `diagrams`, `diagram_versions`, `projects` tables exist; CRUD service stubs in place.
- [ ] Slice F2 complete — `validate_spec`, `render_ascii`, `render_mermaid`, `reverse_sync` service implemented.
- [ ] Slice F4 complete — workspace panel layout, React Flow scaffold, `diagramStore`, `useSpecToFlow` hook all working.
- [ ] Slice F5 complete — response envelope, pagination, error codes, endpoint stubs registered.

---

## Build Sequence

1. **Diagram CRUD endpoints** — Replace the F5 endpoint stubs for `GET/POST/PATCH/DELETE /api/v1/diagrams` and `GET/POST/PATCH/DELETE /api/v1/projects` with real implementations. `PATCH /api/v1/diagrams/{id}` must implement the full spec from D2 §2.4: subscription gate on `spec_json` writes, validate spec, clear caches, re-render synchronously, conditional version snapshot creation. `DELETE` archives (`is_archived=true`), never hard-deletes. `project_id` update in PATCH validates that the target project belongs to the user (DEC-028). (→ D2 §2)

2. **Node/edge interaction wiring** — Wire the React Flow `onNodeDragStop`, `onConnect`, `onNodesDelete`, `onEdgesDelete` callbacks in `WorkspaceEditor.tsx` to the `diagramStore` mutations defined in D2 §3. Each interaction queues the 300ms debounced PATCH. Node deletion cascades to remove referencing edges from the spec in memory before the PATCH fires. (→ D2 §3.1–3.6)

3. **Add-node flow** — Implement the Add Node control in the inspector Selection tab: kind dropdown, label input, ID auto-generation (snake_case from label with collision suffix), placement at first available grid position. (→ D2 §3.3)

4. **Group operations** — Implement group create, rename, dissolve, and drag in the React Flow editor using the `parentNode` pattern (F4 §6.5). Group create: select 2+ nodes → inspector button → group entity added to spec, nodes gain `group_id`. Dissolve: clear `group_id` from children, remove group. (→ D2 §3.7)

5. **Undo / redo** — Implement single-level undo in `diagramStore`: `previousSpec` snapshot taken before each undoable action; `diagramStore.undo()` swaps current spec with `previousSpec`; keyboard shortcut `Ctrl+Z` / `Cmd+Z` calls it; PATCH debounce fires after undo. No redo in V1. (→ D2 §4)

6. **Auto-save and change tracking** — Implement `has_unpersisted_changes` and `has_unversioned_changes` flags in `diagramStore` per D2 §5.1. Auto-save fires 300ms after changes (debounce) and as a 30s fallback. `beforeunload` flushes pending save. Save button (explicit, `Ctrl+S`) calls PATCH with `create_version_snapshot=true`. New unsaved diagrams call `POST /api/v1/diagrams` on first save. Navigation-away guard dialog if `has_unpersisted_changes`. (→ D2 §5, DEC-021)

7. **Inspector panel — selection tab** — Wire the Selection tab to show and edit the selected node's or edge's fields from `diagramStore`. Label changes fire `diagramStore.updateNodeLabel` and queue PATCH. Kind changes re-render the node component. Database diagram column editor (add/remove/reorder columns) updates `spec_json.tables[n].columns`. "Move to project" action sends PATCH with only `project_id`. (→ D2 §6.1, DEC-028)

8. **Inspector panel — validation tab** — Wire the Validation tab to call `validate_spec` on the current spec (debounced at 500ms). Show error/warning counts with paths. Wire the Auto-fix button to call `repair_spec` and apply result to `diagramStore`. (→ D2 §6.2)

9. **Inspector panel — spec editor tab** — Wire the Spec Editor CodeMirror 6 tab to display `diagramStore.spec_json`. Apply button: validate spec first, enable Apply only if valid, then push to `diagramStore` and trigger grid re-render + ASCII re-render. YAML toggle re-serializes for display only; Apply always deserializes back to JSON. (→ D2 §6.3, DEC-011)

10. **Render endpoint** — Implement `POST /api/v1/diagrams/{id}/render`: re-render ASCII and Mermaid from current `spec_json`, write to cache columns, return fresh renders. Used when cache is stale (null) on diagram open, and after node drag. (→ D2 §2.6)

11. **ASCII reverse sync endpoint** — Implement `POST /api/v1/diagrams/{id}/sync-ascii` per D2 §7.4. Calls the F2 `reverse_sync` service: if label-only changes detected, update `spec_json` labels and return `synced=true` with `labels_changed[]`. If structural changes detected, set `ascii_is_detached=true`, return `synced=false`. Wire the ASCII `[Edit ASCII]` toggle and `[Sync]` button in the Preview panel. Show detached warning banner when `ascii_is_detached=true`. Wire `[Reset to spec]` to re-render from spec and clear `ascii_is_detached`. (→ D2 §7, F2 §7, DEC-005)

12. **Title editing** — Wire inline title editing in the workspace header to call `PATCH /api/v1/diagrams/{id}` with only `{ "title": "..." }`. Title changes do not create version snapshots and do not touch `spec_json`. (→ D2 §8)

---

## Done Criteria

- [ ] Dragging a node updates `spec_json` via PATCH within 300ms debounce and triggers ASCII re-render. (→ D2 Acceptance Criteria)
- [ ] Adding a node via the inspector creates a valid spec entry with a unique snake_case id. (→ D2 Acceptance Criteria)
- [ ] Deleting a node also removes all edges referencing it from the spec. (→ D2 Acceptance Criteria)
- [ ] `Ctrl+Z` reverts the spec to the state before the last undoable action. (→ D2 Acceptance Criteria)
- [ ] A new diagram is only persisted on explicit Save or AI generation; navigating away without saving shows a confirmation dialog. (→ D2 Acceptance Criteria)
- [ ] A valid JSON spec pasted into the Spec Editor and Applied updates the grid editor and ASCII preview. (→ D2 Acceptance Criteria)
- [ ] Editing a node label in the ASCII panel and clicking Sync updates `spec_json` with the new label. (→ D2 Acceptance Criteria)
- [ ] Editing a structural element in ASCII and clicking Sync returns `{"synced": false}` and sets `ascii_is_detached=true`. (→ D2 Acceptance Criteria)
- [ ] Detached ASCII banner appears when `ascii_is_detached=true` and is cleared by Reset to spec. (→ D2 Acceptance Criteria)
- [ ] `GET /api/v1/diagrams?sort_by=title&sort_dir=asc` returns diagrams alphabetically. (→ D2 Acceptance Criteria)
- [ ] `PATCH /api/v1/diagrams/{id}` with `spec_json` for a user with `status='canceled'` returns `403 SUBSCRIPTION_REQUIRED`; title and `project_id` updates in the same request are still permitted. (→ D2 §2.4, DEC-022)
- [ ] Moving a diagram to another project via PATCH updates `project_id` and does not create a version snapshot. (→ DEC-028)
