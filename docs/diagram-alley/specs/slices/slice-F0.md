---
phase: 11
slice: F0
spec: specs/F0-tech-stack.md
status: planned
---

# Slice F0 — Tech Stack and Deployment

## Pre-conditions

None. F0 has no spec dependencies. This slice is the starting point for the entire build.

External prerequisites:
- [x] Git repository created and empty (or initialized with only planning docs).
- [x] Fly.io account created; `flyctl` installed locally.
- [x] Cloudflare account created; `wrangler` CLI installed locally (`pnpm add -g wrangler`).
- [x] Neon account created; a project and `main` branch database provisioned; connection string in hand.
- [ ] Stripe account created; a Pro price ID, secret key, and publishable key available (can be test-mode keys for initial setup).
- [ ] Docker Desktop (or Docker Engine) installed locally.

---

## Build Sequence

1. **Scaffold repository structure** — Create `backend/` and `frontend/` directories matching the layout in F0 §4. Add `.gitignore` entries for `.env`, `.env.local`, `__pycache__`, `node_modules`, `dist`, `.venv`. (→ F0 §4)

2. **Docker Compose for local Postgres** — Write `docker-compose.yml` at the repo root with a `postgres:16` service on port 5432, database `diagram_alley`, user/password `postgres`. Confirm `docker compose up -d postgres` starts cleanly. (→ F0 §7)

3. **FastAPI skeleton** — In `backend/`, initialize a `uv` project (`pyproject.toml`). Add FastAPI, Uvicorn, SQLAlchemy 2.x (async), asyncpg, Alembic, Pydantic v2, httpx, fastapi-users[sqlalchemy] as dependencies. Create `app/main.py` with `GET /health → 200 {"status": "ok"}`. Confirm `uv run uvicorn app.main:app --reload` starts and the health endpoint responds. (→ F0 §2.1, §3.2)

4. **Environment variable wiring (backend)** — Create `app/config.py` with a Pydantic `Settings` class reading all variables from F0 §5 (`DATABASE_URL`, `SECRET_KEY`, `ENVIRONMENT`, `ALLOWED_ORIGINS`, `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `STRIPE_PRICE_ID_PRO`). Add `.env` sample file (no real secrets) and `.env.example`. (→ F0 §5)

5. **Database connection and Alembic setup** — Create `app/database.py` with an async SQLAlchemy engine reading `DATABASE_URL`. Initialize Alembic (`alembic init alembic`); point `alembic.ini` at the same `DATABASE_URL`. Run `alembic upgrade head` (empty migration head) against the local Postgres to confirm connectivity. (→ F0 §2.3, §7)

6. **CORS middleware** — Wire `CORSMiddleware` in `app/main.py` reading `ALLOWED_ORIGINS` from config. Confirm that a local `fetch` from `localhost:5173` to `localhost:8000/health` does not produce a CORS error. (→ F0 §6)

7. **Vite + React + TypeScript frontend scaffold** — In `frontend/`, run `pnpm create vite . --template react-ts`. Add dependencies: React Router v6, Zustand, Tailwind CSS v3, React Flow, CodeMirror 6, React Hook Form. Add `VITE_API_BASE_URL` and `VITE_STRIPE_PUBLISHABLE_KEY` to `.env.local`. Confirm `pnpm dev` starts and the app loads in a browser without console errors. (→ F0 §2.2, §5)

8. **Frontend API client stub** — Create `src/api/client.ts` with a thin `apiFetch` wrapper that reads `VITE_API_BASE_URL`, sets `Content-Type: application/json`, and calls `GET /health`. Confirm the health response can be read from the React app. (→ F0 §2.2, F5 §4 — auth header wiring comes in slice-F3/F5)

9. **Dockerfile for backend** — Write `backend/Dockerfile`: Python 3.12 slim base, `uv sync --frozen`, `CMD uvicorn app.main:app --host 0.0.0.0 --port 8080`. Build and run locally: `docker build -t diagram-alley-api . && docker run -p 8080:8080 diagram-alley-api`. Confirm `/health` responds. (→ F0 §3.2)

10. **`fly.toml` and initial Fly.io deploy** — Write `backend/fly.toml` (app: `diagram-alley-api`, region: `lax`, internal port: 8080, health check path `/health`). Run `fly deploy`. Set `DATABASE_URL`, `SECRET_KEY`, `ENVIRONMENT=production`, `ALLOWED_ORIGINS` as Fly.io secrets. Confirm the deployed health endpoint returns 200. (→ F0 §3.2)

11. **Cloudflare Pages deploy for frontend** — Connect the `frontend/` directory to a Cloudflare Pages project (dashboard or `wrangler pages deploy dist`). Set build command `pnpm build`, output directory `dist`. Set `VITE_API_BASE_URL` to the Fly.io backend URL as a Cloudflare Pages environment variable. Deploy and confirm the preview URL loads and the health fetch works without CORS errors. (→ F0 §3.3, §6)

---

## Done Criteria

- [ ] `docker compose up -d postgres` starts a local Postgres that passes `alembic upgrade head` without errors. (→ F0 Acceptance Criteria)
- [ ] `uv run uvicorn app.main:app --reload` starts and `GET /health` returns `200 {"status": "ok"}`. (→ F0 Acceptance Criteria)
- [ ] `pnpm dev` starts the Vite dev server and the app loads in a browser without console errors. (→ F0 Acceptance Criteria)
- [ ] `fly deploy` from `/backend` deploys the Docker container to Fly.io successfully and `/health` responds 200 on the live URL. (→ F0 Acceptance Criteria)
- [ ] Cloudflare Pages deploy produces a working preview URL that loads the app without console errors. (→ F0 Acceptance Criteria)
- [ ] CORS: the Vite dev server can call the local FastAPI (`localhost:5173` → `localhost:8000`) and the Vercel production URL can call the Fly.io backend without CORS errors. (→ F0 Acceptance Criteria, §6)
- [ ] No secrets are committed to the repository. `.env` and `.env.local` are present in `.gitignore`. (→ F0 §5)
- [ ] All F0 §8 future constraints are respected: no hosted AI logic, no team-scoped code, provider abstraction layer directory exists (empty). (→ F0 §8)
