---
phase: 6
artifact: design system
status: complete
applies-to: all mockups, Phase 6 onward
---

# Diagram Alley — Design System

Visual direction for all mockups, and the reference spec for implementation. The aesthetic goal is a **clean, premium, Apple-platform feel**: generous whitespace, layered surfaces, subtle motion, and SF Pro typography. Functional and calm — not flashy, not flat.

---

## Aesthetic Principles

1. **Surface hierarchy over borders.** Depth is communicated through background shade and shadow, not outline borders between regions.
2. **Whitespace is content.** Generous padding makes complex layouts feel approachable. Never cram.
3. **Motion is intentional.** Transitions are fast (150–250ms), easing is ease-out. Nothing bounces. Nothing lingers.
4. **Monospace is special.** Code, ASCII, and spec output live in a dark, slightly warm terminal surface that visually separates "machine output" from "product UI."
5. **One accent.** The brand accent (Diagram Blue) is used for primary actions, focus states, and active nav — not for decoration.

---

## Color Tokens

### Background

| Token | Hex | Usage |
|-------|-----|-------|
| `bg-base` | `#F5F5F7` | App-wide page background (Apple off-white) |
| `bg-surface` | `#FFFFFF` | Cards, panels, modals |
| `bg-elevated` | `#FFFFFF` | Elevated cards (sidebar, dropdowns) — distinguished by shadow |
| `bg-sidebar` | `rgba(255,255,255,0.82)` | Sidebar and navbar, with `backdrop-filter: blur(20px)` |
| `bg-code` | `#1C1C1E` | ASCII output, JSON/YAML editor, monospace blocks |
| `bg-code-muted` | `#2C2C2E` | Secondary areas within dark code surfaces |
| `bg-canvas` | `#FAFAFA` | React Flow canvas background |
| `bg-canvas-grid` | `rgba(0,0,0,0.04)` | Dot grid on canvas |

### Text

| Token | Hex | Usage |
|-------|-----|-------|
| `text-primary` | `#1D1D1F` | Main body text, labels, headings |
| `text-secondary` | `#6E6E73` | Supporting text, metadata, placeholder |
| `text-tertiary` | `#AEAEB2` | Disabled state, subtle hints |
| `text-on-accent` | `#FFFFFF` | Text on accent-colored backgrounds |
| `text-code` | `#E5E5EA` | Text within dark code surfaces |
| `text-code-comment` | `#636366` | Comments, metadata in code surfaces |
| `text-code-key` | `#64D2FF` | JSON keys, spec field names |
| `text-code-string` | `#A8FF60` | JSON string values |
| `text-code-number` | `#FF9F0A` | JSON number values |
| `text-link` | `#0071E3` | Inline links |

### Accent (Diagram Blue)

| Token | Hex | Usage |
|-------|-----|-------|
| `accent` | `#0071E3` | Primary buttons, active nav, focus rings |
| `accent-hover` | `#0077ED` | Hover state for accent elements |
| `accent-muted` | `#EAF2FF` | Accent tint backgrounds (badges, highlights) |
| `accent-text` | `#004FA3` | Accent-colored text on light backgrounds |

### Semantic

| Token | Hex | Usage |
|-------|-----|-------|
| `success` | `#34C759` | Save confirmation, validation pass, sync success |
| `success-muted` | `#EDFBF1` | Success background tint |
| `warning` | `#FF9F0A` | Repair notices, detached export, trial countdown |
| `warning-muted` | `#FFF8EC` | Warning background tint |
| `error` | `#FF3B30` | Validation errors, failed requests |
| `error-muted` | `#FFF0EF` | Error background tint |

### Borders and Dividers

| Token | Value | Usage |
|-------|-------|-------|
| `border-default` | `rgba(0,0,0,0.08)` | Card borders, input outlines |
| `border-subtle` | `rgba(0,0,0,0.04)` | Section dividers, panel separators |
| `border-strong` | `rgba(0,0,0,0.16)` | Active input borders, focus-adjacent |
| `border-code` | `rgba(255,255,255,0.08)` | Borders within dark code surfaces |

---

## Typography

### Font Stack

```css
/* UI text */
font-family: -apple-system, BlinkMacSystemFont, "SF Pro Text", "Segoe UI",
             system-ui, sans-serif;

/* Headings (display) */
font-family: -apple-system, BlinkMacSystemFont, "SF Pro Display", "Segoe UI",
             system-ui, sans-serif;

/* Monospace (ASCII, code, spec) */
font-family: "SF Mono", "Cascadia Code", "Fira Code", Menlo,
             "Courier New", monospace;
```

### Scale

| Name | Size | Weight | Line height | Usage |
|------|------|--------|-------------|-------|
| `caption` | 11px | 400 | 1.36 | Timestamps, badges, metadata chips |
| `footnote` | 12px | 400 | 1.42 | Helper text, tooltips, form hints |
| `subhead` | 13px | 500 | 1.38 | Table headers, form labels, nav items |
| `body` | 14px | 400 | 1.57 | Body text, list items, descriptions |
| `body-strong` | 14px | 600 | 1.57 | Emphasized body, table data |
| `callout` | 15px | 400 | 1.53 | Card content, panel descriptions |
| `headline` | 17px | 600 | 1.41 | Card headings, panel titles, section headers |
| `title3` | 20px | 600 | 1.3 | Page section titles |
| `title2` | 24px | 700 | 1.25 | Page titles, modal headers |
| `title1` | 28px | 700 | 1.21 | Hero headings within the app |
| `large-title` | 34px | 800 | 1.18 | Landing page hero |

---

## Spacing

8px base unit. All spacing values are multiples of 4.

| Token | Value | Usage |
|-------|-------|-------|
| `space-1` | 4px | Tight inline gaps, icon padding |
| `space-2` | 8px | Default inline gap, small padding |
| `space-3` | 12px | Form element padding, tight cards |
| `space-4` | 16px | Standard padding, card gaps |
| `space-5` | 20px | Section padding (compact) |
| `space-6` | 24px | Standard section padding |
| `space-8` | 32px | Large section gaps |
| `space-10` | 40px | Page-level vertical rhythm |
| `space-12` | 48px | Hero spacing |
| `space-16` | 64px | Full-page vertical separation |

---

## Border Radius

| Token | Value | Usage |
|-------|-------|-------|
| `radius-xs` | 6px | Badges, chips, small buttons |
| `radius-s` | 8px | Inputs, dropdowns, secondary buttons |
| `radius-m` | 12px | Cards, panels, primary buttons |
| `radius-l` | 16px | Large cards, modals, sidebars |
| `radius-xl` | 20px | Feature cards, hero elements |
| `radius-full` | 9999px | Pills, avatar circles, toggle switches |

---

## Shadows

| Token | Value | Usage |
|-------|-------|-------|
| `shadow-xs` | `0 1px 2px rgba(0,0,0,0.05)` | Subtle card lift |
| `shadow-s` | `0 1px 4px rgba(0,0,0,0.06), 0 1px 2px rgba(0,0,0,0.04)` | Cards, inputs on hover |
| `shadow-m` | `0 4px 12px rgba(0,0,0,0.08), 0 2px 4px rgba(0,0,0,0.04)` | Dropdowns, floating panels |
| `shadow-l` | `0 8px 32px rgba(0,0,0,0.10), 0 4px 8px rgba(0,0,0,0.05)` | Drawers, popovers |
| `shadow-xl` | `0 24px 64px rgba(0,0,0,0.14), 0 8px 16px rgba(0,0,0,0.06)` | Modals, full-screen overlays |
| `shadow-sidebar` | `1px 0 0 rgba(0,0,0,0.06)` | Sidebar right edge |

---

## Motion

All transitions use `cubic-bezier(0.4, 0, 0.2, 1)` (Material standard ease-in-out, consistent with Apple's feel).

| Token | Duration | Usage |
|-------|----------|-------|
| `duration-fast` | 120ms | Button press states, icon swaps |
| `duration-normal` | 200ms | Hover states, badge changes, tab switches |
| `duration-slow` | 300ms | Panel slides, drawer open/close |
| `duration-slower` | 400ms | Modal enter/exit, page-level transitions |

**Easing:** `cubic-bezier(0.4, 0, 0.2, 1)` for all transitions.

No bounce (`spring` easings). No flash animations on content load. Loading states use subtle pulse (opacity 0.5 → 1, no scale change).

---

## Component Conventions

### Primary Button

- Background: `accent`
- Text: `text-on-accent`, `body-strong`
- Radius: `radius-m`
- Padding: `space-2` vertical, `space-4` horizontal
- Hover: `accent-hover`, `shadow-s`
- Active: scale 0.98
- Disabled: `bg-base`, `text-tertiary`, no pointer

### Secondary Button

- Background: `bg-surface`
- Border: `border-default`
- Text: `text-primary`, `body-strong`
- Radius: `radius-m`
- Hover: `bg-base`, `border-strong`

### Destructive Button

- Background: `error-muted` on hover; transparent default
- Text: `error`
- Border: none default, `error` border on hover

### Input Fields

- Background: `bg-surface`
- Border: `1px solid border-default`
- Border on focus: `1px solid accent`, `shadow-s`
- Radius: `radius-s`
- Padding: `space-2` vertical, `space-3` horizontal
- Font: `body`, `text-primary`
- Placeholder: `text-tertiary`
- Transition: border-color `duration-normal`

### Cards

- Background: `bg-surface`
- Border: `1px solid border-subtle`
- Radius: `radius-m`
- Shadow: `shadow-xs`
- Hover: `shadow-s`, border `border-default`
- Padding: `space-6`
- Transition: shadow + border `duration-normal`

### Nav Items (Sidebar)

- Default: `text-secondary`, no background
- Hover: `text-primary`, background `rgba(0,0,0,0.04)`, `radius-s`
- Active: `accent-text`, background `accent-muted`, `radius-s`
- Font: `subhead`

### Badges / Chips

- Radius: `radius-full`
- Padding: `space-1` vertical, `space-2` horizontal
- Font: `caption`, weight 500
- Semantic colors: use `*-muted` background with matching dark text

### Tooltips

- Background: `#1D1D1F` (same as `text-primary`)
- Text: `#FFFFFF`, `footnote`
- Radius: `radius-xs`
- Padding: `space-1` vertical, `space-2` horizontal
- Shadow: `shadow-m`
- Appear after 400ms hover delay

### Code / ASCII Blocks

- Background: `bg-code`
- Text: `text-code`
- Font: monospace stack, 13px
- Radius: `radius-m`
- Border: `1px solid border-code`
- Padding: `space-4`
- Line height: 1.6
- Horizontal scroll on overflow

### Dividers

- `1px solid border-subtle`
- No margin collapse — use explicit spacing

---

## Dark Mode

V1 implements light mode only. Dark mode is a V2 feature. The design tokens above are defined for light mode. When dark mode is added, each token will have a dark variant.

The code/ASCII surfaces (`bg-code`, `bg-code-muted`) use dark backgrounds in light mode — this is intentional and is not dark mode. It creates a visual "terminal" contrast that is a product design choice, not a theme variant.

---

## Iconography

Use [Lucide](https://lucide.dev/) icons throughout (MIT licensed, clean stroke-based, consistent weight). All icons at `stroke-width: 1.5`. Size: 16px inline, 20px for nav items and buttons, 24px for feature icons.

Do not mix icon sets. If Lucide lacks a needed icon, file it as a gap and use the closest available icon with a label.

---

## Illustration and Empty States

Empty states use a single centered icon (48px, `text-tertiary`) plus a short headline and a CTA. No decorative illustrations in V1. Subtle, not cutesy.

---

## Responsive Breakpoints (reference from F4)

| Breakpoint | Width | Notes |
|------------|-------|-------|
| Desktop | ≥1280px | Full sidebar with labels |
| Desktop narrow | 1024–1279px | Sidebar icon-only |
| Tablet | 768–1023px | Single-column workspace; drawers for panels |
| Mobile | <768px | Share view only; authenticated routes show desktop banner |
