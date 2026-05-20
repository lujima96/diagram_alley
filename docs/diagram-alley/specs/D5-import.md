---
id: D5
tier: domain
status: draft
version: "0.1"
depends-on: [F1, F2, F5]
tags: [import, mermaid, json, yaml, conversion, format-detection]
---

# D5 — Import and Conversion

**Version:** 0.1
**Status:** Draft
**Depends on:** F1, F2, F5

---

## Purpose

Define the import flow for external diagram formats: Mermaid flowchart and ER (best-effort, DEC-006), JSON spec, and YAML spec. Covers format detection, conversion, validation, the conversion preview step, and corruption prevention.

## Non-Goals

- Export (→ D4)
- AI-assisted conversion (the import converter is deterministic; AI is not involved)
- GitHub commit integration (→ D8)
- Full Mermaid import across all diagram types (deferred to V2 per DEC-006)

---

## Owned Concepts

Import endpoint contract; format detection logic; Mermaid-to-spec converter (flowchart, ER); JSON/YAML spec import; import preview step; import warnings; corruption prevention contract; `DIAGRAM_IMPORTED` trigger point (constant owned by F3).

---

## 1. Supported Import Formats

| Format | `input_format` value | Scope | Notes |
|--------|---------------------|-------|-------|
| Mermaid flowchart | `mermaid` | Best-effort; DEC-006 | Supports `flowchart TD/LR` only; unsupported constructs produce warnings |
| Mermaid ER | `mermaid` | Best-effort; DEC-006 | Supports `erDiagram` only |
| JSON spec | `json` | Full fidelity | Must conform to F1 §3 schema |
| YAML spec | `yaml` | Full fidelity | Same as JSON spec; re-serialized to JSON before processing |

**Not supported in V1:**
- Mermaid sequence, state, class, gantt, etc. → `400 IMPORT_UNSUPPORTED_FORMAT`
- Lucidchart, draw.io, PlantUML → `400 IMPORT_UNSUPPORTED_FORMAT`

---

## 2. Import Endpoint

**`POST /api/v1/diagrams/import`**

### 2.1 Request Body

```json
{
  "input_format": "mermaid",
  "content": "flowchart TD\n  A[Start] --> B[Process]\n  B --> C[End]",
  "title": "My Imported Flowchart",
  "project_id": "uuid | null",
  "preview_only": false
}
```

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| `input_format` | Yes | — | `mermaid \| json \| yaml` |
| `content` | Yes | — | Raw import content string; max 50,000 characters |
| `title` | No | Derived (see §5) | Diagram title |
| `project_id` | No | `null` | Assign to project |
| `preview_only` | No | `false` | If true, convert and validate but do not save the diagram |

### 2.2 Service Steps

```
1. Validate request (format value, content length)
2. Detect diagram type from content (§3)
3. Run format-specific converter (§4)
   → DiagramSpec candidate + warnings[]
4. If converter raises ParseError → return 400 IMPORT_PARSE_FAILED
5. Validate candidate spec (F2 validate_spec)
6. If spec has ERRORs → attempt repair pass (F2 repair_spec, max 1 pass)
7. Re-validate after repair
8. If ERRORs remain → return 422 SPEC_VALIDATION_FAILED with detail
9. Apply title (§5)
10. If preview_only=true → return preview response (no save)
11. Create diagrams row (persisted import)
12. Render ASCII and Mermaid synchronously
13. Emit DIAGRAM_IMPORTED audit event (F3 §3.2)
14. Return diagram with warnings
```

### 2.3 Response (Saved)

```json
{
  "data": {
    "diagram_id": "uuid",
    "title": "My Imported Flowchart",
    "diagram_type": "flowchart",
    "spec_json": { "...": "converted spec" },
    "ascii_cache": "...rendered ASCII...",
    "mermaid_cache": "...rendered Mermaid...",
    "import": {
      "input_format": "mermaid",
      "warnings": [
        { "code": "MERMAID_SUBGRAPH_IGNORED", "message": "Subgraph 'Auth' was ignored (not supported in V1)", "line": 5 }
      ],
      "repairs_applied": []
    }
  }
}
```

### 2.4 Response (Preview Only)

When `preview_only=true`, no diagram is saved. The response omits `diagram_id`:

```json
{
  "data": {
    "diagram_type": "flowchart",
    "spec_json": { "...": "converted spec" },
    "ascii_preview": "...rendered ASCII...",
    "import": {
      "input_format": "mermaid",
      "warnings": [...],
      "repairs_applied": []
    }
  }
}
```

---

## 3. Format Detection

The `input_format` field in the request is explicit — there is no auto-detection in V1. The format value is validated immediately; unknown values return `400 BAD_REQUEST`.

For `input_format=mermaid`, the converter detects the Mermaid diagram type from the first non-comment line:
- Starts with `flowchart` or `graph` → flowchart converter
- Starts with `erDiagram` → ER converter
- Anything else → `400 IMPORT_UNSUPPORTED_FORMAT` with message listing supported Mermaid types

---

## 4. Converters

### 4.1 Mermaid Flowchart Converter

Converts Mermaid `flowchart TD` / `flowchart LR` / `graph TD` / `graph LR` to a Diagram Alley `flowchart` spec.

**Supported Mermaid constructs → converted to:**

| Mermaid | Alley equivalent |
|---------|-----------------|
| `A[Label]` | step kind `process`, label "Label" |
| `A([Label])` | step kind `start` or `end` (first/last node heuristic) |
| `A{Label}` | step kind `decision`, label "Label" |
| `A>Label]` | step kind `io`, label "Label" |
| `A --> B` | transition, no label |
| `A -->|label| B` | transition with label |
| `A -.-> B` | transition (dashed style not preserved in flowchart; converted as solid, warning added) |

**Unsupported constructs → warning, element skipped:**
- `subgraph` blocks (DEC-006 — groups not supported in V1 Mermaid import)
- `click` events
- `style` / `classDef` / `class` directives
- `linkStyle` directives

Position assignment: the converter assigns grid positions based on the Mermaid direction (TD → `col=0` for all, incrementing `row`; LR → `row=0`, incrementing `col`). These are approximate — the user can rearrange after import.

**Node ID assignment:** Mermaid node IDs (e.g. `A`, `B`, `my_node`) are used directly as spec step IDs. If a Mermaid ID is not snake_case, the converter converts it (spaces→underscores, lowercased). If the result would be a reserved word or empty, a suffix is added.

### 4.2 Mermaid ER Converter

Converts `erDiagram` to a Diagram Alley `database` spec.

**Supported constructs:**
| Mermaid | Alley equivalent |
|---------|-----------------|
| `ENTITY { ... }` | `tables[]` entry; columns from fields |
| `ENTITY ||--o{ ENTITY2 : "label"` | `relationships[]` entry |
| `string`, `int`, `float`, `date`, `boolean` field types | Normalized to `TEXT`, `INTEGER`, `FLOAT`, `DATE`, `BOOLEAN` |
| `PK`, `FK` annotations | `primary_key`, `foreign_key` flags |

**Unsupported:**
- Inherited relationships (`<|--`)
- `UNIQUE` index annotations (produces warning; not mapped to column)
- Non-standard field types (mapped to `TEXT` with a warning)

**Cardinality mapping:**
| Mermaid notation | `type` |
|-----------------|--------|
| `||--||` | `one_to_one` |
| `||--o{` or `||--|{` | `one_to_many` |
| `}o--o{` | `many_to_many` |
| Others | `one_to_many` with a warning |

### 4.3 JSON Spec Importer

JSON spec import is strict — the content must be a valid Diagram Alley spec (F1 §3):
1. Parse as JSON; on failure → `400 IMPORT_PARSE_FAILED`
2. Check `spec_version` field; if absent or unrecognized → `422 SPEC_VALIDATION_FAILED`
3. Apply migration if `spec_version` is older than current (F1 §1)
4. Run full validation (F2 validate_spec)
5. If validation passes, proceed to save

JSON import does not accept `_export_meta` keys from D4 exports — they are silently stripped before validation.

### 4.4 YAML Spec Importer

YAML is re-serialized to JSON using Python's `PyYAML` (or `ruamel.yaml`). After that, processing follows the JSON importer path (§4.3).

**YAML parsing failure** (invalid YAML) → `400 IMPORT_PARSE_FAILED`.

---

## 5. Title Derivation

Title priority (first match wins):
1. `title` field in request body
2. `title` field in the imported spec (for JSON/YAML imports)
3. First `flowchart`/`erDiagram` comment line in Mermaid (`%% Title: <...>`)
4. `"Imported <type>"` (e.g. "Imported Flowchart")

Max length: 255 characters (truncated with `...` if over).

---

## 6. Corruption Prevention

The import service never overwrites an existing diagram's `spec_json` with import data. Import always creates a **new** diagram row. To prevent corrupting the user's library with a bad import:

1. If validation fails after repair, the spec is never saved. No partial rows are written.
2. All database writes are inside a transaction. On any error, the transaction rolls back.
3. The `preview_only` flag (§2.1) allows users to inspect the conversion result before committing.

---

## 7. Import UI

The import flow is accessible from:
- Sidebar or library: "Import" button
- Workspace: drag-and-drop a file onto the editor (if the file has a recognized extension)

### 7.1 Import Modal

```
+---------------------------------------------+
| Import Diagram                          [×]  |
+---------------------------------------------+
| Format: [Mermaid ▼]                          |
|                                              |
| [Upload file]  or paste below:               |
| +-------------------------------------------+|
| | flowchart TD                              ||
| |   A[Start] --> B[Process]                ||
| +-------------------------------------------+|
|                                              |
| Title: [________________________]            |
| Project: [None ▼]                            |
+---------------------------------------------+
| [Preview]                    [Import & Save] |
+---------------------------------------------+
```

**[Preview]:** Calls `preview_only=true`. Displays the ASCII render of the converted spec and any warnings. The user can then confirm [Import & Save] or cancel.

**Warnings display:** Each warning shows the line (for Mermaid) and a plain-language description. A warning count badge is shown in the modal header.

### 7.2 Drag-and-Drop

If a `.mmd` file is dropped on the workspace canvas:
- Format is set to `mermaid`
- Content is the file's text content
- Modal opens pre-filled; user confirms before import

If a `.json` or `.yaml` file is dropped:
- Format is set accordingly
- Same pre-fill flow

---

## 8. Warning Codes

Import-specific warning codes:

| Code | Meaning |
|------|---------|
| `MERMAID_SUBGRAPH_IGNORED` | A `subgraph` block was ignored |
| `MERMAID_STYLE_IGNORED` | A `style`/`classDef` directive was ignored |
| `MERMAID_UNSUPPORTED_EDGE_STYLE` | A dashed or dotted edge was converted to solid |
| `MERMAID_UNKNOWN_FIELD_TYPE` | A field type was mapped to TEXT |
| `MERMAID_UNKNOWN_CARDINALITY` | A relationship cardinality defaulted to one_to_many |
| `IMPORT_POSITION_APPROXIMATE` | Grid positions were auto-assigned; may need rearranging |

---

## Open Questions

None. All D5 decisions are locked.

---

## Acceptance Criteria

- `POST /api/v1/diagrams/import` with a valid Mermaid flowchart creates a `flowchart` diagram with correct `steps` and `transitions`.
- An `erDiagram` Mermaid import creates a `database` diagram with correct `tables` and `relationships`.
- A Mermaid import with `subgraph` constructs creates the diagram successfully with a `MERMAID_SUBGRAPH_IGNORED` warning.
- Importing an unsupported Mermaid type (e.g. `sequenceDiagram`) returns `400 IMPORT_UNSUPPORTED_FORMAT`.
- `preview_only=true` returns the converted spec and ASCII without creating a `diagrams` row.
- A JSON spec import with an older `spec_version` migrates the spec before validation.
- A JSON spec with validation errors after repair returns `422 SPEC_VALIDATION_FAILED` and no row is created.
- `_export_meta` keys from a D4 JSON export are silently stripped before import validation.
- `DIAGRAM_IMPORTED` audit event is written after every successful (non-preview) import.
- Importing a 60,000-character content string returns `400 BAD_REQUEST` (over 50,000 char limit).
