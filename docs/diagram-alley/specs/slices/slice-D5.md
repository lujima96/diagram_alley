---
phase: 11
slice: D5
spec: specs/D5-import.md
status: planned
---

# Slice D5 — Import and Conversion

## Pre-conditions

- [ ] Slice F1 complete — `diagrams` table and spec schemas exist.
- [ ] Slice F2 complete — `validate_spec` and `repair_spec` callable; F1 spec migration available.
- [ ] Slice F5 complete — response envelope, error codes, endpoint stub for `POST /api/v1/diagrams/import` registered.

---

## Build Sequence

1. **Mermaid subtype detector** — In `app/services/import_service.py`, implement `detect_mermaid_subtype(content: str) -> str`. Strips comments, reads the first non-empty line. Returns `flowchart` for lines starting with `flowchart` or `graph`; `er` for `erDiagram`; raises `ImportUnsupportedFormatError` for anything else. This runs immediately after `input_format=mermaid` is confirmed. (→ D5 §3)

2. **Mermaid flowchart converter** — Implement `app/services/converters/mermaid_flowchart.py`. Parses `flowchart TD/LR/graph TD/LR` using a line-by-line approach (no full parser required). Converts supported constructs to the Diagram Alley `flowchart` spec per D5 §4.1. Unsupported constructs (subgraph, click, style/classDef, linkStyle) are skipped with a warning entry. Node IDs are coerced to snake_case. Grid positions assigned based on TD/LR direction. (→ D5 §4.1, DEC-006)

3. **Mermaid ER converter** — Implement `app/services/converters/mermaid_er.py`. Parses `erDiagram` blocks to produce a `database` spec per D5 §4.2. Maps Mermaid field types to Alley column types; maps cardinality notation to `type` enum values. Unsupported constructs (inherited relationships, UNIQUE index) produce warnings. (→ D5 §4.2, DEC-006)

4. **JSON spec importer** — Implement `app/services/converters/json_spec.py`. Parse JSON; strip `_export_meta` keys silently (D4 JSON exports); check `spec_version`; apply F1 §1 migration if needed; run F2 `validate_spec`. Fail with `400 IMPORT_PARSE_FAILED` on invalid JSON, `422 SPEC_VALIDATION_FAILED` on post-migration validation failure. (→ D5 §4.3)

5. **YAML spec importer** — Implement `app/services/converters/yaml_spec.py`. Parse YAML with `PyYAML`; on parse failure return `400 IMPORT_PARSE_FAILED`. Serialize to JSON dict, then follow the JSON importer path. (→ D5 §4.4)

6. **Import endpoint** — Implement `POST /api/v1/diagrams/import` per D5 §2.2: validate request → detect subtype → run converter → validate spec (F2) → repair if errors (max 1 pass) → re-validate → if `preview_only=true` return preview without saving → otherwise create `diagrams` row + render caches in a single transaction → emit `DIAGRAM_IMPORTED` → return. All DB writes are transactional — partial rows are never written on failure. (→ D5 §2, §6, FM-03)

7. **Title derivation** — Implement `derive_import_title(request_title, spec, content, input_format)` per D5 §5 priority order: request body `title` → spec `title` field → Mermaid comment heuristic (`%% Title: ...`) → `"Imported <type>"`. Truncate to 255 chars. (→ D5 §5)

8. **Import UI** — Implement `<ImportModal>` accessible from the sidebar Import button and the library. Tabs: Paste (textarea for content), Upload (file input for `.mmd`, `.json`, `.yaml` files). Format selector (`mermaid | json | yaml`). Preview button calls `POST .../import?preview_only=true` and shows the ASCII preview and warning list. Warning list shows code and message per warning. Import & Save button calls the same endpoint with `preview_only=false`. Drag-and-drop on the workspace editor: detects file extension (`.mmd` → mermaid, `.json` → json, `.yaml` → yaml) and opens the modal pre-populated. (→ D5 §7, GP-09)

---

## Done Criteria

- [ ] `POST /api/v1/diagrams/import` with `preview_only=true` returns a converted spec and ASCII preview without creating a `diagrams` row. (→ GP-09)
- [ ] Import always creates a new diagram and never overwrites an existing diagram's `spec_json`. (→ D5 §6, GP-09)
- [ ] Unsupported Mermaid type (e.g. sequence) returns `400 IMPORT_UNSUPPORTED_FORMAT`. (→ D5 §3, FM-03)
- [ ] Malformed JSON content returns `400 IMPORT_PARSE_FAILED`. (→ D5 §4.3, FM-03)
- [ ] A converted spec with validation errors that survive the repair pass returns `422 SPEC_VALIDATION_FAILED` and writes no `diagrams` row. (→ D5 §2.2, FM-03)
- [ ] `_export_meta` keys from a D4 JSON export are stripped before validation without causing a parse failure. (→ D5 §4.3, GP-09)
- [ ] Import warnings are returned in the response but are not stored on the `diagrams` row. (→ DEC-031, GP-09)
- [ ] `DIAGRAM_IMPORTED` audit event is emitted on successful save. (→ D5 §2.2, F3 §3.2)
- [ ] A valid Mermaid flowchart with subgraph constructs imports successfully with a warning; the subgraph is skipped. (→ D5 §4.1, DEC-006)
