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

3. **Export endpoint** — Implement `POST /api/v1/diagrams/{id}/export`. Service steps per D4 §3.2 and §3.6: ownership check → **plan gate** (if `format` in `['json', 'yaml', 'ascii', 'markdown']`, call `require_pro(user, db)`; return `403 PLAN_REQUIRED` for free users) → validate `format` + diagram-type compatibility (return `400 EXPORT_FORMAT_UNSUPPORTED` for Mermaid on `file_structure`) → validate `spec_json` (return `400 SPEC_VALIDATION_FAILED` if errors) → select renderer → apply options (title prepend, metadata append, width constraint for ASCII) → emit `DIAGRAM_EXPORTED` audit event with `detail.format` → return inline content with `filename` and `content_type`. PNG content is base64-encoded in the `content` field. Export never mutates `spec_json` or any diagram row field. (→ D4 §3, DEC-038)

4. **Frontend export modal** — Implement `<ExportModal>` (replace stub from F4). Format dropdown, options checkboxes (include title, include metadata), width limit input, preview filename. Copy to clipboard uses the last export response content (or triggers the export call). Download button creates a browser `<a download>` from the response `content` and `filename`. PNG is download-only in V1 — Copy button is hidden for PNG. (→ D4 §6.1–6.2)

5. **Quick-export from preview panel** — Wire the Copy and Download buttons in the Preview panel per D4 §6.3: ASCII tab Copy reads `diagramStore.asciiCache` directly (no API call); Download triggers `POST .../export?format=ascii`; Mermaid tab similar; JSON/YAML tab Copy re-serializes `diagramStore.spec_json`; Visual tab Export SVG and Export PNG call the export endpoint with the respective format. All quick-export paths use default options (`include_title=true`, `include_metadata=false`). (→ D4 §6.3)

6. **Documentation Bundle endpoint** — Implement `POST /api/v1/diagrams/{id}/docs-bundle`. Check `plan='pro'` (return `403 PLAN_REQUIRED` for free users). Validate `spec_json`. Generate all applicable formats using the same renderers as the single-format endpoint (skip Mermaid for `file_structure`). Bundle into a zip stream using Python `zipfile`. Respond with `Content-Type: application/zip` and `Content-Disposition: attachment; filename="<slug>-docs-bundle-<YYYY-MM-DD>.zip"`. Emit `DIAGRAM_EXPORTED` with `detail.format='docs-bundle'`. (→ D4 §7, DEC-043)

7. **Documentation Bundle button in modal** — Add "[⬇ Documentation Bundle (.zip)]" button to `<ExportModal>` below the standard Download button. Visible for all users; disabled with tooltip "Available on Pro — generate every format your docs need from one source" for free users. On click, calls the docs-bundle endpoint and triggers a browser download. (→ D4 §7.3)

---

## Done Criteria

- [ ] `POST /api/v1/diagrams/{id}/export` with `format=mermaid` by a free user returns Mermaid content (free format allowed). (→ DEC-038)
- [ ] `POST /api/v1/diagrams/{id}/export` with `format=json` by a free user returns `403 PLAN_REQUIRED`. (→ DEC-038)
- [ ] `POST /api/v1/diagrams/{id}/export` with `format=ascii` by a free user returns `403 PLAN_REQUIRED`. (→ DEC-038)
- [ ] `POST /api/v1/diagrams/{id}/export` with `format=ascii` by a Pro user returns content identical to `ascii_cache` when cache is fresh. (→ D4 Acceptance Criteria)
- [ ] `format=mermaid` on a `file_structure` diagram returns `400 EXPORT_FORMAT_UNSUPPORTED`. (→ D4 Acceptance Criteria, DEC-035)
- [ ] Export on a diagram with spec validation errors returns `400 SPEC_VALIDATION_FAILED`. (→ D4 Acceptance Criteria)
- [ ] Export with `include_metadata=true` appends a metadata block in the correct format for ASCII, JSON, Mermaid, and SVG. (→ D4 Acceptance Criteria)
- [ ] Filename for `"My DB Schema (v2)!"` + json is `my-db-schema-v2.json`. (→ D4 Acceptance Criteria)
- [ ] `DIAGRAM_EXPORTED` audit event is written with `detail.format` after every successful export. (→ D4 Acceptance Criteria)
- [ ] SVG export returns a complete, valid SVG string that renders the diagram. (→ D4 Acceptance Criteria)
- [ ] PNG export returns base64-encoded PNG bytes generated from the SVG path (not a screenshot). (→ D4 Acceptance Criteria, DEC-019)
- [ ] JSON export with `include_metadata=true` adds a `_export_meta` key alongside the spec fields. (→ D4 Acceptance Criteria)
- [ ] Copy button in the ASCII tab reads `ascii_cache` directly without an API call. (→ D4 Acceptance Criteria)
- [ ] Export never modifies `diagrams.spec_json` or any other diagram field. (→ D4 §3, GP-02)
- [ ] `POST /api/v1/diagrams/{id}/docs-bundle` by a free user returns `403 PLAN_REQUIRED`. (→ D4 §7, DEC-043)
- [ ] `POST /api/v1/diagrams/{id}/docs-bundle` by a Pro user returns a valid zip archive containing `.txt`, `.svg`, `.png`, `.diagram.json`, `.diagram.yaml`. (→ D4 §7)
- [ ] Documentation Bundle for an architecture diagram includes `.mmd`; for a `file_structure` diagram does not. (→ D4 §7.2)
