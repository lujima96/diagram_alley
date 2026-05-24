---
phase: 11
slice: D8
spec: specs/D8-github-integration.md
status: planned
---

# Slice D8 — GitHub Integration

## Pre-conditions

- [ ] Slice F1 complete — `users` table exists; `audit_log` table exists.
- [ ] Slice F3 complete — BYOK encryption service (`app/services/encryption.py`) available; `require_verified` dependency available.
- [ ] Slice F5 complete — endpoint stubs for GitHub routes registered; error code convention established.
- [ ] GitHub OAuth App registered; `GITHUB_CLIENT_ID`, `GITHUB_CLIENT_SECRET`, `GITHUB_REDIRECT_URI` available as env vars.

Note: D8 can be built independently of D1–D7 (except for the pre-conditions above). It reads diagram data and calls GitHub APIs; it does not depend on billing, AI generation, or export endpoints.

---

## Build Sequence

1. **`github_connections` table and migration** — Create the `GithubConnection` ORM model in `app/models/` with all columns from D8 §2.3. Run `alembic revision --autogenerate -m "add github_connections"`. Verify with `\d github_connections` in psql. (→ D8 §2.3)

2. **GitHub OAuth flow** — Implement the OAuth flow in `app/api/github.py`:
   - `GET /api/v1/github/auth` — generates a state token (CSRF), stores it in the session or a short-lived cache, returns the GitHub authorize URL with `client_id`, `scope=repo`, `state`
   - `GET /api/v1/github/callback` — validates state, exchanges `code` for access token via `POST https://github.com/login/oauth/access_token`, calls `GET https://api.github.com/user` to get `id` and `login`, upserts `github_connections` row with encrypted token
   Guard callback with `require_current_user`. (→ D8 §2.2)

3. **Token encryption** — Reuse `app/services/encryption.py` (from F3) for GitHub access tokens. Store encrypted value in `github_connections.access_token_encrypted`. Never return the raw token to the client. (→ D8 §2.3, F3 §4.2)

4. **Connection endpoints** — Implement:
   - `GET /api/v1/github/connection` — returns `{connected: bool, github_username, default_repo, default_commit_path}` (no token in response)
   - `PATCH /api/v1/github/connection` — updates `default_repo` and `default_commit_path`; validates repo is accessible before saving
   - `DELETE /api/v1/github/connection` — removes the row; returns 204
   All require auth. Return `404 GITHUB_NOT_CONNECTED` if no row exists for the user. (→ D8 §2.4, §3.2)

5. **Repo list endpoint** — Implement `GET /api/v1/github/repos`. Decrypt GitHub token; call `GET https://api.github.com/user/repos?per_page=100&sort=updated` with `Authorization: Bearer <token>`. Return simplified list per D8 §3.1. Handle `401` from GitHub as `401 GITHUB_TOKEN_INVALID`. (→ D8 §3.1)

6. **Commit endpoint** — Implement `POST /api/v1/diagrams/{id}/github-commit` per D8 §4:
   - Ownership check on diagram
   - Decrypt GitHub token
   - Validate `spec_json`
   - Derive slug (reuse `derive_filename` from D4 export service, strip extension)
   - Generate requested formats using F2 renderers
   - For each file: GET existing SHA from GitHub Contents API; PUT new content (base64-encoded) with SHA if updating
   - Emit `DIAGRAM_COMMITTED` audit event
   - Return commit result with `commit_sha`, `commit_url`, `files_committed[]`
   Handle GitHub API errors per D8 §4.5. (→ D8 §4)

7. **Workspace commit button** — Add "[Commit to GitHub]" button to the workspace toolbar (next to Export). Grayed with tooltip when not connected. On click, opens `<GitHubCommitModal>`. (→ D8 §6.2)

8. **Commit modal** — Implement `<GitHubCommitModal>`: repo selector (calls `GET /api/v1/github/repos`), path input, branch input, file checkboxes (`.diagram.json` always checked and disabled), commit message input. On submit, calls the commit endpoint. On success, shows toast with "View on GitHub →" link. (→ D8 §6.2)

9. **Settings — providers page GitHub section** — Add GitHub connection UI to `/settings/providers`: show connected/disconnected state, username when connected, default repo/path fields, Disconnect button. "Connect GitHub" button triggers the OAuth flow via `GET /api/v1/github/auth`. (→ D8 §6.1)

---

## Done Criteria

- [ ] `GET /api/v1/github/repos` by a free user returns `403 PLAN_REQUIRED`. (→ DEC-039)
- [ ] `GET /api/v1/github/repos` without a GitHub connection (Pro user) returns `403 GITHUB_NOT_CONNECTED`. (→ D8 Acceptance Criteria)
- [ ] `GET /api/v1/github/repos` with a valid Pro connection returns repos accessible to the user. (→ D8 Acceptance Criteria)
- [ ] `POST /api/v1/diagrams/{id}/github-commit` by a free user returns `403 PLAN_REQUIRED`. (→ DEC-039)
- [ ] `POST /api/v1/diagrams/{id}/github-commit` with no GitHub connection (Pro user) returns `403 GITHUB_NOT_CONNECTED`. (→ D8 Acceptance Criteria)
- [ ] `POST /api/v1/diagrams/{id}/github-commit` on a diagram with validation errors returns `400 SPEC_VALIDATION_FAILED`. (→ D8 Acceptance Criteria)
- [ ] Commit always includes `.diagram.json` regardless of `include_formats`. (→ D8 Acceptance Criteria)
- [ ] A successful commit returns a `commit_sha` and `commit_url` that resolves on GitHub. (→ D8 Acceptance Criteria)
- [ ] Committing the same diagram twice creates an update (uses the existing file's SHA in the second PUT). (→ D8 Acceptance Criteria)
- [ ] `DIAGRAM_COMMITTED` audit event is written after every successful commit. (→ D8 §5)
- [ ] `DELETE /api/v1/github/connection` removes the row; subsequent commit attempts return `403 GITHUB_NOT_CONNECTED`. (→ D8 Acceptance Criteria)
- [ ] An expired/revoked GitHub token returns `401 GITHUB_TOKEN_INVALID`. (→ D8 Acceptance Criteria)
- [ ] GitHub commit is available to Pro users only; free users see an upgrade prompt, not the commit modal. (→ D8 §7, DEC-039)
- [ ] Commit button in workspace toolbar shows an upgrade prompt tooltip for free users; shows "Connect GitHub in Settings" tooltip for connected Pro users without a connection configured. (→ D8 Acceptance Criteria)
