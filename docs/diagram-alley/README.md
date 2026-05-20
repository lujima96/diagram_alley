---
phase: 0
artifact: README
status: active
---

# Diagram Alley — Planning Docs

> **Source-of-truth rules:** See [AGENTS.md](../../AGENTS.md) before editing any file in this folder.

This folder contains all planning, specification, and design artifacts for the Diagram Alley commercial product. Agents and developers should read this file first, then follow the reading order below before proposing or implementing anything.

---

## What Is Diagram Alley

Diagram Alley is a structured, AI-assisted diagram workspace for technical documentation. It converts rough ideas into clean, editable diagrams and produces documentation-ready exports in multiple formats (ASCII, Markdown, Mermaid, SVG, PNG, JSON, YAML).

**Core promise:** Describe a system once. Diagram Alley turns it into clean, editable diagrams that can be regenerated, exported, and maintained without breaking alignment.

**The renderer is the product.** The LLM produces structured intent (a diagram spec). Diagram Alley renders output through deterministic rules — not by accepting raw AI-generated ASCII.

---

## What Is Out of Scope for V1

- Team workspaces, shared projects, role permissions, realtime collaboration
- Desktop app
- Repo scanning and AI-suggested diagrams from repo structure
- GitHub commit/PR integration (stretch goal — included only if core product is complete before launch)
- Full Mermaid import fidelity (flowchart and ER only, best-effort with warnings)
- Full ASCII reverse sync (label-change sync only; node/edge add/remove sync is V2)
- Network diagram as a V1 priority target (supported but not the benchmark)
- PDF export at launch

---

## Reading Order

Before proposing or implementing anything, read in this order:

1. `AGENTS.md` (root) — operating rules for all agents
2. This file — product intent and folder map
3. `glossary.md` — canonical terms
4. `SPEC-INDEX.md` — what specs exist, their status, and their dependencies
5. `decisions-log.md` — locked choices; these override specs when there is a conflict
6. Relevant foundation spec(s) — F0 through F5
7. Relevant domain spec(s) — D1 through D8 (or A1)

**Do not skip steps.** Reading a domain spec without reading its foundation dependencies will cause drift.

---

## Folder Map

```
docs/diagram-alley/
├── README.md              ← this file
├── glossary.md            ← canonical terms
├── SPEC-INDEX.md          ← master spec map
├── decisions-log.md       ← locked decisions
├── specs/
│   ├── F0-tech-stack.md
│   ├── F1-diagram-spec-and-data-model.md
│   ├── F2-rendering-validation-repair.md
│   ├── F3-security-auth-billing.md
│   ├── F4-ui-surface-workspace.md
│   ├── F5-api-standards.md
│   ├── D1-ai-generation.md
│   ├── D2-diagram-editor.md
│   ├── D3-version-history.md
│   ├── D4-export.md
│   ├── D5-import.md
│   ├── D6-sharing-publishing.md
│   ├── D7-billing-subscription.md
│   ├── D8-github-integration.md   ← stretch goal; write last
│   └── A1-admin-console.md
└── sketches/
    ├── personas.md
    ├── surfaces/
    │   ├── desktop.md
    │   └── mobile.md
    ├── stories/
    │   ├── solo-developer.md
    │   ├── technical-writer.md
    │   └── engineering-lead.md
    ├── flows/
    │   ├── ai-generation.md
    │   ├── manual-edit.md
    │   ├── ascii-export-and-reverse-sync.md
    │   ├── version-history.md
    │   ├── sharing.md
    │   ├── billing-upgrade.md
    │   ├── golden-paths.md
    │   ├── failure-modes.md
    │   └── schema-roundup.md
    └── mockups/
        ├── README.md
        ├── index.html
        ├── _route-map.json
        ├── _shared.css
        ├── desktop/
        ├── mobile/
        └── components/
```

---

## Spec Tier System

| Prefix | Tier | Purpose |
|--------|------|---------|
| `F0`–`F5` | Foundation | Cross-cutting contracts; must be written before any domain spec |
| `D1`–`D8` | Domain | Workflow specs for a specific product area |
| `A1` | Admin/Support | How the system is operated after launch |

Foundation specs define reusable contracts. Domain specs consume them. Domain specs must not invent foundation behavior — if a domain needs a new pattern, update the relevant foundation spec first.

---

## Decision Protocol

All locked decisions live in `decisions-log.md`. The log overrides both specs and this document in case of conflict.

When a decision is **not yet defined**, do not invent an answer. Stop and present 2–3 options in the format defined in `AGENTS.md`, then wait for an explicit choice before writing any spec content.

---

## Open Questions

All Phase 0 and audit reconciliation questions are resolved — see `decisions-log.md` DEC-010 through DEC-025.

F1 schema and migration-contract questions are resolved by DEC-018. No open questions currently block F1 from moving toward `ready`.

---

## Build Path Summary

Diagram Alley is planned in these product stages (detail in `OUTLINE.md`):

1. **Engine Proof** — deterministic spec-to-ASCII renderer
2. **Local Full-Stack Test Build** — FastAPI + Vite/React + Postgres + auth + billing foundation
3. **Product Alpha** — AI generation, all diagram types, import/export, templates
4. **Paid V1** — billing, trial, polished UX, launch-ready
5. **V2** — team workspaces, collaboration, deeper GitHub/repo integration
