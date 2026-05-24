# Diagram Alley Commercial Product Overview

## 1. Product Vision

Diagram Alley should become a structured diagramming tool for developers, technical teams, IT professionals, writers, and AI power users who want clean, editable diagrams that can be exported into text-based documentation formats.

The product should not simply be an AI chatbot that creates ASCII art. That would be too fragile and too easy to copy. The strongest version of Diagram Alley is a diagram compiler:

```txt
User prompt or manual input
        ↓
Structured diagram specification
        ↓
Validation and repair layer
        ↓
Deterministic layout engine
        ↓
Multiple exports: ASCII, Markdown, Mermaid, SVG, PNG, JSON, YAML
```

The central promise should be:

> Describe a system once. Diagram Alley turns it into clean, editable diagrams that can be regenerated, exported, and maintained without breaking alignment.

The commercial version needs to solve a real workflow problem, not just generate something visually interesting. The value comes from reliability, editability, exportability, and privacy-friendly model options.

### 1.1 Locked Product Direction for Planning

These decisions are the current source of truth for running the planning process. If any of these change, the decision should be recorded before dependent specs are written.

### Closed Technical Decisions

The following decisions are resolved and should not be reopened without recording the reason in `decisions-log.md`:

* **Backend framework**: FastAPI (Python). Matches the AI/ML ecosystem, has strong async support, and aligns with the example code throughout this document.
* **Billing provider**: Stripe. Industry standard for SaaS subscriptions. Handles trials, metered credits, and subscription management natively.
* **ASCII reverse sync scope (V1)**: Label changes only. Simple node and edge label edits will attempt reverse sync into the structured spec. All other ASCII edits produce a detached export only. Full reverse sync (node add/remove, connection changes) is deferred to V2.

### Product Shape

* V1 is a web SaaS application.
* The web app should be built with Vite and React on the frontend.
* The first local test build should be a full web app with backend/API from the start, not a frontend-only prototype.
* Local testing should use a local Postgres database.
* Production should use cloud-hosted Postgres unless a later technical decision identifies a better fit.
* A more powerful desktop version can be considered later, after the web product proves the workflow.

### First Users

The first users to prioritize are:

1. Developers
2. Technical writers

Other user groups remain useful, but the early product should be judged by whether it helps developers and technical writers create documentation-ready diagrams quickly.

### Core User Win

The first product win should be:

> I describe an architecture, Diagram Alley turns it into a clean structured diagram, and I can immediately use the result in a code editor, README, pull request, documentation site, or GitHub workflow.

### AI Direction

* The product must be useful with or without AI.
* AI generation and AI modification are required for the paid V1.
* The first AI version should support both OpenAI-compatible cloud APIs and local Ollama-style endpoints.
* Users should be able to bring their own cloud API keys.
* The recommended commercial path is to support BYOK first, then add hosted AI credits for onboarding, free trials, and optional paid usage.
* AI failures should trigger automatic retry or repair where possible, then show a clear explanation and fall back to manual editing.

### Editing Direction

* The primary workspace should be a visual grid-based editor/canvas.
* Visual editing should remain deterministic, not freeform whiteboarding.
* Visual drag/edit actions should update the structured spec and influence ASCII layout where feasible.
* Normal users should not be forced to edit raw JSON or YAML.
* Advanced users should be able to inspect and edit the open diagram spec.
* Users should be able to manually edit ASCII output if they need to.
* Manual ASCII edits support label-change reverse sync only in V1. Other edits produce a detached export with a warning. Full reverse sync is V2.

### Spec and Export Direction

* The diagram spec should be an open format.
* V1 should support user choice between JSON and YAML where feasible.
* ASCII is the first export format that must be excellent.
* Paid V1 should target ASCII, Markdown, Mermaid, SVG, PNG, JSON, and YAML exports.
* Exports should include metadata linking them back to source spec/version where appropriate, while still remaining clean enough for documentation.

### Storage and Project Direction

* Diagrams should belong to projects.
* Projects should belong to users.
* The app should support multiple diagrams per project from the start.
* Cloud storage should be opt-in.
* Local download/export should always be available.
* Projects should be generic workspaces first, but designed so they can later link to GitHub repositories or local repository concepts.
* Version history is required for V1.

### Commercial Direction

* The intent is a commercial product, tested locally before launch.
* The first paid version should be single-user.
* Billing is required for the first paid version.
* A free trial should exist before paid conversion.
* The recommended pricing model is recurring Pro subscription revenue, with optional hosted AI credit packs if hosted model usage is offered.
* Team accounts, shared workspaces, permissions, and realtime collaboration are V2 or later.

### Integration Direction

* GitHub integration is part of the product direction because GitHub and local repos are core usage contexts.
* Copy/download/export must work first and are the primary V1 delivery path.
* GitHub commit/PR workflows are a **stretch goal for Paid V1** — included if the core editor, renderer, version history, and exports are solid before launch, otherwise deferred to V2.
* Repo scanning and AI-suggested diagrams are V2 or later.

---

## 2. Core Positioning

### Primary Positioning

Diagram Alley is a web SaaS, AI-assisted diagram workspace that converts rough ideas into structured, editable diagrams and clean documentation-ready exports while supporting both cloud and local model providers.

### Short Product Description

Diagram Alley helps developers and technical writers create clean ASCII, Mermaid, SVG, PNG, JSON/YAML, and Markdown-ready diagrams from prompts, structured specs, imports, or manual editing.

### Stronger Commercial Framing

> A structured diagram generator for technical documentation, powered by AI but rendered by deterministic layout rules.

### What Diagram Alley Is

* A structured diagram editor
* A web SaaS for documentation-ready diagrams
* An AI-assisted diagram planning tool
* A deterministic ASCII renderer
* A visual grid-based diagram editor
* A documentation export tool
* A cloud and local-provider-friendly diagramming workflow
* A bridge between natural language, diagram-as-code, and visual diagrams
* A GitHub/repository-oriented documentation workflow

### What Diagram Alley Is Not

* Not just a chatbot wrapper
* Not just an ASCII art toy
* Not a full Figma replacement
* Not a full Excalidraw clone
* Not a whiteboard-first collaboration platform
* Not a general image generation tool

---

## 3. Commercial Success Requirements

For Diagram Alley to become commercially viable, it needs to do more than produce diagrams. It needs to become a repeatable workflow tool.

The application should be capable of:

1. Turning prompts into structured diagram specs.
2. Rendering clean ASCII diagrams from those specs.
3. Rendering visual previews from the same specs.
4. Letting users manually edit the structured diagram.
5. Letting users export to formats developers actually use.
6. Saving reusable diagrams and templates.
7. Supporting local and cloud AI providers.
8. Supporting privacy-sensitive technical diagrams.
9. Making iteration easier than asking a chatbot repeatedly.
10. Handling diagram updates without breaking layout.
11. Preserving useful version history.
12. Supporting GitHub/repository workflows where they improve documentation work.

The commercial product should be judged by this question:

> Is this easier, cleaner, and more reliable than asking ChatGPT, drawing manually, or writing Mermaid by hand?

If the answer is yes, Diagram Alley has a real product lane.

---

## 4. Target Users

Planning priority for V1:

1. Developers
2. Technical writers

The product can support IT professionals, AI power users, teachers, product managers, and students later, but V1 decisions should optimize for developer and technical-writer documentation workflows.

### 4.1 Primary Users

### Software Developers

Developers need diagrams for:

* README files
* Pull requests
* Architecture documentation
* API documentation
* Code comments
* Internal docs
* Onboarding notes
* Planning documents

Their main need is speed and accuracy. They want diagrams that are easy to copy, edit, and keep in source control.

### Technical Writers

Technical writers need diagrams for:

* Documentation sites
* Knowledge bases
* Internal guides
* User manuals
* Product documentation
* Training material

Their main need is consistency. They want diagrams that match the same formatting style across documents.

### AI Power Users

These users already ask AI chatbots for diagrams. They understand the pain point immediately because they have experienced broken spacing, inconsistent boxes, and fragile follow-up edits.

Their main need is control. They want the AI to help, but they do not want the AI to be the final layout engine.

---

### 4.2 Secondary Users

### IT Professionals and System Administrators

IT users need diagrams for:

* Network layouts
* Device relationships
* Service flows
* Troubleshooting notes
* Infrastructure maps
* Access control flows
* Security review documentation

Their main need is clarity. They need diagrams that explain how systems connect without requiring heavy design tools.

Note: IT professionals are not a V1 priority. Network diagram support is planned, but V1 decisions should not be optimized for this group.

### Teachers and Trainers

Teachers can use Diagram Alley to create simple visual explanations for lessons.

Examples:

* Process diagrams
* Cause-and-effect diagrams
* System diagrams
* Historical event flows
* Classroom handouts

### Product Managers

Product managers can use it to sketch:

* Feature flows
* User journeys
* System handoffs
* Data movement
* MVP scope diagrams

### Students

Students can use it to understand:

* Database relationships
* Programming architecture
* Network basics
* Software design patterns

---

## 5. Core Product Principles

### 5.1 The Renderer Is the Product

The LLM should not be trusted to manually draw final ASCII spacing. It should produce structured intent. Diagram Alley should perform rendering through deterministic rules.

Correct product flow:

```txt
Prompt → Structured Spec → Validation → Renderer → Export
```

Incorrect product flow:

```txt
Prompt → Raw ASCII from LLM
```

The second approach is fragile and not commercially defensible.

### 5.2 Structured Data Must Be Editable

Every generated diagram should remain editable as structured data.

A user should be able to:

* Move nodes on a deterministic grid
* Rename nodes
* Add nodes
* Delete nodes
* Reorder sections
* Change relationships
* Edit labels
* Change diagram type settings
* Inspect and edit JSON or YAML when needed
* Edit generated ASCII when needed
* Reverse-sync simple ASCII edits back into structure where feasible
* Re-render without losing structure

If users cannot edit the diagram after generation, the app becomes a weak chatbot wrapper.

### 5.3 Export Quality Matters

The exports must be genuinely useful. A commercial user should be able to paste the output into real documentation without cleaning it up.

Supported commercial-grade exports should eventually include:

* ASCII
* Markdown
* Mermaid
* SVG
* PNG
* JSON spec
* YAML spec
* Possibly PDF

ASCII quality matters first. The product can support many exports, but paid users should trust the ASCII output enough to paste it directly into source-controlled documentation.

### 5.4 Local-Friendly and Privacy-Friendly

Technical diagrams often contain private architecture details. A strong commercial advantage is letting users connect to:

* Local Ollama models
* llama.cpp servers
* OpenAI-compatible endpoints
* Cloud APIs when desired
* BYOK cloud provider credentials
* Optional hosted AI credits for trial and paid convenience

The app should not force every user into one cloud provider.

### 5.5 Start Simple, Become Powerful

The app should feel simple at first but allow deeper control as the user grows.

Beginner mode:

```txt
Prompt → Generate → Copy
```

Advanced mode:

```txt
Prompt → Edit spec → Modify layout rules → Export multiple formats
```

---

## 6. High-Level Application Map

The commercial version should eventually include these major areas:

```txt
Diagram Alley
├── Public Marketing Site
├── Authentication
├── Onboarding
├── Dashboard
├── Diagram Workspace
├── Diagram Library
├── Template Gallery
├── Diagram Type Pages
├── Model Provider Settings
├── Export Center
├── Import and Conversion
├── GitHub/Repository Integration
├── Project/Workspace Management
├── Version History
├── Account and Billing
├── Documentation and Help
└── Admin/Internal Tools
```

Not all of these are needed for the first engine proof, but they define the full commercial product.

---

## 7. Pages Needed

### 7.1 Public Landing Page

The landing page sells the product.

### Purpose

Convert visitors into users by clearly explaining the problem, showing before/after examples, and demonstrating why Diagram Alley is better than raw chatbot diagram generation.

### Key Sections

1. Hero section
2. Problem section
3. Product demo section
4. Use case examples
5. Export format showcase
6. Local/private model support section
7. Pricing callout
8. FAQ
9. Call-to-action

### Hero Message

Example:

```txt
Clean diagrams from rough prompts.
ASCII, Mermaid, SVG, and Markdown-ready exports without broken spacing.
```

### Components

* Hero headline
* Prompt demo box
* Generated ASCII preview
* Rendered visual preview
* CTA button
* Use case cards
* Export format badges
* Comparison table
* Pricing teaser
* FAQ accordion

### Commercial Requirement

The landing page must quickly communicate that Diagram Alley solves a specific pain: AI-generated diagrams are useful but fragile. Diagram Alley makes them structured, editable, and exportable.

---

### 7.2 Login and Signup Pages

Commercial SaaS versions need authentication.

### Pages

* Sign up
* Log in
* Forgot password
* Reset password
* Email verification

### Components

* Auth form
* Provider login buttons if needed
* Password reset flow
* Error messages
* Terms/privacy links

### V1 Note

Authentication should be included early because the first local test build is intended to mirror the paid SaaS shape. Auth affects saved diagrams, provider credentials, billing, version history, cloud storage, and GitHub integrations.

---

### 7.3 Onboarding Flow

The onboarding flow should help new users get to their first useful diagram quickly.

### Purpose

Reduce confusion and push users into an immediate win.

### Recommended Onboarding Steps

1. Choose primary use case
2. Choose model provider
3. Generate first sample diagram
4. Show how to edit the structure
5. Show copy/export options

### Use Case Options

* Software architecture
* Database schema
* UI wireframe
* Network diagram
* Documentation diagram
* Learning/explaining concepts

### Model Setup Options

* Use built-in cloud model
* Connect OpenAI-compatible API
* Connect Ollama
* Connect llama.cpp
* Skip for now and use manual editor

### Components

* Stepper
* Use case cards
* Provider connection form
* Sample prompt launcher
* First diagram success screen

### Commercial Requirement

A new user should understand the product within five minutes. Ideally, they should produce a useful diagram in under two minutes.

---

### 7.4 Dashboard

The dashboard is the user's home base.

### Purpose

Give users quick access to recent diagrams, templates, projects, and creation actions.

### Dashboard Sections

1. New diagram button
2. Recent diagrams
3. Favorite templates
4. Recent projects
5. Provider status
6. Usage/status summary if SaaS
7. Helpful examples

### Components

* Quick create button
* Diagram cards
* Template cards
* Search bar
* Recent activity list
* Provider status indicator
* Empty state guidance

### Example Layout

```txt
+--------------------------------------------------------------------------------+
| Diagram Alley                                             New Diagram  Settings |
+--------------------------------------------------------------------------------+
| Quick Start                                                                    |
| [Architecture] [Database Schema] [UI Wireframe] [Flowchart]                    |
+--------------------------------------------------------------------------------+
| Recent Diagrams                                                                |
| +------------------+ +------------------+ +------------------+                 |
| | API Architecture| | Inventory Schema | | Dashboard Layout |                 |
| +------------------+ +------------------+ +------------------+                 |
+--------------------------------------------------------------------------------+
| Templates                              | Provider Status                        |
| - FastAPI + React + Postgres           | Ollama: Connected                      |
| - CRUD App Schema                      | Cloud API: Not configured              |
+--------------------------------------------------------------------------------+
```

---

### 7.5 Diagram Workspace

This is the most important page in the app.

### Purpose

Create, edit, render, and export diagrams.

### Main Layout

The workspace should be centered on a visual grid-based editor/canvas. Prompt input, structured data, ASCII output, and export controls should support the canvas rather than replace it.

The workspace should support four major working areas:

1. Visual grid editor/canvas
2. Prompt/input panel
3. Structured spec/inspector panel
4. Preview/export panel

### Recommended Layout

```txt
+--------------------------------------------------------------------------------+
| Diagram Alley | Project: Inventory Docs          Save   Export   Settings       |
+--------------------------------------------------------------------------------+
| Prompt / AI Input      | Visual Grid Editor / Canvas               | Preview     |
|                        |                                           |             |
| [Prompt textarea]      | +-------------+        +-------------+    | [ASCII]     |
| Diagram Type           | | React App   | -----> | API Service |    | [Mermaid]   |
| [Architecture v]       | +------+------+        +------+------+    | [SVG/PNG]   |
| Model Provider         |        |                      |            | [JSON/YAML] |
| [Ollama v]             |        v                      v            |             |
|                        | +-------------+        +-------------+    | Copy        |
| [Generate] [Improve]   | | Auth        |        | Postgres    |    | Download    |
|                        | +-------------+        +-------------+    | Commit PR   |
+--------------------------------------------------------------------------------+
| Inspector: selected node, edge, layout rules, validation, spec escape hatch      |
+--------------------------------------------------------------------------------+
```

### Workspace Modes

The workspace should eventually support multiple modes:

1. Prompt Mode
2. Visual Grid Edit Mode
3. Structured Form Mode
4. JSON/YAML Spec Mode
5. ASCII Edit Mode
6. Export Mode

### Core Workspace Components

#### Prompt Panel

Allows user to describe what they want.

Components:

* Prompt textarea
* Diagram type selector
* Model provider selector
* Model selector
* Generate button
* Improve existing diagram button
* Prompt examples dropdown
* Advanced instruction toggle

Capabilities:

* Generate new diagram
* Modify existing diagram from prompt
* Add section to existing diagram
* Explain current diagram
* Convert diagram type where possible

#### Structured Editor Panel

Allows user to modify diagram data without editing raw JSON.

Components:

* Node list
* Edge list
* Group list
* Grid position controls
* Table editor for database diagrams
* Component tree for UI diagrams
* Properties panel
* Add/delete/reorder controls
* Validation warnings

Capabilities:

* Add node
* Remove node
* Rename node
* Change node kind
* Move node on grid
* Add connection
* Remove connection
* Edit labels
* Reorder layout
* Adjust direction
* Edit diagram title

#### JSON/DSL Editor

Allows advanced users to directly edit the diagram spec.

Components:

* Code editor
* Syntax highlighting
* Format button
* Validate button
* Error panel
* Reset to last valid version
* JSON/YAML format toggle

Capabilities:

* Edit raw spec
* Validate spec
* Copy spec
* Import spec
* Restore previous valid state

#### Preview Panel

Shows output.

Tabs:

* ASCII
* Visual
* Mermaid
* Markdown
* JSON
* YAML

Components:

* Monospace ASCII preview
* Visual preview canvas
* Mermaid preview
* Copy button
* Download button
* Export dropdown
* Zoom controls
* Theme toggle

Capabilities:

* Copy ASCII
* Copy Markdown
* Copy Mermaid
* Download TXT
* Download MD
* Download SVG
* Download PNG
* Download JSON
* Download YAML
* Commit/export to GitHub (stretch goal, where configured)

---

### 7.6 Diagram Library

The diagram library stores the user's saved diagrams.

### Purpose

Let users organize, search, reopen, duplicate, and manage diagrams.

### Components

* Search bar
* Filter sidebar
* Diagram cards
* List/table view toggle
* Tags
* Project selector
* Sort controls
* Bulk actions

### Filters

* Diagram type
* Project
* Tags
* Last updated
* Created by
* Export format availability
* Favorites

### Diagram Card Content

* Title
* Diagram type
* Small preview
* Last updated date
* Tags
* Export badges
* Quick actions

### Quick Actions

* Open
* Duplicate
* Rename
* Export
* Delete
* Favorite

### Commercial Requirement

Saved diagrams are important because they move the product from a one-off generator to a reusable documentation tool.

---

### 7.7 Template Gallery

Templates make the app easier to use and more commercially valuable.

### Purpose

Let users start from common diagram structures instead of a blank page.

### Template Categories

* Software architecture
* API flows
* Database schemas
* UI wireframes
* Network diagrams
* DevOps/infrastructure
* Documentation layouts
* Education/explainer diagrams

### Example Templates

#### Software Architecture

* React + FastAPI + Postgres
* React + Node + MongoDB
* Monolith application
* Microservices with queue
* Local-first desktop app
* ETL pipeline
* AI agent architecture

#### Database Schema

* Blog schema
* Inventory schema
* Orders/customers schema
* User roles and permissions schema
* Reporting database schema

#### UI Wireframes

* Admin dashboard
* CRUD list/detail page
* Settings page
* Login page
* Analytics dashboard
* Mobile drawer layout

#### Network/IT

* Small office network
* Firewall/router/switch topology
* Camera system layout
* VLAN overview
* Device monitoring flow

### Components

* Template cards
* Category tabs
* Search
* Preview modal
* Use template button
* Favorite template button

### Commercial Requirement

Templates reduce friction and make the product feel useful before the user has mastered prompting or specs.

---

### 7.8 Diagram Type Pages

Each diagram type should have a specialized creation flow and editor.

### Architecture Diagram Page

Capabilities:

* Create nodes
* Create edges
* Group services
* Set direction
* Choose node kinds
* Show layers such as Client, API, Data, External Services

Components:

* Node editor
* Edge editor
* Group editor
* Layer controls
* Architecture preview

### Database Schema Page

Capabilities:

* Create tables
* Add columns
* Mark primary keys
* Mark foreign keys
* Define relationships
* Render table boxes
* Export Mermaid ER diagrams

Components:

* Table list
* Column editor
* Relationship editor
* ERD preview
* Schema import option later

### UI Wireframe Page

Capabilities:

* Define page structure
* Add header/sidebar/main/card/table/form/button/input components
* Reorder components
* Render ASCII wireframe
* Render basic visual preview

Components:

* Component tree
* Region editor
* Properties panel
* Layout preview

### Flowchart Page

Capabilities:

* Add process nodes
* Add decision nodes
* Add start/end nodes
* Create yes/no branches
* Render ASCII and Mermaid flowcharts

Components:

* Step editor
* Decision editor
* Branch editor
* Flow preview

### Network Diagram Page

Capabilities:

* Add routers, switches, firewalls, devices, servers
* Group by location or network zone
* Define connections
* Label protocols or VLANs

Components:

* Device list
* Connection editor
* Zone/group editor
* Network preview

---

### 7.9 Model Provider Settings Page

This page is critical for local provider support, BYOK cloud providers, and advanced users.

### Purpose

Let users configure how AI generation works.

### Supported Providers

* Built-in cloud provider if offered
* OpenAI-compatible API
* Ollama
* llama.cpp server
* Direct browser-to-local endpoint connection where browser/network constraints allow
* Anthropic-compatible adapter later
* Gemini-compatible adapter later
* Custom endpoint later

### Components

* Provider list
* Add provider form
* Base URL input
* API key input
* Model name input
* Test connection button
* Default provider selector
* Provider status indicator
* Per-diagram model defaults
* Setting to disable stored credentials and enter credentials per session

### Settings Fields

For OpenAI-compatible endpoints:

* Provider name
* Base URL
* API key
* Model name
* Temperature
* Max tokens

For Ollama:

* Base URL
* Model name
* Test connection

For llama.cpp:

* Base URL
* API key if needed
* Model name
* Context size note

### Commercial Requirement

The model settings experience must be simple enough for normal users but flexible enough for local LLM users.

The default commercial recommendation is BYOK plus optional hosted AI credits. BYOK keeps provider cost predictable and gives power users control. Hosted credits improve trial onboarding and can become an optional paid usage add-on.

---

### 7.10 Export Center

The export center can be either a page or a modal inside the workspace.

### Purpose

Give users control over output formats.

### Export Formats

* ASCII `.txt`
* Markdown `.md`
* Mermaid `.mmd`
* JSON `.json`
* YAML `.yaml`
* SVG `.svg`
* PNG `.png`
* PDF `.pdf` later

### Export Options

* Include title
* Include caption
* Include metadata
* Include source spec
* Include source version reference
* Choose ASCII style
* Choose width constraints
* Choose Mermaid type
* Choose visual theme
* Commit exported files to GitHub where configured

### Components

* Format selector
* Preview panel
* Copy button
* Download button
* Export settings
* Batch export option later

### Commercial Requirement

Exports must be predictable and polished. Users are paying to save time. If they have to clean up exports manually, the product fails.

---

### 7.11 Project and Workspace Management

Projects help organize diagrams by product, team, app, or documentation set.

### Purpose

Turn Diagram Alley from a generator into a documentation workspace.

### Capabilities

* Create project
* Rename project
* Add diagrams to project
* Store multiple diagrams per project
* Tag diagrams
* Duplicate project
* Export project documentation
* Download project specs and exports locally
* Optionally sync project diagrams to cloud storage
* Optionally connect project to a GitHub repository later
* Archive project

### Components

* Project list
* Project detail page
* Project diagram grid
* Project settings
* Tag manager

### Example Project Structure

```txt
Inventory Management System
├── System Architecture
├── Database Schema
├── Dashboard Wireframe
├── Inbound Flow
├── Outbound Flow
└── Reconciliation Flow
```

---

### 7.12 Account and Billing Page

Needed for a commercial SaaS or cloud-sync version.

### Capabilities

* View plan
* Manage subscription
* Update payment method
* View usage
* Manage API/provider options
* Export user data
* Delete account

### Possible Plans

#### Free

* Free trial access before paid conversion
* Limited hosted AI credits if offered
* Limited saved diagrams
* Core editor evaluation

#### Pro

* AI generation
* AI modification of existing diagrams
* More saved diagrams
* Mermaid/SVG/PNG exports
* JSON/YAML spec export
* Local model support
* BYOK cloud provider support
* More templates
* Version history
* GitHub commit/PR workflows (stretch goal for V1)

#### Team

Post-V1.

* Shared projects
* Team templates
* Role permissions
* Centralized billing
* Shared provider configuration

### Commercial Note

V1 is a paid SaaS, so billing and free trial support are required. A desktop/local-first paid app can be revisited later.

---

### 7.13 Documentation and Help Pages

### Purpose

Help users understand how to use prompts, specs, templates, and exports.

### Pages

* Getting started
* Prompting guide
* Diagram type guide
* Export guide
* Local model setup
* Ollama setup
* llama.cpp setup
* Mermaid export guide
* Troubleshooting
* FAQ

### Components

* Searchable docs
* Example gallery
* Copyable prompts
* Copyable JSON specs
* Video/gif demos later

---

### 7.14 Import and Conversion

### Purpose

Let users bring existing diagrams into Diagram Alley before paid launch.

### Engineering Risk: Mermaid Import

Mermaid's syntax has grown organically and is irregular across diagram types (flowchart, ER, sequence, etc.). Parsing Mermaid into a structured Diagram Alley spec is a non-trivial parsing problem — not a simple string transformation. This is a meaningful engineering investment and is scoped in `docs/diagram-alley/specs/D5-import.md`. The risk is that partial or incorrect parsing silently corrupts the imported diagram.

Recommended V1 approach: support Mermaid flowchart and ER diagram import at best-effort, with clear warnings for unsupported constructs. Do not promise full-fidelity Mermaid import at launch.

### Supported Inputs

* Mermaid (flowchart and ER diagram, best-effort — see risk note above)
* JSON spec
* YAML spec
* Supported ASCII diagrams where parsing is feasible

### Components

* Import modal/page
* Paste area
* File upload
* Format detector
* Conversion preview
* Validation warnings
* Accept/reject conversion controls

### Commercial Requirement

Import should never silently corrupt diagrams. If conversion is ambiguous, the app should show warnings and preserve the original input.

---

### 7.15 GitHub and Repository Integration

### Purpose

Connect Diagram Alley to the places developers already maintain documentation.

### V1 Capabilities

* Copy output to clipboard
* Download local files
* Commit diagram specs and exports to GitHub where configured
* Open pull requests with updated diagram files where feasible

### Later Capabilities

* Read repository structure
* Suggest diagrams from repository contents
* Keep exported files synchronized with source specs
* CLI support for repo workflows

---

## 8. Core Components

### 8.1 App Shell

The app shell should provide consistent navigation.

### Components

* Header
* Sidebar
* Main content area
* Settings menu
* Account menu
* Project selector
* Command palette later

### Sidebar Items

* Dashboard
* New Diagram
* Library
* Templates
* Projects
* Model Settings
* Exports
* Imports
* GitHub Integration
* Docs

---

### 8.2 Prompt Input Component

### Purpose

Accept natural language diagram requests.

### Features

* Multi-line prompt textarea
* Prompt examples
* Diagram type selector
* Model selector
* Generate button
* Modify existing diagram option
* Prompt history

### Advanced Options

* Output complexity
* Layout direction
* Include descriptions
* Include labels
* Strict mode
* Repair mode

---

### 8.3 Diagram Type Selector

### Purpose

Let user select the target diagram type.

### Options

* Architecture
* Database Schema
* UI Wireframe
* Flowchart
* Network Diagram
* File Structure
* Sequence Diagram later
* Blender Node Diagram later

Each type should map to:

* Specific schema
* Specific validation rules
* Specific renderer
* Specific editor UI
* Specific export options

---

### 8.4 Structured Spec Editor

### Purpose

Let users edit diagrams without fighting raw text.

### Variants

* Architecture editor
* Database editor
* UI wireframe editor
* Flowchart editor
* Network editor

### Shared Capabilities

* Add item
* Remove item
* Rename item
* Reorder item
* Edit properties
* Validate changes
* Auto-render after changes

---

### 8.5 JSON/DSL Editor

### Purpose

Give advanced users full control.

### Features

* Syntax highlighting
* Auto-format
* Validation
* Error line display
* Reset to last valid version
* Copy JSON or YAML
* Import JSON or YAML

---

### 8.6 ASCII Preview Component

### Purpose

Display copyable plain text diagrams.

### Features

* Monospace display
* Horizontal scrolling
* Copy button
* Download button
* Width indicator
* Line count
* Character count
* Theme toggle

### ASCII Styles Later

* Simple ASCII
* Unicode box drawing
* Compact style
* Wide style
* Markdown fenced block

---

### 8.7 Visual Preview Component

### Purpose

Show and edit a rendered diagram version from the same structured spec.

### Initial Approach

Use a deterministic grid-based visual editor. The visual layout should update the structured spec and influence ASCII layout where feasible.

### Features

* Visual grid editor
* Node drag/edit controls
* Edge creation controls
* Zoom in/out
* Fit to screen
* Theme selector
* Export SVG/PNG

---

### 8.8 Validation Panel

### Purpose

Explain what is wrong with the diagram spec.

### Validation Examples

* Missing node ID
* Duplicate node ID
* Edge references missing node
* Empty label
* Unsupported direction
* Invalid table relationship
* Unsupported UI component type

### Components

* Error list
* Warning list
* Suggested fixes
* Apply auto-fix button

---

### 8.9 Export Controls

### Purpose

Make it easy to move diagrams into other tools.

### Features

* Copy ASCII
* Copy Markdown
* Copy Mermaid
* Download TXT
* Download MD
* Download JSON
* Download YAML
* Download SVG/PNG
* Commit/open PR in GitHub (stretch goal, where configured)

---

### 8.10 Template Card

### Purpose

Preview and start from reusable diagram structures.

### Card Content

* Template title
* Diagram type
* Short description
* Mini preview
* Use button
* Favorite button

---

### 8.11 Diagram Card

### Purpose

Represent saved diagrams in the library.

### Card Content

* Diagram title
* Type badge
* Updated date
* Preview thumbnail
* Tags
* Quick actions

---

## 9. Major Application Flows

### 9.1 Generate New Diagram from Prompt

### Flow

```txt
Dashboard or Workspace
        ↓
User clicks New Diagram
        ↓
User selects diagram type
        ↓
User enters prompt
        ↓
User selects model provider
        ↓
App sends prompt to LLM adapter
        ↓
LLM returns structured spec
        ↓
Validation layer checks spec
        ↓
Repair layer fixes simple issues if needed
        ↓
Renderer generates ASCII and visual preview
        ↓
User edits, saves, or exports
```

### Required Behaviors

* Show loading state during generation
* Validate returned spec
* Show warnings if repairs were made
* Never accept raw unchecked model output as final truth
* Keep the last valid spec if generation fails

---

### 9.2 Create Diagram Manually

### Flow

```txt
New Diagram
        ↓
Choose diagram type
        ↓
Start from blank spec or template
        ↓
Use visual grid editor to add nodes/tables/components
        ↓
Preview updates live
        ↓
Save/export
```

### Commercial Importance

Manual creation matters because the product should remain useful even without AI.

---

### 9.3 Edit Generated Diagram

### Flow

```txt
Generated diagram appears
        ↓
User edits visually on deterministic grid
        ↓
User optionally edits structured fields or ASCII
        ↓
Validation runs
        ↓
Renderer updates preview
        ↓
User copies/exports final version
```

### Required Behaviors

* Edits should not require regeneration from AI
* Preview should update quickly
* Invalid edits should be clearly shown
* User should be able to revert to last valid version
* Visual edits should update the structured spec
* Label-change ASCII edits sync back to the spec; all other ASCII edits produce a detached export with a warning

---

### 9.4 Modify Diagram with AI

### Flow

```txt
Existing diagram
        ↓
User enters change request
        ↓
Example: Add Redis between API and database
        ↓
App sends current spec + change request to LLM
        ↓
LLM returns updated spec
        ↓
Validation and repair run
        ↓
Diff preview shown if possible
        ↓
User accepts or rejects update
```

### Required Behaviors

* Do not overwrite the current diagram without confirmation
* Show what changed
* Allow user to reject the update
* Preserve diagram metadata

### Example Change Requests

* Add Redis between API and database
* Group the backend services
* Rename frontend to React Client
* Add an external USDA API
* Change layout to top-down
* Add edge labels

---

### 9.5 Export Diagram

### Flow

```txt
Diagram workspace
        ↓
User opens export menu
        ↓
User selects format
        ↓
User previews output
        ↓
User copies or downloads
        ↓
User optionally commits export to GitHub or opens PR
```

### Required Behaviors

* Export output should match preview
* Markdown export should include fenced code block for ASCII
* Mermaid export should be valid Mermaid where possible
* JSON/YAML export should preserve full spec
* Export metadata should link output to the source spec/version where appropriate

---

### 9.6 Save and Reopen Diagram

### Flow

```txt
User creates diagram
        ↓
User clicks Save
        ↓
App stores title, type, spec, outputs, metadata
        ↓
User later opens Library
        ↓
User searches or filters
        ↓
User reopens diagram
        ↓
App loads spec and re-renders output
```

### Required Behaviors

* Saved diagrams should store the spec as the source of truth
* Rendered outputs can be cached but should be regeneratable
* Reopening should not depend on AI
* A new version should be created for meaningful changes

---

### 9.7 Use Template

### Flow

```txt
Template Gallery
        ↓
User picks template
        ↓
Template opens in workspace
        ↓
User edits prompt/spec
        ↓
Preview updates
        ↓
User saves as new diagram
```

### Required Behaviors

* Templates should not be overwritten accidentally
* Built-in templates should ship in V1
* User-created templates should ship in V1

---

### 9.8 Configure Local Model

### Flow

```txt
Settings
        ↓
Model Providers
        ↓
Add Provider
        ↓
Choose Ollama or llama.cpp
        ↓
Enter base URL and model name
        ↓
Test connection
        ↓
Save provider
        ↓
Use provider in workspace
```

### Required Behaviors

* Show connection success/failure clearly
* Do not expose API keys in the UI after saving
* Allow users to select default provider
* Allow manual mode if no provider is configured
* Allow users to disable stored credentials and enter provider credentials per session

---

### 9.9 Import Existing Diagram

### Flow

```txt
Workspace or Import page
        ↓
User pastes or uploads Mermaid, JSON, YAML, or supported ASCII
        ↓
App detects format
        ↓
App converts input into structured spec where possible
        ↓
Validation runs
        ↓
User reviews conversion warnings
        ↓
User accepts, edits, saves, or rejects import
```

### Required Behaviors

* Preserve original input
* Show warnings for ambiguous conversion
* Never silently corrupt imported diagrams
* Let user continue manually when conversion is incomplete

---

### 9.10 Commit Diagram to GitHub

### Flow

```txt
Workspace or Export Center
        ↓
User chooses GitHub export
        ↓
User selects repository, branch, and output files
        ↓
App commits source spec and selected exports
        ↓
App optionally opens a pull request
```

### Required Behaviors

* User must preview files before commit
* Source spec should be committed with exports
* Commit/PR should identify the diagram version
* Copy/download must remain available when GitHub is not configured

---

## 10. Diagram Types and Required Capabilities

### 10.1 Architecture Diagrams

### Required Objects

* Nodes
* Edges
* Groups
* Grid positions
* Layers

### Node Kinds

Architecture diagrams should support many node kinds in V1 so users can describe real systems without forcing everything into generic boxes.

**V1 implementation note:** Not all node kinds listed below need distinct visual rendering in V1. Kinds that don't have a meaningful rendering difference from `generic` should render as a labeled generic box in V1 and get specialized rendering in V2. The renderer spec (`F2-rendering-validation-and-repair.md`) should define which kinds get distinct treatment and which collapse to generic. Avoid implementing 20 separate renderers before the engine is proven.

Required node kinds:

* User/client
* Frontend
* Backend
* Client
* Service
* Gateway
* Database
* Cache
* Queue
* Worker
* External API
* Auth
* Storage
* CDN
* Message broker
* Job scheduler
* Search/index
* File/blob storage
* Third-party service
* Internal service
* Generic

### Required Layouts

* Top-down
* Left-to-right

### Later Layouts

* Layered architecture
* Grouped services
* Hub-and-spoke
* Multi-column

### Example Output

```txt
+-------------------+
| React Frontend    |
| client            |
+---------+---------+
          |
          | HTTP
          v
+-------------------+
| FastAPI Backend   |
| service           |
+---------+---------+
          |
          | SQLAlchemy
          v
+-------------------+
| Postgres Database |
| database          |
+-------------------+
```

---

### 10.2 Database Schema Diagrams

### Required Objects

* Tables
* Columns
* Column types
* Primary keys
* Foreign keys
* Relationships

### Required Relationship Types

* One-to-one
* One-to-many
* Many-to-many

### Required Exports

* ASCII table diagram
* Mermaid ER diagram
* JSON spec

### Example Output

```txt
+----------------------+
| users                |
+----------------------+
| id          PK       |
| email       TEXT     |
| created_at  DATE     |
+----------+-----------+
           |
           | one to many
           v
+----------------------+
| orders               |
+----------------------+
| id          PK       |
| user_id     FK       |
| total       DECIMAL  |
+----------------------+
```

---

### 10.3 UI Wireframe Diagrams

### Required Objects

* Page
* Header
* Sidebar
* Main content
* Card
* Table
* Form
* Button
* Input
* Text block

### Required Layouts

* Header + sidebar + main
* Dashboard grid
* Form page
* List/detail page

### Example Output

```txt
+------------------------------------------------------------+
| Header: Inventory Dashboard                                |
+---------------+--------------------------------------------+
| Sidebar       | Main                                       |
|               | +----------------+ +----------------+      |
| - Dashboard   | | On Hand Value  | | Low Stock      |      |
| - Items       | +----------------+ +----------------+      |
| - Inbounds    |                                            |
| - Outbounds   | +--------------------------------------+   |
|               | | Recent Inventory Changes             |   |
|               | +--------------------------------------+   |
+---------------+--------------------------------------------+
```

---

### 10.4 Flowcharts

### Required Objects

* Start
* Process
* Decision
* End
* Branch labels

### Required Layouts

* Top-down
* Basic branching

### Required Exports

* ASCII
* Mermaid flowchart
* JSON spec

---

### 10.5 File Structure Diagrams

### Required Objects

* Folder
* File
* Annotation

### Example Output

```txt
project/
├── frontend/
│   ├── src/
│   │   ├── components/
│   │   └── App.tsx
├── backend/
│   └── app/
└── README.md
```

This is a useful future diagram type because developers frequently need project structure output.

---

### 10.6 Network Diagrams

### Required Objects

* Router
* Firewall
* Switch
* Server
* Client device
* Camera
* Access point
* VLAN
* Network zone

### Required Layouts

* Topology map
* Zone grouping
* Internet-to-LAN flow

---

## 11. Backend Capabilities

The first local test version should include a backend/API from the start. This avoids building a frontend-only prototype that later has to be reworked around accounts, provider credentials, persistence, imports, exports, GitHub integration, billing, and version history.

**Framework: FastAPI (Python).** This decision is closed. FastAPI provides async support, OpenAPI docs generation, and strong compatibility with the Python AI/ML ecosystem used for provider integrations.

The backend should eventually handle:

* LLM provider calls
* Prompt construction
* Spec validation
* Spec repair
* Rendering
* Saving diagrams
* Export generation
* Import and conversion
* Version history
* Provider credential storage with user-level opt-out
* Local Postgres for testing
* Cloud-hosted Postgres for production
* GitHub/repository integration
* User accounts
* Billing
* Team/project permissions later

### Suggested Backend Modules

```txt
backend/
  app/
    main.py
    config.py
    api/
      diagrams.py
      providers.py
      templates.py
      projects.py
      exports.py
    models/
      architecture.py
      database.py
      ui_wireframe.py
      flowchart.py
    providers/
      base.py
      openai_compatible.py
      ollama.py
      llamacpp.py
    validators/
      architecture_validator.py
      database_validator.py
      ui_validator.py
    renderers/
      ascii/
      mermaid/
      svg/
    services/
      diagram_generation_service.py
      diagram_render_service.py
      export_service.py
      repair_service.py
    storage/
      repositories.py
```

---

## 12. Frontend Capabilities

The V1 frontend should be a Vite + React application.

The frontend should eventually handle:

* Prompt entry
* Visual grid editing
* Structured editing
* JSON/YAML editing
* Manual ASCII editing
* Preview tabs
* Diagram library
* Template gallery
* Provider settings
* Export controls
* Import controls
* GitHub commit/PR controls
* Project organization
* Version history
* Error handling
* Onboarding

### Suggested Frontend Modules

```txt
frontend/
  src/
    app/
      App.tsx
      router.tsx
    components/
      layout/
      prompt/
      editor/
      preview/
      export/
      library/
      templates/
      settings/
    pages/
      DashboardPage.tsx
      WorkspacePage.tsx
      LibraryPage.tsx
      TemplatesPage.tsx
      SettingsPage.tsx
      ProjectPage.tsx
    renderers/
      ascii/
      mermaid/
      visual/
    types/
      architecture.ts
      database.ts
      uiWireframe.ts
      flowchart.ts
    stores/
      diagramStore.ts
      providerStore.ts
      projectStore.ts
    api/
      diagramApi.ts
      providerApi.ts
      templateApi.ts
```

---

## 13. Data Model Overview

### Spec Versioning Strategy

The diagram spec format will evolve as new diagram types and features are added. To prevent old diagrams from breaking silently, every spec document must include a `spec_version` field (e.g. `"spec_version": "1.0"`). The backend must check this field on load and either migrate the spec to the current version or return a clear error. Migration logic should be defined in `F1-diagram-spec-and-data-model.md` before committing to a spec format.

### Team Account Forward Compatibility

V1 is single-user. However, to avoid a painful schema migration when V2 team features are added, the `projects` and `diagrams` tables include a nullable `team_id` column from the start. V1 ignores this column entirely. V2 will populate it when team workspaces are introduced.

### 13.1 users

```txt
+----------------+-------------------+
| column         | purpose           |
+----------------+-------------------+
| id             | unique user id    |
| email          | login email       |
| name           | display name      |
| plan           | billing plan      |
| created_at     | creation date     |
| updated_at     | update date       |
+----------------+-------------------+
```

### 13.2 projects

```txt
+----------------+-------------------+
| column         | purpose           |
+----------------+-------------------+
| id             | unique project id |
| user_id        | owner             |
| team_id        | nullable V2 field |
| name           | project name      |
| description    | optional notes    |
| created_at     | creation date     |
| updated_at     | update date       |
+----------------+-------------------+
```

### 13.3 diagrams

```txt
+----------------+-----------------------------+
| column         | purpose                     |
+----------------+-----------------------------+
| id             | unique diagram id           |
| project_id     | project parent              |
| team_id        | nullable V2 field           |
| title          | diagram title               |
| diagram_type   | architecture/database/etc   |
| spec_json      | source of truth             |
| spec_version   | schema version for migration|
| ascii_output   | cached ASCII output         |
| mermaid_output | cached Mermaid output       |
| created_at     | creation date               |
| updated_at     | update date                 |
+----------------+-----------------------------+
```

### 13.4 templates

```txt
+----------------+-----------------------------+
| column         | purpose                     |
+----------------+-----------------------------+
| id             | unique template id          |
| title          | template title              |
| diagram_type   | template diagram type       |
| spec_json      | reusable starting spec      |
| is_system      | built-in or user-created    |
| created_by     | creator user id nullable    |
| created_at     | creation date               |
| updated_at     | update date                 |
+----------------+-----------------------------+
```

### 13.5 model_providers

```txt
+----------------+-----------------------------+
| column         | purpose                     |
+----------------+-----------------------------+
| id             | unique provider id          |
| user_id        | owner                       |
| provider_type  | ollama/openai/llamacpp/etc  |
| name           | display name                |
| base_url       | endpoint                    |
| model          | selected model              |
| api_key_ref    | secret reference            |
| is_default     | default provider flag       |
| created_at     | creation date               |
| updated_at     | update date                 |
+----------------+-----------------------------+
```

### 13.6 diagram_versions

Required for paid V1.

```txt
+----------------+-----------------------------+
| column         | purpose                     |
+----------------+-----------------------------+
| id             | unique version id           |
| diagram_id     | parent diagram              |
| spec_json      | saved spec snapshot         |
| change_summary | description of change       |
| created_at     | version date                |
+----------------+-----------------------------+
```

Version history becomes commercially valuable when users maintain real documentation over time.

### 13.7 user_settings

Useful for provider credential and privacy preferences.

```txt
+-------------------------+----------------------------------+
| column                  | purpose                          |
+-------------------------+----------------------------------+
| id                      | unique settings id               |
| user_id                 | owner                            |
| store_provider_secrets  | whether credentials are stored   |
| default_spec_format     | json or yaml                     |
| default_export_format   | preferred export format          |
| created_at              | creation date                    |
| updated_at              | update date                      |
+-------------------------+----------------------------------+
```

### 13.8 subscriptions

Required for paid SaaS launch.

```txt
+----------------+-----------------------------+
| column         | purpose                     |
+----------------+-----------------------------+
| id             | unique subscription id      |
| user_id        | owner                       |
| plan           | trial/pro/etc               |
| status         | active/trialing/canceled    |
| provider_ref   | billing provider reference  |
| current_period | billing period marker       |
| created_at     | creation date               |
| updated_at     | update date                 |
+----------------+-----------------------------+
```

---

## 14. API Overview

### 14.1 Generate Diagram

```txt
POST /api/diagrams/generate
```

Purpose:

Generate a structured spec from a prompt.

Request:

```json
{
  "prompt": "Create an architecture diagram for a React app with FastAPI and Postgres",
  "diagram_type": "architecture",
  "provider_id": "provider_123"
}
```

Response:

```json
{
  "diagram_spec": {},
  "ascii": "...",
  "mermaid": "...",
  "warnings": []
}
```

### 14.2 Render Diagram

```txt
POST /api/diagrams/render
```

Purpose:

Render an existing spec without calling AI.

### 14.3 Validate Diagram

```txt
POST /api/diagrams/validate
```

Purpose:

Check whether a diagram spec is valid.

### 14.4 Save Diagram

```txt
POST /api/diagrams
```

### 14.5 Update Diagram

```txt
PUT /api/diagrams/{diagram_id}
```

### 14.6 List Diagrams

```txt
GET /api/diagrams
```

### 14.7 Export Diagram

```txt
POST /api/diagrams/{diagram_id}/export
```

### 14.8 Test Model Provider

```txt
POST /api/providers/test
```

### 14.9 Import Diagram

```txt
POST /api/diagrams/import
```

Purpose:

Import Mermaid, JSON, YAML, or supported ASCII formats and convert them into a structured Diagram Alley spec where possible.

### 14.10 List Diagram Versions

```txt
GET /api/diagrams/{diagram_id}/versions
```

Purpose:

Show the version history for a diagram.

### 14.11 Restore Diagram Version

```txt
POST /api/diagrams/{diagram_id}/versions/{version_id}/restore
```

Purpose:

Restore or branch from a previous version.

### 14.12 GitHub Export

```txt
POST /api/integrations/github/export
```

Purpose:

Commit diagram specs and exports to a configured GitHub repository and optionally open a pull request.

---

## 15. AI Generation Requirements

### 15.1 LLM Output Should Be Structured

The model should produce JSON that conforms to a diagram schema.

It should not produce raw ASCII as the primary output.

### 15.2 Prompt Templates Should Be Diagram-Specific

Each diagram type needs a different system prompt.

### Architecture Prompt Should Ask For

* Nodes
* Edges
* Direction
* Optional groups
* Clear labels
* Valid node IDs

### Database Prompt Should Ask For

* Tables
* Columns
* Primary keys
* Foreign keys
* Relationships

### UI Prompt Should Ask For

* Page layout
* Regions
* Components
* Labels
* Hierarchy

### 15.3 Repair Pass

When model output is invalid, the app should be able to:

* Ask the model to repair the JSON
* Run local repair rules
* Remove invalid edges
* Fill missing IDs
* Normalize diagram type
* Default missing values

### 15.4 AI Modify Existing Diagram

This is more valuable than one-time generation.

The app should support prompts like:

```txt
Add Redis between the API and the database.
```

The backend should send:

* Current spec
* User change request
* Diagram schema
* Update rules

The model should return a complete updated spec or a patch operation.

A more advanced future version could use JSON Patch operations.

### 15.5 AI Failure Handling

AI should be treated as an assistive layer, not a single point of failure.

When AI generation or modification fails, the app should:

1. Preserve the user's prompt and current diagram state.
2. Retry or run a repair pass when the failure is recoverable.
3. Explain the failure clearly when it cannot be repaired.
4. Offer the manual visual editor and structured editor as the fallback path.
5. Avoid silently producing corrupted output.

The product should be designed so common failures are prevented through strong prompts, schemas, validation, and repair rules.

---

## 16. Renderer Requirements

### 16.1 ASCII Renderer

The ASCII renderer must be deterministic.

Same spec should always produce same output.

The renderer should use grid/layout metadata from the visual editor where feasible, so visual edits influence ASCII output instead of becoming separate drawing-only state.

### General Rules

* Calculate box width from content
* Pad labels consistently
* Align table columns
* Keep arrows centered where possible
* Respect diagram direction
* Avoid trailing whitespace where possible
* Use simple ASCII by default
* Allow Unicode style later
* Preserve source metadata outside the visible diagram when exported format allows it

### 16.1.1 Manual ASCII Edit Sync

Users should be able to edit ASCII directly when needed, but the structured spec remains the source of truth.

**V1 scope (decided):** Reverse sync is limited to label changes only. If a user edits a node or edge label in the ASCII output, the app will attempt to sync that change back into the structured spec. All other ASCII edits (adding nodes, removing nodes, changing connections, restructuring layout) produce a detached export only — the spec is not updated, and the app will warn the user that their ASCII is now out of sync with the spec.

Node/edge add and remove sync, and connection change sync, are deferred to V2. This boundary is firm for V1 to avoid the engineering rabbit hole of general ASCII parsing.

### 16.2 Mermaid Renderer

Mermaid export is commercially important because many developer documentation tools already support Mermaid.

Required outputs:

* Architecture as flowchart
* Database as ER diagram
* Flowchart as Mermaid flowchart

### 16.3 Visual Renderer

The visual renderer should use the same structured spec, not separate drawing data.

The first visual renderer can be simple visually, but it should still be the primary grid-based editing surface.

**Design note:** The visual grid editor is the most architecturally complex piece of the product. It is now owned by `docs/diagram-alley/specs/F4-ui-surface-workspace.md`, with editor interactions in `docs/diagram-alley/specs/D2-diagram-editor.md`. Key questions that spec must answer: how grid coordinates map to text layout positions, what library drives the canvas, how drag operations update the structured spec, and how layout metadata flows back to the renderer.

The visual renderer should support:

* SVG
* Zoom
* Pan
* Export PNG
* Grid-based drag positioning
* Node and edge selection
* Layout metadata that can inform ASCII rendering

---

## 17. Validation Requirements

Validation is one of the core product differentiators.

### Architecture Validation

* diagram_type must be architecture
* title required
* direction must be supported
* node IDs must be unique
* nodes must have labels
* edges must reference existing nodes
* edge labels optional

### Database Validation

* table names required
* column names required
* primary key markers valid
* relationships reference existing tables and columns
* relationship type supported

### UI Validation

* layout root required
* component IDs unique
* component types supported
* children valid where allowed

### Validation Output

Validation should return:

* errors
* warnings
* repair suggestions
* auto-fix availability

---

## 18. Commercial Feature Set by Version

### 18.1 Engine Proof

Goal: prove the core structured-spec-to-ASCII engine.

Features:

* Architecture spec type
* Architecture ASCII renderer
* Deterministic layout rules
* Basic grid positioning
* JSON/YAML spec shape
* Live preview
* Copy ASCII
* Basic validation

This is not the product launch. It is the proof that Diagram Alley is more than a chatbot wrapper.

### 18.2 Local Full-Stack Test Build

Goal: test the real application shape locally before commercial launch.

Features:

* Vite + React frontend
* Backend/API from the start
* Local Postgres database
* Authentication
* Dashboard
* Project and diagram storage
* Visual grid editor/canvas
* Architecture diagram support first
* ASCII export quality pass
* Provider settings for OpenAI-compatible APIs and Ollama/local endpoints
* Version history foundation
* Billing integration foundation

This build should be close enough to the real product architecture that it does not need to be thrown away.

### 18.3 Product Alpha

Goal: make the app useful enough for serious internal testing and selected early users.

Features:

* Prompt-to-spec AI generation
* AI modification of existing diagrams
* Manual visual editing
* Structured inspector/editor
* JSON/YAML spec editing
* ASCII edit reverse sync for label changes (V1 scope)
* Architecture, database, flowchart, UI wireframe, file structure, and network diagram coverage
* Built-in templates
* User-created templates
* Import existing Mermaid, JSON, YAML, and supported ASCII diagrams
* Copy/download exports
* Mermaid export
* SVG/PNG export
* Metadata in exports where appropriate
* Improved validation and repair
* Clear AI failure handling

This version should prove broad coverage while continuing to make architecture diagrams the engine-quality benchmark.

### 18.4 Paid V1

Goal: commercial release.

Required launch features:

* Single-user SaaS account model
* Billing and free trial
* Dashboard-first experience
* Polished visual grid editor/canvas
* Deterministic ASCII renderer
* Architecture diagram support with many node kinds
* Multiple diagram types
* AI generation
* AI modification of existing diagrams
* Works without AI through manual editing
* Local model support
* OpenAI-compatible cloud provider support
* BYOK provider credentials
* Optional hosted AI credits if offered
* Diagram library
* Project organization
* Multiple diagrams per project
* Built-in templates
* User-created templates
* Import and conversion
* Export center
* ASCII, Markdown, Mermaid, SVG, PNG, JSON, and YAML exports
* Export metadata linking outputs to source spec/version where appropriate
* Version history
* Optional cloud storage
* Local download/export
* GitHub commit/PR integration (stretch goal — included if core product is complete before launch)
* Strong onboarding
* Documentation

Paid V1 should be judged by whether a developer or technical writer can create, revise, save, version, and export a documentation-ready diagram faster and more reliably than using ChatGPT or Mermaid directly.

### 18.5 V2 and Team Version

Goal: expand revenue.

Features:

* Team workspaces
* Shared templates
* Shared projects
* Comments
* Role permissions
* Realtime collaboration if justified
* Documentation bundle export
* Organization model/provider settings
* Audit/version history
* Deeper GitHub/repository scanning
* AI-suggested diagrams from repo structure
* Desktop app exploration
* Template marketplace
* CLI for repo workflows

---

## 19. Product Differentiators

Diagram Alley needs a clear reason to exist.

Strong differentiators:

1. Structured specs instead of fragile raw diagrams.
2. Deterministic ASCII rendering.
3. Multiple exports from one source spec.
4. Local LLM support.
5. Manual editing after AI generation.
6. Developer documentation focus.
7. Diagram templates for real technical workflows.
8. AI modification of existing diagrams without breaking layout.
9. Works even without AI.
10. Designed for Markdown, README, PRs, and internal docs.
11. Visual grid editing backed by deterministic layout.
12. Versioned open specs that can live alongside code.
13. Import and GitHub/repository workflows.

Weak differentiators:

* AI-powered diagram generation alone
* Pretty diagrams alone
* Basic ASCII editing alone
* Generic prompt box alone

---

## 20. What the App Must Do Better Than Alternatives

Diagram Alley must be better than a chatbot at:

* Keeping alignment correct
* Preserving structure across edits
* Exporting to documentation formats
* Re-rendering after changes
* Letting users manually fix details

It must be better than manual ASCII tools at:

* Getting started quickly
* Generating initial structure
* Converting prompts into diagram specs
* Supporting multiple output formats

It must be better than Mermaid alone at:

* Producing ASCII output
* Offering form-based structured editing
* Supporting local AI generation
* Providing diagram templates and repairs

It does not need to beat full visual diagram tools at polished whiteboard collaboration.

---

## 21. Key User Experience Requirements

### 21.1 Fast First Win

A user should be able to create a useful diagram quickly.

Ideal first session:

```txt
Open app
↓
Choose Architecture
↓
Enter prompt
↓
Generate
↓
Copy Markdown-ready ASCII
```

### 21.2 Clear Error Handling

If something fails, the user should know why.

Bad error:

```txt
Invalid input.
```

Good error:

```txt
The edge from "api" to "db" references node "db", but no node with id "db" exists.
```

### 21.3 No Silent Diagram Corruption

The app should never quietly produce broken output from invalid specs. Validation must happen before rendering.

### 21.4 Easy Copy/Export

Copy and export should be one-click where possible.

### 21.5 Reversible AI Changes

When using AI to modify an existing diagram, users should be able to preview and reject changes.

---

## 22. Commercial Pricing Direction

**Status: Locked.** Pricing model decided via DEC-034 and DEC-037.

The V1 pricing model is a permanent free tier + Pro subscription.

### 22.1 Free Tier (DEC-034)

Replaces the original 14-day trial concept (DEC-010 superseded by DEC-034).

Free tier includes:

* 5 private diagrams (hard cap)
* Unlimited public diagrams
* Full export access (all formats)
* BYOK AI generation (OpenAI-compatible, Anthropic, Ollama/llama.cpp)
* 10 version snapshots per diagram
* No expiry — permanent

No hosted AI credits in V1; AI requires BYOK or local model setup (DEC-010).

### 22.2 Pro Subscription — $9/month (DEC-037)

Primary paid plan.

Includes:

* Unlimited private diagrams
* Full export bundle + Export Pack zip
* Unlimited version history
* Advanced templates
* No Diagram Alley branding on public share links
* GitHub commit integration (V1 scope per DEC-036)
* BYOK AI generation

### 22.3 Hosted AI Credits — V2 Deferred

Hosted AI is explicitly prohibited in V1 (DEC-010, F0 §8). Users must bring their own API key or use a local model.

If Diagram Alley offers built-in hosted AI in V2, usage would be controlled through included monthly credits plus optional credit packs.

### 22.4 Team Plan Later

Team billing should wait until V2 features exist:

* Shared workspaces
* Shared templates
* Role permissions
* Organization provider settings
* Admin/audit controls

---

## 23. Recommended Build Path

### Phase 1: Engine Proof

Build:

* Architecture spec
* ASCII box renderer
* Top-down architecture renderer
* Grid layout model
* JSON/YAML spec model
* Live preview
* Copy button

Goal:

Prove structured spec to ASCII rendering.

### Phase 2: Full-Stack Foundation

Build:

* Vite + React app shell
* Backend/API
* Local Postgres setup
* Authentication
* Users, projects, diagrams, diagram versions
* Provider settings
* Basic billing foundation
* Dashboard

Goal:

Test the real SaaS shape locally.

### Phase 3: Product Workspace

Build:

* Visual grid editor/canvas
* Node and edge editing
* Inspector panel
* Prompt panel
* Preview tabs for ASCII and Mermaid
* Export controls
* Save/load diagrams
* Version snapshots

Goal:

Make the app useful without AI.

### Phase 4: AI Generation and Modification

Build:

* Provider abstraction
* Ollama support
* OpenAI-compatible support
* Prompt-to-spec generation
* Repair pass
* AI modify existing diagram
* Clear AI failure handling

Goal:

Make the app faster than manual diagram creation.

### Phase 5: Broad Diagram and Import Coverage

Build:

* Database schema renderer
* Flowchart renderer
* UI wireframe renderer
* File structure renderer
* Network diagram renderer
* Many architecture node kinds
* Import Mermaid, JSON, YAML, and supported ASCII
* Best-effort ASCII reverse sync

Goal:

Support the wide-coverage product promise while keeping architecture diagrams the quality benchmark.

### Phase 6: Commercial Exports and Integrations

Build:

* Mermaid export
* SVG/PNG export
* Markdown export
* JSON/YAML export
* Export metadata
* GitHub commit/PR workflow
* Local download/export

Goal:

Make outputs immediately usable in code editors, local repos, and GitHub.

### Phase 7: Paid V1 Launch

Build:

* Billing and free trial
* Template gallery
* Built-in templates
* User-created templates
* Onboarding
* Documentation
* Polished UX
* Production cloud Postgres deployment
* Hosted AI credits if offered

Goal:

Launch paid version.

---

## 24. Commercial Success Checklist

Diagram Alley is ready to be taken seriously commercially when it can do the following:

```txt
[ ] Generate architecture diagrams from prompts
[ ] Generate database schemas from prompts
[ ] Generate UI wireframes from prompts
[ ] Render clean ASCII deterministically
[ ] Provide a visual grid editor/canvas
[ ] Export Markdown-ready diagrams
[ ] Export Mermaid diagrams
[ ] Export JSON specs
[ ] Export YAML specs
[ ] Export SVG and PNG files
[ ] Include useful source metadata in exports where appropriate
[ ] Save and reopen diagrams
[ ] Organize multiple diagrams by project
[ ] Let users manually edit generated diagrams
[ ] Let users manually edit ASCII with label-change reverse sync (V1 scope)
[ ] Validate specs and explain errors clearly
[ ] Support local model providers
[ ] Support API-based model providers
[ ] Support BYOK provider credentials
[ ] Provide useful templates
[ ] Let users create templates
[ ] Import existing Mermaid, JSON, YAML, and supported ASCII diagrams
[ ] Let users modify diagrams with AI prompts
[ ] Preserve diagram structure across edits
[ ] Provide version history
[ ] Provide billing and free trial
[ ] Provide GitHub commit/PR workflow (stretch goal — included if core product is complete before launch)
[ ] Provide a polished workspace UI
[ ] Provide onboarding and documentation
[ ] Make copying/exporting faster than using a chatbot alone
```

---

## 25. Final Product Definition

Diagram Alley should be a structured, AI-assisted diagram workspace for technical documentation.

The successful commercial version should let users move smoothly between:

```txt
Dashboard
Project
Prompt
Structured Spec
Visual Grid Editor
Manual ASCII Editor
ASCII Preview
Visual Preview
Markdown Export
Mermaid Export
SVG/PNG Export
GitHub Export
Version History
Saved Diagram Library
```

The core value is not that the app can draw boxes. The core value is that diagrams remain clean, editable, repeatable, exportable, and safe to regenerate.

Final positioning:

> Diagram Alley turns rough technical ideas into clean, editable, documentation-ready diagrams using structured specs, deterministic renderers, and optional AI assistance.

---

## 26. Planning Artifacts

This outline seeds the planning process. The planning corpus should turn this product vision into source-of-truth implementation docs.

Required planning artifacts:

1. `README.md` for Diagram Alley planning docs
2. `SPEC-INDEX.md`
3. `decisions-log.md`
4. `specs/F0-tech-stack.md`
5. `specs/F1-diagram-spec-and-data-model.md` — defines spec versioning strategy and exact JSON schema
6. `specs/F2-rendering-validation-repair.md` — defines validation, text rendering, visual export rendering, and node-kind rendering
7. `specs/F3-security-auth-billing.md` — defines auth, Stripe setup, free tier limits ($9/mo Pro per DEC-037), feature gate, permissions, and audit events
8. `specs/F4-ui-surface-workspace.md` — designs the visual grid editor and workspace surface
9. `specs/F5-api-standards.md`
10. `specs/D1-ai-generation.md`
11. `specs/D2-diagram-editor.md`
12. `specs/D3-version-history.md`
13. `specs/D4-export.md`
14. `specs/D5-import.md` — scopes Mermaid import risk
15. `specs/D6-sharing-publishing.md`
16. `specs/D7-billing-subscription.md`
17. `specs/A1-admin-console.md`
18. `specs/D8-github-integration.md` — **V1 scope** (elevated from stretch goal per DEC-036)

Resolved decisions (no longer open):

* ~~Hosted AI credit budget and limits~~ — No hosted AI in V1 (DEC-010); V2 consideration only
* ~~Exact JSON/YAML schema format and spec_version migration path~~ — Resolved in F1
* ~~Which node kinds get distinct rendering vs. collapsing to generic in V1~~ — All 20 kinds render distinctly (DEC-012); resolved in F2
* ~~Grid editor library/approach~~ — React Flow selected (DEC-013); resolved in F4
* ~~Production storage and secret-management details~~ — Fly.io secrets + Neon (DEC-015); resolved in F0
