---
phase: 5
artifact: stories — P1 Solo Developer
status: complete
persona: P1
depends-on: [personas, flows/ai-generation.md, flows/manual-edit.md, flows/ascii-export-and-reverse-sync.md, flows/version-history.md]
---

# Stories: Solo Developer (P1)

User story set for **Priya**, the P1 Solo Developer persona. Stories are organized by workflow area. Each story is followed by the acceptance criteria that would confirm it works from the user's perspective.

For the full persona profile see [sketches/personas.md](../personas.md). For the detailed flow walkthroughs see the flows listed in the depends-on header above.

---

## Vignette

Priya is three weeks into a new contract. Her team lead has asked for an architecture diagram of the new auth service before tomorrow's design review. She opens Diagram Alley, describes the service in a prompt, gets a clean ASCII diagram in 90 seconds, and pastes it into the PR description. The next day, after the review, she reopens the diagram, asks the AI to add the rate-limiter node that the team agreed on, checks the diff, accepts, and saves a new version. The ASCII is already in the PR — she replaces the old block with the new copy. Total time across both sessions: under five minutes.

---

## Story Set

### AI Generation

**S-P1-01** — As a solo developer, I want to describe a system architecture in plain English and receive a clean ASCII diagram, so that I can paste it directly into a README or PR description without manual formatting.

*Acceptance criteria:*
- Submitting a prompt with a model provider configured returns a valid ASCII diagram within a reasonable wait.
- The ASCII output uses consistent box widths, centered labels, and clean edge routing.
- The diagram renders deterministically — submitting the same prompt again produces the same layout structure (though exact AI output may vary, the renderer is consistent for any given spec).
- No raw unchecked LLM output is displayed as the final result.

---

**S-P1-02** — As a solo developer, I want to see a clear repair notice if the AI returned something that needed fixing, so that I know the diagram I'm looking at is reliable.

*Acceptance criteria:*
- If the repair pass modified the spec, a visible warning appears explaining what was changed (e.g., "1 edge removed: referenced node does not exist").
- The warning does not block the user from using or saving the diagram.
- If no repairs were needed, no repair notice appears.

---

**S-P1-03** — As a solo developer, I want to use my own OpenAI-compatible API key so that my architecture diagrams never leave my provider and I control model costs.

*Acceptance criteria:*
- Settings → Providers allows adding an OpenAI-compatible endpoint (base URL, API key, model name).
- A "Test connection" action confirms the key and endpoint work before saving.
- The API key is not shown in the UI after saving (masked).
- All AI generation requests in the workspace use the configured provider.

---

**S-P1-04** — As a solo developer, I want to connect a local Ollama instance as my AI provider, so that I can generate diagrams for private codebases without sending data to a cloud service.

*Acceptance criteria:*
- Settings → Providers supports Ollama as a provider type (base URL + model name, no API key required).
- Connection test confirms the Ollama server is reachable and the model is available.
- AI generation works with an Ollama provider the same as a cloud provider.

---

### Editing

**S-P1-05** — As a solo developer, I want to rename a node label directly on the visual canvas, so that I can fix a label without opening the inspector form.

*Acceptance criteria:*
- Double-clicking a node label in the visual grid editor makes it editable inline.
- Pressing Enter or clicking outside commits the change.
- The spec updates and the preview panel re-renders.

---

**S-P1-06** — As a solo developer, I want to use AI to modify just one part of an existing diagram, so that I can add a new component without regenerating the whole diagram from scratch.

*Acceptance criteria:*
- With a diagram open, entering a change request (e.g., "Add Redis between API and Postgres") sends the current spec + request to the AI provider.
- The result is shown as a proposed update (not silently applied).
- The visual grid editor and preview panel update to show the proposed change before the user confirms.
- The user can accept or reject the change.
- Rejecting the change restores the exact previous state.

---

**S-P1-07** — As a solo developer, I want to undo accidental edits during a session, so that I don't have to restore a version snapshot for small mistakes.

*Acceptance criteria:*
- Ctrl+Z (or Cmd+Z on Mac) undoes the last editor action.
- Undo works for: node move, node delete, label rename, edge add/delete.
- Undo history is within the current session only; it does not create version snapshots.

---

### ASCII Export and Reverse Sync

**S-P1-08** — As a solo developer, I want to copy the ASCII output with one click, so that I can paste it into a README or code comment immediately.

*Acceptance criteria:*
- A visible Copy button is present in the ASCII preview panel.
- Clicking it copies the current ASCII output to the clipboard.
- A brief "Copied!" confirmation appears.

---

**S-P1-09** — As a solo developer, I want to edit a node label directly in the ASCII panel and have it sync back to the diagram spec, so that I can make a quick label fix without switching to the visual editor.

*Acceptance criteria:*
- The ASCII panel has an Edit mode that makes the text editable.
- Editing only a node or edge label and clicking Apply syncs the change back to the spec.
- The visual grid editor updates to reflect the label change.
- A confirmation indicator ("Label synced to diagram") appears briefly.

---

**S-P1-10** — As a solo developer, I want to be clearly warned when an ASCII edit cannot be synced back to the spec, so that I know my diagram and my ASCII are out of sync.

*Acceptance criteria:*
- Any ASCII edit that adds nodes, removes nodes, or changes connections triggers the Detached Export state.
- A visible banner explains that the edit cannot be synced and offers a Reset option.
- The visual grid editor is unaffected by the detached edit.
- The Export button in detached state exports the edited ASCII text, not the spec-rendered output.
- Reset discards the detached edit and returns to spec-rendered output.

---

### Export

**S-P1-11** — As a solo developer, I want to export my diagram as Mermaid so that I can embed it in docs sites that support Mermaid natively.

*Acceptance criteria:*
- The export modal offers Mermaid (.mmd) as a format option.
- The Mermaid output is syntactically valid for the diagram type (architecture as flowchart, database as ER).
- Download produces a file that opens in a Mermaid renderer without errors.

---

**S-P1-12** — As a solo developer, I want to export the diagram spec as JSON so that I can commit it to source control alongside my code.

*Acceptance criteria:*
- The export modal offers JSON (.json) spec export.
- The exported JSON conforms to the F1 schema (includes `spec_version`).
- The exported JSON can be re-imported to recreate the diagram.

---

### Saving and Projects

**S-P1-13** — As a solo developer, I want to organize diagrams by project so that I can find all diagrams for a given codebase in one place.

*Acceptance criteria:*
- A project can be created and named.
- New diagrams can be assigned to a project.
- The project detail page lists all diagrams in the project.
- Diagrams can be moved between projects.

---

**S-P1-14** — As a solo developer, I want my edits to be auto-saved continuously so that I don't lose work if I close the browser tab.

*Acceptance criteria:*
- The live `spec_json` is auto-saved in the background as changes occur (no manual save required for persistence).
- Reopening a diagram shows the last auto-saved state.
- Auto-save does not create version history entries — only the explicit Save action creates a snapshot.

---

### Version History

**S-P1-15** — As a solo developer, I want to restore a previous version of my diagram after an AI update changed more than I intended, so that I can get back to a known-good state quickly.

*Acceptance criteria:*
- The version history drawer shows all snapshots in reverse-chronological order with labels and dates.
- Previewing a version shows its rendered ASCII output before committing to restore.
- Restoring creates an auto-named "Before restore" snapshot of the current state before applying the restore.
- After restore, the workspace shows the restored version's canvas and preview.

---

### Onboarding

**S-P1-16** — As a solo developer starting my trial, I want to connect a model provider and generate my first diagram within the first session, so that I understand the product value before the trial ends.

*Acceptance criteria:*
- The onboarding path surfaces provider setup as step 1 for AI features.
- After provider setup, a "Try your first prompt" prompt/example is offered.
- A new user can go from account creation to a copied ASCII diagram in under 3 minutes without documentation.
