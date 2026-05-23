---
phase: 11
slice: F4
spec: specs/F4-ui-surface-workspace.md
status: planned
---

# Slice F4 — UI Surface and Workspace Architecture

## Pre-conditions

- [ ] Slice F0 complete — Vite + React + TypeScript scaffold running, Tailwind CSS configured, `pnpm dev` starts without errors.
- [ ] Slice F3 is at least far enough to call `POST /auth/login` and get a token (needed to test protected route guards).

Note: F4 is primarily frontend scaffolding. It can be built in parallel with F3 as long as auth stub responses are available (mock or real).

---

## Build Sequence

1. **React Router setup and route table** — Define all routes from F4 §2 in `src/router.tsx`. Use `createBrowserRouter`. Apply an `<AuthGuard>` wrapper to all `Required` routes that redirects to `/login` if no access token is in the Zustand store. Apply a `<SuperuserGuard>` to all `/admin/*` routes. The 404 catch-all route renders `NotFoundPage`. (→ F4 §2)

2. **Zustand stores** — Create the three stores from F4 §6.6 in `src/stores/`: `diagramStore` (spec, unsaved flag, selected node/edge, ascii cache), `workspaceUiStore` (panel visibility, sizes, active tab, inspector tab), `providerStore` (provider list, selected provider). Keep stores typed with TypeScript. `diagramStore` is the single source of truth for the spec — React Flow state is always derived from it. (→ F4 §6.6)

3. **App shell** — Build the `<AppShell>` component: 64px header bar, 240px fixed sidebar, flex main content area (F4 §3.1). Sidebar includes the navigation links from F4 §3.2 (New Diagram CTA, Dashboard, Library, Templates, Projects, Settings group). Header includes logo, breadcrumb slot, trial badge slot, upgrade button slot, user avatar menu (profile, settings, logout). Sidebar collapses to icon-only between 768–1023px. (→ F4 §3)

4. **Public layout** — Build a `<PublicLayout>` (minimal shell, no sidebar) used by `/`, `/login`, `/register`, `/verify-email`, `/reset-password`, and `/share/:shareToken`. (→ F4 §2)

5. **Page scaffolds — authenticated** — Create stub page components for each authenticated route: `DashboardPage`, `ProjectsPage`, `ProjectDetailPage`, `LibraryPage`, `TemplatesPage`, `AccountSettingsPage`, `BillingSettingsPage`, `ProvidersSettingsPage`. Each renders its title, a loading skeleton, and an empty state. Real data comes in domain slices. (→ F4 §4)

6. **Workspace panel layout** — Build the core workspace layout shell in `src/pages/WorkspacePage.tsx`: three horizontal panels (Prompt 300px, Grid Editor flex, Preview 320px) with drag handles and minimum widths (240px / 400px / 260px). Bottom inspector panel at 200px default height, collapsible. Panel collapsed/expanded state persisted in `localStorage`. Panels are empty shells at this stage — content comes from domain slices. (→ F4 §5.1)

7. **Prompt panel shell** — Build `<PromptPanel>` with the diagram type selector, prompt textarea, model provider selector, Generate button, and Improve button. Wire the four states (empty, generating, error, success) to `workspaceUiStore` flags. No real generation logic yet — Generate button calls a stub that sets the generating state for 1s then returns. (→ F4 §5.2)

8. **Preview panel shell** — Build `<PreviewPanel>` with the four tabs (ASCII, Mermaid, Visual, JSON/YAML). ASCII tab shows a monospace `<pre>` block reading from `diagramStore.asciiCache`. Mermaid tab is a monospace block. Visual tab is a read-only React Flow canvas (no drag handles). JSON/YAML tab is a read-only CodeMirror 6 editor. Copy and Download buttons are stubbed. (→ F4 §5.4)

9. **Inspector panel shell** — Build `<InspectorPanel>` with three tabs: Selection (shows selected node/edge ID as placeholder), Validation (shows error/warning count from `diagramStore`), Spec Editor (CodeMirror 6, read-only by default, Apply button stubbed). (→ F4 §5.5)

10. **React Flow grid editor scaffold** — Build `WorkspaceEditor.tsx` using `<ReactFlow>` with `snapToGrid={true}` and `snapGrid={[220, 120]}`. Import the constants from `src/renderers/visual/constants.ts` (created in slice-F2). Register stub versions of all 20 `nodeTypes` custom components (renders a box with the node label only — visual styling comes when F2 frontend work is complete). Wire `onNodeDragStop` to update `diagramStore` positions and call the debounced spec patch stub. (→ F4 §6)

11. **`useSpecToFlow` hook** — Implement the hook from F4 §6.2 in `src/renderers/visual/useSpecToFlow.ts`. Converts a `DiagramSpec` to React Flow `nodes[]` and `edges[]` using the grid-to-pixel formula. Wire it as the source of nodes/edges in `WorkspaceEditor`. (→ F4 §6.2–6.3)

12. **Loading, empty, and error states** — Implement the three states for every page scaffold: skeleton loader (Tailwind animate-pulse divs), empty state with a primary CTA button, and an error state with a retry button. Workspace adds the four extra states listed in F4 §8. (→ F4 §8)

13. **Mobile guard** — On the workspace route (`/workspace/*`), if `window.innerWidth < 768`, render the "Please use a desktop browser" banner instead of the editor. The `/share/:shareToken` route renders the read-only view on all viewports. (→ F4 §9)

---

## Done Criteria

- [ ] Workspace renders with three panels (Prompt, Grid Editor, Preview) simultaneously visible on a 1280px viewport. (→ F4 Acceptance Criteria)
- [ ] A node dragged to col 2, row 1 snaps to `x = 480, y = 160` and updates `diagramStore` to `position: { col: 2, row: 1 }`. (→ F4 Acceptance Criteria, §6.1)
- [ ] `diagramStore` spec is the single source of truth — refreshing the page and reloading from the backend produces identical React Flow layout. (→ F4 Acceptance Criteria)
- [ ] All 20 node kind custom components render without errors in the grid editor. (→ F4 Acceptance Criteria)
- [ ] Inspector Spec Editor accepts a valid JSON edit and updates the grid editor and ASCII preview. (→ F4 Acceptance Criteria)
- [ ] On mobile (<768px), authenticated workspace routes show the "use desktop" banner instead of the editor. (→ F4 Acceptance Criteria)
- [ ] `/share/:shareToken` renders a view-only diagram on mobile without auth. (→ F4 Acceptance Criteria)
- [ ] All authenticated routes redirect to `/login` when no access token is present. (→ F4 §2)
- [ ] All `/admin/*` routes return 403 for non-superuser authenticated users. (→ F4 §2, F3 §2.2)
- [ ] Panel collapsed states persist across page reloads via `localStorage`. (→ F4 §5.1)
