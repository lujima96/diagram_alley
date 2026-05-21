---
phase: 3
artifact: surface — mobile
status: complete
depends-on: [F4, D6]
---

# Surface: Mobile

V1 mobile surface definition. V1 mobile scope is **read-only share viewing only**. No diagram creation, editing, or AI generation on mobile in V1.

Source of truth for the mobile scope decision: F4 §1 (Surface Scope table).

---

## V1 Mobile Scope

| Category | In scope | Out of scope |
|----------|----------|-------------|
| Public share link viewing (`/share/:shareToken`) | Yes | — |
| Read-only diagram rendering (ASCII, visual, Mermaid) | Yes | — |
| Copy ASCII output from share view | Yes | — |
| New diagram creation | No | Deferred to V2 |
| Diagram editing (visual, structured, JSON/YAML) | No | Deferred to V2 |
| AI generation | No | Deferred to V2 |
| Dashboard / Library / Projects | No — "use desktop" banner shown to authenticated mobile users | V2 |
| Settings / Billing | No — "use desktop" banner | V2 |
| Export downloads (PNG, SVG, file types) | No — mobile browser limitations; copy ASCII only | V2 |

Authenticated users who visit any authenticated route on a mobile device see a banner: "Diagram Alley is optimized for desktop. Open this page on a wider screen to create and edit diagrams." The banner does not block the page, but most functionality is hidden or disabled.

---

## Device Assumptions

| Property | Value |
|----------|-------|
| Viewport | < 768px width |
| Input | Touch; no physical keyboard assumed |
| Connection | Online required to load share view data |
| Browser support | Mobile Safari (iOS 16+), Chrome for Android (current) |
| Share view orientation | Portrait primary; landscape renders correctly but is not optimized |

---

## Share View (`/share/:shareToken`)

This is the only fully supported mobile route in V1.

**Purpose:** Allow a diagram to be shared with someone (team member, stakeholder, reviewer) who only has a phone or tablet. The viewer does not need an account.

**Source:** D6 (sharing and publishing spec), F4 §2 (route map)

### Layout

```
+--------------------------------+
| [Diagram Alley logo]           |
| diagram title                  |
+--------------------------------+
| Tabs: ASCII  |  Visual  |  ...  |
+--------------------------------+
|                                |
|   [Active tab content]         |
|                                |
|   ASCII: scrollable monospace  |
|   Visual: React Flow read-only |
|                                |
+--------------------------------+
| [Copy ASCII]  [View on desktop]|
+--------------------------------+
```

- Tabs shown depend on what outputs are available for the shared diagram.
- ASCII tab is the default (most readable on mobile without zoom).
- Visual tab shows a React Flow read-only render; pinch-to-zoom is supported.
- The "View on desktop" link copies the share URL for sending to a desktop browser.

### Behaviors

| Behavior | Detail |
|----------|--------|
| Expired share token | Show "This link has expired or is no longer available." No redirect. |
| Invalid share token | Same message as expired. Do not distinguish valid-but-expired from never-valid (security). |
| Diagram load error | "Could not load diagram. Try refreshing." with a retry button. |
| Long ASCII output | Horizontal scroll within the monospace block; scroll hint text on first load. |
| Copy ASCII button | Copies ASCII text to clipboard; shows "Copied!" toast for 2 seconds. |

### What is not shown in share view

- Edit button (share view is read-only — see D6)
- Export download buttons (mobile browser inconsistency; copy ASCII is the mobile path)
- Version history
- Prompt / AI panel
- Inspector panel
- Any pricing or upgrade CTAs on public share links (keep share view clean)

---

## Authenticated Mobile Users

Authenticated users who visit the app on mobile are not locked out, but they see degraded UI:

- **Any authenticated route except `/share/:shareToken`**: full-width banner at top of page — "Diagram Alley works best on a desktop browser. Some features are not available on this device."
- Navigation sidebar is hidden; replaced by a minimal header with account menu and logo.
- Dashboard page renders a simplified read-only recent-diagrams list (diagram titles only, no previews). Tapping a diagram opens its share view equivalent (read-only rendering).
- Settings/Billing pages render their content without the sidebar; forms are usable for account and billing management tasks.

The goal is that a user who receives a share link or needs to check their billing on mobile is not completely blocked, but creation and editing are explicitly deferred to desktop.

---

## V2 Mobile Considerations (Out of Scope for V1)

These are noted here so Phase 6 mockups do not need to anticipate them, but they are on the radar for V2:

- Mobile-optimized prompt entry for quick diagram generation
- Simplified mobile editor for minor label edits
- Download PNG/SVG from mobile share view
- PWA (add to home screen) support
- Offline diagram viewing for saved diagrams
