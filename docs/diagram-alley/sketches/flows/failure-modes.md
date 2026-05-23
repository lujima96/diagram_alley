---
phase: 9
artifact: failure-modes
status: complete
depends-on: [F1, F2, F3, F5, D1, D3, D5, D7]
---

# Phase 9 — Failure Modes

Catalog of the failure scenarios that must be handled correctly before Diagram Alley can be considered commercially ready. Each entry describes what goes wrong, how the system detects it, what the user experiences, what invariants must be preserved, and which specs own the handling.

These are not new product decisions. They document already-specified behavior from the foundation and domain specs. Contradictions or gaps found here should be flagged as drift, not resolved silently.

---

## Failure Mode Index

| ID | Scenario | Trigger | Severity |
|----|----------|---------|----------|
| FM-01 | AI provider outage | Provider returns 5xx, times out, or is unreachable | High — blocks core feature |
| FM-02 | Spec validation failure | AI returns unparseable or invalid JSON | High — blocks persistence |
| FM-03 | Import corruption | Import input is malformed, unsupported, or produces an invalid spec | Medium — blocks import only |
| FM-04 | Payment failure | Stripe renewal fails or webhook is missing/delayed | High — blocks continued access |
| FM-05 | Version conflict on restore | A version snapshot's spec no longer validates under the current schema | Medium — rare; blocks restore |

---

## FM-01 — AI Provider Outage

### What Happens

The user submits a generation or improve prompt. The provider adapter fires the request and receives one of:
- A network-level error (connection refused, DNS failure)
- HTTP 5xx from the provider
- No response within the 30-second per-request timeout (D1 §1.2)

### Detection

The provider adapter in `app/providers/<type>.py` catches `httpx.RequestError` and HTTP 5xx responses. The service layer receives a `ProviderError`. One retry is attempted. If the retry also fails, the service returns without creating or mutating any diagram rows.

### API Response

| Condition | Status | Error Code |
|-----------|--------|-----------|
| Connection/network failure after retry | 502 | `PROVIDER_UNAVAILABLE` |
| Provider response timeout (>30s) | 503 | `PROVIDER_TIMEOUT` |

(F5 §9.1)

### User Experience

1. The loading indicator in the prompt panel stops.
2. A toast error appears: "Generation failed: [provider error message]. Your prompt has been kept."
3. The prompt textarea retains the user's input.
4. The canvas/editor remains in its previous state — empty for a new diagram, unchanged for an improve attempt.
5. A **Retry** button appears below the error toast.
6. If retry also fails, additional options are offered: "Try a different provider" (links to `/settings/providers`) and "Continue manually" (focuses the blank visual grid editor).

For an **improve** request, the existing diagram's `spec_json` is never touched — the proposal is never returned.

### Invariants

- No `diagrams` row is created or mutated on a provider failure.
- No `diagram_versions` snapshot is created.
- The user's prompt is preserved in the UI for retry.
- The error does not crash the workspace; the editor remains functional.

### Owners

D1 §7 (AI failure handling), F5 §9.1 (error codes), F3 §3.2 (audit: `DIAGRAM_AI_GENERATION_FAILED` emitted if the diagram row was created before the failure — which it is not in V1's flow; emission is pre-persistence, so no orphaned rows).

---

## FM-02 — Spec Validation Failure

### What Happens

The AI returns a response that is either:
1. **Not parseable as JSON** — the model emitted prose, markdown fences, or a partial JSON fragment.
2. **Parseable JSON but fails validation** — a valid JSON object that violates F2 validation rules (e.g., duplicate node IDs, an edge that references a non-existent node, a missing required field).

### Detection

**Parse failure:** The service's JSON extraction step fails to parse the raw LLM response as JSON. A regex fallback attempts to extract a `{...}` block; if that also fails, `GENERATION_PARSE_FAILED` is returned immediately with no repair attempt.

**Validation failure:** After parsing, `F2.validate_spec()` runs. If validation errors exist, one repair pass is attempted (`F2.repair_spec()`). After repair, validation runs again. If errors remain, the generation is rejected.

### API Response

| Condition | Status | Error Code |
|-----------|--------|-----------|
| LLM output not parseable as JSON | 422 | `GENERATION_PARSE_FAILED` |
| Spec fails validation after repair | 422 | `GENERATION_INVALID_SPEC` |
| Client submits invalid spec directly | 400 | `SPEC_VALIDATION_FAILED` (with `detail.errors[]`) |

(F5 §9.1)

### What the Repair Pass Fixes

From F2 §3, the repair pass handles a defined set of common AI-output errors without user involvement:
- Removes edges that reference non-existent node IDs.
- Fills missing required `id` fields using a slugified label.
- Strips unknown `node.kind` values back to the default kind (`service`).
- Fills a missing `spec_version` field with the current version.

Repairs are listed in `generation.repairs_applied[]` in the response. Non-blocking warnings are listed in `generation.warnings[]`.

### User Experience

**Successful repair (non-blocking warnings):**
1. The workspace renders the diagram normally.
2. A yellow repair banner appears at the top of the preview panel: "1 repair was made: [repair description]. Check your diagram."
3. The user can accept the diagram, edit it manually, or re-generate.
4. The banner is dismissable. Warnings are not stored on the diagram row (DEC-031).

**Irrecoverable failure:**
1. No diagram is created or persisted.
2. An error toast: "We couldn't generate a valid diagram from that prompt. Try rephrasing or choosing a different diagram type."
3. A **Retry** and a **Try a different prompt** option are offered.
4. The prompt is preserved for editing.

**Client-submitted invalid spec (manual PATCH):**
1. The API returns `400 SPEC_VALIDATION_FAILED` with a structured `detail.errors[]` list.
2. The frontend displays inline validation errors in the spec/JSON editor (if applicable) or an error toast.

### Invariants

- A spec with unresolved validation errors is never rendered. The renderer is never called with an invalid spec (F2 §1).
- No `diagrams` row is created if the final spec fails validation.
- No version snapshot is created on failure.
- The repair pass runs at most once per generation request.

### Owners

F2 §2 (validation rules), F2 §3 (repair pass), D1 §3.2 (service steps), D1 §7 (failure handling), F5 §9.1 (error codes), DEC-031 (warnings transient-only).

---

## FM-03 — Import Corruption

### What Happens

A user imports a Mermaid, JSON, or YAML diagram that is malformed, unsupported, or converts to a spec that fails validation after repair.

Sub-cases:
1. **Unsupported format** — Mermaid sequence, state, PlantUML, Lucidchart, draw.io.
2. **Parse failure** — The Mermaid parser encounters syntax it cannot interpret, or the JSON/YAML is structurally invalid.
3. **Conversion warning** — The Mermaid content is parseable but contains constructs the converter cannot map (e.g., a Mermaid styling directive). These produce warnings, not failures.
4. **Post-conversion validation failure** — The converted spec has ERRORs that survive the repair pass.

### Detection

The import service in D5 §2.2 runs format detection, then the format-specific converter, then F2 validation. Each stage can terminate the flow early with a distinct error code.

### API Response

| Condition | Status | Error Code |
|-----------|--------|-----------|
| Format not supported in V1 | 400 | `IMPORT_UNSUPPORTED_FORMAT` |
| Content cannot be parsed | 400 | `IMPORT_PARSE_FAILED` |
| Converted spec fails validation after repair | 422 | `SPEC_VALIDATION_FAILED` (with `detail.errors[]`) |

(F5 §9.1, D5 §2.2)

### Conversion Warnings vs. Errors

| Category | What It Means | Outcome |
|----------|--------------|---------|
| WARNING | Unsupported Mermaid construct skipped (e.g., click event, style directive) | Import proceeds; warning shown |
| ERROR (repaired) | Validation error fixed by repair pass | Import proceeds; repair listed |
| ERROR (unrepaired) | Validation error survives repair | Import blocked; no `diagrams` row |

### User Experience

**Preview-only mode (recommended flow):**
1. User submits import with `preview_only=true`.
2. Response includes the converted diagram, a rendered ASCII preview, and any warnings.
3. Warning list is shown in the import UI: "3 constructs were skipped during import. [Show details]"
4. If validation fails entirely, the preview shows an error state: "This diagram could not be imported. [Show errors]"
5. User reviews and clicks **Import & Save** (or cancels).

**Fatal import error (unsupported / parse failure):**
1. A clear inline error appears: "This format is not supported in V1" or "The file could not be parsed."
2. No `diagrams` row is written.
3. Existing diagrams are not touched — imports always create new rows (D5 §2.2).

**Post-import (success with warnings):**
1. User lands in `/workspace/:diagramId`.
2. A yellow warning indicator is visible in the toolbar.
3. Warnings are transient (response data only) — they are not stored on the diagram row (DEC-031).

### Invariants

- `preview_only=true` never creates a `diagrams` row (D5 §2.2 step 10).
- A failed import never overwrites or mutates an existing diagram.
- Partial state is never written: if validation fails after parsing, the transaction rolls back (D5 §2.2 step 8).
- `_export_meta` fields from D4 JSON exports are stripped before validation to prevent false validation errors (D5 §4.3, GP-09).

### Owners

D5 (full import spec), F2 §2–3 (validation and repair), F5 §9.1 (error codes), DEC-006 (Mermaid import scope), DEC-031 (warnings transient-only).

---

## FM-04 — Payment Failure

### What Happens

A Pro subscriber's renewal payment fails. Stripe attempts to charge the card on file and is declined. This triggers Stripe's retry schedule and a series of webhook events.

Sub-cases:
1. **Initial payment failure** — First attempt fails; retries are pending.
2. **All retries exhausted** — Stripe fires `customer.subscription.deleted` after the retry window ends.
3. **Webhook delivery failure** — Stripe fires the event but the backend webhook handler returns a non-2xx response, causing Stripe to retry webhook delivery.
4. **Checkout abandoned** — User initiates upgrade but abandons at Stripe Checkout; no subscription is created.

### Detection

The webhook handler at `POST /webhooks/stripe` (D7 §3.3) receives Stripe events. The handler verifies the webhook signature (`STRIPE_WEBHOOK_SECRET`) before processing. Unsigned or tampered events are rejected with 400.

### Subscription State Progression

| Event | Database Update | Access Effect |
|-------|----------------|--------------|
| `invoice.payment_failed` | `status = 'past_due'` | Access continues; banner shown |
| `customer.subscription.deleted` (retries exhausted) | `status = 'canceled'` | Access restricted per DEC-022 |
| `customer.subscription.updated` (card updated, paid) | `status = 'active'`; `past_due` cleared | Access fully restored |

(D7 §7, DEC-025)

### User Experience

**`past_due` state:**
1. A persistent "Your payment failed" banner appears on `/settings/billing`.
2. The banner includes a [Manage Billing] button that opens the Stripe Billing Portal.
3. From the portal, the user can update their payment method.
4. All features remain accessible during `past_due` — access is not interrupted until retries are exhausted.

**Access ends (`status = 'canceled'`):**
1. On next app load or route navigation, the subscription gate (F3 §5.5) detects canceled status.
2. The behavior is identical to trial expiry (DEC-022): view and export existing diagrams; block AI generation, spec edits, and new diagram creation.
3. The billing page shows the expired/canceled state (D7 §9.3) with a resubscribe option.
4. The user can resubscribe via `POST /api/v1/billing/checkout-session`, which is available for `status = 'canceled'` users.

**Webhook delivery failure / delayed delivery:**
1. Stripe retries webhook delivery automatically with exponential backoff.
2. The backend webhook handler is idempotent: processing the same event twice produces the same database state.
3. During the gap between a payment event and its webhook delivery, the user's in-database status may lag behind Stripe's truth. This is acceptable for V1 — the lag is typically seconds to minutes, not hours.
4. Webhook signature verification prevents replay attacks from invalid sources.

**Checkout abandoned:**
1. Stripe does not create a subscription; no webhook fires.
2. The user is redirected to `/settings/billing?checkout=canceled`.
3. A dismissable "Checkout was canceled" banner is shown.
4. The subscription row is unchanged.

### Invariants

- Diagram data is never deleted on payment failure (D7 §6).
- Subscription state is driven by Stripe webhooks, not by polling.
- The webhook handler must be idempotent.
- Feature gating is enforced by API, not only UI overlay (D7 §2.3, GP-04).
- A `past_due` subscription does not block access until retries are exhausted.

### Owners

D7 §7 (payment failure), D7 §3.3–3.4 (webhook handling), D7 §8 (resubscription), F3 §5.5 (feature gate), F3 §3.2 (`SUBSCRIPTION_CANCELED` audit event), DEC-022 (expired access scope), DEC-025 (cancel-at-period-end semantics).

---

## FM-05 — Version Conflict on Restore

### What Happens

A user attempts to restore a historical version snapshot whose `spec_json` no longer passes validation under the current spec schema. This can occur if:
- The diagram spec schema (F1 §3) evolved and the migration function for that `spec_version` is missing or incomplete.
- A repair pass was not applied when the snapshot was created, leaving a dormant error.
- Manual edits to the raw spec (via JSON editor) created an invalid snapshot at save time (if the gate was not enforced correctly).

This scenario is rare in V1 since `spec_version` is `"1.0"` throughout the launch period. It becomes more likely in V2+ when `spec_version` advances.

### Detection

The restore service in D3 §4.1 runs the spec migration (F1 §1) and then `F2.validate_spec()` on the migrated snapshot. If validation errors exist after migration, the restore is blocked.

### API Response

| Condition | Status | Error Code |
|-----------|--------|-----------|
| Migrated snapshot fails validation | 422 | `SPEC_VALIDATION_FAILED` (with `detail.errors[]`) |
| Version not found or wrong owner | 404 | `VERSION_NOT_FOUND` |

### User Experience

1. The user clicks **Restore** on a version in the history drawer and confirms.
2. The backend attempts migration and validation.
3. If validation fails, a modal or toast error is shown: "This version could not be restored because it contains errors that are no longer valid. [Show errors]"
4. The current diagram's `spec_json` is unchanged — the pre-restore snapshot (D3 §4.1 step 5) is **not** created when restore is blocked before that step. This means the error occurs before any mutation.
5. The user can try a different version or continue editing the current diagram.

### Invariants

- The current `diagrams.spec_json` is never mutated if restore fails validation (D3 §4.1 — step 4 halts before step 5).
- No pre-restore or post-restore version snapshots are created if the restore is blocked.
- The version history is append-only; blocking a restore does not delete any version rows.
- The backend never renders from an invalid spec.

### Spec Note

The restore step ordering in D3 §4.1 is critical to this invariant:
- Step 3 validates the migrated spec.
- Step 4 checks for failure and returns early **before** step 5 (pre-restore snapshot creation).
- No mutation occurs before the validation gate.

### Owners

D3 §4 (restore endpoint), F1 §1 (spec migration), F2 §2 (validation), F5 §9.1 (error codes), DEC-032 (restore response contract), GP-05 (golden path: version restore).

---

## Cross-Cutting Failure Rules

These rules apply across all failure modes:

1. **No partial writes.** If a service step fails after beginning a database transaction, the transaction rolls back. No orphaned rows are written.
2. **API gates are authoritative.** UI state (disabled buttons, overlays) is a convenience. The API always enforces subscription, ownership, and validation checks independently.
3. **Error responses never expose stack traces in production.** F5 §9.1 defines `INTERNAL_ERROR` for unhandled exceptions; the `detail` field is null.
4. **User prompts are preserved on failure.** For all AI generation failures, the prompt textarea retains its content so the user can retry or edit without retyping.
5. **Audit events are emitted for failures where specs require it.** The absence of a success event is itself meaningful for operator debugging.
6. **Warnings are transient.** Repair and import warnings appear in the current UI state and may appear in audit event detail, but are not persisted on the diagram row (DEC-031).

---

## Phase 9 Status

Phase 9 is complete. No new decisions were discovered while drafting this artifact.
