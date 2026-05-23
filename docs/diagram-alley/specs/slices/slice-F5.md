---
phase: 11
slice: F5
spec: specs/F5-api-standards.md
status: planned
---

# Slice F5 — API and Integration Standards

## Pre-conditions

- [ ] Slice F0 complete — FastAPI app starts, `GET /health` responds.
- [ ] Slice F3 complete — JWT auth middleware, ownership dependency, and subscription gate are implemented and testable.

Note: F5 is cross-cutting infrastructure applied to all routes. Wire it before any domain router (D1–D7) is added.

---

## Build Sequence

1. **Response envelope middleware** — Establish the standard success envelope from F5 §6.1. Create `app/schemas/response.py` with generic `DataResponse[T]` and `ListResponse[T]` Pydantic models. All endpoints return these shapes — no bare model returns, no top-level arrays. Action endpoints return `{"data": {...}}`. List endpoints return `{"data": [...], "meta": {...}}`. (→ F5 §6)

2. **Error shape and exception handlers** — Create `app/api/errors.py`. Register FastAPI exception handlers for: `RequestValidationError` → 422 `UNPROCESSABLE_ENTITY`, `HTTPException` → pass-through with standardised `{"error": {"code": ..., "message": ..., "detail": null}}` body. Create a catch-all handler for unhandled exceptions that returns 500 `INTERNAL_ERROR` with no stack trace in production (`ENVIRONMENT != 'local'`). Define all error code string constants in `app/constants/error_codes.py`. (→ F5 §8–9)

3. **`TOKEN_EXPIRED` distinction** — Ensure the JWT auth middleware returns `401 TOKEN_EXPIRED` (not the generic `UNAUTHORIZED`) specifically when the token is present but expired. FastAPI Users' default is a generic 401; override the `on_error` callback to check for expiry and set the correct code. (→ F5 §9.1)

4. **Pagination helpers** — Create `app/services/pagination.py` with a `paginate(query, page, page_size)` helper that applies `.offset().limit()` to a SQLAlchemy query and returns `(items, total)`. Create a `PaginationMeta` Pydantic model and a `paginated_response(items, total, page, page_size)` builder. Apply to every list endpoint uniformly. Default `page_size=24`, max `page_size=100`. (→ F5 §7)

5. **`slowapi` rate limiting** — Install `slowapi`. Register a `Limiter` instance with `key_func=get_remote_address` for IP-based limits and `key_func=get_user_id` for user-based limits. Apply the limits from F5 §12 to the auth and AI endpoints as decorators. All other routes share the 300 req/min per-user default. Return `429 RATE_LIMITED` with a `Retry-After` header on breach. (→ F5 §12)

6. **Frontend API client** — Build out `src/api/client.ts` beyond the health-check stub. Implement `apiFetch(path, options)` that: prepends `VITE_API_BASE_URL + '/api/v1'`, attaches `Authorization: Bearer <token>` from the Zustand auth store, parses the `{"data": ...}` envelope and returns the inner data, and handles errors per F5 §9.2: `401 TOKEN_EXPIRED` triggers silent token refresh + one retry; `403 SUBSCRIPTION_REQUIRED` redirects to `/settings/billing`; `502 PROVIDER_UNAVAILABLE` surfaces in the prompt panel without crashing. All other errors surface as toasts. (→ F5 §4, §9.2)

7. **Endpoint stubs for all routes** — Register all routes from the F5 §10 endpoint index as FastAPI router stubs that return `501 Not Implemented` with `{"error": {"code": "NOT_IMPLEMENTED", "message": "..."}}`. This ensures the full URL surface exists and can be tested for auth and routing correctness before domain logic is added. Domain slices replace stubs one by one. (→ F5 §10)

8. **CORS pre-flight** — Confirm all routes accept OPTIONS pre-flight from allowed origins. This was wired in F0 §6 but validate it holds now that all routes are registered. A pre-flight `OPTIONS /api/v1/diagrams` from `localhost:5173` must return 200 with correct CORS headers. (→ F5 §13, F0 §6)

9. **Integration test harness** — Set up `pytest` with `httpx.AsyncClient` against the FastAPI app (in-memory test client, not a running server). Add fixtures for: a test database (SQLite or a test Postgres), a registered verified user with an active trial subscription, an auth token. These fixtures are the base for all domain spec tests. (→ F5 Acceptance Criteria)

---

## Done Criteria

- [ ] `GET /api/v1/diagrams` without a token returns `401 {"error": {"code": "UNAUTHORIZED", ...}}`. (→ F5 Acceptance Criteria)
- [ ] `GET /api/v1/diagrams/{id}` for a diagram owned by a different user returns `403 {"error": {"code": "FORBIDDEN", ...}}`. (→ F5 Acceptance Criteria)
- [ ] `GET /api/v1/diagrams?page=2&page_size=10` returns at most 10 results with `meta.page = 2`. (→ F5 Acceptance Criteria)
- [ ] `POST /api/v1/diagrams/generate` from a user with `status = 'canceled'` returns `403 SUBSCRIPTION_REQUIRED`. (→ F5 Acceptance Criteria)
- [ ] `POST /auth/login` attempted 11 times in 1 minute from the same IP returns `429 RATE_LIMITED` on the 11th attempt. (→ F5 Acceptance Criteria)
- [ ] A request with an expired access token returns `401 TOKEN_EXPIRED` (not generic `UNAUTHORIZED`). (→ F5 Acceptance Criteria)
- [ ] `GET /health` returns `200 {"status": "ok"}` without any auth token. (→ F5 Acceptance Criteria)
- [ ] All error responses follow the `{"error": {"code": ..., "message": ..., "detail": ...}}` shape — no stack traces in production. (→ F5 §8–9)
- [ ] Frontend `apiFetch` silently retries once after a `401 TOKEN_EXPIRED` using the refresh cookie. (→ F5 §9.2)
- [ ] `OPTIONS /api/v1/diagrams` from `localhost:5173` returns 200 with correct CORS headers. (→ F5 §13)
