---
phase: 0
artifact: glossary
status: active
---

# Diagram Alley — Glossary

Canonical definitions for all terms used across specs, flows, and code. When a term appears in a spec, it means exactly what this file defines. If a spec uses a term differently, the spec is wrong and should be corrected.

Terms are listed alphabetically. Add new terms here when they are first introduced anywhere in the planning corpus.

---

## ASCII Output

The strict-ASCII text rendering of a diagram, using printable ASCII characters only. Produced deterministically by the ASCII Renderer from the Diagram Spec. Must be reproducible from the same spec every time with no variation. The primary text export format for V1.

See also: [Text Output](#text-output), [Detached Export](#detached-export), [ASCII Renderer](#ascii-renderer).

---

## ASCII Renderer

The component responsible for converting a validated Diagram Spec into strict ASCII Output or UTF-8 Text Output. The renderer is deterministic — given the same spec, layout metadata, and text mode, it always produces the same output. The LLM is never the final layout engine. Owner spec: `F2-rendering-validation-repair.md`.

---

## Audit Event

A record written to the audit log when a significant action occurs. Each event captures the actor, the entity affected, the action taken, and a timestamp. Audit events are defined in Foundation spec `F3` and referenced (not redefined) in domain specs.

---

## BYOK (Bring Your Own Key)

A provider configuration where the user supplies their own API key for a cloud AI provider (e.g., OpenAI, Anthropic). The key is stored server-side with user-level opt-out for credential persistence. BYOK is a V1 commercial feature. Owner spec: `F3-security-auth-billing.md`.

---

## Detached Export

An ASCII export that is no longer synchronized with the Diagram Spec. This state occurs when a user makes ASCII edits beyond the supported reverse-sync scope (label changes only in V1). The app warns the user when a diagram is in detached export state. The spec remains unchanged; the ASCII is a one-off snapshot. Owner spec: `F2-rendering-validation-repair.md`.

See also: [Reverse Sync](#reverse-sync).

---

## Diagram

A structured visual artifact consisting of a Diagram Spec plus rendered outputs (ASCII, Mermaid, SVG, etc.). A diagram belongs to one Project. A diagram can have many Versions. Owner table: `diagrams`. Owner spec: `F1-diagram-spec-and-data-model.md`.

---

## Diagram Spec (also: Spec)

The structured JSON or YAML document that is the source of truth for a diagram's content. The spec describes nodes, edges, groups, layout direction, and diagram-type-specific data. The spec is stored in the `diagrams.spec_json` column. All rendering, editing, and export starts from the spec. The LLM produces a spec candidate; the app validates and may repair it before accepting it. Owner spec: `F1-diagram-spec-and-data-model.md`.

---

## Diagram Type

The category of a diagram. Determines which schema, validation rules, renderer, editor UI, and export options apply. V1 diagram types:

- `architecture` — nodes, edges, groups, layers
- `database` — tables, columns, relationships
- `flowchart` — process, decision, branch nodes
- `ui_wireframe` — page layout, regions, components
- `file_structure` — folders, files, annotations
- `network` — devices, zones, connections (supported but not V1 priority benchmark)

Owner spec: `F1-diagram-spec-and-data-model.md`.

---

## Edge

A directed or undirected connection between two Nodes in a diagram. Edges have an optional label and belong to one diagram. Owner spec: `F1-diagram-spec-and-data-model.md`.

---

## Export

A rendered artifact produced from the Diagram Spec in a specific output format. Exports include ASCII, Markdown, Mermaid, SVG, PNG, JSON spec, and YAML spec. Exports may include metadata linking them to their source spec and version. Owner spec: `D4-export.md`.

---

## Foundation Spec

A spec file in the `F*` tier that defines cross-cutting contracts (tech stack, data model, rendering, security, UI, API). Foundation specs must be written and stabilized before any Domain spec that depends on them. Domain specs reference foundation contracts; they do not redefine them.

---

## Group

An optional container in an architecture diagram that holds one or more Nodes. Used to represent layers, service clusters, or zones. Owner spec: `F1-diagram-spec-and-data-model.md`.

---

## Layout Engine

The part of the ASCII/Text Renderer responsible for positioning nodes and edges on a grid and calculating spacing, box widths, and arrow routes. The layout engine is deterministic. It uses grid position metadata from the Visual Grid Editor where available. Owner spec: `F2-rendering-validation-repair.md`.

---

## Local Provider

An AI model endpoint running on the user's own machine or local network, such as Ollama or llama.cpp. Local providers allow users to keep diagram prompts private without sending data to cloud services. Owner spec: `F3-security-auth-billing.md`, `D1-ai-generation.md`.

---

## Model Provider (also: Provider)

A configured AI model endpoint used for diagram generation and modification. Can be a BYOK cloud API, a local Ollama/llama.cpp server, or (if offered) hosted credits on Diagram Alley's own infrastructure. Each user can configure multiple providers. Owner spec: `F3-security-auth-billing.md`, `D1-ai-generation.md`.

---

## Node

A discrete element within an architecture or network diagram. Nodes have an ID, a label, a kind (e.g., `service`, `database`, `gateway`), and optional grid position metadata. Owner spec: `F1-diagram-spec-and-data-model.md`.

---

## Node Kind

The semantic type of an architecture or network Node. Node kinds determine the icon, rendering style, and optional validation rules. All named architecture node kinds get distinct rendering in V1 (DEC-012). The full list of V1 node kinds and rendering treatments is defined in `F2-rendering-validation-repair.md`. Owner spec: `F2-rendering-validation-repair.md`.

---

## Text Output

A deterministic plain-text rendering of a diagram. Text Output can be strict ASCII or UTF-8. UTF-8 text mode may use Unicode box-drawing or tree characters when they materially improve readability; strict ASCII mode must not. Owner spec: `F2-rendering-validation-repair.md`.

---

## Open Question

An unresolved decision that is explicitly tracked inside a spec, the decisions log, or this README. Open questions block the spec from advancing to `ready` status until resolved by the user. Format: `[OPEN — <owning spec>] Question text. Blocks: <artifact>`.

---

## Permission Key

A namespaced string that identifies a single authorized action. Convention: `domain.entity.action`. All permission keys are defined in the Foundation spec `F3`. Domain specs reference keys; they do not define new ones without updating F3. Owner spec: `F3-security-auth-billing.md`.

---

## Project

A named workspace that contains one or more Diagrams. Projects belong to a User. In V1, projects include a nullable `team_id` column for forward compatibility with V2 team workspaces, but team features are not active in V1. Owner spec: `F1-diagram-spec-and-data-model.md`.

---

## Repair Pass

An automated step that attempts to fix a Diagram Spec that has failed validation. The repair pass runs after LLM output is received and can: fill missing node IDs, remove edges referencing non-existent nodes, normalize diagram type, and default missing required values. If the spec cannot be repaired automatically, the user is shown a clear error. Owner spec: `F2-rendering-validation-repair.md`.

---

## Renderer

Any component that converts a validated Diagram Spec into a specific output format. V1 renderers: ASCII/Text Renderer, Mermaid Renderer, Visual Renderer (SVG/grid), SVG export renderer, and PNG export conversion. Owner spec: `F2-rendering-validation-repair.md`.

---

## Reverse Sync

The process of applying a user's manual ASCII edit back into the Diagram Spec. **V1 scope (locked):** Label changes only. If a node or edge label is edited in the ASCII output, the app attempts to sync that change back into the spec. All other ASCII edits (add/remove nodes, change connections, restructure layout) produce a Detached Export with a warning. Full reverse sync is V2. Owner spec: `F2-rendering-validation-repair.md`.

---

## Spec Version (`spec_version`)

A string field on every Diagram Spec document that records the schema version in use (e.g., `"1.0"`). The backend checks this field on load and either migrates the spec to the current version or returns a clear error. Migration logic is defined in `F1-diagram-spec-and-data-model.md`.

---

## Template

A reusable starting Diagram Spec that can be opened in the workspace. Templates are either system-provided (built-in) or user-created. Templates are not overwritten when a user opens them — a copy is made. Owner spec: `F1-diagram-spec-and-data-model.md`.

---

## User

An authenticated account in Diagram Alley. V1 is single-user (no teams). Users have a plan (free trial or pro), an optional default model provider, and optional stored provider credentials. Owner table: `users`. Owner spec: `F1-diagram-spec-and-data-model.md`.

---

## Validation

The process of checking a Diagram Spec against the rules for its diagram type before rendering or saving. Validation returns errors, warnings, and repair suggestions. A spec with unresolved errors cannot be rendered. Owner spec: `F2-rendering-validation-repair.md`.

---

## Version

A snapshot of a Diagram Spec at a point in time. Versions are created when a diagram is saved with meaningful changes. Users can restore or branch from a previous version. Owner table: `diagram_versions`. Owner spec: `D3-version-history.md`.

---

## Visual Grid Editor

The primary editing surface in the workspace. A canvas that renders nodes and edges on a deterministic grid, allows drag-and-drop repositioning, and updates the Diagram Spec on every edit. The visual grid editor is backed by layout metadata that also informs the ASCII/Text Renderer. The grid editor architecture must be specced in `F4-ui-surface-workspace.md` before any domain UI implementation. Owner spec: `F4-ui-surface-workspace.md`, `D2-diagram-editor.md`.

---

## Workspace

The main page of the Diagram Alley UI where a user creates, edits, previews, and exports a single diagram. The workspace contains the Prompt Panel, Visual Grid Editor, Inspector Panel, and Preview/Export Panel. Owner spec: `F4-ui-surface-workspace.md`.
