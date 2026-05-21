---
phase: 3
artifact: personas (full entries)
status: complete
supersedes: Phase 0 stub (same file)
---

# Diagram Alley — Personas

Full persona entries for V1 planning. Each entry includes role code, primary surface, goals, key flows, pain points, permission cluster, and specs touched.

**V1 decision drivers:** P1 (Solo Developer) and P2 (Technical Writer). All product and UX tradeoffs optimize for these two personas first.

**Reading order:** Read OUTLINE.md §4 for background, then this file. Specs listed under "Specs touched" are the domain specs most relevant to each persona's workflows.

---

## V1 Priority Personas

### P1 — Solo Developer

**Role code:** `P1`
**Primary surface:** Desktop web app (`/workspace`, `/dashboard`, `/library`)
**Subscription model:** Free trial → Pro

#### Goals

1. Get to a clean, paste-ready diagram in under two minutes from a natural language description.
2. Export ASCII output that can go directly into a README, PR description, or code comment without cleanup.
3. Maintain a small library of diagrams across projects so they can be reopened and updated as systems evolve.

#### Key Flows

1. **Generate from prompt** — Enter a system description, pick diagram type (usually Architecture), select model provider (BYOK OpenAI-compatible or Ollama), generate, copy ASCII to clipboard.
2. **AI modify existing** — Reopen a saved diagram, type a change request ("Add Redis between API and database"), review the diff, accept or reject.
3. **Manual grid edit** — Drag nodes in the visual grid editor, rename labels, add or remove edges without touching AI again.
4. **ASCII reverse sync (label-change only)** — Edit a label directly in the ASCII panel, confirm sync back to the spec, re-export.
5. **Export to Mermaid** — One-click Mermaid export for docs sites.
6. **Provider setup** — Connect Ollama or a BYOK OpenAI-compatible key in Settings → Providers before first AI use.

#### Pain Points (from OUTLINE.md §4.1 and §19)

- AI chatbot diagrams break spacing and lose structure on follow-up edits.
- Re-generating a changed diagram produces a different layout — no determinism.
- Cannot reliably modify just one part of a chatbot-generated diagram.
- Mermaid output from chatbots is often syntactically invalid or inconsistently structured.

#### Permission Cluster

Standard `user` role. Relevant permission keys:

| Key | When used |
|-----|-----------|
| `diagram.diagram.create` | New diagram from prompt or blank |
| `diagram.diagram.update` | Manual edits, AI modification |
| `diagram.diagram.delete` | Library cleanup |
| `diagram.version.restore` | Undo AI change by restoring previous version |
| `diagram.diagram.export` | Copy/download any format |
| `project.project.create` | Organizing diagrams by repo or project |
| `provider.provider.create` | BYOK / Ollama setup |

#### Specs Touched

D1 (AI generation), D2 (diagram editor), D3 (version history), D4 (export), F2 (ASCII renderer), F4 (workspace surface), F3 (BYOK credential storage, trial limits)

---

### P2 — Technical Writer

**Role code:** `P2`
**Primary surface:** Desktop web app (`/workspace`, `/templates`, `/library`, `/projects`)
**Subscription model:** Free trial → Pro

#### Goals

1. Maintain a consistent visual language for diagrams across all docs — same box style, same arrow style, same formatting.
2. Re-export from one source spec in multiple formats (ASCII for code docs, PNG for wikis, SVG for design tools) without starting from scratch.
3. Update diagrams when the underlying system changes without losing structure or layout.

#### Key Flows

1. **Template start** — Pick a template from the gallery that matches the documentation structure (e.g., "FastAPI + React + Postgres"), customize labels with the structured editor or a prompt, export.
2. **Multi-format export** — Export the same diagram as ASCII, PNG, and Mermaid in one session.
3. **Reopen and update** — Open a saved diagram, use AI to apply a system change, review the diff, accept and re-export.
4. **Version history** — Review version history to understand how a diagram changed alongside content changes; restore an older version if the AI update was wrong.
5. **User-created template** — Save a customized diagram as a template for reuse across similar doc sets.

#### Pain Points (from OUTLINE.md §4.1 and §19)

- Diagrams created in one-off chatbot sessions are not reproducible with the same formatting.
- Regenerating an updated diagram from a chatbot produces a different layout and requires cleanup.
- Exporting to multiple formats requires separate tools and manual reformatting.
- No version history means diagram changes are hard to track alongside documentation updates.

#### Permission Cluster

Standard `user` role. Relevant permission keys:

| Key | When used |
|-----|-----------|
| `diagram.diagram.create` | New diagram from template or prompt |
| `diagram.diagram.update` | AI modification, label edits |
| `diagram.version.restore` | Restore a clean version after a bad AI update |
| `diagram.diagram.export` | Multi-format export workflow |
| `project.project.create` | Organizing diagrams per docs set or product |
| `diagram.template.create` | Saving custom templates for reuse |

#### Specs Touched

D1 (AI generation), D2 (diagram editor), D3 (version history), D4 (export), F2 (renderer), F4 (workspace surface)

---

## Secondary Personas

These personas are not V1 decision drivers. V1 should not actively break for them, but UX decisions are not optimized for them.

---

### P3 — AI Power User

**Role code:** `P3`
**Primary surface:** Desktop web app (workspace — mainly the spec editor and AI generation panel)
**Subscription model:** Pro (likely early adopter — understands the problem before seeing the product)

#### Goals

1. Use AI to produce an initial spec, then take full control of the structured JSON or YAML spec to refine it manually.
2. Understand exactly what the AI returned and what was repaired or modified by the system.
3. Avoid silent AI output corruption — wants transparent validation and repair messages.

#### Key Flows

1. **Prompt → raw spec review** — Generate a diagram, open the JSON/YAML spec editor to inspect and adjust the AI output directly.
2. **Repair transparency** — See which fields the repair pass modified before accepting the generated diagram.
3. **BYOK configuration** — Use their own OpenAI key or a locally hosted model (Ollama) to avoid hosted AI costs and maintain privacy.
4. **Export spec** — Export the JSON or YAML spec to version control alongside the diagram.

#### Pain Points

- AI tools that silently produce invalid or wrong structure are the core pain point this persona already understands.
- Wanting to see what the AI actually returned vs. what got sanitized.

#### Permission Cluster

Standard `user` role — same as P1. Also uses `provider.provider.create` frequently.

#### Specs Touched

D1 (AI generation, repair transparency), D4 (JSON/YAML spec export), F1 (spec schema), F2 (repair pass), F3 (BYOK)

---

### P4 — IT Professional / System Administrator

**Role code:** `P4`
**Primary surface:** Desktop web app (workspace — network diagram type)
**Subscription model:** Pro (if network diagram coverage meets their needs)

#### Goals

1. Document infrastructure: network topology, device relationships, VLAN layouts, service flows.
2. Export diagrams for internal IT documentation or security reviews.

#### Key Flows

1. **Network diagram creation** — Use the network diagram type to add routers, switches, servers, firewalls, and connections.
2. **Export for docs** — Export as ASCII or PNG for internal IT documentation.

#### Pain Points

- Network diagramming tools are either too heavy (Visio, draw.io) or too limited (ASCII chatbots).
- Diagrams go out of date when infrastructure changes.

#### V1 note

Not a V1 priority. Network diagram type is supported but not optimized for IT-specific workflows. V1 decisions should not be blocked by IT-specific requirements.

#### Specs Touched

D2 (diagram editor — network diagram type), D4 (export), F2 (network diagram renderer)

---

### P5 — Product Manager

**Role code:** `P5`
**Primary surface:** Desktop web app (workspace — flowchart and architecture types)
**Subscription model:** Pro (if workflow fits)

#### Goals

1. Quickly sketch feature flows, user journeys, and system handoffs for stakeholder communication.
2. Export as PNG or SVG to embed in slide decks or wiki pages.

#### Key Flows

1. **Flowchart from prompt** — Describe a feature flow in plain English, generate a flowchart, export as PNG.
2. **Dashboard embed** — Export a PNG or SVG for a product spec or wiki page.

#### Pain Points

- Speed matters over precision; rough diagrams quickly.

#### V1 note

Secondary persona. V1 optimizes for developer/technical-writer accuracy over PM sketch speed. The product is useful for PMs but they are not the reason any tradeoff is made.

#### Specs Touched

D1 (AI generation), D4 (export — PNG/SVG), F2 (flowchart renderer)

---

### P6 — Engineering Lead / Architect

**Role code:** `P6`
**Primary surface:** Desktop web app (full workspace — architecture and database types; version history; library)
**Subscription model:** Pro

#### Goals

1. Maintain authoritative, versioned architecture diagrams that the team can trust.
2. Update diagrams as the system evolves and re-export for onboarding docs, design reviews, and PRs.
3. Organize multiple diagrams (system architecture, database schema, service flows) within a project.

#### Key Flows

1. **Multi-diagram project** — Create a project for a codebase, add architecture, database schema, and flow diagrams to it.
2. **Version history** — Review how architecture evolved over time; restore a clean version before a problematic refactor was reflected.
3. **Share link** — Share a read-only link to the current architecture diagram for a design review.

#### Pain Points

- Chatbot architecture diagrams are one-off screenshots, not maintainable documentation.
- No version history means no record of how the system changed.

#### V1 note

Strong use case, especially for version history and multi-diagram project organization. Slightly lower V1 priority than P1/P2 because V1 is single-user — the collaboration and team-sharing workflows P6 ultimately wants are V2 features.

#### Permission Cluster

Standard `user` role. Also uses `diagram.share.create` (D6) for sharing links with team members.

#### Specs Touched

D1, D2, D3 (version history — critical), D4, D6 (sharing — read-only links), F1, F4

---

### P7 — Student / Learner

**Role code:** `P7`
**Primary surface:** Desktop web app (workspace — architecture and database types)
**Subscription model:** Free trial (likely does not convert to Pro)

#### Goals

1. Create visual diagrams to understand technical concepts: database relationships, application architectures, programming patterns.
2. Export diagrams for notes, assignments, or sharing with peers.

#### Pain Points

- Visual diagram tools have steep learning curves or require paid accounts.

#### V1 note

Lowest priority. The product is useful for learners but V1 does not optimize for educational or beginner-specific UX. Onboarding should not be worse for them, but dedicated beginner flows are V2 or later.

#### Specs Touched

D1, D4 (export), F4 (workspace surface)

---

## Persona Priority for V1 Decisions

| Priority | Persona | Optimize V1 for? |
|----------|---------|-----------------|
| 1 | P1 — Solo Developer | Yes |
| 2 | P2 — Technical Writer | Yes |
| 3 | P6 — Engineering Lead | Indirectly (single-user constraint limits team workflows) |
| 4 | P3 — AI Power User | Indirectly (transparency and spec control) |
| 5 | P4 — IT Professional | No — do not break, but no optimization |
| 6 | P5 — Product Manager | No |
| 7 | P7 — Student | No |
