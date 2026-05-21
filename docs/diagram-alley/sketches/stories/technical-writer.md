---
phase: 5
artifact: stories — P2 Technical Writer
status: complete
persona: P2
depends-on: [personas, flows/manual-edit.md, flows/ai-generation.md, flows/ascii-export-and-reverse-sync.md, flows/version-history.md]
---

# Stories: Technical Writer (P2)

User story set for **Marcus**, the P2 Technical Writer persona. Stories focus on consistency, multi-format export, template reuse, and long-running documentation maintenance — the patterns that distinguish the technical writer use case from the solo developer's single-session workflow.

For the full persona profile see [sketches/personas.md](../personas.md). For the detailed flow walkthroughs see the flows listed in the depends-on header above.

---

## Vignette

Marcus maintains the architecture documentation for a 20-service platform. Every sprint that adds or removes a service, he has to update four diagrams and re-export them in three formats for two different docs platforms. Before Diagram Alley, this meant re-running a chatbot prompt, getting a different layout every time, and manually reformatting the output. Now he opens each affected diagram, uses AI to apply the sprint's change, reviews the diff, accepts it, saves a new named version ("Sprint 14 — added notification service"), and exports ASCII, PNG, and Mermaid in one session. The docs sites are updated in 20 minutes instead of two hours, and every diagram matches the same formatting style.

---

## Story Set

### Templates and Starting Points

**S-P2-01** — As a technical writer, I want to start from a template that matches the kind of documentation I'm creating, so that I have a correct structure before I add any content.

*Acceptance criteria:*
- The template gallery shows system templates organized by category (architecture, database schema, UI wireframe, etc.).
- Clicking "Use template" opens the workspace with the template's spec pre-loaded as a new unsaved diagram.
- The template is not modified — it remains available for reuse.
- The new diagram from the template is initially titled "[Template name] — copy."

---

**S-P2-02** — As a technical writer, I want to save a customized diagram as a template, so that I can reuse the same structure and style conventions across a whole docs set.

*Acceptance criteria:*
- A "Save as template" action is available from the workspace (diagram menu or export modal).
- The saved template appears in the template gallery under a "My templates" category.
- Using the template forks it into a new diagram — changes to the new diagram do not affect the saved template.
- Templates can be renamed and deleted.

---

### Multi-Format Export

**S-P2-03** — As a technical writer, I want to export the same diagram in multiple formats in a single session, so that I can update ASCII docs, a PNG wiki embed, and a Mermaid docs site from one source.

*Acceptance criteria:*
- The export modal allows switching between formats without closing and reopening.
- Selecting ASCII + Copy, then switching to PNG + Download, then switching to Mermaid + Download all work in sequence without re-rendering from scratch.
- All three exports reflect the current saved spec.

---

**S-P2-04** — As a technical writer, I want the PNG export to be clean enough to embed in a wiki or slide deck without post-processing, so that I don't spend time editing screenshots.

*Acceptance criteria:*
- PNG output has a white or transparent background (configurable).
- All node labels are fully visible — no text clipping.
- Exported PNG matches the visual grid editor's current layout.
- Resolution is sufficient for standard wiki display (≥96 DPI equivalent).

---

**S-P2-05** — As a technical writer, I want to choose between strict ASCII and UTF-8 text output modes, so that I can use Unicode box-drawing characters for rendered docs sites while keeping pure ASCII for code-embedded diagrams.

*Acceptance criteria:*
- The export modal offers Strict ASCII and UTF-8 Text as output mode options for text exports.
- Both modes produce correct output for the current diagram type.
- The mode selection is per-export and does not permanently change the diagram.

---

### Updating Existing Diagrams

**S-P2-06** — As a technical writer, I want to use AI to apply a specific change to an existing diagram without regenerating the whole thing, so that the rest of the diagram's structure stays exactly as I approved it.

*Acceptance criteria:*
- With a saved diagram open, an AI modify action sends the current spec + change request to the provider.
- The response is previewed as a proposed update before being applied.
- Only the described change appears in the diff — the rest of the diagram is unchanged.
- The user can reject the update and the diagram returns to its exact previous state.

---

**S-P2-07** — As a technical writer, I want to add and rename nodes manually without using AI, so that I can make precise targeted edits when the change is too small to warrant a prompt.

*Acceptance criteria:*
- Nodes can be added via the inspector panel's Add Node action.
- Existing node labels can be edited inline on the canvas (double-click) or via the inspector form.
- The spec updates and the preview re-renders after each change.
- No AI call is made during manual editing.

---

### Consistency and Standards

**S-P2-08** — As a technical writer, I want every diagram I create to use the same box style and edge style, so that documentation diagrams look like they came from the same source.

*Acceptance criteria:*
- The renderer applies consistent formatting rules to all diagrams of the same type — box widths calculated from content, labels padded consistently, arrows centered.
- Two diagrams created from the same spec produce identical ASCII output.
- There is no per-diagram style setting that can accidentally produce inconsistent formatting across the docs set (V1 — style customization is V2).

---

**S-P2-09** — As a technical writer, I want to validate a diagram before exporting it, so that I don't publish documentation with a broken diagram.

*Acceptance criteria:*
- Validation runs automatically as edits are made and shows results in the inspector panel.
- Errors (e.g., edge referencing a missing node) are shown with the specific field or element that caused them.
- The export modal warns if there are unresolved validation errors before allowing download.
- Warnings (non-blocking issues) are listed separately from errors.

---

### Version History for Documentation Tracking

**S-P2-10** — As a technical writer, I want to save a named version of a diagram after each significant update, so that I can track how the architecture changed sprint-by-sprint.

*Acceptance criteria:*
- The Save action prompts for an optional change summary.
- Saved version summaries (e.g., "Sprint 14 — added notification service") are shown in the version history drawer.
- The history drawer shows date, label, and a snapshot preview for each version.

---

**S-P2-11** — As a technical writer, I want to preview an older version of a diagram before restoring it, so that I can confirm it is the correct state without committing to the restore.

*Acceptance criteria:*
- Clicking a version in the history drawer shows a preview of that version's rendered ASCII output.
- The preview is read-only and does not affect the current live diagram.
- A "Restore this version" button is available from the preview.
- Confirming restore creates an auto-named pre-restore snapshot before applying.

---

### Projects and Organization

**S-P2-12** — As a technical writer, I want to organize diagrams by documentation set or product, so that all diagrams for a product's docs live in one project.

*Acceptance criteria:*
- Projects can be created, named, and have diagrams added to them.
- The project detail page lists all diagrams with type, last-updated date, and quick-open action.
- Diagrams can be moved between projects or removed from a project.

---

**S-P2-13** — As a technical writer, I want to search and filter my diagram library by type and last-updated date, so that I can quickly find the diagram I need across a large library.

*Acceptance criteria:*
- The library page has a search bar that filters by diagram title.
- Filter options include diagram type (architecture, database schema, etc.) and last-updated range.
- Results update as filters are applied without a full-page reload.

---

### Import

**S-P2-14** — As a technical writer, I want to import an existing Mermaid diagram and convert it into an editable Diagram Alley spec, so that I can migrate diagrams from our old docs without recreating them from scratch.

*Acceptance criteria:*
- The import flow accepts Mermaid flowchart and ER diagram syntax (best-effort, per DEC-006).
- After import, a conversion preview shows the resulting diagram before the user accepts it.
- Any unsupported Mermaid constructs are listed as warnings — not silently dropped.
- The original Mermaid text is preserved (not destroyed) if the user rejects the conversion.
- The imported diagram can be edited in the visual grid editor like any other diagram.

---

### Trial and Billing

**S-P2-15** — As a technical writer evaluating the product, I want the full export capability to be available during the trial, so that I can validate whether the outputs meet our documentation standards before committing to a paid plan.

*Acceptance criteria:*
- All export formats (ASCII, Mermaid, PNG, SVG, JSON, YAML) are available during the free trial (DEC-010).
- Export quality during the trial is identical to Pro.
- No watermarks or trial branding appear in exported files.
