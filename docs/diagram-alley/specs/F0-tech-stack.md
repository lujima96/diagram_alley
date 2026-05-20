---
id: F0
tier: foundation
status: draft
version: "0.1"
depends-on: []
tags: [tech-stack, deployment, infra]
---

# F0 — Tech Stack and Deployment

**Version:** 0.1
**Status:** Draft
**Depends on:** —

---

## Purpose

Define the languages, frameworks, runtimes, databases, hosting platforms, environment model, and future constraints for Diagram Alley V1. All other specs assume the choices made here.

## Non-Goals

- Detailed API or data model contracts (→ F1, F5)
- Auth implementation details (→ F3)
- Frontend UI architecture (→ F4)
- Rendering engine design (→ F2)

---

## Canonical Terms

All terms used here are defined in `glossary.md`. No new terms are introduced in F0.

---

## Owned Decisions

All decisions in this spec are locked in `decisions-log.md`. References are cited inline.

---

## 1. Languages and Runtimes

| Layer | Language | Runtime | Version target |
|-------|----------|---------|----------------|
| Backend | Python | CPython | 3.12+          |
| Frontend | TypeScript | Node.js (build only) | Node 20 LTS |
| Database | SQL | Postgres | 16+ |

**Python package manager:** `uv` (fast, deterministic lockfile, replaces pip + venv for V1).
**JS package manager:** `pnpm` (fast installs, strict lockfile, workspace support for potential monorepo).

---

## 2. Frameworks and Libraries

### 2.1 Backend — FastAPI

- **Framework:** FastAPI (Python) — DEC-003
- **ASGI server:** Uvicorn (production: `uvicorn --workers 2`)
- **ORM:** SQLAlchemy 2.x (async) with Alembic for migrations
- **Validation:** Pydantic v2 (already a FastAPI dependency)
- **Auth:** FastAPI Users — DEC-017
- **HTTP client (for provider calls):** `httpx` (async)
- **SVG/PNG conversion:** `cairosvg` or equivalent lightweight SVG-to-PNG converter for V1 PNG export — DEC-019
- **Task queue:** None in V1. Long-running export jobs run inline if under 5s; async background tasks via FastAPI's `BackgroundTasks` for anything longer. A proper queue (Celery + Redis, or Dramatiq) is deferred to V2 if needed.

### 2.2 Frontend — Vite + React

- **Build tool:** Vite 5+
- **UI framework:** React 18+ with TypeScript
- **Routing:** React Router v6
- **State management:** Zustand (lightweight, no boilerplate — preferred over Redux for V1 scope)
- **Grid editor:** React Flow — DEC-013
- **Styling:** Tailwind CSS v3
- **Code editor (JSON/YAML inspector):** CodeMirror 6
- **HTTP client:** `fetch` via a thin wrapper; no Axios dependency
- **Form handling:** React Hook Form

### 2.3 Database — Postgres

- **ORM layer:** SQLAlchemy 2.x (async engine with `asyncpg` driver)
- **Migrations:** Alembic
- **Local dev:** Local Postgres instance (Docker-compose `postgres:16` service)
- **Production:** Neon free tier (serverless Postgres) — DEC-014

---

## 3. Deployment Targets

### 3.1 Environments

| Environment | Purpose | Backend host | Frontend host | Database |
|-------------|---------|-------------|--------------|----------|
| `local` | Developer machine | `uvicorn` on `localhost:8000` | `vite dev` on `localhost:5173` | Local Postgres via Docker |
| `preview` | Branch deploy (automatic) | — (not deployed) | Vercel preview URL | — |
| `production` | Live product | Fly.io — DEC-015 | Vercel — DEC-016 | Neon free tier — DEC-014 |

`staging` is deferred. If needed before V2, it will be a second Fly.io app + Neon branch.

### 3.2 Backend (Fly.io)

- Deployed as a Docker container (`Dockerfile` in `/backend`)
- Config: `fly.toml` in `/backend`
- App name: `diagram-alley-api`
- Region: `lax` (US West) as default; can change before launch
- Health check: `GET /health` → 200 JSON `{"status": "ok"}`
- Secrets: injected via `fly secrets set KEY=VALUE` — never committed to source

### 3.3 Frontend (Vercel)

- Root: `/frontend`
- Build command: `pnpm build`
- Output dir: `dist`
- Environment variables set in Vercel dashboard
- CORS: FastAPI must whitelist `https://diagram-alley.vercel.app` and `https://*.vercel.app` (preview URLs) — see §6

### 3.4 Database (Neon)

- Connection string stored as `DATABASE_URL` env var on Fly.io and in `.env.local`
- Neon branching used for dev/staging if needed (one branch = one Postgres database)
- Connection pooling: Neon's built-in pooler (`?sslmode=require&pgbouncer=true`) for production

---

## 4. Repository Structure

```
diagram-alley/
├── backend/
│   ├── Dockerfile
│   ├── fly.toml
│   ├── pyproject.toml          # uv project manifest
│   ├── alembic.ini
│   ├── alembic/
│   │   └── versions/
│   └── app/
│       ├── main.py
│       ├── config.py
│       ├── database.py
│       ├── api/
│       │   ├── diagrams.py
│       │   ├── projects.py
│       │   ├── providers.py
│       │   ├── templates.py
│       │   ├── exports.py
│       │   ├── imports.py
│       │   └── versions.py
│       ├── models/             # SQLAlchemy ORM models
│       ├── schemas/            # Pydantic request/response schemas
│       ├── services/
│       │   ├── diagram_service.py
│       │   ├── render_service.py
│       │   ├── export_service.py
│       │   ├── import_service.py
│       │   ├── ai_service.py
│       │   └── version_service.py
│       ├── renderers/
│       │   ├── ascii/
│       │   ├── mermaid/
│       │   └── visual/
│       ├── validators/
│       ├── providers/
│       │   ├── base.py
│       │   ├── openai_compatible.py
│       │   ├── ollama.py
│       │   └── llamacpp.py
│       └── auth/               # FastAPI Users config
└── frontend/
    ├── package.json
    ├── vite.config.ts
    ├── tailwind.config.ts
    ├── tsconfig.json
    └── src/
        ├── main.tsx
        ├── App.tsx
        ├── router.tsx
        ├── components/
        ├── pages/
        ├── stores/
        ├── api/
        ├── types/
        └── renderers/
```

---

## 5. Environment Variables

### Backend (`.env` locally; Fly.io secrets in production)

| Variable | Required | Description |
|----------|----------|-------------|
| `DATABASE_URL` | Yes | Postgres connection string (Neon in prod, local in dev) |
| `SECRET_KEY` | Yes | FastAPI Users JWT signing secret (min 32 chars, random) |
| `ENVIRONMENT` | Yes | `local` \| `production` |
| `ALLOWED_ORIGINS` | Yes | Comma-separated CORS origins |
| `STRIPE_SECRET_KEY` | Yes (prod) | Stripe secret key |
| `STRIPE_WEBHOOK_SECRET` | Yes (prod) | Stripe webhook signing secret |
| `STRIPE_PRICE_ID_PRO` | Yes (prod) | Stripe Price ID for Pro plan |

All secrets are **never committed to source control**. Local `.env` is in `.gitignore`. Production secrets are set via `fly secrets set`.

### Frontend (Vercel dashboard or `.env.local`)

| Variable | Description |
|----------|-------------|
| `VITE_API_BASE_URL` | Backend API URL (`http://localhost:8000` locally; Fly.io URL in prod) |
| `VITE_STRIPE_PUBLISHABLE_KEY` | Stripe publishable key |

Frontend env vars prefixed `VITE_` are bundled at build time — do not put secrets here.

---

## 6. CORS Configuration

FastAPI must allow cross-origin requests from:

- `http://localhost:5173` (local Vite dev server)
- `https://diagram-alley.vercel.app` (production Vercel)
- `https://*.vercel.app` (Vercel preview deployments)

`ALLOWED_ORIGINS` env var controls this list. The backend reads it and configures `CORSMiddleware` at startup. Preview URL wildcard (`*.vercel.app`) is acceptable in staging context only; production should pin the exact domain.

---

## 7. Local Development Setup

```
# Clone repo
git clone <repo>

# Start Postgres
docker compose up -d postgres

# Backend
cd backend
uv sync
uv run alembic upgrade head
uv run uvicorn app.main:app --reload --port 8000

# Frontend (separate terminal)
cd frontend
pnpm install
pnpm dev
```

`docker-compose.yml` in the root provides:
- `postgres:16` on port 5432
- `DATABASE_URL=postgresql+asyncpg://postgres:postgres@localhost:5432/diagram_alley`

---

## 8. Future Constraints

These constraints are locked for planning. Implementations must not violate them.

1. **No AI provider lock-in.** The provider abstraction layer (`providers/base.py`) must allow adding or swapping providers without touching diagram generation logic.
2. **No hosted AI in V1.** The product ships BYOK-first (DEC-010). Do not design billing or generation logic around hosted AI credits — that is a V2 commercial decision.
3. **Desktop app is out of scope for V1.** All stack decisions assume a web SaaS. If a desktop Electron/Tauri app is explored in V2, the backend must remain a separate service (no Electron-bundled backend).
4. **Team accounts are V2.** All data models include forward-compat columns but V1 code must not implement team-scoped queries or permissions (DEC-001, DEC-009).
5. **Neon free tier limits.** Free Neon allows 3 GB storage and 10 branches. If these are hit before revenue, upgrade to Neon's paid tier. This is not an architecture concern — just an ops watch item.
6. **Fly.io free tier limits.** 3 shared VMs, 160 GB/month outbound. Monitor with Fly.io metrics. No code changes required if limits are hit — upgrade to a paid machine.
7. **PNG export is V1.** PNG export is generated from the deterministic SVG export path, not from a screenshot-only browser workflow (DEC-019).

---

## Open Questions

None. All F0 decisions are locked in `decisions-log.md`.

---

## Acceptance Criteria

- `docker compose up` starts a local Postgres that passes `alembic upgrade head` without errors.
- `uvicorn app.main:app --reload` starts and `GET /health` returns `200 {"status": "ok"}`.
- `pnpm dev` starts the Vite dev server and the app loads in a browser without console errors.
- `fly deploy` from `/backend` deploys the Docker container to Fly.io successfully.
- `vercel deploy` from `/frontend` produces a working preview URL.
- CORS: the Vite dev server can call the local FastAPI and the Vercel production URL can call the Fly.io backend without CORS errors.
