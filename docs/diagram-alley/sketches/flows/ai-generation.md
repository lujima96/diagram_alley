---
phase: 4
artifact: flow — AI generation
status: complete
depends-on: [D1, F1, F2, F3, F4, personas]
---

# Flow: AI Generation

End-to-end narrative for generating a new diagram from a prompt using an AI provider. Cast: **Priya**, a solo developer (P1) who just signed up, connected her OpenAI-compatible key, and wants to document the architecture of her side project.

---

## Cast and Context

| | |
|-|-|
| **User** | Priya, P1 — Solo Developer |
| **Session** | Day 2 of free trial (DEC-010); BYOK OpenAI-compatible provider configured |
| **Goal** | Generate an architecture diagram for a FastAPI + React + Postgres project and copy the ASCII to her README |
| **Provider** | BYOK OpenAI-compatible (GPT-4o), set as default |

---

## Pre-condition

Priya has already completed provider setup under Settings → Providers: entered a base URL, API key, and model name, and confirmed "Connection OK." Her default provider is set. She is on the `/dashboard`.

---

## Step-by-Step Flow

### 1. Initiate

Priya clicks **New Diagram** in the sidebar. The app navigates to `/workspace/new`. The prompt panel is focused and example prompts appear in a suggestion list. The visual grid editor shows an empty canvas with a "start with a prompt or blank diagram" hint.

### 2. Set Diagram Type

She selects **Architecture** from the diagram type selector in the prompt panel. The prompt placeholder updates to "Describe the system you want to diagram…"

### 3. Enter Prompt

She types:

> "FastAPI backend, React frontend, Postgres database, Redis cache between API and Postgres. Frontend talks to API over HTTP. API talks to Redis and Postgres."

Model selector shows her configured provider (GPT-4o via BYOK). She clicks **Generate**.

### 4. Request Processing (Backend — D1)

The app posts to `POST /api/v1/diagrams/generate` with:

```json
{
  "prompt": "FastAPI backend, React frontend, Postgres database, Redis cache between API and Postgres. Frontend talks to API over HTTP. API talks to Redis and Postgres.",
  "diagram_type": "architecture",
  "provider_id": "provider_priya_openai"
}
```

The backend:
1. Constructs a diagram-type-specific system prompt (D1 §prompt templates — architecture prompt asks for nodes, edges, direction, optional groups, valid node IDs, clear labels).
2. Sends the prompt + system prompt to the BYOK provider adapter.
3. Receives JSON from the model.
4. Runs the **validation pass** (F2): checks node IDs are unique, edges reference existing nodes, all required fields are present.
5. Runs the **repair pass** (F2): no issues found in this case. If there had been (e.g., a missing node ID), repair would fill defaults and attach a warning.
6. Runs the **ASCII renderer** (F2) to produce deterministic text output.
7. Returns the spec, ASCII output, and an empty warnings array.

### 5. Result Renders

The workspace updates:

- The **visual grid editor** renders the diagram as React Flow nodes and edges (DEC-013). Four nodes appear: React Frontend, FastAPI Backend, Redis Cache, Postgres Database. Edges connect them per the spec.
- The **preview panel** (ASCII tab, default) shows:

```
+-------------------+
| React Frontend    |
| frontend          |
+---------+---------+
          |
          | HTTP
          v
+-------------------+
| FastAPI Backend   |
| service           |
+--------+----------+
         |          |
         | cache    | SQLAlchemy
         v          v
+----------+  +-------------------+
| Redis    |  | Postgres Database |
| cache    |  | database          |
+----------+  +-------------------+
```

- No validation warnings are shown. The repair indicator is hidden.
- The inspector panel shows the diagram title ("Untitled Architecture") as editable.

### 6. Review and Minor Edit

Priya notices the diagram title is "Untitled Architecture." She clicks it in the header bar and types "Inventory API Architecture." The title updates in the spec via auto-save (DEC-021 — auto-save flushes to `diagrams.spec_json` but does not create a version snapshot).

She also wants to rename "FastAPI Backend" to "API Service." She double-clicks the node in the visual grid editor. The node label becomes an inline editable field. She types "API Service" and presses Enter. The spec updates and the preview panel re-renders.

### 7. Save (Version Snapshot)

She clicks **Save** in the header bar. The app:

1. Flushes any pending auto-save.
2. Creates a `diagram_versions` snapshot (DEC-020 — initial AI generation snapshot is created here as the first version). The snapshot is labeled "Initial generation."
3. The route changes from `/workspace/new` to `/workspace/da_7f3a19c` (the new diagram ID).
4. The Save button returns to its inactive state.

### 8. Copy ASCII

Priya clicks the **Copy** button in the ASCII preview panel. The ASCII text is copied to the clipboard. She switches to her code editor and pastes it into her README inside a fenced code block.

**Value test met:** "I described an architecture, got a clean structured diagram, and pasted it into my PR in under 2 minutes."

---

## Alternate Path: Generation Fails (Repair Warning)

If the model returns JSON with an edge that references a node ID that does not exist:

1. Validation flags the error.
2. The repair pass removes the invalid edge and adds it to the `warnings` array.
3. The workspace renders the diagram with a **yellow repair banner** at the top of the preview panel: "1 repair was made: edge 'api→db' was removed because node 'db' does not exist. Check your diagram."
4. Priya can accept the diagram as-is or use the inspector to add the missing node manually.
5. The repair warning is response data for the current UI state and may be recorded in audit event detail; it is not stored on the diagram row (DEC-031).

## Alternate Path: Generation Fails Completely (D1 AI failure handling)

If the provider returns an error (rate limit, bad key, timeout):

1. The app shows a toast error: "Generation failed: [provider error message]. Your prompt has been kept."
2. The prompt textarea retains her input.
3. The canvas remains in its previous state (empty for a new diagram).
4. A **Retry** button appears below the error toast.
5. If retry also fails, the error panel offers: "Try a different provider" (links to Settings → Providers) and "Continue manually" (focuses the visual grid editor with an empty canvas).

---

## Spec References

| Behavior | Owner |
|----------|-------|
| Prompt-to-spec flow | D1 |
| Provider routing and BYOK credential use | D1, F3 |
| Validation and repair pass | F2 |
| Deterministic ASCII renderer | F2 |
| Auto-save (live `spec_json`, no snapshot) | D2, DEC-021 |
| Version snapshot on initial save | D3, DEC-020 |
| Workspace layout and panel behavior | F4, D2 |
| AI failure handling | D1 |
