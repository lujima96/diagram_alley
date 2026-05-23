---
phase: 11
slice: D4
spec: specs/D4-export.md
status: planned
---

# Slice D4 — Export

## Pre-conditions

- [ ] Slice F1 complete — `diagrams` table exists.
- [ ] Slice F2 complete — all renderers callable: `render_text`, `render_mermaid`, `render_svg`, `render_png`, `validate_spec`.
- [ ] Slice F5 complete — response envelope, error codes, endpoint stub for `POST /api/v1/diagrams/{id}/export` registered.
- [ ] Slice D2 complete — diagrams exist and are saveable; preview panel Copy buttons need export integration.

---

## Build Sequence

1. **Filename derivation** — Implement `export_service.py::derive_filename(title, format)` per D4 §5: lowercase, replace special chars with hyphens, collapse consecutive hyphens, strip leading/trailing hyphens, truncate at 64 chars at a word boundary, append extension. Unit-test the edge cases from D4 §5 examples. (→ D4 §5)

2. **Metadata block builder** — Implement `build_metadata_block(title, diagram_type, exported_at, spec_version, format)` for each format variant: text comment block for ASCII/Markdown, `_export_meta` key for JSON/YAML, `%%` comment block for Mermaid, `<desc>` element for SVG, PNG text chunk for PNG (skip silently with a non-blocking warning if unsupported by `cairosvg`). (→ D4 §4)

3. **Export endpoint** — Implement `POST /api/v1/diagrams/{id}/export`. Service steps per D4 §3.2: ownership check → validate `format` + diagram-type compatibility (return `400 EXPORT_FORMAT_UNSUPPORTED` for Mermaid on wireframe/file_structure) → validate `spec_json` (return `400 SPEC_VALIDATION_FAILED` if errors) → select renderer → apply options (title prepend, metadata append, width constraint for ASCII) → emit `DIAGRAM_EXPORTED` audit event with `detail.format` → return inline content with `filename` and `content_type`. PNG content is base64-encoded in the `content` field. Export never mutates `spec_json` or any diagram row field. (→ D4 §3)

4. **Frontend export modal** — Implement `<ExportModal>` (replace stub from F4). Format dropdown, options checkboxes (include title, include metadata), width limit input, preview filename. Copy to clipboard uses the last export response content (or triggers the export call). Download button creates a browser `<a download>` from the response `content` and `filename`. PNG is download-only in V1 — Copy button is hidden for PNG. (→ D4 §6.1–6.2)

5. **Quick-export from preview panel** — Wire the Copy and Download buttons in the Preview panel per D4 §6.3: ASCII tab Copy reads `diagramStore.asciiCache` directly (no API call); Download triggers `POST .../export?format=ascii`; Mermaid tab similar; JSON/YAML tab Copy re-serializes `diagramStore.spec_json`; Visual tab Export SVG and Export PNG call the export endpoint with the respective format. All quick-export paths use default options (`include_title=true`, `include_metadata=false`). (→ D4 §6.3)

---

## Done Criteria

- [ ] `POST /api/v1/diagrams/{id}/export` with `format=ascii` returns content identical to `ascii_cache` when cache is fresh. (→ D4 Acceptance Criteria)
- [ ] `format=mermaid` on a `ui_wireframe` diagram returns `400 EXPORT_FORMAT_UNSUPPORTED`. (→ D4 Acceptance Criteria)
- [ ] Export on a diagram with spec validation errors returns `400 SPEC_VALIDATION_FAILED`. (→ D4 Acceptance Criteria)
- [ ] Export with `include_metadata=true` appends a metadata block in the correct format for ASCII, JSON, Mermaid, and SVG. (→ D4 Acceptance Criteria)
- [ ] Filename for `"My DB Schema (v2)!"` + json is `my-db-schema-v2.json`. (→ D4 Acceptance Criteria)
- [ ] `DIAGRAM_EXPORTED` audit event is written with `detail.format` after every successful export. (→ D4 Acceptance Criteria)
- [ ] SVG export returns a complete, valid SVG string that renders the diagram. (→ D4 Acceptance Criteria)
- [ ] PNG export returns base64-encoded PNG bytes generated from the SVG path (not a screenshot). (→ D4 Acceptance Criteria, DEC-019)
- [ ] JSON export with `include_metadata=true` adds a `_export_meta` key alongside the spec fields. (→ D4 Acceptance Criteria)
- [ ] Copy button in the ASCII tab reads `ascii_cache` directly without an API call. (→ D4 Acceptance Criteria)
- [ ] Export never modifies `diagrams.spec_json` or any other diagram field. (→ D4 §3, GP-02)
