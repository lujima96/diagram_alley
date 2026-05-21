---
phase: 4
artifact: flow — sharing
status: complete
depends-on: [D6, F1, F3, F4, personas]
---

# Flow: Sharing and Publishing

Narrative covering public share link creation, the share view experience, and share link management. Cast: **Rafael** (P6, Engineering Lead) who wants to share a read-only architecture diagram with his team for a design review without requiring them to create accounts.

---

## Cast and Context

| | |
|-|-|
| **User** | Rafael, P6 — Engineering Lead |
| **Session** | Active Pro subscriber; "Payments Service Architecture" diagram is saved and in a clean state |
| **Goal** | Create a public share link for the diagram, send it to three colleagues who don't have Diagram Alley accounts, and be able to revoke the link after the review |
| **Viewers** | Three colleagues on mobile and desktop — no accounts required |

---

## Pre-condition

Rafael has opened `/workspace/da_5d1f8b2` ("Payments Service Architecture"). The diagram has been saved and has version history. He is on the workspace page.

---

## Step-by-Step Flow: Creating a Share Link

### 1. Open Share Settings

Rafael clicks the **Share** button in the header bar. A share settings panel (modal or drawer) opens.

Current state: "This diagram is private." No active share links.

### 2. Create a Share Link

Rafael clicks **Create share link**. The backend:

1. Creates a `diagram_shares` row with a randomly generated `share_token` (URL-safe random string, e.g. `shr_4xK9p2mQr`).
2. Sets `is_active = true`, `access_level = 'view_only'`, `created_at = now()`.
3. The share link is: `https://diagramalley.com/share/shr_4xK9p2mQr`

The share settings panel now shows:

- The full share URL with a **Copy Link** button.
- Share status: "Anyone with the link can view this diagram (read-only)."
- A **Revoke link** button.
- Creation date.

### 3. Send the Link

Rafael clicks **Copy Link**. He pastes the URL into a Slack message to his three colleagues.

---

## Viewer Experience: Desktop

One colleague, Sam, opens the link on her laptop. She is not logged into Diagram Alley.

The `/share/shr_4xK9p2mQr` route renders the **share view**:

- App header shows the Diagram Alley logo and the diagram title ("Payments Service Architecture") — no sidebar, no nav, no account menu.
- Tabs: **ASCII** (default) | **Visual** | **Mermaid** | **JSON**
- ASCII tab shows the current rendered ASCII output (read-only monospace block with horizontal scroll).
- Visual tab shows the React Flow read-only canvas. She can zoom and pan; she cannot select, move, or edit nodes.
- A "**View in Diagram Alley**" call-to-action appears at the bottom of the share view (not intrusive; links to `/register`).

She reviews the diagram and adds a comment in Slack. No account required.

---

## Viewer Experience: Mobile

Another colleague, Tomas, opens the link on his phone. He gets the mobile share view (per the mobile surface spec):

- The ASCII tab is shown by default (best for mobile — no zoom required for text).
- Visual tab is available; pinch-to-zoom works.
- A **Copy ASCII** button is shown at the bottom of the screen.
- Download buttons for PNG/SVG are not shown on mobile (mobile browser limitations; V2 scope).

---

## Viewer Experience: Expired or Revoked Link

Rafael revokes the link after the design review (Step 4 below). If Tomas tries to open the link again:

The share view shows: "This link has expired or is no longer available." No other information is shown (do not distinguish revoked from never-valid for security — D6).

---

## Step-by-Step Flow: Revoking a Share Link

### 4. Revoke After the Review

Two days later, Rafael wants to revoke the link before sharing a revised version.

He opens the workspace, clicks **Share**, and sees the active share link listed. He clicks **Revoke link**. A confirmation dialog:

> "Revoke this share link? Anyone with this link will no longer be able to view the diagram."

He confirms. The backend sets `is_active = false` on the `diagram_shares` row. The link immediately returns a 404-equivalent share view message.

### 5. Create a New Link for the Next Review

Rafael makes changes to the diagram (adds two new nodes from the design review feedback), saves a new version, and clicks **Create share link** again. A new token is generated (`shr_9rWpL7nXt`). He sends the new link.

---

## Share Link and Diagram Version

Share links always show the **current live state** of the diagram — the live `spec_json`, not a specific version snapshot. If Rafael edits the diagram and saves, the share link will show the updated diagram.

This is intentional for V1: share links are living views of the current diagram, not point-in-time snapshots. Point-in-time sharing (share a specific version) is a V2 consideration.

If viewers need a stable snapshot, Rafael should export PNG or ASCII and attach it directly.

---

## Trial-Expired Users and Sharing (DEC-022)

Rafael's trial expires while a share link is active:

- The share link remains accessible to viewers (D6 — share viewing is not gated behind an active subscription).
- Rafael can still manage share links (create, revoke) from the workspace.
- He cannot edit the diagram or create new diagrams until he upgrades.

Per DEC-022: expired trial users can view, export, and manage share links. They cannot edit `spec_json`, use AI, or create new diagrams.

---

## Permission Rules

| Action | Required permission | Notes |
|--------|---------------------|-------|
| Create share link | `diagram.share.create` | Owner of the diagram only (V1 single-user) |
| Revoke share link | `diagram.share.delete` | Owner of the diagram only |
| View via share link | None (public route) | No auth required |
| View diagram share settings | `diagram.share.read` | Owner only |

---

## What Share View Does Not Show

- Edit button or any editing controls
- Version history
- AI generation panel
- Inspector panel
- Pricing or upgrade banners (keep share view clean and professional)
- The viewer's identity or any tracking of who viewed the link (V1; analytics is V2)

---

## Spec References

| Behavior | Owner |
|----------|-------|
| Share token data model (`diagram_shares` table) | F1 |
| Share link creation, revocation, and API contract | D6 |
| Share view route and auth rules | F4, D6 |
| Mobile share view layout | sketches/surfaces/mobile.md |
| Trial-expired user share access | F3, DEC-022 |
| Share view showing current live state (not versioned) | D6 |
