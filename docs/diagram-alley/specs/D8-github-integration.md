---
id: D8
tier: domain
status: draft
version: "0.1"
depends-on: [F1, F3, F5]
tags: [github, oauth, commit, export, docs-as-code]
---

# D8 — GitHub Integration

**Version:** 0.1
**Status:** Draft
**Depends on:** F1, F3, F5

---

## Purpose

Enable users to commit diagram files directly to a GitHub repository, storing the blueprint and selected exports alongside source code. This turns Diagram Alley into a docs-as-code tool: diagrams live in the repo, versioned alongside the code they document.

## Non-Goals

- PR creation workflow (V2)
- GitLab or Bitbucket support (V2)
- Auto-sync on diagram save (V2; commit is always manual in V1)
- GitHub Actions integration (V2; "generate diagrams on push" workflow)
- CLI tool (`diagram-alley export docs/diagrams/payment-flow.diagram.yaml`) (V2)
- Repo scan / auto-generate diagrams from repo structure (V2; DEC-045)

---

## Owned Concepts

GitHub OAuth; GitHub connection storage; repo selection; commit operation; GitHub-connected diagram state.

---

## 1. Overview

The GitHub integration is a V1 feature (DEC-036). It allows a user to:
1. Connect their GitHub account via OAuth.
2. Select a target repo and default commit path.
3. Commit a diagram's blueprint (`.diagram.json`) and chosen export files to the repo.

The commit operation is **manual and on-demand** — it does not run automatically on every save. Users choose when to commit and which exports to include.

**File pattern committed to repo:**
```
<commit_path>/
  <slug>.diagram.json    # The canonical Diagram Alley blueprint
  <slug>.mmd             # Mermaid (if diagram type supports it)
  <slug>.md              # Markdown text output
  <slug>.svg             # SVG export
```

Any subset of these can be committed; `.diagram.json` is always included (it is the source of truth for re-import).

---

## 2. GitHub OAuth

### 2.1 OAuth App

A separate GitHub OAuth App is registered for Diagram Alley. It is distinct from the main FastAPI Users auth flow.

- **Scopes:** `repo` (full repo access for private repos; use `public_repo` for public-only users)
- **Env vars:** `GITHUB_CLIENT_ID`, `GITHUB_CLIENT_SECRET`, `GITHUB_REDIRECT_URI`

### 2.2 Connection Flow

```
1. User clicks "Connect GitHub" in Settings → Providers (F4 route: /settings/providers)
2. Frontend redirects to GitHub OAuth authorize URL with state parameter (CSRF protection)
3. User approves on GitHub; GitHub redirects to GITHUB_REDIRECT_URI with code + state
4. Backend validates state, exchanges code for access token via GitHub token endpoint
5. Backend stores encrypted GitHub access token in a new `github_connections` table row
6. Frontend shows "Connected as <github_username>" in settings
```

### 2.3 `github_connections` Table

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | UUID | PK | — |
| `user_id` | UUID | FK → users.id, NOT NULL, UNIQUE | One connection per user |
| `github_user_id` | TEXT | NOT NULL | GitHub user ID (numeric) |
| `github_username` | TEXT | NOT NULL | GitHub login for display |
| `access_token_encrypted` | TEXT | NOT NULL | GitHub OAuth token, encrypted at rest (same scheme as BYOK: AES-256-GCM via F3 §4.2) |
| `default_repo` | TEXT | NULLABLE | `owner/repo` slug of the user's default repo |
| `default_commit_path` | TEXT | NOT NULL, default `'docs/diagrams'` | Default path inside repo for committed files |
| `created_at` | TIMESTAMPTZ | NOT NULL, default now() | — |
| `updated_at` | TIMESTAMPTZ | NOT NULL, default now() | — |

**Index:** `(user_id)` for connection lookup.

**Encryption:** GitHub access tokens are encrypted using the same AES-256-GCM scheme as BYOK keys (F3 §4.2). The token is never sent to the client.

### 2.4 Disconnect

`DELETE /api/v1/github/connection` removes the `github_connections` row and clears `default_repo`. Previously committed files remain in GitHub.

---

## 3. Repo Selection

### 3.1 List User Repos

**`GET /api/v1/github/repos`** (requires auth + GitHub connection)

Calls GitHub API `GET /user/repos?per_page=100&sort=updated` using the user's stored access token. Returns a simplified list:

```json
{
  "data": [
    {
      "full_name": "lujima96/my-project",
      "description": "Main service repo",
      "private": false,
      "default_branch": "main",
      "updated_at": "2026-05-22T10:00:00Z"
    }
  ]
}
```

Repos are sorted by last updated (most recent first). Response is not cached in V1 — each call hits the GitHub API.

### 3.2 Set Default Repo and Path

**`PATCH /api/v1/github/connection`** (requires auth + GitHub connection)

```json
{
  "default_repo": "lujima96/my-project",
  "default_commit_path": "docs/diagrams"
}
```

Updates `github_connections.default_repo` and `default_commit_path`. Validates that the repo exists in the user's accessible repos before saving. Returns `404 REPO_NOT_FOUND` if not accessible.

---

## 4. Commit Operation

### 4.1 Endpoint

**`POST /api/v1/diagrams/{id}/github-commit`** (requires auth + GitHub connection)

### 4.2 Request Body

```json
{
  "repo": "lujima96/my-project",
  "commit_path": "docs/diagrams",
  "branch": "main",
  "commit_message": "Update payment-flow diagram",
  "include_formats": ["json", "mermaid", "markdown", "svg"]
}
```

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| `repo` | No | `default_repo` from connection | `owner/repo` slug |
| `commit_path` | No | `default_commit_path` from connection | Path inside repo |
| `branch` | No | repo's default branch | Target branch |
| `commit_message` | No | `"Update <slug> diagram via Diagram Alley"` | Commit message |
| `include_formats` | No | `["json", "mermaid", "markdown", "svg"]` | Which files to commit |

`"json"` (`.diagram.json`) is always included regardless of `include_formats`.

### 4.3 Service Steps

```
1. Load diagram (ownership check)
2. Check GitHub connection exists; decrypt access token
3. Validate spec_json (F2 validate_spec); fail if errors present
4. Derive slug from diagram title (same slugification as D4 §5)
5. Generate requested formats using D4 export logic (same renderers)
6. For each format, call GitHub Contents API:
   GET /repos/{owner}/{repo}/contents/{path}/{slug}.{ext}
     → if file exists: record its SHA
   PUT /repos/{owner}/{repo}/contents/{path}/{slug}.{ext}
     → body: { message, content (base64), branch, sha (if update) }
7. Emit DIAGRAM_COMMITTED audit event with detail: {repo, path, formats_committed}
8. Return commit result
```

### 4.4 Response

```json
{
  "data": {
    "repo": "lujima96/my-project",
    "branch": "main",
    "commit_path": "docs/diagrams",
    "files_committed": [
      "docs/diagrams/payment-flow.diagram.json",
      "docs/diagrams/payment-flow.mmd",
      "docs/diagrams/payment-flow.md",
      "docs/diagrams/payment-flow.svg"
    ],
    "commit_sha": "abc123def456...",
    "commit_url": "https://github.com/lujima96/my-project/commit/abc123..."
  }
}
```

### 4.5 Error Cases

| Condition | Response |
|-----------|----------|
| No GitHub connection | `403 GITHUB_NOT_CONNECTED` |
| Repo not found or not accessible | `404 REPO_NOT_FOUND` |
| Branch not found | `404 BRANCH_NOT_FOUND` |
| Spec has validation errors | `400 SPEC_VALIDATION_FAILED` |
| GitHub API rate limit hit | `429 UPSTREAM_RATE_LIMITED` |
| GitHub token expired/revoked | `401 GITHUB_TOKEN_INVALID` (prompt reconnect) |

---

## 5. Audit Event

| Event | Trigger | Actor | Entity |
|-------|---------|-------|--------|
| `DIAGRAM_COMMITTED` | GitHub commit succeeded | user | diagrams |

`detail` shape: `{ "repo": "owner/repo", "branch": "main", "commit_path": "docs/diagrams", "formats": ["json", "mermaid", "markdown", "svg"] }`

This event is appended to F3 §3.2 audit events table.

---

## 6. GitHub Integration UI

### 6.1 Settings — Providers Page

The `/settings/providers` page (F4) gains a "GitHub" section below the AI provider list:

```
+------------------------------------------------------------------+
| GitHub                                                           |
+------------------------------------------------------------------+
| Not connected                                                    |
| [Connect GitHub]                                                 |
+------------------------------------------------------------------+
```

Once connected:

```
+------------------------------------------------------------------+
| GitHub                                                           |
+------------------------------------------------------------------+
| Connected as: lujima96                                          |
| Default repo:  lujima96/my-project          [Change ▼]          |
| Default path:  docs/diagrams                [Edit]               |
|                                                                  |
| [Disconnect]                                                     |
+------------------------------------------------------------------+
```

### 6.2 Workspace — Commit Button

The workspace toolbar (F4 §3.1) gains a **[Commit to GitHub]** button, visible when the user has a GitHub connection. It is grayed out with a tooltip "Connect GitHub in Settings" when not connected.

Clicking opens the commit modal:

```
+-------------------------------------------------------+
| Commit to GitHub                                [×]   |
+-------------------------------------------------------+
| Repo:    lujima96/my-project          [Change ▼]      |
| Path:    docs/diagrams/                               |
| Branch:  main                         [Change]        |
|                                                       |
| Files to commit:                                      |
| ☑ payment-flow.diagram.json  (blueprint — always)    |
| ☑ payment-flow.mmd           (Mermaid)               |
| ☑ payment-flow.md            (Markdown)              |
| ☑ payment-flow.svg           (SVG)                   |
|                                                       |
| Commit message:                                       |
| [Update payment-flow diagram via Diagram Alley____]   |
+-------------------------------------------------------+
| [Commit]                            [Cancel]          |
+-------------------------------------------------------+
```

On commit success, show a toast: "Committed to lujima96/my-project · [View on GitHub →]" (links to `commit_url`).

### 6.3 Availability

GitHub integration is a **Pro-only feature** (DEC-039). The commit button is visible to free users but grayed out with an upgrade prompt tooltip: "GitHub commit requires Pro — upgrade to unlock."

Free users who click the button see an inline upgrade prompt, not the commit modal.

---

## 7. Feature Gate

GitHub integration requires:
- Verified email
- `plan = 'pro'` active subscription (DEC-039; returns `403 PLAN_REQUIRED` for free users)
- Active GitHub connection (`github_connections` row exists; returns `403 GITHUB_NOT_CONNECTED` if missing)

`PLAN_REQUIRED` is checked before `GITHUB_NOT_CONNECTED`. A free user without a GitHub connection sees `403 PLAN_REQUIRED`, not `403 GITHUB_NOT_CONNECTED`.

The `github_connections` table row can only be created by Pro users. Free users cannot initiate the OAuth flow.

---

## Open Questions

None. D8 is V1 scope (DEC-036); Pro-only gate set by DEC-039.

---

## Acceptance Criteria

- `GET /api/v1/github/repos` without a GitHub connection returns `403 GITHUB_NOT_CONNECTED`.
- `GET /api/v1/github/repos` with a valid connection returns at least the list of repos accessible to the user.
- `POST /api/v1/diagrams/{id}/github-commit` with no connection returns `403 GITHUB_NOT_CONNECTED`.
- `POST /api/v1/diagrams/{id}/github-commit` on a diagram with validation errors returns `400 SPEC_VALIDATION_FAILED`.
- `POST /api/v1/diagrams/{id}/github-commit` commits `.diagram.json` regardless of `include_formats`.
- A successful commit returns a `commit_sha` and `commit_url` that resolves to a real GitHub commit.
- Committing the same diagram twice creates an update (not a duplicate file) — the SHA from the first commit is used in the second `PUT` request.
- `DIAGRAM_COMMITTED` audit event is written after every successful commit.
- Revoking the GitHub connection via `DELETE /api/v1/github/connection` removes the `github_connections` row; subsequent commit attempts return `403 GITHUB_NOT_CONNECTED`.
- An expired GitHub token returns `401 GITHUB_TOKEN_INVALID` and prompts the user to reconnect.
- The commit button in the workspace toolbar is grayed out for users without a GitHub connection.
- Free users can commit diagrams (GitHub is not Pro-gated).
