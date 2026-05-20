# Plan: Create planning_roadmap.md for Diagram Alley

## Context

The user has a complete product intent document (`outline.md`) and a reusable planning process (`planning.md` — 11 phases). Before creating any specs or mockups, they need a single sequenced roadmap that maps the planning.md phases onto the specific artifacts required for Diagram Alley. This roadmap becomes the working checklist for all planning work.

## Output File

`/home/paul/Desktop/diagrams/planning_roadmap.md`

## What the roadmap must contain

For each phase: the objective, the ordered list of artifact files to produce (with paths), any hard dependencies on prior phase outputs, and a checkbox so progress is trackable.

## Artifact sequence derived from planning.md + outline.md

### Phase 0 — Intake and Scope (unblock everything else)
- `docs/diagram-alley/README.md` — entry point for all agents/devs; wraps outline.md intent into a navigable doc
- `docs/diagram-alley/glossary.md` — canonical terms (spec, diagram, node, edge, renderer, reverse sync, detached export, BYOK, etc.)
- `docs/diagram-alley/SPEC-INDEX.md` — master map; initially just stubs with status: planned
- `docs/diagram-alley/decisions-log.md` — seed with all closed decisions already in outline.md §1.1
- `docs/diagram-alley/sketches/personas.md` — initial list only (full persona work is Phase 3)
- Open questions list (inline in README or as a section)

### Phase 1 — Foundation Lock (must all precede any domain spec)
Order matters within this phase — F1 before F3, F4 before domain UI work:

1. `docs/diagram-alley/specs/F0-tech-stack.md` — FastAPI, Vite+React, Postgres, deployment targets, env model, future constraints
2. `docs/diagram-alley/specs/F1-diagram-spec-and-data-model.md` — spec_version strategy, JSON/YAML schema, diagrams/projects/versions/users tables, state machines, lifecycle rules
3. `docs/diagram-alley/specs/F2-rendering-validation-repair.md` — deterministic renderer contract, validation rules, repair heuristics, V1 node kind rendering list
4. `docs/diagram-alley/specs/F3-security-auth-billing.md` — auth provider, BYOK credential model, Stripe setup, free trial limits (must be resolved here), permission key convention, audit event convention
5. `docs/diagram-alley/specs/F4-ui-surface-workspace.md` — desktop-first surface, visual grid editor architecture (must be specced before Phase 3 implementation), navigation pattern, responsive rules
6. `docs/diagram-alley/specs/F5-api-standards.md` — naming convention, auth on every route, pagination, error codes, background job standards, webhook shape

### Phase 2 — Domain Decomposition
Each domain spec depends on the F-specs it lists. Write in dependency order:

1. `docs/diagram-alley/specs/D1-ai-generation.md` — prompt → structured spec flow; provider routing; BYOK; local model support (depends: F1, F2, F3, F5)
2. `docs/diagram-alley/specs/D2-diagram-editor.md` — visual grid editor interactions, spec round-trip, ASCII panel, reverse sync boundary (depends: F1, F2, F4, F5)
3. `docs/diagram-alley/specs/D3-version-history.md` — save, diff, restore, named versions (depends: F1, F3, F5)
4. `docs/diagram-alley/specs/D4-export.md` — PNG, SVG, ASCII, JSON/YAML spec, PDF; export state machine (depends: F1, F2, F5)
5. `docs/diagram-alley/specs/D5-import.md` — Mermaid flowchart+ER (best-effort with warnings), JSON spec import (depends: F1, F2, F5)
6. `docs/diagram-alley/specs/D6-sharing-publishing.md` — public links, embed, view-only (depends: F1, F3, F5)
7. `docs/diagram-alley/specs/D7-billing-subscription.md` — free/paid tiers, Stripe checkout, upgrade/downgrade, trial expiry (depends: F1, F3, F5)
8. `docs/diagram-alley/specs/A1-admin-console.md` — user management, feature flags, audit log review, support tools (depends: F1, F3)
9. `docs/diagram-alley/specs/D8-github-integration.md` — **stretch goal only; write after core product specs are complete** (depends: F1, F3, F5)

### Phase 3 — Personas and Surface Reality
- `docs/diagram-alley/sketches/personas.md` — full entries: role code, primary surface, goals, key flows, pain points, permission cluster, specs touched
- `docs/diagram-alley/sketches/surfaces/desktop.md`
- `docs/diagram-alley/sketches/surfaces/mobile.md` (read-only / share viewing)

### Phase 4 — Flow Narratives
One file per major workflow (these are the highest-value ones for Diagram Alley):
- `docs/diagram-alley/sketches/flows/ai-generation.md`
- `docs/diagram-alley/sketches/flows/manual-edit.md`
- `docs/diagram-alley/sketches/flows/ascii-export-and-reverse-sync.md`
- `docs/diagram-alley/sketches/flows/version-history.md`
- `docs/diagram-alley/sketches/flows/sharing.md`
- `docs/diagram-alley/sketches/flows/billing-upgrade.md`

### Phase 5 — Stories
- `docs/diagram-alley/sketches/stories/solo-developer.md`
- `docs/diagram-alley/sketches/stories/technical-writer.md`
- `docs/diagram-alley/sketches/stories/engineering-lead.md`

### Phase 6 — Mockups
- `docs/diagram-alley/sketches/mockups/index.html` (browser index)
- `docs/diagram-alley/sketches/mockups/_route-map.json`
- `docs/diagram-alley/sketches/mockups/_shared.css`
- Desktop screens (landing, dashboard, editor, version history, export, billing, settings)
- Mobile screens (share view, read-only)
- Component files (toolbar, AI prompt panel, ASCII panel, version drawer, export modal)

### Phase 7 — Schema Proposals and Consolidation
- Per-flow proposal files as needed
- `docs/diagram-alley/sketches/flows/schema-roundup.md`

### Phase 8 — Golden Paths
- `docs/diagram-alley/sketches/flows/golden-paths.md`
  - Trial signup → first diagram generated
  - Prompt → spec → ASCII export → Git commit (stretch)
  - Free user hits limit → upgrades → diagram unlocked
  - Version restore → re-export

### Phase 9 — Failure Modes
- `docs/diagram-alley/sketches/flows/failure-modes.md`

### Phase 10 — Audit and Reconciliation
- Run after Phase 1, after Phase 2, and after Phase 6 mockups
- Audit report files as needed

### Phase 11 — Implementation Handoff
- One slice file per domain, created on demand as build begins

## Format of planning_roadmap.md

- One `##` section per phase with objective
- Checkbox list of artifact files under each phase (`- [ ] path/to/file.md — one-line purpose`)
- Dependency callouts where phase order matters
- Status column or label (Not Started / In Progress / Done) per phase
- A note at the top pointing to AGENTS.md for source-of-truth rules

## Verification

After writing: confirm the file exists, opens cleanly, and that the phase sequence matches planning.md's ordering. No code to run — purely document-based.
