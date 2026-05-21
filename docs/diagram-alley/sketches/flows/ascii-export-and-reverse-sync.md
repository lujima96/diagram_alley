---
phase: 4
artifact: flow — ASCII export and reverse sync
status: complete
depends-on: [D2, D4, F2, F4, personas]
decisions: [DEC-005, DEC-023]
---

# Flow: ASCII Export and Reverse Sync

Narrative covering the ASCII/text output export path and the V1 reverse sync boundary. Two sub-flows are shown: (A) a successful label-change reverse sync, and (B) a structural edit that produces a detached export with a warning.

Cast: **Priya** (P1, Solo Developer) working on her architecture diagram after pasting ASCII into her README and realizing she needs to correct one label.

---

## V1 Boundary (DEC-005)

Reverse sync in V1 is **label changes only**. If a user edits a node or edge label in the ASCII text panel, the app attempts to sync that change back into the structured spec. All other ASCII edits (adding nodes, removing nodes, changing connections, restructuring layout) produce a **Detached Export** with a warning. The spec is not modified for non-label edits.

Text output modes (DEC-023): the app supports **Strict ASCII mode** (printable ASCII only) and **UTF-8 text mode** (Unicode box-drawing characters). The export and reverse sync logic applies to both. Specs must not call UTF-8 text output "ASCII output."

---

## Cast and Context

| | |
|-|-|
| **User** | Priya, P1 — Solo Developer |
| **Session** | Active trial; working on "Inventory API Architecture" from the AI generation flow |
| **Goal (Sub-flow A)** | She pasted the ASCII into her README, then realized the node label "API Service" should be "Inventory API" — she wants to fix it directly in the ASCII panel and sync the change back |
| **Goal (Sub-flow B)** | She tries to add a new node by typing it into the ASCII panel, triggering the detached export warning |

---

## Sub-flow A: Label Change — Successful Reverse Sync

### 1. Open ASCII Panel in Edit Mode

Priya is in the workspace at `/workspace/da_7f3a19c`. The preview panel shows the ASCII tab with the current output. She clicks **Edit ASCII** (a button that changes the ASCII panel from read-only to an editable textarea).

The panel header shows: "Editing ASCII — label changes will sync back to the diagram. Other edits will create a detached export."

### 2. Make a Label Edit

The current ASCII contains:

```
+-------------------+
| API Service       |
| service           |
+-------------------+
```

Priya changes "API Service" to "Inventory API":

```
+-------------------+
| Inventory API     |
| service           |
+-------------------+
```

She clicks **Apply**.

### 3. Reverse Sync Runs

The backend receives the edited ASCII text. The reverse sync parser (F2):

1. Compares the new ASCII against the last rendered output from the current spec.
2. Identifies a changed label: the node with ID `api_service` had label "API Service" and now has label "Inventory API."
3. Determines this is a **label-only change** — box structure, edges, and all other content are unchanged.
4. Updates `spec_json`: sets `nodes[api_service].label = "Inventory API"`.
5. Re-renders the ASCII and visual outputs from the updated spec.

### 4. Result

- The preview panel re-renders with the updated label.
- The visual grid editor node label updates to "Inventory API."
- A green success indicator appears briefly: "Label synced to diagram."
- The diagram is marked dirty (auto-save will flush the change to `spec_json`).
- No detached export state is set.

Priya clicks **Save** to create a version snapshot.

---

## Sub-flow B: Structural Edit — Detached Export Warning

### 1. Open ASCII Panel in Edit Mode

Priya clicks **Edit ASCII** again. She decides to try adding a new "Load Balancer" box by typing it into the ASCII text:

```
+-------------------+
| Load Balancer     |
| gateway           |
+--------+----------+
         |
         v
+-------------------+
| Inventory API     |
| service           |
+-------------------+
```

She clicks **Apply**.

### 2. Reverse Sync Detects a Structural Change

The reverse sync parser compares the new ASCII against the last rendered spec output. It detects:

- A new box labeled "Load Balancer" that has no matching node in the current spec.
- A new edge connecting it to the existing "Inventory API" node.

This is not a label-only change. It is a structural modification (new node, new edge). This is outside the V1 reverse sync scope (DEC-005).

### 3. Detached Export Warning Appears

The app does **not** update the spec. Instead:

- The ASCII panel content is accepted as an **edited text snapshot** only.
- A yellow **Detached Export** banner appears in the preview panel:

> "Your ASCII has been edited in a way that can't be synced back to the diagram spec (structural changes — new nodes or edges are V2). Your ASCII text is saved as a detached export and will not update if you edit the diagram. To keep your changes, download this ASCII. To discard it and return to the spec-rendered output, click Reset."

- The **Export** button in the preview panel now exports the detached ASCII text (not the spec-rendered output).
- The visual grid editor is unaffected — it still shows the original 4-node diagram.
- A "Detached" badge appears on the ASCII tab.

### 4. Priya's Options

**Option A — Keep detached export:** She downloads the detached ASCII as a `.txt` file. The detached state remains until she clicks Reset or edits the diagram through the spec.

**Option B — Reset to spec output:** She clicks **Reset**. The ASCII panel discards the detached text and re-renders from the current spec. The detached badge disappears.

**Option C — Add the node through the editor:** She clicks Reset, then uses the visual grid editor to add the "Load Balancer" node properly. The spec updates, the ASCII re-renders deterministically with the new node in the correct position, and no detached state is created.

Priya chooses Option C — she wants the spec to stay in sync.

---

## Export Format: Strict ASCII vs. UTF-8 Text (DEC-023)

When Priya opens the Export modal (D4), she can choose the text output mode:

| Mode | Characters used | Best for |
|------|-----------------|---------|
| Strict ASCII | `+`, `-`, `|`, `>`, `v`, `^`, printable ASCII only | README files, code comments, terminals with limited encoding |
| UTF-8 Text | `┌`, `─`, `│`, `└`, `├`, `→`, Unicode box-drawing | Docs sites, Markdown renderers that support Unicode, file trees |

The export mode is a per-export setting (D4), not a permanent diagram property. It defaults to Strict ASCII. The selection does not affect the spec or the visual grid editor.

The ASCII panel in the workspace shows the current mode's output. Both modes go through the same reverse sync boundary rules — editing text in the UTF-8 text panel also supports label-change sync only.

---

## Spec References

| Behavior | Owner |
|----------|-------|
| Reverse sync scope — label changes only | F2, DEC-005 |
| Detached export state definition and rendering | F2 |
| Detached export warning UX and reset action | D2 |
| Strict ASCII vs. UTF-8 text output modes | F2, DEC-023 |
| Export modal format selector | D4 |
| ASCII renderer determinism contract | F2 |
| Auto-save (live spec_json) | D2, DEC-021 |
