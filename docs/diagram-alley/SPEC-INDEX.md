---
phase: 0
artifact: SPEC-INDEX
status: active
---

# Diagram Alley — Spec Index

Master map of all planning artifacts. Agents and developers use this file to find the owner of any concept and to understand which specs must be read before a given domain can be touched.

> **Rule:** Every model, enum, permission key, audit event, API operation, and UI route has exactly one owner spec. If no owner is listed here, assign one before proceeding.

Ordering within each tier follows the required reading/writing sequence. Foundation specs must be complete before any domain spec that depends on them.

---

## Phase 0 — Intake and Scope

| ID | File | Purpose | Status |
|----|------|---------|--------|
| — | `README.md` | Entry point; product intent, folder map, open questions | **Done** |
| — | `glossary.md` | Canonical term definitions | **Done** |
| — | `SPEC-INDEX.md` | This file | **Done** |
| — | `decisions-log.md` | Locked decisions; overrides specs on conflict | **Done** |
| — | `sketches/personas.md` | Initial persona list (full entries in Phase 3) | **Done** |

---

## Phase 1 — Foundation Specs

All foundation specs must reach `ready` status before dependent domain specs are written.

| ID | File | Purpose | Status | Depends On |
|----|------|---------|--------|-----------|
| F0 | `specs/F0-tech-stack.md` | FastAPI, Vite+React, Postgres, deployment targets, env model, secret management, future constraints | **Draft 0.1** | — |
| F1 | `specs/F1-diagram-spec-and-data-model.md` | Diagram spec schema, `spec_version` migration strategy, all database tables (`users`, `projects`, `diagrams`, `diagram_versions`, `templates`, `model_providers`, `user_settings`, `subscriptions`), lifecycle rules, state machines | **Draft 0.1** | F0 |
| F2 | `specs/F2-rendering-validation-repair.md` | Deterministic renderer contract, validation rules per diagram type, repair heuristics, V1 node kind rendering list, ASCII style rules, Mermaid renderer contract, visual renderer contract | **Draft 0.1** | F1 |
| F3 | `specs/F3-security-auth-billing.md` | Auth provider, session model, BYOK credential storage, Stripe setup, free trial limits (resolved DEC-010), permission key convention, audit event convention, provider credential opt-out | **Draft 0.1** | F0, F1 |
| F4 | `specs/F4-ui-surface-workspace.md` | Desktop-first surface map, React Flow grid editor architecture, grid coordinate↔ASCII mapping, navigation pattern, workspace layout, responsive rules, component conventions | **Draft 0.1** | F0 |
| F5 | `specs/F5-api-standards.md` | REST naming convention, auth on every route, pagination contract, error codes, background job standards, rate limit policy, full endpoint index | **Draft 0.1** | F0, F3 |

---

## Phase 2 — Domain Specs

Write in the order listed. Each spec depends on the foundation specs shown.

| ID | File | Purpose | Status | Depends On |
|----|------|---------|--------|-----------|
| D1 | `specs/D1-ai-generation.md` | Prompt → structured spec flow, provider routing, BYOK config, local model support, repair pass integration, AI failure handling | **Draft 0.1** | F1, F2, F3, F5 |
| D2 | `specs/D2-diagram-editor.md` | Visual grid editor interactions, spec round-trip, ASCII panel editing, reverse sync boundary enforcement, inspector panel | **Draft 0.1** | F1, F2, F4, F5 |
| D3 | `specs/D3-version-history.md` | Save, diff, restore, named versions, version snapshot triggers | **Draft 0.1** | F1, F3, F5 |
| D4 | `specs/D4-export.md` | ASCII, Markdown, Mermaid, SVG, PNG, JSON spec, YAML spec exports; export state machine; metadata in exports | **Draft 0.1** | F1, F2, F5 |
| D5 | `specs/D5-import.md` | Mermaid flowchart+ER import (best-effort, with warnings), JSON/YAML spec import, format detection, conversion preview, corruption prevention | **Draft 0.1** | F1, F2, F5 |
| D6 | `specs/D6-sharing-publishing.md` | Public share links, embed, view-only access | **Draft 0.1** | F1, F3, F5 |
| D7 | `specs/D7-billing-subscription.md` | Free trial, Pro subscription, Stripe checkout, upgrade/downgrade, trial expiry | **Draft 0.1** | F1, F3, F5 |
| A1 | `specs/A1-admin-console.md` | User management, audit log review, subscription overrides, support tools, bulk export | **Draft 0.1** | F1, F3 |
| D8 | `specs/D8-github-integration.md` | GitHub OAuth, repo selection, commit diagram spec+exports, open PR workflow — **stretch goal; write only after D1–D7 and A1 are complete** | Planned | F1, F3, F5 |

---

## Phase 3 — Personas and Surface Reality

| File | Purpose | Status |
|------|---------|--------|
| `sketches/personas.md` | Full persona entries (expands Phase 0 stub) | Planned |
| `sketches/surfaces/desktop.md` | Desktop route list, device assumptions, nav pattern, offline behavior | Planned |
| `sketches/surfaces/mobile.md` | Mobile: read-only / share viewing only (V1) | Planned |

---

## Phase 4 — Flow Narratives

| File | Purpose | Status |
|------|---------|--------|
| `sketches/flows/ai-generation.md` | End-to-end AI generation flow with realistic cast and data | Planned |
| `sketches/flows/manual-edit.md` | Manual create and edit flow | Planned |
| `sketches/flows/ascii-export-and-reverse-sync.md` | ASCII export, label-change sync, detached export warning | Planned |
| `sketches/flows/version-history.md` | Save, restore, branch from version | Planned |
| `sketches/flows/sharing.md` | Public link creation and view-only access | Planned |
| `sketches/flows/billing-upgrade.md` | Trial start, limit hit, upgrade, diagram unlock | Planned |

---

## Phase 5 — Stories

| File | Purpose | Status |
|------|---------|--------|
| `sketches/stories/solo-developer.md` | Developer persona user story set | Planned |
| `sketches/stories/technical-writer.md` | Technical writer persona user story set | Planned |
| `sketches/stories/engineering-lead.md` | Engineering lead persona user story set | Planned |

---

## Phase 6 — Mockups

| File | Purpose | Status |
|------|---------|--------|
| `sketches/mockups/index.html` | Browser index for all mockup screens | Planned |
| `sketches/mockups/_route-map.json` | Machine-readable route registry | Planned |
| `sketches/mockups/_shared.css` | Shared styles across mockup files | Planned |
| `sketches/mockups/desktop/` | Desktop screen files (landing, dashboard, editor, version history, export, billing, settings) | Planned |
| `sketches/mockups/mobile/` | Mobile screen files (share view, read-only) | Planned |
| `sketches/mockups/components/` | Reusable component files (toolbar, AI prompt panel, ASCII panel, version drawer, export modal) | Planned |

---

## Phase 7 — Schema Proposals and Consolidation

| File | Purpose | Status |
|------|---------|--------|
| `sketches/flows/schema-roundup.md` | Deduplicated schema changes from flow work | Planned |
| Per-flow proposal files | Created on demand during Phase 4 if flows discover schema gaps | Planned |

---

## Phase 8 — Golden Paths

| File | Purpose | Status |
|------|---------|--------|
| `sketches/flows/golden-paths.md` | Trial signup → first diagram; prompt → export → Git commit; free limit → upgrade; version restore → re-export | Planned |

---

## Phase 9 — Failure Modes

| File | Purpose | Status |
|------|---------|--------|
| `sketches/flows/failure-modes.md` | Failure catalog: AI outage, validation failure, import corruption, payment failure, version conflict | Planned |

---

## Phase 10 — Audit and Reconciliation

Run audits after Phase 1, after Phase 2, and after Phase 6 mockups.

| File | Purpose | Status |
|------|---------|--------|
| Audit report files | Created on demand; named `audit-YYYY-MM-DD.md` | Planned |

---

## Phase 11 — Implementation Handoff

| File | Purpose | Status |
|------|---------|--------|
| Per-domain slice files | One implementation slice file per domain, created when build begins | Planned |

---

## Concept Ownership Table

Quick lookup for "who owns this concept." If a concept is not listed here, add it before proceeding.

| Concept | Owner Spec |
|---------|-----------|
| `users` table | F1 |
| `projects` table | F1 |
| `diagrams` table | F1 |
| `diagram_versions` table | F1 |
| `templates` table | F1 |
| `model_providers` table | F1 |
| `user_settings` table | F1 |
| `subscriptions` table | F1 |
| Diagram spec schema | F1 |
| `spec_version` field and migration | F1 |
| Diagram type enum | F1 |
| Node, edge, group models | F1 |
| ASCII Renderer | F2 |
| Mermaid Renderer | F2 |
| Visual Renderer | F2 |
| Validation rules | F2 |
| Repair pass | F2 |
| Node kind list and rendering | F2 |
| Reverse sync scope | F2 |
| Detached export state | F2 |
| Auth provider and session | F3 |
| BYOK credential storage | F3 |
| Permission key convention | F3 |
| Audit event convention | F3 |
| Stripe integration | F3 |
| Free trial limits | F3 |
| Workspace layout | F4 |
| Visual grid editor architecture | F4 |
| Navigation pattern | F4 |
| API naming convention | F5 |
| Error codes | F5 |
| Pagination contract | F5 |
| Background job standards | F5 |
| AI generation flow | D1 |
| Provider adapter interface | D1 |
| Provider routing | D1 |
| AI retry policy | D1 |
| Editor interactions | D2 |
| Diagram CRUD endpoint contract | D2 |
| Undo/redo scope | D2 |
| Auto-save policy | D2 |
| ASCII reverse sync trigger | D2 |
| Version history | D3 |
| Version snapshot triggers | D3 |
| Version pruning | D3 |
| Export state machine | D4 |
| Export format list | D4 |
| Metadata embedding in exports | D4 |
| Import and conversion | D5 |
| Mermaid converter | D5 |
| Import corruption prevention | D5 |
| Public share links | D6 |
| `diagram_shares` table | D6 |
| Share token model | D6 |
| Embed support | D6 |
| Billing and subscription tiers | D7 |
| Trial countdown UX | D7 |
| Stripe checkout flow | D7 |
| Billing portal delegation | D7 |
| Admin tools | A1 |
| User management actions | A1 |
| Subscription override | A1 |
| Audit log query interface | A1 |
| GitHub integration | D8 |
