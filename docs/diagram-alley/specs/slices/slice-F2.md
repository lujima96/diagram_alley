---
phase: 11
slice: F2
spec: specs/F2-rendering-validation-repair.md
status: planned
---

# Slice F2 — Rendering, Validation, and Repair

## Pre-conditions

- [ ] Slice F1 complete — all ORM models exist, `alembic upgrade head` runs cleanly, Pydantic spec schemas are defined for all six diagram types.

---

## Build Sequence

1. **`validate_spec` — universal and per-type rules** — Implement `app/services/render_service.py::validate_spec(spec: dict) -> ValidationResult`. Run universal checks first (F2 §2.1), then dispatch to type-specific checks (F2 §2.2–2.7). Return `ValidationResult` with `valid`, `errors[]`, `warnings[]`, and `repair_suggestions[]` matching the shape in F2 §2.8. Errors block rendering; warnings do not. (→ F2 §2)

2. **`repair_spec` — ordered repair rules** — Implement `repair_spec(spec: dict) -> tuple[dict, list[RepairAction]]`. Apply the nine repair rules in order (F2 §3.1): fill missing node IDs, remove invalid edges, remove invalid relationships, normalize diagram type, default direction, default edge style, remove duplicate IDs, truncate oversized labels, strip file children. Re-validate after each full pass. Run at most two passes before returning failure (F2 §3.2). Return the repaired spec and `repairs_applied[]`. (→ F2 §3)

3. **ASCII renderer — architecture and database** — Implement `app/renderers/ascii/renderer.py`. Start with `architecture` and `database` diagram types; these are the most-used and establish the box-drawing conventions. Strict ASCII mode uses `+`, `-`, `|`, `v`, `>` only. Apply the node kind → border style table (F2 §4.3) and column layout for database tables (F2 §4.4). Auto-assign positions when `position` is absent using a top-down flow layout (F2 §4.7). (→ F2 §4.1–4.4, §4.7)

4. **ASCII renderer — flowchart, network, file structure** — Extend the ASCII renderer for the remaining three types. File structure uses strict ASCII tree fallback (`|--`, `` ` ``--`) and UTF-8 tree glyphs (`├──`, `└──`) based on the `mode` parameter (F2 §4.6). Edge and arrow rendering from F2 §4.5 applies to flowchart and network. (→ F2 §4.5–4.6)

5. **UTF-8 text mode** — Add `mode: str = "ascii"` parameter to `render_text`. When `mode="utf8"`, allow Unicode tree/box glyphs. Strict ASCII mode must contain only printable ASCII + `\n` — enforce with an assertion in tests. Document that UTF-8 output is never labeled as ASCII output (DEC-023). (→ F2 §4.1, DEC-023)

6. **Mermaid renderer** — Implement `app/renderers/mermaid/renderer.py::render_mermaid(spec: dict) -> str`. Cover `architecture` (→ `graph TD/LR`) and `database` (→ `erDiagram`) first, then `flowchart` (→ `flowchart TD/LR`) and `network` (→ `graph TD` best-effort). Return `UnsupportedDiagramTypeError` for `ui_wireframe` and `file_structure`. Apply the Mermaid node shape map (F2 §5.2) and ER relationship type map (F2 §5.3). (→ F2 §5)

7. **SVG renderer** — Implement `app/renderers/visual/svg_renderer.py::render_svg(spec: dict) -> str`. Generates a standalone SVG string using the same grid coordinate system as the React Flow canvas (F2 §6.2 constants: `CELL_WIDTH=220`, `CELL_HEIGHT=120`, `CELL_PADDING=40`). Node kind border styles from F2 §4.3 are expressed as SVG `rect` attributes. No browser dependency — this runs server-side for export. (→ F2 §6.1–6.3)

8. **PNG renderer** — Implement `render_png(spec: dict) -> bytes` that calls `render_svg` then converts the SVG to PNG bytes using `cairosvg` (F0 §2.1). Confirm PNG bytes are valid via `PIL.Image.open(BytesIO(...))` in tests. (→ F2 §6.1, F0 §2.1, DEC-019)

9. **React Flow visual renderer (frontend)** — Create `frontend/src/renderers/visual/` with the grid coordinate constants (`constants.ts`), a `specToFlow(spec)` function that converts a Diagram Spec to React Flow `nodes[]` and `edges[]`, and custom node components for each kind (styled `div`s matching the border styles in F2 §4.3). Enable `snapToGrid` with `[CELL_WIDTH, CELL_HEIGHT]` so drags resolve to integer grid positions. For `file_structure` and `ui_wireframe`, implement the tree-row and nested-box rendering per F2 §6.3 (DEC-030). (→ F2 §6.2–6.4)

10. **Reverse sync — label change detection** — Implement `app/services/reverse_sync.py`. On ASCII edit, compare the edited string against the original rendered output using a `sync_map` (rendered label → node/edge ID). If only label-position changes are detected, update `spec_json` labels and persist. If any non-label change is detected, set `diagrams.ascii_is_detached = true` and store the edited ASCII in `ascii_cache` without touching `spec_json`. (→ F2 §7.1–7.3, DEC-005)

11. **Render service integration** — Wire all renderers into `render_service.py` behind the public interface in F2 §8. Ensure `validate_spec` is called before every render call and raises `ValidationError` for invalid specs — the renderer must never be called with an invalid spec (F2 §1). Wire cache invalidation: when `spec_json` is written, null out `ascii_cache` and `mermaid_cache`. (→ F2 §1, §8)

---

## Done Criteria

- [ ] Same architecture spec input produces identical text output on every call for the same text mode. (→ F2 Acceptance Criteria)
- [ ] Strict ASCII mode output passes `all(32 <= ord(c) <= 126 or c == '\n' for c in output)`. (→ F2 Acceptance Criteria)
- [ ] UTF-8 text mode renders file-tree glyphs (`├──`, `└──`) for `file_structure` diagrams. (→ F2 Acceptance Criteria)
- [ ] A spec with a missing node ID passes through the repair pass, receives a generated ID, and renders successfully. (→ F2 Acceptance Criteria)
- [ ] An edge referencing a non-existent node is removed by the repair pass; the diagram renders without that edge. (→ F2 Acceptance Criteria)
- [ ] A `database` kind node renders with `(( ))` cylinder border in ASCII. (→ F2 Acceptance Criteria)
- [ ] A `gateway` kind node renders with `>--<` chevron border in ASCII. (→ F2 Acceptance Criteria)
- [ ] A node with `position: {col: 1, row: 0}` appears at `x = 260, y = 40` in the React Flow canvas. (→ F2 Acceptance Criteria, §6.2)
- [ ] Dragging a node in React Flow to column 2 updates `spec_json.nodes[n].position.col = 2` in the database. (→ F2 Acceptance Criteria)
- [ ] Editing a node label in ASCII output updates `spec_json.nodes[n].label` (label-change sync). (→ F2 Acceptance Criteria)
- [ ] Editing a node's border in ASCII triggers `ascii_is_detached = true` without touching `spec_json`. (→ F2 Acceptance Criteria)
- [ ] Architecture → Mermaid renders a `graph TD` string accepted by the Mermaid parser. (→ F2 Acceptance Criteria)
- [ ] Database → Mermaid renders a valid `erDiagram` string. (→ F2 Acceptance Criteria)
- [ ] SVG export returns a standalone SVG string from a validated spec. (→ F2 Acceptance Criteria)
- [ ] PNG export returns valid PNG bytes generated from the SVG path (not a browser screenshot). (→ F2 Acceptance Criteria, DEC-019)
