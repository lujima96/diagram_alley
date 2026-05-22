---
phase: 7
artifact: schema-roundup
status: complete
depends-on: [F1, F2, F3, F5, D1, D2, D3, D4, D5, D6, D7, A1, audit-2026-05-22]
---

# Phase 7 — Schema Proposals and Consolidation

This file consolidates schema, data-model, and API-shape implications discovered during Phase 4 flows, Phase 6 mockups, and the reconciled Phase 10 audit.

Specs remain the source of truth. This roundup does not redefine owner specs; it records whether a flow implies a schema change, confirms the owner, and identifies follow-up work before implementation handoff.

---

## 1. Outcome Summary

No new database tables are proposed from Phase 4-6 reconciliation.

Most flow-derived needs are already covered by F1/F5 and the domain specs:

- Core tables exist: `users`, `projects`, `diagrams`, `diagram_versions`, `templates`, `model_providers`, `user_settings`, `diagram_shares`, `subscriptions`, `audit_log`.
- Diagram specs for all six V1 diagram types are owned by F1.
- Live save, version snapshots, import, export, sharing, billing, and admin workflows have endpoint owners.
- Phase 10 decisions DEC-026 through DEC-030 were recorded and propagated to owner specs.

The Phase 7 follow-ups are resolved:

1. Warnings/repairs remain transient response data plus optional audit detail (DEC-031).
2. Restore responses return both created version IDs (DEC-032).

---

## 2. Consolidated Schema Decisions

| Topic | Flow/Audit Source | Owner | Disposition |
|-------|-------------------|-------|-------------|
| User-created templates | Phase 10 audit | F1, F5, D2 | Closed by DEC-026. Existing `templates` table supports system and private user templates. No schema change. |
| Project archive and restore | Phase 10 audit | F1, F4, F5 | Closed by DEC-027. Existing `projects.is_archived` supports archive/restore. No schema change. |
| Diagram move between projects | Phase 10 audit | D2, F4 | Closed by DEC-028. Existing `diagrams.project_id` and `PATCH /diagrams/{id}` support moves. No schema change. |
| Display-name fallback | Phase 10 audit | F1, D6 | Closed by DEC-029. Fallback is derived from email prefix. No schema change. |
| File structure and UI wireframe visual rendering | Phase 10 audit | F2, F4 | Closed by DEC-030. Existing F1 schemas support tree-row and nested-box rendering. No schema change. |
| Detached ASCII state | ASCII reverse-sync flow | F1, F2, D2 | Covered by `diagrams.ascii_is_detached`. No schema change. |
| Cancel-at-period-end access | Billing flow | F1, D7, A1 | Covered by `subscriptions.cancel_at_period_end` and `access_ends_at`. No schema change. |
| Share links show live diagram | Sharing flow | F1, D6 | Covered by `diagram_shares` pointing to `diagrams`. No version snapshot link needed in V1. |
| Import warnings | D5 import flow | D5 | Response-level warning objects are defined. Persistence decision remains explicit follow-up (§4.1). |
| AI repair warnings | AI generation flow | D1, F2 | Response-level warning objects are defined. Persistence decision remains explicit follow-up (§4.1). |

---

## 3. Flow-by-Flow Schema Review

### 3.1 AI Generation

**Flow need:** Persist a new generated diagram, return generated spec, render caches, provider metadata, repair warnings, and initial version snapshot.

**Covered by current specs:**

- `diagrams` row stores `spec_json`, `spec_version`, `ascii_cache`, `mermaid_cache`.
- `diagram_versions` stores the initial snapshot (DEC-020).
- D1 response includes transient `generation.repairs_applied` and `generation.warnings`.
- F3 owns audit event constants; D1 identifies generation audit trigger points.

**Resolution:** Closed by DEC-031. Repair warnings are response/audit data only and are not stored on the diagram row.

### 3.2 Manual Edit

**Flow need:** Add/edit database tables and columns, drag nodes, update layout, create version snapshots, export current spec.

**Covered by current specs:**

- F1 database schema spec includes `tables[]`, `columns[]`, and `relationships[]`.
- F1 architecture and other graph-like specs include position metadata where needed.
- D2 owns `PATCH /diagrams/{id}` for `spec_json`, title, and `project_id`.
- D3 owns version snapshot creation.

**No schema change proposed.**

### 3.3 ASCII Export and Reverse Sync

**Flow need:** Label-only reverse sync and detached export state for structural text edits.

**Covered by current specs:**

- DEC-005 locks label-only reverse sync.
- F1 includes `diagrams.ascii_is_detached`.
- F2 owns detached export state semantics.
- D2 owns reset behavior and ASCII sync endpoint behavior.

**No schema change proposed.**

### 3.4 Version History

**Flow need:** Append-only snapshots, restore, pre-restore snapshot, post-restore snapshot, named summaries.

**Covered by current specs:**

- F1 `diagram_versions` stores immutable `spec_json` snapshots and `change_summary`.
- D3 restore creates pre-restore and post-restore snapshots.
- D3 allows `PATCH /versions/{vid}` to update only `change_summary`.

**Follow-up:**

- Restore response may need both pre-restore and post-restore version IDs (§4.2).

### 3.5 Sharing and Publishing

**Flow need:** Create/revoke public share links and show current live diagram state.

**Covered by current specs:**

- F1 `diagram_shares` stores token, access level, revoked timestamp, and access counters.
- D6 defines live current-state sharing, not point-in-time version sharing.
- DEC-022 keeps share management available after trial expiry.

**No schema change proposed.**

### 3.6 Billing and Upgrade

**Flow need:** Trial lifecycle, expired access, Pro checkout, cancel-at-period-end, resubscribe/resume renewal.

**Covered by current specs:**

- F1 `subscriptions` includes `plan`, `status`, Stripe IDs, trial/current period timestamps, `cancel_at_period_end`, `access_ends_at`, and `canceled_at`.
- DEC-025 locks cancel-at-period-end semantics.
- D7 distinguishes expired trial from canceled Pro.

**No schema change proposed.**

---

## 4. Follow-Up Proposals

### 4.1 Warning and Repair Persistence

**Source:** `ai-generation.md`, D1, D5.

**Issue:** D1 and D5 define `warnings` and `repairs_applied` in API responses. Earlier flow wording implied persisted repair-warning metadata, but F1 has no `diagram_metadata`, `warnings`, or `repairs_applied` column.

**Resolution:** Closed by DEC-031. Warnings/repairs are transient response data plus optional audit details. No database schema change.

**Owner updates:** `ai-generation.md` now removes the stale persistence wording. F1 intentionally has no diagnostics column.

---

### 4.2 Restore Response Shape

**Source:** D3 restore contract.

**Issue:** D3 restore steps create two version rows during restore: a pre-restore snapshot and a post-restore snapshot. The earlier response shape only included one new version identifier, which was ambiguous.

**Resolution:** Closed by DEC-032. D3 now returns both `pre_restore_version_id` and `restored_version_id`.

**Owner updates:** D3 restore response contract updated. No database schema change.

---

## 5. Reconciled During Phase 7

### 5.1 Version Pruning Policy Drift

**Source:** `version-history.md`, F1 §4.4, D3 acceptance criteria.

**Issue:** The version-history flow says version history is unlimited in V1 and pruning is future scope. F1 and D3 define a soft limit of 100 versions per diagram with pruning on the 101st snapshot.

**Resolution:** Closed. `version-history.md` now matches F1/D3: V1 keeps up to 100 versions per diagram and prunes the oldest when the 101st snapshot is created.

---

## 6. Phase 7 Closure

Phase 7 is complete:

- No new database tables or columns are proposed.
- No Alembic/database migration proposal is required beyond the F1 Draft 0.1 schema already defined.
- All schema-roundup follow-ups are resolved by DEC-031 and DEC-032.

Next planning phase: Phase 8 Golden Paths.
