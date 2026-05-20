---
id: F2
tier: foundation
status: draft
version: "0.1"
depends-on: [F0, F1]
tags: [renderer, validation, repair, ascii, mermaid, visual]
---

# F2 — Rendering, Validation, and Repair

**Version:** 0.1
**Status:** Draft
**Depends on:** F0, F1

---

## Purpose

Define the deterministic rendering contract for all output formats, the validation rules for every diagram type, the repair heuristics applied to AI-generated specs, and the reverse sync boundary. This spec is the authority for what "correct output" means.

## Non-Goals

- AI generation prompts (→ D1)
- Editor interactions (→ D2)
- Export file packaging (→ D4)
- Import and conversion (→ D5)

---

## Owned Concepts

ASCII/Text Renderer, Mermaid Renderer, Visual Renderer; validation rules per diagram type; repair pass; reverse sync scope; detached export state; node kind rendering table.

---

## 1. Core Rendering Principle

**The renderer is the product.** The LLM produces structured intent (a Diagram Spec). Diagram Alley renders output through deterministic rules. Given the same spec and the same layout metadata, the renderer always produces the same output.

The flow is always:

```
Diagram Spec (JSON)
    → Validation
    → (Repair Pass if invalid)
    → Re-validation
    → Renderer
    → Output (ASCII text | UTF-8 text | Mermaid | SVG | PNG)
```

A spec with unresolved validation errors is never rendered. The renderer is never called with an invalid spec.

---

## 2. Validation

### 2.1 Universal Validation (all diagram types)

These checks run before type-specific checks:

| Rule | Error message |
|------|--------------|
| `spec_version` must be present and recognized | `"spec_version is missing or unrecognized: {value}"` |
| `diagram_type` must be one of the six defined values | `"diagram_type '{value}' is not supported"` |
| `title` must be present and non-empty | `"title is required"` |

### 2.2 Architecture Diagram Validation

| Rule | Severity | Error |
|------|----------|-------|
| Node `id` must be unique within spec | ERROR | `"Duplicate node id: '{id}'"` |
| Node `id` must be non-empty snake_case | ERROR | `"Node id must be non-empty snake_case: '{id}'"` |
| Node `label` must be non-empty | ERROR | `"Node '{id}' has an empty label"` |
| Node `kind` must be one of the 20 defined values | ERROR | `"Unknown node kind '{kind}' on node '{id}'"` |
| Edge `from` must reference an existing node id | ERROR | `"Edge '{id}': source node '{from}' does not exist"` |
| Edge `to` must reference an existing node id | ERROR | `"Edge '{id}': target node '{to}' does not exist"` |
| Edge `id` must be unique within spec | ERROR | `"Duplicate edge id: '{id}'"` |
| Group `node_ids` must all reference existing node ids | WARNING | `"Group '{id}' references unknown node '{node_id}'"` |
| `position.col` and `position.row` must be non-negative integers if present | WARNING | `"Node '{id}' has an invalid position"` |
| `direction` must be `top_down` or `left_right` if present | WARNING | `"Unknown direction '{value}'; defaulting to top_down"` |

### 2.3 Database Schema Validation

| Rule | Severity | Error |
|------|----------|-------|
| Table `id` unique | ERROR | `"Duplicate table id: '{id}'"` |
| Table `label` non-empty | ERROR | `"Table '{id}' has an empty label"` |
| Each table must have at least one column | WARNING | `"Table '{id}' has no columns"` |
| Column `name` unique within table | ERROR | `"Duplicate column name '{name}' in table '{id}'"` |
| Relationship `from_table` and `to_table` must exist | ERROR | `"Relationship '{id}': table '{table}' does not exist"` |
| Relationship `from_column` must exist in `from_table` | ERROR | `"Relationship '{id}': column '{col}' not found in table '{table}'"` |
| Relationship `type` must be valid | ERROR | `"Unknown relationship type: '{type}'"` |

### 2.4 Flowchart Validation

| Rule | Severity | Error |
|------|----------|-------|
| Step `id` unique | ERROR | Duplicate step id |
| Step `kind` must be valid | ERROR | Unknown step kind |
| Step `label` non-empty | ERROR | Empty label |
| Exactly one `start` step | WARNING | `"Flowchart should have exactly one start step"` |
| Exactly one `end` step | WARNING | `"Flowchart should have exactly one end step"` |
| Transition `from` and `to` must reference existing step ids | ERROR | Transition references unknown step |
| Decision steps should have exactly two outgoing transitions | WARNING | `"Decision step '{id}' has {n} outgoing transitions; expected 2"` |

### 2.5 UI Wireframe Validation

| Rule | Severity | Error |
|------|----------|-------|
| Component `id` unique across entire tree | ERROR | Duplicate component id |
| `root` must be present | ERROR | `"root is required for ui_wireframe"` |
| Root `kind` must be `page` | ERROR | `"root kind must be 'page'"` |
| Component `kind` must be valid | ERROR | Unknown component kind |
| Component `label` non-empty | ERROR | Empty label |
| `layout` must be `column`, `row`, or null if present | WARNING | Unknown layout value |

### 2.6 File Structure Validation

| Rule | Severity | Error |
|------|----------|-------|
| Node `id` unique | ERROR | Duplicate node id |
| `root` must be present and `kind: folder` | ERROR | Root must be a folder |
| File nodes must have no children | WARNING | `"File '{id}' has children; files cannot have children"` |
| Node `label` non-empty | ERROR | Empty label |

### 2.7 Network Diagram Validation

| Rule | Severity | Error |
|------|----------|-------|
| Device `id` unique | ERROR | Duplicate device id |
| Device `kind` must be valid | ERROR | Unknown device kind |
| Device `label` non-empty | ERROR | Empty label |
| Connection `from` and `to` must reference existing device ids | ERROR | Connection references unknown device |
| Zone `device_ids` must all reference existing devices | WARNING | Zone references unknown device |

### 2.8 Validation Output Shape

```json
{
  "valid": false,
  "errors": [
    { "severity": "error", "code": "DUPLICATE_NODE_ID", "message": "Duplicate node id: 'api'", "path": "nodes[1].id" }
  ],
  "warnings": [
    { "severity": "warning", "code": "MISSING_END_STEP", "message": "Flowchart should have exactly one end step", "path": "steps" }
  ],
  "repair_suggestions": [
    { "code": "REMOVE_INVALID_EDGE", "message": "Edge 'e3' removed (target node 'db' not found)", "auto_fixable": true }
  ]
}
```

A spec is valid only when `errors` is empty. `warnings` do not block rendering. `repair_suggestions` are available to the repair pass.

---

## 3. Repair Pass

The repair pass runs automatically after AI-generated spec validation fails with recoverable errors. It is also available as a manual "Auto-fix" action in the UI.

### 3.1 Repair Rules (applied in order)

1. **Fill missing node IDs.** If a node has no `id`, generate one as `node_{index}`.
2. **Remove invalid edges.** If an edge `from` or `to` references a non-existent node id, remove the edge.
3. **Remove invalid relationship references.** Same logic for database diagrams.
4. **Normalize diagram type.** If `diagram_type` is a known variant spelling (e.g. `"arch"`, `"architecture_diagram"`), coerce to the canonical value.
5. **Default missing `direction`.** If `direction` is absent or invalid, set `"top_down"`.
6. **Default missing `style` on edges.** Set `"solid"`.
7. **Remove duplicate IDs.** Keep the first occurrence, remove subsequent duplicates, log a warning.
8. **Truncate oversized labels.** If a label exceeds 100 chars, truncate to 97 + `"..."`.
9. **Strip file children.** If a file node in a file_structure spec has children, move children to be siblings and log a warning.

### 3.2 Repair Limits

The repair pass runs at most **twice** before giving up. If validation still fails after two repair attempts, the app returns the validation error to the client without rendering and preserves the last valid spec.

### 3.3 Repair Transparency

Every repair action is logged in the API response as a `repair_applied` entry:

```json
"repairs_applied": [
  { "rule": "REMOVE_INVALID_EDGE", "detail": "Removed edge 'e3' (target 'db' not found)" }
]
```

The UI displays repairs as a non-blocking warning banner.

---

## 4. ASCII/Text Renderer

### 4.1 Contract

- **Input:** A validated Diagram Spec (JSON) + optional layout metadata (grid positions from the Visual Grid Editor).
- **Output:** A deterministic plain-text string in either strict ASCII mode or UTF-8 text mode (DEC-023).
- **Strict ASCII mode:** Uses printable ASCII characters only (+ `\n`).
- **UTF-8 text mode:** May use Unicode tree/box glyphs for readability. UTF-8 output must not be labeled ASCII.
- **Determinism:** Same spec + same positions + same text mode → identical output, always.
- **Width:** Box widths are calculated from content (label length + fixed padding of 2 chars each side). No hard max width in V1 — the consumer (UI, export) controls truncation or scrolling.
- **Direction:** `top_down` renders nodes in rows; `left_right` renders nodes in columns.

### 4.2 Box Drawing Characters

V1 uses simple ASCII box drawing:

```
+------------------+
| Node Label       |
| kind_value       |
+------------------+
```

- Top/bottom border: `+` at corners, `-` for runs
- Side borders: `|`
- Arrows down: `|` then `v`
- Arrows right: `-` then `>`
- Arrow labels appear inline on the connecting line

### 4.3 Architecture Node Kind Rendering

All 20 named kinds get distinct visual treatment in V1 (DEC-012). Rendering is expressed as: the **box border style** and a **kind annotation line** inside the box.

| Kind | Border style | Kind annotation (shown below label) |
|------|-------------|-------------------------------------|
| `user_client` | `[ ]` brackets | `[user]` |
| `frontend` | standard `+--+` | `frontend` |
| `backend` | standard | `backend` |
| `service` | standard | `service` |
| `gateway` | `>--<` chevron | `gateway` |
| `database` | `(( ))` cylinder hint | `database` |
| `cache` | standard | `cache` |
| `queue` | `{ }` braces | `queue` |
| `worker` | standard | `worker` |
| `external_api` | `[[ ]]` double bracket | `external` |
| `auth` | standard | `auth` |
| `storage` | standard | `storage` |
| `cdn` | standard | `cdn` |
| `message_broker` | `{ }` braces | `broker` |
| `job_scheduler` | standard | `scheduler` |
| `search_index` | standard | `search` |
| `file_blob_storage` | standard | `blob` |
| `third_party_service` | `[[ ]]` double bracket | `3rd party` |
| `internal_service` | standard | `internal` |
| `generic` | standard | *(no annotation)* |

**ASCII example:**

```
+--------------------+      +====================+      +--------------------+
| React Frontend     |      | FastAPI Backend    |      | Postgres           |
| frontend           | ---> | service            | ---> | ((  database  ))   |
+--------------------+      +====================+      +--------------------+
```

*(The exact border characters for each kind are finalized in the renderer implementation; the above is illustrative. The renderer spec table above is the authority for which style maps to which kind.)*

### 4.4 Database Table Box Rendering

```
+----------------------+
| users                |
+----------------------+
| id          UUID  PK |
| email       TEXT     |
| created_at  DATE     |
+----------+-----------+
```

Column alignment: name and type are left-aligned, annotation (PK, FK) is right-aligned. Separator line between table name header and columns.

### 4.5 Edge and Arrow Rendering

- **Top-down edges:** Vertical `|` runs with `v` arrowhead, edge label centered on the line.
- **Left-right edges:** Horizontal `-` runs with `>` arrowhead, edge label above the line.
- **Cross edges** (connecting non-adjacent columns): routed with `+` junctions at turns.
- **Dashed edges in strict ASCII mode:** Use `.` instead of `-` or `|`.
- **Dashed edges in UTF-8 text mode:** May use `·` for a clearer dotted line.

### 4.6 File Structure Rendering

```
project/
├── frontend/
│   ├── src/
│   │   └── App.tsx
└── backend/
    └── main.py
```

UTF-8 text mode uses Unicode box-drawing for tree lines (`├──`, `│`, `└──`) because it is significantly more readable.

Strict ASCII mode uses this fallback:

```
project/
|-- frontend/
|   |-- src/
|   |   `-- App.tsx
`-- backend/
    `-- main.py
```

### 4.7 Layout Metadata

When the Visual Grid Editor provides `position.col` and `position.row` for nodes, the ASCII renderer:
1. Groups nodes by `row` value.
2. Within each row, orders nodes by `col` value.
3. Renders each row as a horizontal group of boxes.
4. Draws edges between rows as vertical connections, between same-row nodes as horizontal connections.

When positions are absent, the renderer assigns positions automatically using a simple top-down flow layout (each edge adds one row between source and target).

---

## 5. Mermaid Renderer

### 5.1 Contract

- **Input:** Validated Diagram Spec (JSON).
- **Output:** A valid Mermaid string.
- **Diagram type mapping:**
  - `architecture` → `graph TD` or `graph LR` (based on direction)
  - `database` → `erDiagram`
  - `flowchart` → `flowchart TD` or `flowchart LR`
  - `ui_wireframe` → Not supported in V1 (export returns an error with explanation)
  - `file_structure` → Not supported in V1
  - `network` → `graph TD` (best effort)

### 5.2 Architecture → Mermaid

```
graph TD
    api["FastAPI Backend\nservice"]
    db[("Postgres\ndatabase")]
    frontend["React Frontend\nfrontend"]
    frontend -->|HTTP| api
    api -->|SQLAlchemy| db
```

Node shape map to Mermaid node syntax:
- `database` kind → `[(label)]` (cylindrical)
- `queue` / `message_broker` kind → `{{label}}` (hexagon)
- `external_api` / `third_party_service` kind → `[[label]]` (subroutine)
- `gateway` kind → `>label]` (asymmetric)
- `user_client` kind → `([label])` (stadium)
- All others → `[label]` (rectangle)

### 5.3 Database → Mermaid ER

```
erDiagram
    USERS {
        UUID id PK
        TEXT email
        DATE created_at
    }
    ORDERS {
        UUID id PK
        UUID user_id FK
        DECIMAL total
    }
    USERS ||--o{ ORDERS : "has many"
```

Relationship type mapping:
- `one_to_one` → `||--||`
- `one_to_many` → `||--o{`
- `many_to_many` → `}o--o{`

---

## 6. Visual Renderer

### 6.1 Contract

The browser visual renderer is powered by **React Flow** (DEC-013). It reads the Diagram Spec and the same layout metadata used by the ASCII/Text Renderer.

- **Input:** Diagram Spec (JSON) + layout positions → React Flow `nodes[]` and `edges[]` arrays.
- **Output:** An interactive SVG canvas rendered in the browser.
- **Determinism:** Same spec + same positions → same initial render. User drag operations update positions in the spec (round-trip).

For export, the visual export renderer serializes the same validated spec and layout metadata into a standalone SVG string. PNG export is generated from that deterministic SVG output (DEC-019); it is not a screenshot-only workflow.

### 6.2 Grid Coordinate → React Flow Position Mapping

This mapping is the critical bridge between ASCII layout and the visual editor.

Each node's `position.col` and `position.row` maps to pixel coordinates in the React Flow canvas:

```
x = col * CELL_WIDTH  + CELL_PADDING
y = row * CELL_HEIGHT + CELL_PADDING
```

Constants (defined in `frontend/src/renderers/visual/constants.ts`):

| Constant | Value | Rationale |
|----------|-------|-----------|
| `CELL_WIDTH` | 220px | Wide enough for most labels + kind annotation |
| `CELL_HEIGHT` | 120px | Tall enough for 2-line node boxes |
| `CELL_PADDING` | 40px | Margin from canvas edge |

When a user drags a node in React Flow, the new pixel position is converted back to `(col, row)` by reversing the formula (integer division). The spec is updated and the ASCII renderer re-runs with the new positions.

**Snapping:** React Flow's snap-to-grid is enabled with `snapToGrid={true}` and `snapGrid={[CELL_WIDTH, CELL_HEIGHT]}` so visual drag positions always resolve to integer grid coordinates.

### 6.3 Node Appearance in React Flow

Each `node.kind` maps to a custom React Flow node component. The component renders the label, kind annotation, and the border style defined in §4.3. In V1, these are simple styled `div` elements — no SVG icons required. Icons are a V2 enhancement.

### 6.4 Edge Appearance in React Flow

- Solid edges: `type="smoothstep"` with solid stroke
- Dashed edges: same type with `strokeDasharray="5,5"`
- Labels rendered as React Flow `EdgeLabel` components

---

## 7. Reverse Sync

### 7.1 Scope (DEC-005)

**V1 scope is label changes only.** If a user edits a node or edge label in the ASCII output, the app attempts to sync that change back into the Diagram Spec. All other ASCII edits produce a Detached Export with a warning. Full reverse sync is V2.

### 7.2 Label Change Detection Algorithm

1. The app stores a `sync_map`: a mapping from each rendered ASCII label string to the node or edge ID it came from.
2. On ASCII edit, the app diffs the edited ASCII against the original rendered output.
3. If the only changes are in label positions tracked in `sync_map`, each changed label value is written back to the corresponding node or edge `label` field in the spec.
4. If any change is detected outside tracked label positions, the spec is not updated. The diagram enters **detached export state**.

### 7.3 Detached Export State

When a diagram's ASCII output has been edited in ways beyond label changes:

- The `diagrams.ascii_cache` column stores the edited ASCII as-is.
- The `diagrams.ascii_is_detached` column (F1 §4.3) is set to `true`.
- The UI shows a persistent warning banner: *"This ASCII has been manually edited and is no longer synced with the diagram spec. Edits beyond label changes are not reflected in the spec. Re-render from spec to discard manual changes."*
- The user can: (a) re-render from spec (loses ASCII edits), or (b) keep the detached ASCII for export only.
- Exporting a detached diagram shows the same warning in the export UI.

---

## 8. Render Service Interface

The render service (`app/services/render_service.py`) exposes these functions:

```python
def render_text(spec: dict, mode: str = "ascii") -> str:
    """Validates spec, renders strict ASCII or UTF-8 text. Raises ValidationError if spec is invalid."""

def render_ascii(spec: dict) -> str:
    """Compatibility wrapper for render_text(spec, mode="ascii")."""

def render_mermaid(spec: dict) -> str:
    """Validates spec, renders Mermaid. Raises UnsupportedDiagramTypeError for unsupported types."""

def render_svg(spec: dict) -> str:
    """Validates spec, renders standalone SVG for export."""

def render_png(spec: dict) -> bytes:
    """Validates spec, renders PNG bytes from the standalone SVG output."""

def validate_spec(spec: dict) -> ValidationResult:
    """Returns ValidationResult with errors, warnings, repair_suggestions."""

def repair_spec(spec: dict) -> tuple[dict, list[RepairAction]]:
    """Attempts repair. Returns (repaired_spec, repairs_applied). Runs validation after repair."""
```

---

## Open Questions

None. All F2 decisions are locked in `decisions-log.md`.

---

## Acceptance Criteria

- The same architecture spec input produces identical text output on every call for the same text mode (determinism).
- Strict ASCII mode emits only printable ASCII characters plus newlines.
- UTF-8 text mode can emit file-tree glyphs for file structure diagrams.
- A spec with a missing node ID passes through the repair pass, receives a generated ID, and renders successfully.
- An edge referencing a non-existent node is removed by the repair pass; the diagram renders without that edge.
- A `database` kind node renders with `((  ))` cylinder border characters in ASCII.
- A `gateway` kind node renders with `>--<` chevron border in ASCII.
- A node with `position: {col: 1, row: 0}` appears at `x = 260, y = 40` in the React Flow canvas (using the constants above).
- Dragging a node in React Flow to column 2 updates `spec_json.nodes[n].position.col = 2` in the database.
- Editing a node label in ASCII output updates `spec_json.nodes[n].label` in the spec (label-change sync).
- Editing a node's border in ASCII output triggers detached export state without updating the spec.
- Architecture → Mermaid renders a valid Mermaid `graph TD` string accepted by the Mermaid parser.
- Database → Mermaid renders a valid `erDiagram` string.
- SVG export returns a standalone SVG string from the validated spec.
- PNG export returns valid PNG bytes generated from the SVG export path.
