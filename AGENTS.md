# Agent Operating Rules — Diagram Alley

## Source of Truth Hierarchy

1. **Specs are the source of truth** once created. A spec is any file under `docs/diagram-alley/specs/` that has been written and not marked `status: draft-abandoned`.
2. **`outline.md`** is the pre-spec product intent document. It is authoritative for decisions that have not yet been promoted into a spec.
3. **`decisions-log.md`** records locked choices. A decision in the log overrides both the spec and `outline.md` if there is a conflict — the log entry is the authoritative version; update the spec to match.
4. **Chat history is not authoritative.** If something was discussed but not written into a spec, the log, or `outline.md`, it does not exist.

## Decision Protocol

**When a decision is already defined** (in a spec, `decisions-log.md`, or the Closed Technical Decisions block of `outline.md`):
- Use the recorded decision. Do not re-open it.
- Do not ask the user to re-decide something already decided.

**When a decision is NOT yet defined:**
- Do not invent an answer and proceed silently.
- Stop and ask the user. Present the smallest set of viable options (2–3), recommend one with a one-line rationale, and wait for an explicit choice.
- Once the user chooses, write a decision-log entry before continuing.

**Option format to use when asking:**

```
## Decision Needed: <Topic>

**Context:** <Why this matters and what it blocks.>

**Option A:** <Choice> — <one-line pro/con>
**Option B:** <Choice> — <one-line pro/con>

**Recommendation:** Option A/B because <reason>.
```

## Authorization Required to Override Specs

Specs may only be changed if one of the following is true:

- The user explicitly says to change the spec (e.g., "update the spec", "revise F1", "change that decision").
- A decision-log entry exists that supersedes the current spec content.
- The change is a non-breaking clarification (wording, examples) that does not alter behavior, data shape, permissions, or workflow.

**Never silently change a spec to match your implementation preference.** If you believe a spec is wrong, flag it to the user and wait for authorization.

## Reading Order

Before proposing or implementing anything:

1. Read `outline.md` (product intent and closed decisions).
2. Read `docs/diagram-alley/SPEC-INDEX.md` (what exists and what it depends on).
3. Read `docs/diagram-alley/decisions-log.md` (locked choices).
4. Read the relevant foundation spec(s) before any domain spec.
5. Read the relevant domain spec before touching implementation.

## One Owner Per Concept

Every model, enum, permission key, audit event, API operation, and UI route has exactly one owner spec. If you need to reference something from another spec, cite it — do not redefine it. If no owner exists yet, assign one before proceeding and note it in SPEC-INDEX.md.

## When You Find Drift

If a spec, the decisions log, and `outline.md` contradict each other:

1. Flag the contradiction to the user with the specific files and lines involved.
2. Do not silently pick a winner.
3. Wait for the user to resolve it before writing code or updating docs.
