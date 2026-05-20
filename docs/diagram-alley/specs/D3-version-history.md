---
id: D3
tier: domain
status: draft
version: "0.1"
depends-on: [F1, F3, F5]
tags: [versions, history, save, restore, snapshots, diff]
---

# D3 — Version History

**Version:** 0.1
**Status:** Draft
**Depends on:** F1, F3, F5

---

## Purpose

Define the save, diff, restore, named version, and version snapshot trigger behavior for diagram version history. Domain spec D3 governs all versioning workflows, the version endpoint contracts, and the rules for creating and pruning version rows.

## Non-Goals

- AI generation (which triggers a snapshot — D1)
- Editor interactions (D2)
- Export of versions (D4)
- The `diagram_versions` table structure (owned by F1 §4.4)

---

## Owned Concepts

Version snapshot trigger rules; version list and restore endpoints (full contract); named versions; version pruning policy enforcement; version diff computation; the `change_summary` field population.

---

## 1. Version Snapshot Triggers

A new `diagram_versions` row is created when (F1 §5.3):

| Trigger | Who Creates | `change_summary` |
|---------|-------------|-----------------|
| User clicks Save (`Ctrl+S` or Save button) | User action (D2 §5.2) | `"Manual save"` |
| Successful new AI generation | D1 §3.3 generation flow | `"Initial AI generation"` |
| User accepts an AI improve proposal | D1 §4.2 accept flow | `"AI modification: <first 80 chars of change_request>"` |
| User restores a previous version | This spec §4 | `"Before restore to <timestamp>"` and `"Restored from version <short_id>"` |

**No snapshot on:**
- Live auto-save (D2 §5.3)
- ASCII label sync (D2 §7.2)
- Title-only PATCH
- Render calls

### 1.1 Minimum Interval Guard

To prevent accidental duplicate snapshots (double-click Save), a new snapshot is not created if the previous snapshot for this diagram was created less than 5 seconds ago. The PATCH still persists through live auto-save behavior, but no version row is written. The 5-second guard applies only to explicit-Save triggers; AI modification and restore always create a snapshot regardless of timing.

---

## 2. Version List Endpoint

**`GET /api/v1/diagrams/{id}/versions`**

**Query parameters:**
- `page`, `page_size` (F5 §7; default page_size 50)
- `sort_dir`: `asc | desc` (default `desc` — newest first)

**Response:**

```json
{
  "data": [
    {
      "id": "uuid",
      "diagram_id": "uuid",
      "change_summary": "Manual save",
      "created_by": "uuid",
      "created_at": "2026-05-20T14:32:00Z",
      "spec_version": "1.0"
    }
  ],
  "meta": { "total": 12, "page": 1, "page_size": 50, "has_next": false }
}
```

`spec_json` is **not** included in list responses (only in the single-version GET) to keep the list lightweight.

---

## 3. Get Version Endpoint

**`GET /api/v1/diagrams/{id}/versions/{vid}`**

Returns the full version snapshot including `spec_json`.

```json
{
  "data": {
    "id": "uuid",
    "diagram_id": "uuid",
    "spec_json": { "...": "full spec at this version" },
    "spec_version": "1.0",
    "change_summary": "Manual save",
    "created_by": "uuid",
    "created_at": "2026-05-20T14:32:00Z"
  }
}
```

If `spec_version` of the stored snapshot is older than the current spec version, the backend migrates it before returning (same F1 §1 migration path as live diagram specs).

---

## 4. Restore Version Endpoint

**`POST /api/v1/diagrams/{id}/versions/{vid}/restore`**

Restores the diagram to the state captured in version `{vid}`.

### 4.1 Restore Steps

```
1. Load version {vid} (ownership check: diagram belongs to user)
2. Migrate version's spec_json to current spec_version if needed (F1 §1)
3. Validate migrated spec (F2 validate_spec)
4. If validation fails → return 422 SPEC_VALIDATION_FAILED (with detail)
   (Spec may have become invalid if spec schema changed since the snapshot — rare)
5. Create a pre-restore snapshot of the current diagrams.spec_json with
   change_summary = "Before restore to <version_created_at timestamp>"
   (preserves in-progress work; always created even if diagram has no unsaved changes)
6. Update diagrams.spec_json to the restored spec
7. Clear diagrams.ascii_cache and mermaid_cache (NULL)
8. Clear diagrams.ascii_is_detached (set to false)
9. Re-render ASCII and Mermaid synchronously
10. Create a post-restore snapshot with change_summary = "Restored from version {short_vid}"
11. Emit VERSION_RESTORED audit event (F3 §3.2)
12. Return the updated diagram
```

### 4.2 Response

```json
{
  "data": {
    "diagram_id": "uuid",
    "spec_json": { "...": "restored spec" },
    "ascii_cache": "...rendered ASCII...",
    "mermaid_cache": "...rendered Mermaid...",
    "restored_from_version_id": "uuid",
    "new_version_id": "uuid"
  }
}
```

### 4.3 Restore Does Not Delete History

Restoring a version does not remove any existing version rows. The history is always append-only. After restore, the post-restore snapshot (created in step 10) becomes the latest version.

---

## 5. Named Versions

Users can give a version a human-readable `change_summary` by editing it directly. This is a convenience feature — the `change_summary` field is the only user-editable field on a version row.

**Endpoint:** `PATCH /api/v1/diagrams/{id}/versions/{vid}`

**Request:**
```json
{ "change_summary": "Before Redis migration" }
```

**Response:** The updated version object (no `spec_json` — not re-returned).

Max length: 255 characters.

---

## 6. Version Pruning

The version pruning policy is defined in F1 §4.4: soft limit of 100 versions per diagram; oldest version is pruned when the limit is exceeded.

Pruning is enforced in the service layer at write time (before inserting a new version row):

```python
def prune_if_over_limit(diagram_id: UUID, db: Session, limit: int = 100) -> None:
    count = db.query(DiagramVersion).filter_by(diagram_id=diagram_id).count()
    if count >= limit:
        oldest = (
            db.query(DiagramVersion)
            .filter_by(diagram_id=diagram_id)
            .order_by(DiagramVersion.created_at.asc())
            .first()
        )
        db.delete(oldest)
```

Pruning runs inside the same transaction as the new version insert — atomic.

---

## 7. Version Diff Display (Frontend)

The version history drawer (UI) shows a version list with diff indicators. The diff is computed client-side from consecutive versions' `spec_json` when a user expands a version row.

**Diff shown as:**
- Count of nodes/entities added, removed, changed
- Count of edges/relationships added, removed, changed
- `change_summary` text (editable in-place, §5)

A full side-by-side spec diff (JSON) is available behind a "Show diff" toggle using the same diff logic as D1 §4.3.

**Preview on hover:** Hovering a version row in the drawer shows the ASCII render of that version's spec (rendered client-side by calling `GET /api/v1/diagrams/{id}/versions/{vid}` lazily when expanded).

---

## 8. Version History UI (Drawer)

The version history is surfaced as a slide-in drawer (right side), opened from the workspace header (a clock or history icon). The drawer overlays the right panel but does not replace the workspace.

Contents:
- Version list (paginated, 20 per page in the drawer)
- Each row: timestamp, `change_summary`, [Restore] button
- Inline `change_summary` edit (pencil icon → text input → save)
- "Restore" confirms with a dialog: "Restore to this version? The current diagram will be saved as a new version first."

The "saved as a new version first" behavior: before restore, the service creates a snapshot of the *current* spec with summary `"Before restore to <timestamp>"`, then performs the restore. This ensures no work is ever lost.

---

## Open Questions

None. All D3 decisions are locked.

---

## Acceptance Criteria

- `GET /api/v1/diagrams/{id}/versions` returns versions in descending order by `created_at`.
- Clicking Save creates a new version row; a second click within 5 seconds does not create a duplicate.
- `POST /api/v1/diagrams/{id}/versions/{vid}/restore` updates `diagrams.spec_json` and creates pre-restore and restored version rows.
- After restore, `ascii_is_detached` is reset to `false`.
- A version snapshot at spec_version `"1.0"` is migrated before restore if the current spec version has advanced.
- Pruning removes the oldest version when the 101st version is created, leaving exactly 100.
- `PATCH /api/v1/diagrams/{id}/versions/{vid}` updates `change_summary` without modifying `spec_json`.
- `VERSION_CREATED` audit event is written for every new snapshot; `VERSION_RESTORED` is written on every restore.
