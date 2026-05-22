# AI Software Build Planning Outline

This document is a reusable process for  preparing, specifying, auditing, and handing off a software build. Use it before implementation when the project is large enough that ad hoc prompting would create drift, duplicated decisions, missed edge cases, or unclear ownership.

The goal is not to create paperwork for its own sake. The goal is to turn a vague product idea into a coherent build plan with:

- explicit decisions and rationale
- stable source-of-truth documents
- clear domain boundaries
- user-centered workflow narratives
- implementation-ready data, API, permission, and UI contracts
- audit loops that catch contradictions before code is written

## How To Use This File

1. Create a planning folder for the project, usually `docs/<project-name>/`.
2. Copy the artifact structure in this document.
3. Work from foundation specs to domain specs to admin/support specs.
4. Keep open questions inline until the user decides them.
5. Move every locked answer into a decision log with rationale.
6. Run consistency audits before implementation.
7. Only start code once the first implementation slice has clear specs, data shape, workflows, permissions, UI surfaces, and acceptance criteria.

For small projects, compress the process into fewer files. For large projects, keep the separation. The separation is what lets multiple AI agents or developers work without stepping on each other.

## Core Rules

- One concept has one owner. Other documents may reference it, but they must not redefine it.
- Record decisions when they happen. Do not rely on chat history.
- Every decision needs a why, not just a what.
- Foundation contracts come before feature specs.
- Workflows must describe real user behavior, not just ideal database operations.
- Data models, API contracts, permissions, audit events, and UI states must agree.
- Cross-cutting behavior belongs in foundation specs, not copied into every domain spec.
- Open questions stay visible until resolved.
- Deferred work must be explicitly marked as deferred with the reason and re-entry condition.
- Audit the whole corpus after major changes. Local edits often create global drift.

## Recommended Folder Structure

```text
docs/<project-name>/
|-- README.md
|-- SPEC-INDEX.md
|-- decisions-log.md
|-- schema.md or schema.prisma or data-model.md
|-- specs/
|   |-- F0-tech-stack-and-future.md
|   |-- F1-data-model-foundation.md
|   |-- F2-domain-math-and-rules.md
|   |-- F3-security-permissions-audit.md
|   |-- F4-ui-surface-architecture.md
|   |-- F5-api-and-integration-standards.md
|   |-- D1-first-domain-workflow.md
|   |-- D2-second-domain-workflow.md
|   `-- A1-admin-and-support.md
`-- sketches/
    |-- README.md
    |-- personas.md
    |-- surfaces/
    |   |-- desktop.md
    |   |-- tablet.md
    |   `-- mobile.md
    |-- stories/
    |   `-- <persona>.md
    |-- flows/
    |   |-- <workflow>.md
    |   |-- <workflow>-schema-proposals.md
    |   |-- golden-paths.md
    |   |-- failure-modes.md
    |   `-- schema-roundup.md
    `-- mockups/
        |-- README.md
        |-- desktop/
        |-- tablet/
        |-- mobile/
        `-- components/
```

Adjust names for the project. Keep the idea: index, decision log, foundation specs, domain specs, admin specs, personas, surfaces, stories, flows, mockups, schema proposals, audits.

## Artifact Purposes

### README.md

Use the README as the entry point. It should explain:

- what is being built
- what is deliberately out of scope
- how the docs are organized
- the spec tier system
- how decisions are locked
- how schema or data contracts are managed
- how to run any lint, validation, or generation scripts
- how implementation teams should read the folder

The README should be written for a new AI assistant or developer who has no chat history.

### SPEC-INDEX.md

Use the index as the master map of content. It should list every spec with:

- ID
- title
- tier
- status
- version
- dependencies
- owner if known
- short description

For larger projects, use stable IDs:

- `F*` for foundation and cross-cutting specs
- `D*` for domain workflow specs
- `A*` for admin, support, or platform-operation specs

Example:

```markdown
| ID | Title | Status | Depends On |
|---|---|---|---|
| F1 | Data Model Foundation | Draft 0.1 | F0 |
| F3 | Security, Permissions, Audit | Draft 0.1 | F0, F1 |
| D1 | Customer Onboarding | Draft 0.1 | F1, F3, F4, F5 |
| A1 | Admin Console | Draft 0.1 | F1, F3, F4 |
```

### decisions-log.md

Use the decision log as the canonical record of user-approved choices. It should be chronological or reverse-chronological, but the ordering rule must be stated at the top.

Each decision entry should include:

- date
- spec or flow affected
- decision
- rationale
- citations to docs changed
- downstream consequences
- deferred follow-ups if any

Template:

```markdown
## YYYY-MM-DD - <Area or Pass Name>

### <Decision Title>

**Decision:** <What is now locked. Be explicit.>

**Why:** <Business, user, technical, regulatory, cost, or workflow reason.>

**Consequences:** <What other specs, schema fields, APIs, permissions, or UI flows must align.>

**Citations:** `<file>`, `<file>`, `<spec-id> section`.
```

Rules:

- If an AI and user decide something in chat, write it here.
- If a later decision reverses an earlier one, do not delete the old decision. Add a superseding entry.
- If a proposal is rejected, record that too. Rejected options prevent repeat debates.
- If a feature is deferred, record the trigger that would bring it back.

### schema or data-model document

For any build with persistent data, create one canonical data-model source. This can be:

- `schema.prisma`
- `schema.sql`
- `data-model.md`
- generated `schema.md`
- OpenAPI or JSON Schema files

Rules:

- A model, table, enum, event, or field must have one canonical owner.
- Feature specs may quote fields, but should not redefine entire models independently.
- Every field should have an origin or owner when the corpus is large.
- Cross-cutting enums belong in foundation specs.
- Proposals can live separately until approved, but approved schema must move into the canonical source.

Recommended field annotation:

```prisma
// from: D3 section 2.1
customerId Int
```

### Foundation Specs

Foundation specs define reusable contracts. They should be locked before domain specs depend on them.

Typical foundation specs:

- `F0` Tech stack, deployment, environments, future constraints
- `F1` Data model, identity, state machines, lifecycle rules
- `F2` domain math, pricing, scoring, allocation, rules engines
- `F3` security, permissions, audit, compliance, privacy
- `F4` UI surface architecture, layout conventions, navigation, devices
- `F5` API standards, integration standards, background job standards
- optional infrastructure specs for notifications, files, search, realtime, imports, exports, reporting, AI processing

Foundation specs should include:

- purpose
- scope and non-scope
- canonical terms
- owned models or contracts
- state machines
- conventions
- examples
- dependencies
- conflicts audit
- open questions

### Domain Specs

Domain specs define a business workflow or product capability. They consume foundation specs.

Each domain spec should include:

- purpose
- what it owns
- what it does not own
- dependencies
- page or route map
- primary personas
- data model references
- workflow steps
- state transitions
- permissions
- audit events
- API operations
- UI states
- edge cases
- failure handling
- notifications
- reporting impacts
- open questions
- conflicts audit
- acceptance criteria

Domain specs should not silently invent foundation behavior. If a domain needs a new permission pattern, audit event shape, model lifecycle, UI convention, notification channel, or API standard, update the relevant foundation spec or create a schema proposal.

### Admin and Support Specs

Admin/support specs define how the system is operated after launch. They are often the difference between a demo and a real product.

Include:

- user and role management
- system settings
- feature flags
- integrations
- import/export tools
- audit log review
- support tools
- recovery workflows
- configuration safety patterns
- bulk operations
- dry runs
- revert paths
- diagnostics
- help or documentation systems

For every admin action, answer:

- Who can do it?
- Is a second approval, PIN, or MFA step required?
- What is audited?
- Can it be previewed before commit?
- Can it be reverted?
- What downstream workflows does it affect?

## Spec Template

Use this template for each spec.

```markdown
---
id: D1
tier: domain
status: draft
version: "0.1"
depends-on: [F1, F3, F4, F5]
tags: [tier/domain, area/<area-name>]
---

# D1 - <Spec Name>

**Version:** 0.1
**Status:** Draft
**Depends on:** F1, F3, F4, F5

## Purpose

<What this spec owns and why it exists.>

## Non-Goals

<What this spec explicitly does not own.>

## Personas

<Primary and secondary users.>

## Page Map or Surface Map

| Route or Screen | Purpose | Primary Surface |
|---|---|---|

## Data Contracts

<Canonical model references or proposed additions.>

## Workflow

<Happy path, with numbered steps.>

## State Machine

<States, legal transitions, owners, triggers.>

## Permissions

| Permission | Allows | PIN or Approval | Audit Event |
|---|---|---|---|

## API or Service Contracts

<Queries, mutations, endpoints, events, jobs, webhooks.>

## UI Requirements

<Required controls, empty states, loading states, errors, accessibility, responsive behavior.>

## Notifications

<Who is notified, on which channel, under which severity, with what actions.>

## Reports and Analytics

<Metrics, dashboards, exports, operational reporting.>

## Edge Cases and Failure Modes

<What breaks, how users recover, what the system does automatically.>

## Audit Events

| Event | Trigger | Actor | Entity | Details |
|---|---|---|---|---|

## Conflicts Audit

<Potential overlap with other specs. Say which spec wins.>

## Open Questions

- [OPEN] <Question, options, default recommendation, affected files.>

## Acceptance Criteria

- <Observable behavior that proves this spec is implemented.>
```

## Planning Phases

### Phase 0: Intake and Scope

Objective: understand what is being built and what success means.

Ask:

- Who are the users?
- What problem does this solve?
- What workflows exist today?
- What must be preserved?
- What can be changed?
- What are the regulatory, security, compliance, or contractual constraints?
- What existing systems are sources of truth?
- What integrations are required?
- What is out of scope for version 1?
- What does a successful first release look like?

Outputs:

- project README
- initial glossary
- initial persona list
- initial spec index
- first open questions list

### Phase 1: Foundation Lock

Objective: define the contracts every feature depends on.

Create foundation specs for:

- tech stack and deployment
- data model and identity
- state machines
- permissions and audit
- UI surface architecture
- API and integration standards
- background jobs and events
- file storage and document handling if needed
- notifications if needed
- reporting if needed

Do not let domain specs each invent their own version of these.

Outputs:

- foundation specs
- canonical data-model shell
- permission key convention
- audit event convention
- API convention
- UI surface convention
- decision-log entries for locked choices

### Phase 2: Domain Decomposition

Objective: split the product into coherent workflow specs.

A good domain spec boundary usually matches one of:

- a user workflow
- a business object lifecycle
- an operational department
- a bounded context
- a major external integration

For each domain, define:

- owner persona
- dependent foundation specs
- owned models or fields
- consumed models or fields
- lifecycle states
- screens
- APIs
- permissions
- audit events
- edge cases

Outputs:

- domain specs
- route map updates
- model references
- open questions
- conflict notes

### Phase 3: Personas and Surface Reality

Objective: make sure specs describe real users, devices, and constraints.

Create `personas.md` with:

- role code
- primary surface
- goals
- key flows
- pain points
- permissions cluster
- specs touched
- notification expectations

Create surface maps for:

- desktop
- tablet
- mobile
- customer portal or public app if relevant
- command line, embedded device, kiosk, or API-only clients if relevant

For each surface, define:

- route or screen list
- primary user
- device assumptions
- input method
- navigation pattern
- offline behavior if needed
- accessibility requirements
- performance expectations

Do not design only for a generic browser if the real user is gloved, driving, in a warehouse, on a kiosk, in a call center, using screen readers, or switching tabs across monitors.

Outputs:

- personas
- surface maps
- revised domain specs if reality changes the workflow

### Phase 4: Flow Narratives

Objective: turn specs into end-to-end user stories that expose missing details.

Each flow narrative should include:

- physical or operational reality
- trigger
- happy path
- action count or effort budget
- UI surface used at each step
- data created or updated
- permissions checked
- audit events emitted
- notifications sent
- edge cases
- failure recovery
- after-hours behavior
- open questions

Flow narratives should be concrete. Use realistic names, timestamps, sample records, and actual screens. This is how contradictions become visible.

Template:

```markdown
# <Workflow> Flow

## 1. Physical or User Reality

<Where the user is, what device they use, what pressures exist.>

## 2. Happy Path

1. <Step>
2. <Step>
3. <Step>

## 3. Action Budget

| Persona | Target | Actual Narrative Count |
|---|---:|---:|

## 4. Data Chain

<Entities created or updated in order.>

## 5. Audit Chain

<Events emitted in order.>

## 6. Failure Modes

<What goes wrong and how the system recovers.>

## 7. Open Questions

- [OPEN] <Question>
```

Outputs:

- flow narratives
- schema proposals
- UI mockup requirements
- decisions for the log

### Phase 5: Stories

Objective: create implementation-sized user stories without losing context.

Organize by persona, feature, or epic.

Each story should include:

- user role
- goal
- trigger
- acceptance criteria
- route or API
- relevant spec section
- dependencies
- edge case notes

Template:

```markdown
## Story: <Short Name>

As a <persona>, I want <capability>, so that <outcome>.

**Spec:** D1 section 4
**Surface:** Tablet
**Route:** `/example/:id`
**Permission:** `domain.entity.action`

### Acceptance Criteria

- Given <state>, when <action>, then <result>.
- Audit event `<EVENT_NAME>` is emitted with <details>.
- Unauthorized users cannot perform the action.
```

Outputs:

- persona story files
- backlog seeds
- test seeds

### Phase 6: Mockups and Interaction References

Objective: show enough UI to remove ambiguity before code.

Mockups can be HTML, screenshots, Figma files, markdown sketches, or component prototypes. The format matters less than consistency and traceability.

For each important screen, capture:

- route
- surface
- persona
- primary action
- secondary actions
- loading state
- empty state
- error state
- permission-denied state
- destructive confirmation
- audit or saved confirmation
- responsive constraints

Create reusable components for repeated device or workflow elements.

Examples:

- search and filter bars
- import dry-run table
- notification drawer
- scanner or barcode input panel
- file upload panel
- approval modal
- audit-event detail panel
- settings diff preview

### Mockup Browser Index

When mockups are file-based HTML, create a `sketches/mockups/index.html` entry point. This gives users, developers, and AI assistants one browser-openable place to inspect the whole UI corpus without hunting through folders.

**Organize the index by route or page, not by surface.** Group all surface variants of the same route together. A route that exists on desktop, tablet, and mobile should show all three links in one row or card cluster, labeled by surface. This makes it immediately obvious which routes have surface gaps and lets reviewers compare variants without hunting through separate surface sections.

Each route group should show:

- the canonical route path
- the page title or screen name
- one link per surface variant (desktop, tablet, mobile, embedded, etc.), each labeled by surface
- access level badge (public, auth, admin)
- persona or spec reference
- short purpose note

The index should:

- open directly from `file://` without a dev server
- avoid build steps unless the project already has a frontend toolchain
- distinguish active mockups from deferred or archived mockups
- list reusable component mockups in a separate section at the bottom, since they are not tied to a single route
- include a filter or section jump when the route count is large

**Surface coverage rule:** every route that a user can reach on more than one surface must have a mockup for each surface. If a surface variant does not yet exist, mark it as `[planned]` in the index rather than omitting it. This makes gaps visible and prevents the index from silently misrepresenting coverage.

Recommended files:

```text
sketches/mockups/
|-- index.html
|-- README.md
|-- _route-map.json
|-- _route-map.js
|-- _shared.css
|-- _shared.js
|-- desktop/
|-- tablet/
|-- mobile/
|-- customer/
|-- components/
`-- _deferred/
```

Use `_route-map.json` as the source of truth when possible. Generate `_route-map.js` from it if `index.html` needs a browser-friendly script without a server.

Route-map entry template:

```json
{
  "id": "sales-order-new",
  "title": "New Sales Order",
  "surface": "desktop",
  "path": "desktop/sales-order-new.html",
  "route": "/sales/orders/new",
  "spec": "D8",
  "persona": "sales_clerk",
  "status": "active"
}
```

Index quality checks:

- every route-map path exists as a file
- every active mockup is listed in the route map
- every listed mockup has a valid surface label
- routes accessible on multiple surfaces show all surface variants in one group
- missing surface variants are marked `[planned]`, not silently omitted
- deferred mockups live under `_deferred/` or are marked `status: deferred`
- component mockups are listed separately from full-screen routes
- links work from the index page

Outputs:

- mockup README
- screen files
- browser index page organized by route
- reusable components section
- route map

### Phase 7: Schema Proposals and Consolidation

Objective: let flow work discover data needs without immediately corrupting the canonical schema.

When a flow needs new data, create a proposal file:

```text
sketches/flows/<workflow>-schema-proposals.md
```

Each proposal should include:

- origin flow
- problem solved
- proposed models or fields
- enums
- indexes
- migrations
- owner spec
- conflicts
- priority
- open questions

Use a `schema-roundup.md` to merge proposals:

- deduplicate models
- resolve naming conflicts
- choose canonical owners
- bucket changes by priority
- identify destructive migrations
- list open product decisions
- define merge order

Priority buckets:

- P0: required before implementation
- P1: needed for first release feature parity
- P2: optimization or future hardening
- Deferred: explicitly not in current release

Outputs:

- proposal files
- roundup file
- canonical schema edits after approval
- decision-log entries for accepted or rejected changes

### Phase 8: Golden Paths

Objective: verify that major real-world outcomes work across multiple specs.

A golden path follows one object, customer, order, incident, payment, file, or workflow through the whole system.

Each golden path should include:

- cast and sample data
- ASCII or tabular timeline
- surfaces used
- data chain
- audit chain
- notifications
- failure branches
- time budget
- acceptance criteria

Pick 3 to 7 golden paths that stress the system. Good golden paths cross boundaries.

Examples:

- trial signup to paid subscription
- order placed to delivered to invoice paid
- security incident detected to account locked to audit exported
- document uploaded to parsed to approved to downstream record created
- failed integration to admin repair to backlog replayed

Outputs:

- `golden-paths.md`
- missing spec updates
- missing schema updates
- missing admin tools

### Phase 9: Failure Modes

Objective: design recovery before implementation.

Create `failure-modes.md` for cross-cutting failures:

- network outage
- background job failure
- external API outage
- partial import
- permission misconfiguration
- duplicate submission
- stale data
- failed payment
- failed notification
- locked account
- bad deployment or feature flag rollback
- unavailable device
- user abandons workflow midway
- concurrent edits

For each failure, define:

- detection
- user-facing state
- automatic system action
- manual recovery action
- audit event
- notification
- data integrity guarantee
- test case

Outputs:

- failure-mode catalog
- admin recovery requirements
- test scenarios

### Phase 10: Audit and Reconciliation

Objective: catch drift across the corpus.

Run audits after every major pass. Audits can be done by humans, AI agents, scripts, or all three.

Audit categories:

- foundation audit: do domain specs contradict foundation specs?
- schema audit: do specs reference fields that do not exist?
- permission audit: are all permission keys defined and consistently named?
- audit-event audit: are event names defined and emitted consistently?
- API audit: duplicate operations, inconsistent naming, missing authorization
- UI audit: routes without specs, specs without routes, mockups without links
- persona audit: workflows assigned to missing or wrong roles
- flow audit: narratives stale against locked decisions
- decision-log audit: decisions not reflected in specs
- deferred-work audit: deferred items still referenced as active
- implementation-readiness audit: can engineers build from this without chat history?

Finding levels:

- BLOCKER: implementation would fail or data model is contradictory
- MAJOR: behavior is ambiguous or cross-spec drift exists
- MINOR: stale wording, missing citation, incomplete edge case
- NIT: formatting, typo, link cleanup

Audit output template:

```markdown
# Audit - <Scope>

## Summary

- BLOCKER: 0
- MAJOR: 0
- MINOR: 0
- NIT: 0

## Findings

### BLOCKER-1: <Title>

**Files:** `<file>`, `<file>`
**Problem:** <What is inconsistent or broken.>
**Impact:** <Why it matters.>
**Fix:** <Specific change.>
```

After fixes:

- update affected specs
- update decision log
- update schema or generated docs
- update index if status or dependencies changed
- rerun audits or lint

### Phase 11: Implementation Handoff

Objective: produce buildable slices.

Before starting a slice, confirm:

- spec exists and is current
- decisions are locked
- data model fields exist or are explicitly part of the slice
- API operations are named
- permissions are defined
- audit events are defined
- UI routes and states are known
- failure modes are known
- acceptance criteria are testable
- migration order is clear
- dependencies are known

Handoff template:

```markdown
# Implementation Slice: <Name>

## Goal

<User-visible outcome.>

## Source Docs

- Spec: `docs/<project>/specs/D1-...md`
- Flow: `docs/<project>/sketches/flows/...md`
- Mockups: `docs/<project>/sketches/mockups/...`
- Schema: `docs/<project>/schema...`

## Build Order

1. Migration or model update
2. Server validation and service logic
3. API operations
4. Permissions and audit events
5. UI route and components
6. Tests
7. Seed/demo data

## Acceptance Criteria

- <Criterion>

## Risks

- <Risk and mitigation>
```

## Decision Workflow for AI Assistants

When the user asks a product or technical question:

1. Identify which spec owns the concept.
2. Check whether a decision already exists.
3. If undecided, present the smallest set of viable options.
4. Recommend one option with rationale.
5. When the user chooses, update the decision log.
6. Update all affected specs and sketches.
7. Add audit notes if the decision may create drift.

Option format:

```markdown
## Decision Needed: <Topic>

**Context:** <Why this matters.>

**Option A:** <Choice>
- Pros:
- Cons:
- Best when:

**Option B:** <Choice>
- Pros:
- Cons:
- Best when:

**Recommendation:** <A or B, with reason.>

**Files affected if locked:** <list>
```

Avoid asking the user questions that can be answered from existing docs. Ask only when the answer changes product behavior, implementation cost, compliance posture, or user workflow.

## Naming and Versioning

Use stable names.

- Spec IDs do not change after creation unless the index is updated everywhere.
- Route names should be consistent with product navigation.
- Model names should be singular nouns.
- Enum values should be explicit and stable.
- Permission keys should follow one convention, such as `domain.entity.action`.
- Audit events should be past-tense or event-style constants, such as `ORDER_CONFIRMED`.
- Feature flags should use namespaced keys, such as `checkout.express.enabled`.
- System settings should use namespaced keys, such as `notifications.quietHours.enabled`.

Version rules:

- Major: breaking behavior or data contract change
- Minor: additive behavior or fields
- Patch: wording, examples, clarifications

When a spec changes version, note why.

## Open Questions

Every unresolved question should be visible in the owning spec.

Format:

```markdown
- [OPEN] Should <question>?
  - Option A: <choice>
  - Option B: <choice>
  - Recommendation: <choice>
  - Blocks: <implementation slice or spec>
```

When answered:

1. remove or mark resolved in the spec
2. add a decision-log entry
3. update dependent docs

Do not leave resolved questions floating as stale TODOs.

## Deferred Work

Deferred work is not deleted. It is moved or marked so future agents know it was considered.

A deferral note should include:

- what is deferred
- why it is deferred
- what replaces it for now
- what would cause it to return
- which specs or mockups are affected

Template:

```markdown
> Deferred for v1: <feature>. Reason: <why>. Current source of truth or workaround: <what>. Re-entry condition: <when to revisit>.
```

## Permissions and Audit Checklist

For every write action:

- permission key exists
- permission key is assigned to expected roles
- unauthorized behavior is defined
- PIN or approval requirement is defined if risky
- audit event exists
- audit event includes before and after when applicable
- actor is captured
- entity is captured
- reason is captured for destructive actions
- bulk actions produce batch-level traceability
- revert or compensation path is defined where possible

## API Checklist

For each API operation:

- operation name follows project convention
- input type is explicit
- output type is explicit
- authorization is defined
- validation is defined
- idempotency is defined for retries where needed
- pagination is defined for lists
- sorting and filtering are defined
- error codes are defined
- audit events are emitted where needed
- rate limits are defined for public or partner APIs
- webhook events are defined if downstream systems need updates

## UI Checklist

For each screen:

- primary persona is known
- primary task is obvious
- route is defined
- data loading state exists
- empty state exists
- error state exists
- permission-denied state exists
- destructive actions require confirmation
- saved state or audit link is visible when useful
- keyboard behavior is defined for desktop-heavy flows
- touch target requirements are defined for tablet and mobile
- responsive behavior is defined
- offline or degraded behavior is defined if relevant
- screen links back to spec or story in mockup metadata if possible

## Data Model Checklist

For each model or table:

- owner spec is known
- lifecycle states are defined
- legal state transitions are defined
- required fields are justified
- nullable fields are justified
- unique constraints are defined
- indexes are defined from query patterns
- retention rules are defined
- soft delete, void, archive, or immutable history behavior is defined
- multi-tenant behavior is defined if applicable
- migration impact is known
- seed or sample data exists for important flows

## Background Job Checklist

For each job:

- trigger is defined
- schedule or event source is defined
- retry policy is defined
- idempotency key is defined
- failure state is visible
- admin recovery path exists
- audit or job log exists
- notifications exist for human-required failures
- concurrency behavior is defined
- backfill or replay behavior is defined

## Integration Checklist

For each external system:

- source of truth is defined
- auth method is defined
- credential storage is defined
- refresh and rotation are defined
- timeout and retry policy are defined
- rate limits are defined
- mapping tables are defined
- manual override path is defined
- reconciliation path is defined
- outage behavior is defined
- admin diagnostics are defined
- audit events are defined

## Implementation Readiness Gate

Do not start implementation for a slice until these are true:

- The owner spec has no blocking open questions.
- The decision log contains all user-approved choices affecting the slice.
- The data model has either canonical fields or approved migration tasks.
- Permissions and audit events are named.
- At least one happy path and one failure path are documented.
- UI route and primary states are documented.
- Tests can be written from acceptance criteria.
- Dependencies on deferred features are removed or replaced.

## AI Agent Operating Instructions

When using this outline as an AI assistant:

1. Read `README.md`, `SPEC-INDEX.md`, and `decisions-log.md` first.
2. Read foundation specs before domain specs.
3. Read the relevant persona, surface, flow, and mockup docs before proposing implementation.
4. Prefer updating existing owner docs over creating duplicate explanations.
5. When adding a new concept, assign an owner immediately.
6. When changing a locked decision, add a superseding decision rather than silently editing history.
7. When you find drift, fix the smallest set of files that restores one source of truth.
8. Keep citations to affected files in decision-log entries.
9. After major edits, audit for stale references.
10. Before coding, produce or verify an implementation slice.

## Common Anti-Patterns

- Building from chat history instead of docs.
- Letting each feature define its own permissions or audit format.
- Repeating full model definitions across specs.
- Writing happy paths without failure recovery.
- Creating mockups that do not map to routes or specs.
- Treating admin tools as optional.
- Deferring a feature without documenting what replaces it.
- Updating a flow after a decision but forgetting the spec.
- Updating the schema without updating generated docs or references.
- Leaving open questions hidden in chat.
- Starting implementation before data, permissions, and workflow states agree.

## Minimal Version for Small Projects

If the project is small, use one `docs/build-plan.md` with these sections:

- Purpose
- Users
- Decisions
- Data model
- Workflows
- Screens
- API
- Permissions
- Edge cases
- Open questions
- Implementation slices

Still keep a decision log section. Even small projects drift when decisions live only in conversation.

## Final Output Expected Before Build

Before implementation begins, the planning corpus should answer:

- What are we building?
- Who uses it?
- What does each user do?
- What screens exist?
- What data exists?
- What states can the data be in?
- What actions can change those states?
- Who is allowed to perform those actions?
- What is audited?
- What happens when things fail?
- What integrations exist?
- What is deferred?
- What should be built first?
- How will we know it works?
