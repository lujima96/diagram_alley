---
phase: 4
artifact: flow — version history
status: complete
depends-on: [D3, F1, F3, D2, personas]
decisions: [DEC-021, DEC-020]
---

# Flow: Version History

Narrative covering version snapshot creation, browsing version history, previewing a previous version, and restoring. Cast: **Marcus** (P2, Technical Writer) who made an AI-assisted update to his architecture diagram that changed more than he intended, and wants to restore the version from before the AI change.

---

## Cast and Context

| | |
|-|-|
| **User** | Marcus, P2 — Technical Writer |
| **Session** | Active Pro subscriber; working on "Platform Architecture v2" in his "Core Docs" project |
| **Goal** | Roll back the diagram to the version that existed before yesterday's AI update, which added several nodes he didn't want |
| **Version history state** | 4 versions exist: initial generation, two manual saves, and a recent AI modification |

---

## Pre-condition

Marcus opens `/workspace/da_3c8b70f` (the "Platform Architecture v2" diagram). The visual grid editor shows the current state — more nodes than he wanted. He can see from the canvas that yesterday's AI "improve" run added three extra services.

---

## Step-by-Step Flow

### 1. Open Version History

Marcus clicks the **History** icon in the header bar (or the keyboard shortcut). A **version history drawer** slides in from the right side of the workspace, partially overlapping the preview panel.

The drawer shows a reverse-chronological list of snapshots:

| # | Label | Date | Created by |
|---|-------|------|------------|
| 4 | "AI: Added monitoring services" | Yesterday 14:32 | AI modification |
| 3 | "Updated node labels for sprint 4" | 3 days ago | Manual save |
| 2 | "Added Redis and CDN nodes" | 5 days ago | Manual save |
| 1 | "Initial generation" | 7 days ago | AI generation |

Each row shows a small thumbnail preview (ASCII snippet or node count).

### 2. Preview a Previous Version

Marcus clicks on version 3 ("Updated node labels for sprint 4"). The drawer expands to show a **side-by-side comparison**:

- Left: the current (version 4) diagram
- Right: version 3 preview (read-only ASCII rendering)

The comparison highlights differences. Nodes that exist in v4 but not v3 are shown in a different shade. Nodes that exist in v3 but not v4 are also highlighted.

Marcus can see that version 4 added "Prometheus," "Alertmanager," and "Grafana" nodes — those are not part of the diagram he approved. Version 3 is exactly what he wants.

### 3. Restore the Version

Marcus clicks **Restore this version** on the version 3 row. A confirmation dialog appears:

> "Restore 'Updated node labels for sprint 4' (version 3)?
> The current diagram will be saved as a new version before restoring.
> This cannot be undone from within version history."

He clicks **Confirm Restore**.

### 4. Restore Executes

The backend:

1. Creates a new version snapshot of the current state (version 4 is preserved — it is never deleted by a restore). The auto-generated label is "Before restore to v3."
2. Copies the `spec_json` from version 3's snapshot into `diagrams.spec_json` (the live diagram row).
3. Re-renders the diagram from the restored spec.

The workspace updates:
- The visual grid editor shows the version 3 canvas (three fewer nodes).
- The preview panel re-renders the ASCII output from the restored spec.
- A green toast appears: "Diagram restored to 'Updated node labels for sprint 4'."
- The version history drawer now shows 5 entries (the new "Before restore to v3" snapshot at the top).

### 5. Save After Restore

The restored state is now the live `spec_json` (auto-saved). Marcus clicks **Save** to create a new named version snapshot:
- He types the summary: "Restored — removed monitoring services added by AI."
- A new version 6 snapshot is created.

Version history now has 6 entries: the original 4, the auto-generated "Before restore to v3," and his new labeled snapshot.

---

## Sub-flow: Named Version on Manual Save

Any time Marcus clicks **Save**, the app:

1. Flushes any pending auto-save queue (DEC-021).
2. Prompts for an optional change summary (default: blank, stored as null).
3. Creates a `diagram_versions` row with `spec_json` snapshot, `change_summary`, and `created_at`.
4. The Save button returns to inactive.

Auto-save (continuous background persistence to `diagrams.spec_json`) does **not** create version snapshots — only the explicit Save action does. This keeps version history meaningful (DEC-021).

---

## Sub-flow: Initial AI Generation Creates Version 1

When Priya (P1) generates a new diagram for the first time (from the AI generation flow), the backend:

1. Creates the `diagrams` row (persisted for the first time).
2. Immediately creates a `diagram_versions` snapshot labeled "Initial generation" (DEC-020).

This ensures the pre-edit state is always recoverable, even if the user immediately starts making changes without an explicit Save.

---

## Version Snapshot Triggers (Summary)

| Trigger | Snapshot created? | Label |
|---------|------------------|-------|
| First successful AI generation (new diagram) | Yes | "Initial generation" |
| Manual Save button | Yes | User-supplied or blank |
| Restore action (auto-pre-restore snapshot) | Yes | "Before restore to v[N]" |
| Auto-save (background flush) | No | — |
| Import (new diagram from import) | Yes | "Imported from [format]" |

---

## Version Pruning

Version history keeps up to 100 snapshots per diagram in V1 (F1 §4.4, D3 acceptance criteria). When the 101st snapshot is created, the oldest snapshot is pruned so exactly 100 remain.

---

## Permission and Access Rules

| Action | Required permission | Notes |
|--------|---------------------|-------|
| View version history | `diagram.version.read` | Any user who can read the diagram |
| Restore a version | `diagram.version.restore` | Requires write access to the diagram; trial-expired users cannot restore (DEC-022 — edit is blocked) |
| Delete a version | Not supported in V1 | Deferred to V2 |

---

## Spec References

| Behavior | Owner |
|----------|-------|
| Version snapshot data model (`diagram_versions` table) | F1 |
| Snapshot triggers and rules | D3, DEC-021, DEC-020 |
| Restore operation (pre-restore snapshot + spec copy) | D3 |
| Version history drawer UX | D3 (behavior), F4 (panel placement) |
| Auto-save (live spec_json, no snapshot) | D2, DEC-021 |
| Trial-expired users blocked from restore | F3, DEC-022 |
