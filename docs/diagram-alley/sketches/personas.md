---
phase: 0
artifact: personas (initial list)
status: stub — full entries written in Phase 3
---

# Diagram Alley — Personas (Initial List)

This is the Phase 0 stub. It lists all personas and their basic profile so they can be referenced in specs before Phase 3 produces full entries.

**Phase 3 will add:** role code, primary surface, goals, key flows, pain points, permissions cluster, specs touched, notification expectations.

---

## V1 Priority Personas

These personas drive all V1 decisions. When a product or UX choice has to be made, optimize for these two first.

### P1 — Solo Developer

**Who:** Individual software developer or small-team dev working on documentation, architecture notes, README files, pull requests, or internal docs.

**Primary surface:** Desktop web app.

**Core need:** Speed and accuracy. Get to a useful diagram fast, export it in a format that can be pasted into source control without cleanup.

**Pain point:** AI chatbot diagrams have broken spacing, lose structure on follow-up edits, and cannot be regenerated reliably.

**Key workflows:**
- Describe an architecture → generate → copy ASCII to README
- Modify existing diagram with AI → review diff → accept or reject
- Export to Mermaid for docs site
- Save diagrams by project, reopen later

**V1 value test:** "I described an architecture, got a clean structured diagram, and pasted it into my PR in under 2 minutes."

---

### P2 — Technical Writer

**Who:** Documentation specialist maintaining docs sites, knowledge bases, user manuals, or internal guides for a product or engineering team.

**Primary surface:** Desktop web app.

**Core need:** Consistency. Diagrams that match the same formatting style across documents and can be regenerated when the underlying system changes.

**Pain point:** Diagrams go out of date. Re-creating them from scratch or asking an AI chatbot each time is inconsistent and time-consuming.

**Key workflows:**
- Start from a template → customize with prompt or form editing → export
- Re-open a saved diagram and update it as the system changes
- Export to multiple formats from one source (ASCII for code docs, PNG for wiki, SVG for design tools)
- Use version history to track diagram changes alongside content changes

**V1 value test:** "I updated a system architecture diagram and re-exported it in three formats without starting from scratch."

---

## Secondary Personas

Not V1 decision drivers, but the product should not actively break for them.

### P3 — AI Power User

**Who:** Someone who already uses AI chatbots to generate diagrams and understands the pain points of fragile, non-editable AI output.

**Core need:** Control. Wants the AI to help, but wants a deterministic renderer to own the layout.

**Key worry:** AI output that silently produces wrong structure or spacing.

**V1 fit:** Natural early adopter once they understand the spec-first approach.

---

### P4 — IT Professional / System Administrator

**Who:** IT or network admin who needs to document infrastructure, network topology, or access control flows.

**Core need:** Clarity. Diagrams that explain how systems connect without requiring heavy design tools.

**V1 note:** Not a V1 priority target. Network diagram support exists but V1 decisions are not optimized for this persona. V1 should not block them, but it will not be tailored to IT-specific workflows.

---

### P5 — Product Manager

**Who:** PM sketching feature flows, user journeys, system handoffs, or MVP scope diagrams.

**Core need:** Speed. Rough diagrams quickly, good enough for stakeholder communication.

**V1 note:** Secondary persona. The product is useful for PMs but V1 prioritizes developer/technical-writer accuracy over PM sketch speed.

---

### P6 — Engineering Lead / Architect

**Who:** Senior developer or tech lead who creates and maintains architecture documentation for a team or project.

**Core need:** Authority. Diagrams that are versioned, shareable, and canonical — not one-off chatbot screenshots.

**Key workflows:**
- Create authoritative architecture diagrams for the team
- Update them as the system evolves
- Export for onboarding docs, PRs, and design reviews

**V1 fit:** Strong use case, especially for version history and multi-diagram project organization. Slightly lower priority than P1/P2 because V1 is single-user.

---

### P7 — Student / Learner

**Who:** Someone learning programming, database design, networking, or system design who wants visual aids.

**Core need:** Understanding. Diagrams that help explain or explore technical concepts.

**V1 note:** Lowest priority. The product is useful for learners but V1 does not optimize for educational or beginner-specific UX.

---

## Persona Priority for V1 Decisions

| Priority | Persona | Optimize V1 for? |
|----------|---------|------------------|
| 1 | P1 — Solo Developer --------| Yes |
| 2 | P2 — Technical Writer ------| Yes |
| 3 | P6 — Engineering Lead | Indirectly (single-user constraint) |
| 4 | P3 — AI Power User | Indirectly |
| 5 | P4 — IT Professional | No (but do not break) |
| 6 | P5 — Product Manager | No |
| 7 | P7 — Student | No |
