---
id: D4
tier: domain
status: draft
version: "0.1"
depends-on: [F1, F2, F5]
tags: [export, ascii, mermaid, svg, png, json, yaml, markdown]
---

# D4 — Export

**Version:** 0.1
**Status:** Draft
**Depends on:** F1, F2, F5

---

## Purpose

Define all export formats, the export endpoint contract, the export state machine, metadata included in exports, and the file naming and download behavior. Export is non-destructive and idempotent — it never modifies the diagram spec.

## Non-Goals

- Rendering rules (→ F2)
- Import and conversion (→ D5)
- GitHub commit workflow (→ D8)
- Version selection for export (users export the current live spec; D3 handles restoring a version before export)

---

## Owned Concepts

Export format list; export endpoint contract; export state machine; metadata embedding; file naming convention; export modal UI contract; DIAGRAM_EXPORTED audit event trigger.

---

## 1. Export Formats

V1 supports the following export formats. Format availability by diagram type is noted.

| Format | `format` value | Extension | Availability |
|--------|---------------|-----------|--------------|
| ASCII text | `ascii` | `.txt` | All types |
| Markdown (ASCII in fenced block) | `markdown` | `.md` | All types |
| Mermaid | `mermaid` | `.mmd` | architecture, database, flowchart, network only |
| JSON spec | `json` | `.json` | All types |
| YAML spec | `yaml` | `.yaml` | All types |
| SVG | `svg` | `.svg` | All types (visual render) |
| PNG | `png` | `.png` | Deferred to V2 (headless browser render not in V1) |
| PDF | `pdf` | `.pdf` | Deferred to V2 |

**Mermaid availability:** `ui_wireframe` and `file_structure` do not have a meaningful Mermaid equivalent. Requesting `mermaid` for these types returns `400 EXPORT_FORMAT_UNSUPPORTED`.

**PNG/PDF deferral:** These require a headless browser (Puppeteer/Playwright) which adds significant infrastructure. SVG is the V1 visual export. PNG/PDF are listed as planned V2 exports.

---

## 2. Export State Machine

```
idle
  → exporting       (user triggers export)
  → ready           (content ready; download offered)
  → error           (generation failed)

ready → idle        (user downloads or dismisses)
error → idle        (user dismisses or retries)
```

For synchronous exports (ASCII, Markdown, Mermaid, JSON, YAML, SVG), the state transitions from `idle → exporting → ready` in a single API call (<1s typical). The frontend shows a brief spinner on the export button.

PNG and PDF (V2) will use the background task pattern (F5 §11) since render takes seconds.

---

## 3. Export Endpoint

**`POST /api/v1/diagrams/{id}/export`**

### 3.1 Request Body

```json
{
  "format": "ascii",
  "options": {
    "include_title": true,
    "include_metadata": false,
    "ascii_width": null
  }
}
```

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| `format` | Yes | — | One of the format values in §1 |
| `options.include_title` | No | `true` | Prepend diagram title to output |
| `options.include_metadata` | No | `false` | Append metadata block (see §4) |
| `options.ascii_width` | No | `null` | Max width for ASCII output in characters; null = no limit |

### 3.2 Service Steps

```
1. Load diagram (ownership check)
2. Validate format value; check diagram_type compatibility
3. Validate current spec_json (F2 validate_spec); fail if errors present
4. Select renderer:
   ascii     → F2 render_ascii
   markdown  → F2 render_ascii; wrap in markdown fenced block
   mermaid   → F2 render_mermaid
   json      → serialize spec_json with 2-space indent
   yaml      → re-serialize spec_json as YAML
   svg       → F2 visual renderer → SVG string
5. Apply options (title prepend, metadata append, width constraint)
6. Emit DIAGRAM_EXPORTED audit event (F3 §3.2) with detail.format
7. Return export content
```

### 3.3 Response

Export content is returned inline (not as a file download — the client handles the download):

```json
{
  "data": {
    "format": "ascii",
    "content": "...the export content as a string...",
    "filename": "three-tier-web-app.txt",
    "content_type": "text/plain"
  }
}
```

SVG content is a full SVG string. The `content_type` is `image/svg+xml`.

The frontend triggers a browser download using the `filename` from the response.

### 3.4 Spec Validation Gate

If the current `spec_json` has validation errors, the export is blocked and returns:
```json
{
  "error": {
    "code": "SPEC_VALIDATION_FAILED",
    "message": "Cannot export a diagram with validation errors. Fix the errors first.",
    "detail": { "errors": [...] }
  }
}
```

### 3.5 Format Not Supported

```json
{
  "error": {
    "code": "EXPORT_FORMAT_UNSUPPORTED",
    "message": "Mermaid export is not supported for ui_wireframe diagrams.",
    "detail": null
  }
}
```

New error code `EXPORT_FORMAT_UNSUPPORTED` must be added to F5 §9.1 error code table.

---

## 4. Metadata Block

When `options.include_metadata = true`, a metadata block is appended to the export. Format varies by export type:

**ASCII / Markdown:**
```
---
Diagram: Three-Tier Web App
Type: architecture
Exported: 2026-05-20T14:32:00Z
Spec version: 1.0
Source: diagram-alley
---
```

**JSON / YAML:** Metadata is added as a top-level `_export_meta` key in the JSON/YAML output (not part of the spec schema — prefixed with `_` to distinguish):
```json
{
  "_export_meta": {
    "title": "Three-Tier Web App",
    "diagram_type": "architecture",
    "exported_at": "2026-05-20T14:32:00Z",
    "spec_version": "1.0"
  },
  "spec_version": "1.0",
  "diagram_type": "architecture",
  ...
}
```

**Mermaid:** Comment block at the top:
```
%% Diagram: Three-Tier Web App
%% Type: architecture
%% Exported: 2026-05-20T14:32:00Z
```

**SVG:** Appended as an SVG `<desc>` element.

---

## 5. File Naming

The filename is derived from the diagram title:
- Lowercased
- Spaces and special characters replaced with hyphens
- Consecutive hyphens collapsed to one
- Trailing/leading hyphens stripped
- Max 64 characters (truncated at word boundary)
- Extension appended per format

Examples:
- "Three-Tier Web App" + ascii → `three-tier-web-app.txt`
- "My DB Schema (v2)!" + json → `my-db-schema-v2.json`
- "API" + mermaid → `api.mmd`

---

## 6. Export Modal (UI)

The Export Modal is opened from the workspace header `[Export]` button or from the Preview panel Download button.

### 6.1 Modal Layout

```
+---------------------------------------------+
| Export Diagram                          [×]  |
+---------------------------------------------+
| Format:  [ASCII ▼]                           |
|                                              |
| Options:                                     |
| ☑ Include title                              |
| ☐ Include metadata                           |
| Width limit: [_______] chars (blank = none)  |
|                                              |
| Preview filename: three-tier-web-app.txt     |
+---------------------------------------------+
| [Copy to clipboard]     [Download]           |
+---------------------------------------------+
```

### 6.2 Copy to Clipboard

For text formats (ASCII, Markdown, Mermaid, JSON, YAML): copies the export content string to the clipboard. No API call — uses the cached content from the last export call (or triggers the export call if not yet done).

For SVG: copies the SVG string.

### 6.3 Quick Export from Preview Panel

The Preview panel (F4 §5.4) has per-tab copy/download buttons:
- **ASCII tab:** [Copy] copies `ascii_cache`; [Download] triggers export with `format=ascii`
- **Mermaid tab:** [Copy] copies `mermaid_cache`; [Download] triggers export with `format=mermaid`
- **JSON/YAML tab:** [Copy] copies spec JSON or YAML re-serialization; [Download] triggers export with `format=json` or `format=yaml`
- **Visual tab:** [Export SVG] triggers export with `format=svg`

These quick-export paths bypass the modal and use the default options (`include_title=true`, `include_metadata=false`).

---

## 7. Reconciliation Notes (F5 Drift)

### 7.1 New Error Code

`EXPORT_FORMAT_UNSUPPORTED` (HTTP 400) must be added to F5 §9.1. Added to F5 here as a tracked reconciliation item.

**Action required:** Add to F5 §9.1:
```
| 400 | `EXPORT_FORMAT_UNSUPPORTED` | Requested format is not supported for this diagram type |
```

---

## Open Questions

None. All D4 decisions are locked.

---

## Acceptance Criteria

- `POST /api/v1/diagrams/{id}/export` with `format=ascii` returns content identical to `ascii_cache` when the cache is fresh.
- `POST /api/v1/diagrams/{id}/export` with `format=mermaid` on a `ui_wireframe` diagram returns `400 EXPORT_FORMAT_UNSUPPORTED`.
- `POST /api/v1/diagrams/{id}/export` on a diagram with spec validation errors returns `400 SPEC_VALIDATION_FAILED`.
- Export with `include_metadata=true` appends a metadata block in the correct format for each output type.
- Filename for "My DB Schema (v2)!" is `my-db-schema-v2.json` (special characters stripped).
- `DIAGRAM_EXPORTED` audit event is written with `detail.format` after every successful export.
- SVG export returns a complete, valid SVG string that renders the diagram visually.
- JSON export with `include_metadata=true` includes a `_export_meta` key alongside the spec fields.
- Copy button in the Preview panel copies ASCII cache without an API call.
