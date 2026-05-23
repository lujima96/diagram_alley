---
phase: 11
slice: D3
spec: specs/D3-version-history.md
status: planned
---

# Slice D3 — Version History

## Pre-conditions

- [ ] Slice F1 complete — `diagram_versions` table exists; `prune_if_over_limit` implemented.
- [ ] Slice F3 complete — subscription gate available; audit event helper available.
- [ ] Slice F5 complete — pagination helper and endpoint stubs registered.
- [ ] Slice D2 complete — `PATCH /api/v1/diagrams/{id}` with `create_version_snapshot=true` is the primary snapshot trigger; the D3 restore flow creates two snapshots that D2's auto-save must not interfere with.

---

## Build Sequence

1. **Version list endpoint** — Implement `GET /api/v1/diagrams/{id}/versions`. Ownership check on diagram. Return list of version summary objects (no `spec_json` in list — id, diagram_id, change_summary, created_by, created_at, spec_version). Default `sort_dir=desc`, `page_size=50`. (→ D3 §2)

2. **Version get endpoint** — Implement `GET /api/v1/diagrams/{id}/versions/{vid}`. Returns the full version including `spec_json`. If `spec_version` of the stored snapshot is older than current, apply F1 §1 migration before returning. (→ D3 §3)

3. **Version snapshot write path** — Implement `version_service.create_snapshot(diagram_id, spec_json, change_summary, created_by, db)`. Calls `prune_if_over_limit` first (from slice-F1), then inserts the `diagram_versions` row. Emits `VERSION_CREATED` audit event. This is called by: D2 PATCH with `create_version_snapshot=true`, D1 generate endpoint (initial snapshot), D1 improve accept, and the restore flow below. (→ D3 §1, F1 §5.3)

4. **Minimum interval guard** — In `create_snapshot`, before inserting, check whether the previous snapshot for this diagram was created within the last 5 seconds. If so, skip the insert (no-op). Apply only to explicit-Save triggers; AI modification and restore always create a snapshot regardless. (→ D3 §1.1)

5. **Restore endpoint** — Implement `POST /api/v1/diagrams/{id}/versions/{vid}/restore` per the 12-step flow in D3 §4.1 exactly: load version → migrate spec to current version → validate (fail with `422 SPEC_VALIDATION_FAILED` if invalid) → create pre-restore snapshot → update `diagrams.spec_json` → clear caches (`ascii_cache=NULL`, `mermaid_cache=NULL`, `ascii_is_detached=false`) → re-render synchronously → create post-restore snapshot → emit `VERSION_RESTORED` → return both version IDs (DEC-032). No mutation occurs if validation fails at step 3 (pre-mutation guard). (→ D3 §4, DEC-032, FM-05)

6. **Named version rename endpoint** — Implement `PATCH /api/v1/diagrams/{id}/versions/{vid}` accepting only `{ "change_summary": "string (max 255)" }`. Returns the updated version object (no `spec_json`). (→ D3 §5)

7. **Version drawer UI** — Implement `<VersionDrawer>` (referenced in the workspace but stubbed in F4). The drawer opens from the workspace header or Version History button. Shows the version list (paginated, newest first) from `GET .../versions`. Each row: timestamp, change_summary (editable inline), created_by display name. Clicking a row opens a preview panel showing the version's `spec_json` ASCII render. Restore button shows a confirmation modal; on confirm calls the restore endpoint. After restore, close drawer and reload the workspace from the restored diagram. (→ D3, GP-05)

---

## Done Criteria

- [ ] `GET /api/v1/diagrams/{id}/versions` returns version list without `spec_json`. (→ D3 §2)
- [ ] `GET /api/v1/diagrams/{id}/versions/{vid}` returns the full version including a migrated `spec_json` if the version's `spec_version` is older than current. (→ D3 §3)
- [ ] Explicit Save creates a version snapshot; auto-save does not. (→ DEC-021, D3 §1)
- [ ] Two rapid Save clicks within 5 seconds create only one version snapshot. (→ D3 §1.1)
- [ ] `POST .../versions/{vid}/restore` returns both `pre_restore_version_id` and `restored_version_id` in the response. (→ DEC-032)
- [ ] Restore never deletes any existing version rows — history is append-only. (→ D3 §4.3, GP-05)
- [ ] Restore of a version whose migrated spec fails validation returns `422 SPEC_VALIDATION_FAILED` and does not touch `diagrams.spec_json`. (→ FM-05)
- [ ] A trial-expired user attempting restore receives `403 SUBSCRIPTION_REQUIRED`. (→ GP-05, DEC-022)
- [ ] Version count reaches 100 → next snapshot prunes the oldest row before inserting. (→ F1 §4.4, D3 §6)
- [ ] `PATCH .../versions/{vid}` with a new `change_summary` updates only that field; `spec_json` is not returned. (→ D3 §5)
