---
phase: 4
artifact: flow — manual edit
status: complete
depends-on: [D2, F1, F2, F4, personas]
---

# Flow: Manual Edit

End-to-end narrative for creating and editing a diagram without AI — starting from a blank canvas or a template, using the visual grid editor and the structured inspector. Cast: **Marcus**, a technical writer (P2) who wants to update a database schema diagram after a sprint added a new table, without regenerating from a prompt.

---

## Cast and Context

| | |
|-|-|
| **User** | Marcus, P2 — Technical Writer |
| **Session** | Active Pro subscriber; diagram "Orders Schema v3" already exists in his "Checkout Redesign" project |
| **Goal** | Add a `discount_codes` table to an existing database schema diagram and re-export it in ASCII and Mermaid formats |
| **Reason for manual edit** | He knows exactly what to add; using AI to modify the whole diagram would risk changing existing table layouts he has already approved |

---

## Pre-condition

Marcus opens `/workspace/db_88c2a41` (the existing "Orders Schema v3" diagram). The visual grid editor shows four tables: `users`, `orders`, `order_items`, `products`. The ASCII preview shows the current schema.

---

## Step-by-Step Flow

### 1. Open the Diagram

The workspace loads with "Orders Schema v3" in the header. The visual grid editor shows the four existing tables rendered as React Flow nodes (DEC-013). The preview panel (ASCII tab) shows the current schema. The diagram is in its last saved state.

### 2. Assess the Existing Layout

Marcus scrolls the canvas to see the full layout. He wants to add `discount_codes` to the right of `orders`. He notes that `orders` has an `id PK` column and he will add a `discount_code_id FK` to `orders`.

### 3. Add a New Table Node

Marcus clicks **Add Node** in the inspector panel (or right-clicks an empty area of the canvas and selects "Add table"). A new table node appears at a default position on the canvas, labeled "New Table."

He immediately opens the **node properties form** in the inspector panel:
- Sets the table name to `discount_codes`.
- Adds columns:
  - `id` — type `UUID`, primary key checked
  - `code` — type `TEXT`
  - `discount_pct` — type `DECIMAL`
  - `expires_at` — type `TIMESTAMPTZ`, nullable
  - `created_at` — type `TIMESTAMPTZ`

The visual grid editor updates live as he adds each column — the table node in the canvas grows to show the column list.

The preview panel re-renders the ASCII output each time he commits a change. Validation runs silently; no errors.

### 4. Position the Node

Marcus drags the `discount_codes` node to the right of `orders` on the canvas. The React Flow grid snaps the node to the grid (F4 — grid coordinate system). The ASCII renderer uses the updated grid position to produce a horizontally-adjacent layout.

### 5. Add a Foreign Key Column to `orders`

Marcus clicks the `orders` node in the canvas to select it. The inspector panel shows the `orders` table's column list. He clicks **Add Column** and adds:
- `discount_code_id` — type `UUID`, nullable, marks as FK

He sets the FK target to `discount_codes.id` using the relationship editor in the inspector.

The visual grid editor adds a dashed edge from the `orders` node to `discount_codes` representing the foreign key relationship. The ASCII preview updates to show the relationship line.

### 6. Validation Check

A green checkmark appears in the inspector panel: "Diagram is valid." No errors or warnings.

If Marcus had accidentally typed a FK target that referenced a non-existent table, the validation panel would show: "Edge from `orders.discount_code_id` references table `discount_code` — no table with that name exists. Did you mean `discount_codes`?"

### 7. Review ASCII Output

Marcus clicks over to the ASCII tab in the preview panel. He reviews the output:

```
+---------------------+         +----------------------------+
| orders              |         | discount_codes             |
+---------------------+         +----------------------------+
| id           PK     | ----->  | id            PK           |
| user_id      FK     |         | code          TEXT         |
| discount_    FK     |         | discount_pct  DECIMAL      |
|   code_id           |         | expires_at    TIMESTAMPTZ  |
| total        DECIMAL|         | created_at    TIMESTAMPTZ  |
| created_at   DATE   |         +----------------------------+
+---------------------+
```

He is satisfied with the layout.

### 8. Undo a Mistake

Marcus accidentally deletes the `expires_at` column while clicking. He presses **Ctrl+Z** (undo). The column reappears. The undo/redo scope is within the current session — it does not create or restore version snapshots (D2 owns undo/redo scope).

### 9. Save (Version Snapshot)

Marcus clicks **Save**. He is prompted for an optional change summary. He types: "Added discount_codes table and FK on orders." The app creates a `diagram_versions` snapshot with this summary. The header bar Save button returns to inactive.

### 10. Export

Marcus opens the **Export** modal from the header bar:

1. He selects **ASCII (.txt)** and clicks **Copy**. The ASCII output is copied to the clipboard for the internal docs wiki.
2. He selects **Mermaid (.mmd)** and clicks **Download**. The file `orders-schema-v3.mmd` downloads.

Both exports complete from the current saved spec without calling AI.

---

## Alternate Path A: Start From a Blank Diagram

If Marcus were starting from scratch instead of editing an existing diagram:

1. He navigates to `/workspace/new` and selects **Database Schema** as the diagram type.
2. The visual grid editor shows an empty canvas. The inspector panel shows a form-based "Add first table" prompt.
3. He adds tables one by one using the inspector form.
4. The flow continues from Step 3 above.

## Alternate Path B: Start From a Template

If Marcus uses a template as a starting point:

1. He browses `/templates`, finds "Blog Schema" (a system template), and clicks **Use Template**.
2. The workspace opens at `/workspace/new` with the template's spec pre-loaded.
3. The visual grid editor shows the template's tables. The diagram is unsaved (title shows "Blog Schema — copy").
4. He edits the tables as needed using the inspector panel (Steps 3–8 above).
5. He saves, which creates the diagram row and the initial version snapshot.

---

## Spec References

| Behavior | Owner |
|----------|-------|
| Visual grid editor interactions (add, delete, drag, select, inline rename) | D2 |
| Database schema node kind and column editor | D2, F2 |
| Grid coordinate system and ASCII position mapping | F4 |
| Validation rules for database diagram | F2 |
| Undo/redo scope (session only, not version snapshot) | D2 |
| Auto-save (live `spec_json`, no snapshot) | D2, DEC-021 |
| Version snapshot on manual Save | D3 |
| Export modal (ASCII copy, Mermaid download) | D4 |
| Template loading and fork | D2 (load), F1 (spec fork) |
