---
phase: 11
slice: F3
spec: specs/F3-security-auth-billing.md
status: planned
---

# Slice F3 — Security, Auth, Provider Credentials, and Billing

## Pre-conditions

- [ ] Slice F0 complete — Fly.io app deployed, env var wiring in place (`SECRET_KEY`, `STRIPE_*` vars set as Fly.io secrets).
- [ ] Slice F1 complete — `users`, `subscriptions`, `model_providers`, and `audit_log` tables exist and migrations run cleanly.
- [ ] Stripe account created; test-mode `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, and `STRIPE_PRICE_ID_PRO` available.
- [ ] Transactional email provider configured (Resend recommended); `RESEND_API_KEY` or SMTP credentials available as env vars.

---

## Build Sequence

1. **FastAPI Users wiring** — Install and configure `fastapi-users[sqlalchemy]` in `app/auth/`. Define the `UserManager` class, register the JWT backend (HS256, 15-minute access tokens, 30-day rotating refresh tokens). Wire `POST /auth/register`, `/auth/login`, `/auth/logout`, `/auth/refresh`, `/auth/forgot-password`, `/auth/reset-password`, `/auth/verify` routes. Confirm registration creates a `users` row with `is_verified = false`. (→ F3 §1.1–1.3)

2. **Refresh token cookie delivery** — Configure FastAPI Users to set an httpOnly, Secure, SameSite=Lax cookie for the refresh token on login. Access token is returned in the response body only — not set as a cookie. Confirm a login response sets the cookie and a `/auth/refresh` call with the cookie returns a new access token and rotates the refresh token. (→ F3 §1.1)

3. **`on_after_register` hook** — In `UserManager.on_after_register`, create the `subscriptions` row (`plan='free'`, `status='active'`, no `trial_ends_at`, no Stripe interaction), create the `user_settings` row with defaults, and emit the `USER_REGISTERED` audit event. Stripe Customer creation is deferred to the first checkout session (F3 §5.3). (→ F3 §5.2, F1 §4.9, DEC-034)

4. **Audit log write helper** — Create `app/services/audit_service.py::log_event(db, event, entity_type, entity_id, user_id, detail, ip_address)`. Uses the event constants from F3 §3.2, stored as string literals in `app/constants/audit_events.py`. Confirm appending a row and that no update/delete paths exist on the audit log. (→ F3 §3.1–3.3)

5. **BYOK encryption** — Create `app/services/encryption.py`. Derive an AES-256 Fernet key from `SECRET_KEY` using HKDF with a stable salt. Expose `encrypt(value: str) -> str` and `decrypt(value: str) -> str`. Store base64 ciphertext in `api_key_encrypted`. Confirm round-trip: `decrypt(encrypt("sk-abc"))` returns `"sk-abc"`. Never return decrypted keys in API responses. (→ F3 §4.2)

6. **Ownership dependency** — Create `app/dependencies/auth.py` with a `get_current_user` FastAPI dependency (from FastAPI Users) and a `require_owner(resource_user_id, current_user)` helper that raises HTTP 403 `FORBIDDEN` if the IDs do not match. All service-layer functions that read or write user-owned resources must call this check. (→ F3 §2.2)

7. **Superuser guard** — Create a `require_superuser(current_user)` dependency that raises HTTP 403 if `user.is_superuser` is false. Apply to all `/api/v1/admin/*` routes. (→ F3 §2.2)

8. **Subscription feature gate** — Implement four gate functions in `app/services/billing_service.py` per F3 §5.5: `require_verified(user)` (blocks diagram creation and AI for unverified users), `require_diagram_quota(user, db)` (counts non-archived private diagrams; raises `DiagramLimitError` → `403 DIAGRAM_LIMIT_REACHED` at 5 for free users), `require_pro(user, db)` (raises `PlanRequiredError` → `403 PLAN_REQUIRED` for free users; applied to JSON/YAML/ASCII/Markdown export and GitHub commit), `require_active_for_ai(user, db)` (checks verified + active status). Apply diagram quota gate to diagram creation endpoints; apply AI gate to AI generation endpoints; apply Pro gate to blueprint export and GitHub commit endpoints. (→ F3 §5.5, DEC-034, DEC-038, DEC-039)

9. **Stripe webhook handler** — Implement `POST /webhooks/stripe` in `app/api/webhooks.py`. Verify signature with `stripe.Webhook.construct_event`. Dispatch on event type per F3 §5.4: `checkout.session.completed` (upgrade to Pro), `invoice.payment_succeeded` (extend period), `invoice.payment_failed` (past_due), `customer.subscription.updated` (cancel_at_period_end), `customer.subscription.deleted` (revert to `plan='free', status='active'`). Handler must be idempotent — processing the same event twice must produce the same DB state. Emit the appropriate audit event for each. (→ F3 §5.4, DEC-034)

---

## Done Criteria

- [ ] `POST /auth/register` creates a user with `is_verified = false` and a `subscriptions` row with `plan = 'free'`, `status = 'active'` (no `trial_ends_at`). (→ F3 Acceptance Criteria, DEC-034)
- [ ] `POST /auth/login` returns an access token (response body) and sets a `refresh_token` httpOnly cookie. Access token expires in 15 minutes. (→ F3 Acceptance Criteria)
- [ ] A request with an expired access token but a valid refresh cookie receives a new access token from `POST /auth/refresh`. (→ F3 Acceptance Criteria)
- [ ] `POST /auth/refresh` with an already-used refresh token returns 401 (rotation enforced). (→ F3 Acceptance Criteria)
- [ ] `GET /api/v1/diagrams` without a token returns 401. (→ F3 Acceptance Criteria)
- [ ] Accessing a diagram owned by a different user returns 403 `FORBIDDEN`. (→ F3 Acceptance Criteria)
- [ ] `model_providers.api_key_encrypted` decrypts to the original API key using the server's encryption key. (→ F3 Acceptance Criteria)
- [ ] A free user who already has 5 private diagrams calling `POST /api/v1/diagrams` receives 403 `DIAGRAM_LIMIT_REACHED`. (→ F3 Acceptance Criteria, DEC-034)
- [ ] A free user calling `POST /api/v1/diagrams/{id}/export` with `format=json` receives 403 `PLAN_REQUIRED`. (→ DEC-038)
- [ ] A free user calling `POST /api/v1/diagrams/{id}/github-commit` receives 403 `PLAN_REQUIRED`. (→ DEC-039)
- [ ] An unverified user calling `POST /api/v1/diagrams` receives 403 `EMAIL_VERIFICATION_REQUIRED`. (→ F3 Acceptance Criteria)
- [ ] A Stripe `checkout.session.completed` webhook with a valid signature updates the subscription to `plan = 'pro'`, `status = 'active'`. (→ F3 Acceptance Criteria)
- [ ] A `customer.subscription.deleted` webhook reverts the subscription to `plan = 'free'`, `status = 'active'`. (→ F3 Acceptance Criteria, DEC-034)
- [ ] A Stripe webhook with an invalid signature returns 400 without processing. (→ F3 §5.4)
- [ ] Processing the same Stripe webhook event twice produces the same DB state (idempotency). (→ FM-04)
- [ ] Audit log rows are never updated or deleted — only appended. Confirmed by absence of update/delete code paths. (→ F3 §3.3)
