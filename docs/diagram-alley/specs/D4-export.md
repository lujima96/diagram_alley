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

Export format list; export endpoint contract; export state machine; metadata embedding; file naming convention; export modal UI contract; `DIAGRAM_EXPORTED` trigger point (constant owned by F3).

---

## 1. Export Formats

V1 supports the following export formats. Format availability by diagram type is noted.

| Format | `format` value | Extension | Availability |
|--------|---------------|-----------|--------------|
| ASCII text | `ascii` | `.txt` | All types |
| Markdown (text output in fenced block) | `markdown` | `.md` | All types |
| Mermaid | `mermaid` | `.mmd` | architecture, database, flowchart, network only |
| JSON spec | `json` | `.json` | All types |
| YAML spec | `yaml` | `.yaml` | All types |
| SVG | `svg` | `.svg` | All types (visual render) |
| PNG | `png` | `.png` | All types (generated from SVG export path) |
| PDF | `pdf` | `.pdf` | Deferred to V2 |

**Mermaid availability:** `file_structure` does not have a meaningful Mermaid equivalent. Requesting `mermaid` for this type returns `400 EXPORT_FORMAT_UNSUPPORTED`. `network` renders as `graph TD` (best effort). (`ui_wireframe` is deferred to V2 per DEC-035.)

**PNG support:** PNG is V1 (DEC-019). It is generated from the deterministic SVG export path, not from a queued screenshot-only browser workflow. PDF remains deferred to V2.

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

For synchronous exports (ASCII, Markdown, Mermaid, JSON, YAML, SVG, PNG), the state transitions from `idle → exporting → ready` in a single API call (<1s typical for text/SVG; PNG may take longer but remains synchronous in V1). The frontend shows a brief spinner on the export button.

PDF (V2) will use the background task pattern (F5 §11) if render takes seconds.

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
    "ascii_width": null,
    "text_mode": "ascii"
  }
}
```

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| `format` | Yes | — | One of the format values in §1 |
| `options.include_title` | No | `true` | Prepend diagram title to output |
| `options.include_metadata` | No | `false` | Append metadata block (see §4) |
| `options.ascii_width` | No | `null` | Max width for ASCII output in characters; null = no limit |
| `options.text_mode` | No | `ascii` | `ascii \| utf8`; strict ASCII output stays printable ASCII only (DEC-023) |

### 3.2 Service Steps

```
1. Load diagram (ownership check)
2. Validate format value; check diagram_type compatibility
3. Validate current spec_json (F2 validate_spec); fail if errors present
4. Select renderer:
   ascii     → F2 render_text(mode=options.text_mode)
   markdown  → F2 render_text(mode=options.text_mode); wrap in markdown fenced block
   mermaid   → F2 render_mermaid
   json      → serialize spec_json with 2-space indent
   yaml      → re-serialize spec_json as YAML
   svg       → F2 visual renderer → SVG string
   png       → F2 render_svg → SVG-to-PNG conversion → base64 PNG
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
    "content_type": "text/plain",
    "content_encoding": "utf-8"
  }
}
```

SVG content is a full SVG string with `content_type = "image/svg+xml"` and `content_encoding = "utf-8"`. PNG content is base64-encoded PNG bytes with `content_type = "image/png"` and `content_encoding = "base64"`.

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
    "message": "Mermaid export is not supported for file_structure diagrams.",
    "detail": null
  }
}
```

Error code `EXPORT_FORMAT_UNSUPPORTED` is defined in F5 §9.1.

### 3.6 Plan Gate (DEC-038, DEC-049)

Export formats are tiered by plan:

| Format | Free | Pro |
|--------|------|-----|
| Mermaid (`.mmd`) — download | Yes | Yes |
| PNG — download | Yes | Yes |
| SVG — download | Yes | Yes |
| ASCII (`.txt`) — **copy only**, includes attribution comment | Yes | Yes (clean, no comment) |
| Markdown (`.md`) — **copy only**, includes attribution comment | Yes | Yes (clean, no comment) |
| ASCII (`.txt`) — **file download** | No | Yes |
| Markdown (`.md`) — **file download** | No | Yes |
| JSON (`.diagram.json`) | No | Yes |
| YAML (`.diagram.yaml`) | No | Yes |

**Attribution comment** appended to free-tier ASCII/Markdown copy (not present for Pro):
```
# Generated by Diagram Alley — diagram-alley.com
```

**API behavior for ASCII/Markdown:** The export endpoint always generates the content. For free users requesting `format=ascii` or `format=markdown`, the response `content` field includes the attribution comment appended. The frontend's "Copy" action works for free users; the "Download" button for these formats is shown but disabled with a tooltip "File download requires Pro."

For JSON/YAML, all access (copy and download) requires Pro:
```json
{
  "error": {
    "code": "PLAN_REQUIRED",
    "message": "Blueprint export (JSON/YAML) requires a Pro plan.",
    "detail": null
  }
}
```

Service step 1.5 (after ownership check, before format validation):
```
1.5 If format in ['json', 'yaml']: require_pro(current_user, db)
    If format in ['ascii', 'markdown'] AND request.download_mode == True: require_pro(current_user, db)
```

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

**PNG:** Metadata is written to PNG text chunks when the conversion library supports it. If unsupported, export still succeeds and returns a non-blocking warning.

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

PNG is download-only in V1; copy-to-clipboard is not supported for PNG.

### 6.3 Quick Export from Preview Panel

The Preview panel (F4 §5.4) has per-tab copy/download buttons:
- **ASCII tab:** [Copy] copies `ascii_cache`; [Download] triggers export with `format=ascii`
- **Mermaid tab:** [Copy] copies `mermaid_cache`; [Download] triggers export with `format=mermaid`
- **JSON/YAML tab:** [Copy] copies spec JSON or YAML re-serialization; [Download] triggers export with `format=json` or `format=yaml`
- **Visual tab:** [Export SVG] triggers export with `format=svg`; [Export PNG] triggers export with `format=png`

These quick-export paths bypass the modal and use the default options (`include_title=true`, `include_metadata=false`).

---

## 7. Documentation Bundle (DEC-043)

The Documentation Bundle generates a zip archive containing all applicable export formats for the current diagram in a single download. This is a Pro plan feature (free users can use individual exports for Mermaid, PNG, and SVG).

One click generates every format your docs need from one source.

### 7.1 Endpoint

**`POST /api/v1/diagrams/{id}/docs-bundle`** (requires auth, Pro plan)

No request body. All formats are included by default using default options (`include_title=true`, `include_metadata=false`).

Service steps:
```
1. Load diagram (ownership check)
2. Check plan='pro' (gate: return 403 PLAN_REQUIRED if free user)
3. Validate spec_json (F2 validate_spec); fail if errors present
4. Generate all applicable formats:
   - ascii   → <slug>.txt
   - markdown → <slug>.md
   - mermaid  → <slug>.mmd  (only if diagram_type supports it)
   - svg      → <slug>.svg
   - png      → <slug>.png
   - json     → <slug>.diagram.json
   - yaml     → <slug>.diagram.yaml
5. Bundle into zip stream
6. Emit DIAGRAM_EXPORTED audit event with detail.format='docs-bundle'
7. Return zip stream
```

### 7.2 Response

```
Content-Type: application/zip
Content-Disposition: attachment; filename="<slug>-docs-bundle-<YYYY-MM-DD>.zip"
```

File naming inside the zip uses the same slug logic as §5. Example for "Payment Flow" diagram bundled 2026-05-23:

```
payment-flow-docs-bundle-2026-05-23.zip
  ├── payment-flow.txt
  ├── payment-flow.md
  ├── payment-flow.mmd         (architecture/database/flowchart/network only)
  ├── payment-flow.svg
  ├── payment-flow.png
  ├── payment-flow.diagram.json
  └── payment-flow.diagram.yaml
```

### 7.3 Documentation Bundle in the UI

The Export Modal (§6) includes a "Documentation Bundle" button for Pro users:
```
+---------------------------------------------+
| ...format options...                         |
+---------------------------------------------+
| [Copy to clipboard]  [Download]              |
+---------------------------------------------+
| ─────────────────────────────               |
| [⬇ Documentation Bundle (.zip)]  (Pro only) |
+---------------------------------------------+
```

For free users the Documentation Bundle button is shown but disabled with a tooltip: "Available on Pro. Generate every format your docs need from one source."

---

## Open Questions

None. All D4 decisions are locked.

---

## Acceptance Criteria

- `POST /api/v1/diagrams/{id}/export` with `format=ascii` returns content identical to `ascii_cache` when the cache is fresh.
- `POST /api/v1/diagrams/{id}/export` with `format=mermaid` on a `file_structure` diagram returns `400 EXPORT_FORMAT_UNSUPPORTED`.
- `POST /api/v1/diagrams/{id}/export` on a diagram with spec validation errors returns `400 SPEC_VALIDATION_FAILED`.
- Export with `include_metadata=true` appends a metadata block in the correct format for each output type.
- Filename for "My DB Schema (v2)!" is `my-db-schema-v2.json` (special characters stripped).
- `DIAGRAM_EXPORTED` audit event is written with `detail.format` after every successful export.
- SVG export returns a complete, valid SVG string that renders the diagram visually.
- PNG export returns base64-encoded PNG bytes generated from the SVG export path.
- JSON export with `include_metadata=true` includes a `_export_meta` key alongside the spec fields.
- Copy button in the Preview panel copies ASCII cache without an API call.
- `POST /api/v1/diagrams/{id}/docs-bundle` by a free user returns `403 PLAN_REQUIRED`.
- `POST /api/v1/diagrams/{id}/docs-bundle` by a Pro user returns a valid zip archive containing at minimum `.txt`, `.svg`, `.png`, `.diagram.json`, `.diagram.yaml`.
- The zip archive for an architecture diagram includes a `.mmd` file; a file_structure diagram does not.
- `DIAGRAM_EXPORTED` audit event is written with `detail.format='docs-bundle'` after Documentation Bundle download.
