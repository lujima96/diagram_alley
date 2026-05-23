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

3. **`on_after_register` hook** — In `UserManager.on_after_register`, create a Stripe Customer (via `stripe.Customer.create(email=user.email, metadata={"user_id": str(user.id)})`), create the `subscriptions` row (`plan='trial'`, `status='trialing'`, `trial_ends_at=now()+14d`, `access_ends_at=trial_ends_at`, `stripe_customer_id=<id>`), create the `user_settings` row with defaults, and emit the `SUBSCRIPTION_TRIAL_STARTED` and `USER_REGISTERED` audit events. (→ F3 §5.2, F1 §4.9, DEC-010)

4. **Audit log write helper** — Create `app/services/audit_service.py::log_event(db, event, entity_type, entity_id, user_id, detail, ip_address)`. Uses the event constants from F3 §3.2, stored as string literals in `app/constants/audit_events.py`. Confirm appending a row and that no update/delete paths exist on the audit log. (→ F3 §3.1–3.3)

5. **BYOK encryption** — Create `app/services/encryption.py`. Derive an AES-256 Fernet key from `SECRET_KEY` using HKDF with a stable salt. Expose `encrypt(value: str) -> str` and `decrypt(value: str) -> str`. Store base64 ciphertext in `api_key_encrypted`. Confirm round-trip: `decrypt(encrypt("sk-abc"))` returns `"sk-abc"`. Never return decrypted keys in API responses. (→ F3 §4.2)

6. **Ownership dependency** — Create `app/dependencies/auth.py` with a `get_current_user` FastAPI dependency (from FastAPI Users) and a `require_owner(resource_user_id, current_user)` helper that raises HTTP 403 `FORBIDDEN` if the IDs do not match. All service-layer functions that read or write user-owned resources must call this check. (→ F3 §2.2)

7. **Superuser guard** — Create a `require_superuser(current_user)` dependency that raises HTTP 403 if `user.is_superuser` is false. Apply to all `/api/v1/admin/*` routes. (→ F3 §2.2)

8. **Subscription feature gate** — Implement `require_active_subscription(user, db)` in `app/services/billing_service.py` exactly as specified in F3 §5.5: check `sub.status in ('trialing', 'active')` and `sub.access_ends_at > now()`. Raises `SubscriptionRequiredError` caught by the API layer as `403 SUBSCRIPTION_REQUIRED`. Apply to AI generation endpoints only in this slice; domain specs wire it to their own endpoints. (→ F3 §5.5)

9. **Stripe webhook handler** — Implement `POST /webhooks/stripe` in `app/api/webhooks.py`. Verify signature with `stripe.Webhook.construct_event`. Dispatch on event type per F3 §5.4: `checkout.session.completed`, `invoice.payment_succeeded`, `invoice.payment_failed`, `customer.subscription.updated`, `customer.subscription.deleted`. Handler must be idempotent — processing the same event twice must produce the same DB state. Emit the appropriate audit event for each. (→ F3 §5.4)

10. **Trial expiry job** — Create a simple daily check in `app/jobs/expire_trials.py`: query `subscriptions` rows where `plan='trial'`, `status='trialing'`, `trial_ends_at <= now()`, update `status='canceled'`, emit `SUBSCRIPTION_EXPIRED`. Wire as a FastAPI startup background task or, preferably, as a Fly.io cron machine (one daily `fly machine run` invocation). (→ F3 §5.2)

---

## Done Criteria

- [ ] `POST /auth/register` creates a user with `is_verified = false` and a `subscriptions` row with `status = 'trialing'`. (→ F3 Acceptance Criteria)
- [ ] `POST /auth/login` returns an access token (response body) and sets a `refresh_token` httpOnly cookie. Access token expires in 15 minutes. (→ F3 Acceptance Criteria)
- [ ] A request with an expired access token but a valid refresh cookie receives a new access token from `POST /auth/refresh`. (→ F3 Acceptance Criteria)
- [ ] `POST /auth/refresh` with an already-used refresh token returns 401 (rotation enforced). (→ F3 Acceptance Criteria)
- [ ] `GET /api/v1/diagrams` without a token returns 401. (→ F3 Acceptance Criteria)
- [ ] Accessing a diagram owned by a different user returns 403 `FORBIDDEN`. (→ F3 Acceptance Criteria)
- [ ] `model_providers.api_key_encrypted` decrypts to the original API key using the server's encryption key. (→ F3 Acceptance Criteria)
- [ ] A user with `status = 'canceled'` calling a gated AI endpoint receives 403 `SUBSCRIPTION_REQUIRED`. (→ F3 Acceptance Criteria)
- [ ] A Stripe `checkout.session.completed` webhook with a valid signature updates the subscription to `status = 'active'`. (→ F3 Acceptance Criteria)
- [ ] A Stripe webhook with an invalid signature returns 400 without processing. (→ F3 §5.4)
- [ ] Processing the same Stripe webhook event twice produces the same DB state (idempotency). (→ FM-04)
- [ ] Audit log rows are never updated or deleted — only appended. Confirmed by absence of update/delete code paths. (→ F3 §3.3)
