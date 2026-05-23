---
phase: 11
artifact: slice-template
status: reference
---

# Implementation Slice Template

Use this template to create one slice file per spec when Phase 11 begins. Files live in `docs/diagram-alley/specs/slices/` and are named `slice-<id>.md` (e.g., `slice-F0.md`, `slice-D1.md`).

Slice files **sequence and gate** spec content. They do not repeat it. Every statement in a slice file that describes behavior should cite the spec section that defines it.

---

## How to Fill This In

- **Pre-conditions:** List what must be merged, deployed, or confirmed working before a developer can start this slice. Be specific — name the slice it depends on, not just the spec.
- **Build sequence:** Ordered steps, coarsest to finest. Each step should be completable and testable independently. Link to the spec section that defines the behavior. Aim for 5–12 steps.
- **Done criteria:** Pull directly from golden paths (Phase 8) and spec acceptance conditions. Each criterion must be falsifiable. No vague items like "it works."

---

```
---
phase: 11
slice: <spec ID>          # e.g. F0, D1
spec: <spec file>         # e.g. specs/F0-tech-stack.md
status: planned           # planned | in-progress | complete
---

# Slice <ID> — <Spec Title>

## Pre-conditions

<!-- What must be done before this slice can start. Name specific slices or external prerequisites. -->

- [ ] Slice <X> complete (reason: <why this unblocks the current slice>)
- [ ] <External prerequisite, e.g., "Fly.io app created and `fly.toml` committed">

## Build Sequence

<!-- Ordered. Each step is independently testable. Cite the spec section. -->

1. **<Step name>** — <One-sentence description of what to build.> (→ <spec ID> §<section>)
2. **<Step name>** — <Description.> (→ <spec ID> §<section>)
...

## Done Criteria

<!-- Pulled from golden paths and specs. Each item is falsifiable. -->

- [ ] <Criterion from GP-XX or spec acceptance condition>
- [ ] <Criterion>
...
```

---

## Sequencing Guide

Foundation slices must reach `complete` before the domain slices that depend on them. The required order matches the SPEC-INDEX dependency table:

```
F0  ──►  F1  ──►  F2
     ├──►  F3
     ├──►  F4
     └──►  F5

F1 + F2 + F3 + F5  ──►  D1
F1 + F2 + F4 + F5  ──►  D2
F1 + F3 + F5       ──►  D3
F1 + F2 + F5       ──►  D4
F1 + F2 + F5       ──►  D5
F1 + F3 + F5       ──►  D6
F1 + F3 + F5       ──►  D7
F1 + F3            ──►  A1
D1–D7 + A1         ──►  D8  (stretch)
```

Within the foundation group, F0 must be complete first (it defines the runtime environment everything else is deployed into). F1, F3, F4, F5 can proceed in parallel once F0 is done. F2 depends on F1.
