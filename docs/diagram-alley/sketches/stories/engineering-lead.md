---
phase: 5
artifact: stories — P6 Engineering Lead
status: complete
persona: P6
depends-on: [personas, flows/version-history.md, flows/sharing.md, flows/ai-generation.md, flows/manual-edit.md]
---

# Stories: Engineering Lead / Architect (P6)

User story set for **Rafael**, the P6 Engineering Lead persona. Stories focus on authoritative versioned documentation, multi-diagram project organization, team sharing, and the long-lived lifecycle of architecture diagrams across a product's evolution.

Rafael is not a V1 decision driver — P1 and P2 take priority when there are tradeoffs. However, his use cases are a natural extension of P1/P2 workflows and should work well without any V1 compromises. His team-sharing and collaboration needs (shared projects, comments, concurrent editing) are V2 features and are excluded from this story set.

For the full persona profile see [sketches/personas.md](../personas.md). For the detailed flow walkthroughs see the flows listed in the depends-on header above.

---

## Vignette

Rafael maintains five architecture diagrams for the platform — one per service cluster. Before each design review, he updates the relevant diagram using AI for the structural changes and the manual editor for label and layout refinements, saves a named version, and sends a share link to the team. During the review, someone asks what the diagram looked like before last quarter's refactor. He opens version history, shows the pre-refactor version on his screen, and confirms the team's mental model. After the review, he regenerates the exports and commits the ASCII versions alongside the PR description. Everything is versioned and reproducible.

---

## Story Set

### Multi-Diagram Project Organization

**S-P6-01** — As an engineering lead, I want to organize all architecture diagrams for a product in a single project, so that new team members can find all system documentation in one place.

*Acceptance criteria:*
- Projects can contain multiple diagrams of different types (architecture, database schema, flowchart, etc.).
- The project detail page lists all diagrams with type, last-updated date, and a preview thumbnail.
- Diagrams can be sorted by type, title, or last-updated date.
- New diagrams created from within a project are automatically assigned to that project.

---

**S-P6-02** — As an engineering lead, I want to add a diagram to a project after it was created, so that I can organize diagrams that were originally made without a project context.

*Acceptance criteria:*
- A "Move to project" or "Add to project" action is available from the library and from within the workspace.
- Moving a diagram to a project does not change its content, version history, or share links.

---

**S-P6-03** — As an engineering lead, I want to give each diagram a clear title and an optional description, so that the project view is self-explanatory without opening every diagram.

*Acceptance criteria:*
- Diagram titles are editable from the header bar in the workspace and from the library card.
- An optional description field is available in diagram settings (not prominently surfaced during editing — accessible via diagram settings or inspector).
- The description is shown on the library card on hover or in the project detail view.

---

### Authoritative Version History

**S-P6-04** — As an engineering lead, I want to add a meaningful label to each version snapshot when I save, so that version history reads as a clear changelog of architecture decisions.

*Acceptance criteria:*
- The Save action prompts for an optional change summary before creating the snapshot.
- The version history drawer shows the summary alongside the date and creator.
- If no summary is provided, the snapshot is labeled with the date/time only — no auto-generated label text.

---

**S-P6-05** — As an engineering lead, I want to view a side-by-side comparison of two versions, so that I can explain to the team exactly what changed between a pre-refactor and post-refactor diagram.

*Acceptance criteria:*
- Selecting a version in the history drawer shows it in a side-by-side view next to the current diagram.
- Nodes present in one version but not the other are visually distinguished.
- The comparison is read-only and does not affect the live diagram.

---

**S-P6-06** — As an engineering lead, I want version history to be unlimited for Pro subscribers, so that I never have to choose between keeping old architecture snapshots and creating new ones.

*Acceptance criteria:*
- No version count limit is enforced for Pro subscribers in V1.
- All versions remain accessible in the history drawer regardless of how many exist.

---

### Sharing for Design Reviews

**S-P6-07** — As an engineering lead, I want to share a read-only link to a diagram with my team, so that they can review the current architecture without needing a Diagram Alley account.

*Acceptance criteria:*
- A "Share" action in the workspace creates a public share link.
- The share link opens a read-only view that shows the diagram's current rendered outputs (ASCII, visual, Mermaid tabs).
- No Diagram Alley account is required to view the link.
- The share view shows no editing controls, no version history, and no pricing prompts.

---

**S-P6-08** — As an engineering lead, I want to revoke a share link after a design review, so that outdated versions of the diagram are not circulated after I've made changes.

*Acceptance criteria:*
- The Share settings panel lists all active share links with creation dates.
- Each link has a Revoke action that immediately invalidates it.
- Revoking a link does not affect the diagram itself or its version history.
- After revocation, the link returns a "no longer available" message — not a 404.

---

**S-P6-09** — As an engineering lead, I want share links to always show the current live diagram, so that when I update the diagram after a review, the team automatically sees the updated version on the same link.

*Acceptance criteria:*
- Share links render the live `spec_json` of the diagram, not a frozen version snapshot.
- Saving a new version and then loading the share link shows the updated diagram.
- There is no "publish" step required to push changes to the share link.

---

### Using AI for Architecture Updates

**S-P6-10** — As an engineering lead, I want to use an AI prompt to add a new service to an existing architecture diagram, so that I can apply sprint-by-sprint changes quickly without redrawing the whole diagram.

*Acceptance criteria:*
- The AI modify action sends the current spec and the change request to the configured provider.
- The result is previewed before being applied (not auto-applied).
- Only the described change appears in the update — existing nodes and edges that were not mentioned are unchanged.
- If the AI update changes more than requested, the user can reject it and the diagram is unmodified.

---

**S-P6-11** — As an engineering lead, I want to connect a local Ollama model for architecture diagrams that describe internal infrastructure, so that sensitive architecture details are never sent to a cloud provider.

*Acceptance criteria:*
- Ollama is supported as a provider type in Settings → Providers.
- The workspace provider selector can switch between multiple configured providers per diagram session.
- AI generation and modification work with the Ollama provider without requiring a cloud API key.

---

### Exporting for PRs and Documentation

**S-P6-12** — As an engineering lead, I want to export the architecture diagram as ASCII and include it in a PR description alongside the code changes, so that reviewers understand the system impact without switching to a separate tool.

*Acceptance criteria:*
- ASCII copy is one-click from the workspace or from the share view.
- The ASCII output is clean enough to paste directly into a GitHub PR description inside a fenced code block without adjustment.
- The diagram's title is optionally included in the export as a heading.

---

**S-P6-13** — As an engineering lead, I want to export the diagram spec as JSON alongside the code changes, so that the diagram's source of truth is version-controlled with the system it describes.

*Acceptance criteria:*
- JSON spec export produces a valid F1-schema-compliant file with the `spec_version` field.
- The exported JSON can be re-imported to recreate the diagram exactly.
- The filename defaults to a slug of the diagram title.

---

### Long-Lived Documentation Maintenance

**S-P6-14** — As an engineering lead, I want to reopen a diagram created six months ago and update it with confidence that the structure and version history are intact, so that architecture documentation can be maintained over the full product lifecycle.

*Acceptance criteria:*
- Saved diagrams persist indefinitely for Pro subscribers.
- Reopening a diagram from six months ago shows its last saved state and full version history.
- The spec can be re-rendered from the stored JSON without requiring the original AI prompt.
- No diagram data is lost due to spec format migration — `spec_version` migration is handled transparently (F1, DEC-008).

---

**S-P6-15** — As an engineering lead, I want diagrams to be organized by project so I can archive older projects without deleting their diagrams, so that historical documentation remains accessible even after a project ends.

*Acceptance criteria:*
- Projects can be archived (hidden from the main project list but not deleted).
- Archived project diagrams remain accessible via the library.
- Share links to diagrams in archived projects continue to work.

---

### Onboarding New Team Members (Single-User V1 Constraint)

**S-P6-16** — As an engineering lead, I want to share a read-only link to the onboarding architecture diagram with new hires, so that they can explore the system structure before their first week's setup is complete.

*Acceptance criteria:*
- Share links work for any diagram in any project.
- The share view is accessible without an account.
- The visual tab in the share view allows zoom and pan on a complex multi-node architecture diagram.
- The share view is readable on both desktop and mobile (mobile shows ASCII tab by default per the mobile surface spec).

*V1 note:* Team workspaces, shared editing, and role-based access to diagrams within a team are V2 features. In V1, Rafael sends read-only share links and manages all diagrams himself.
