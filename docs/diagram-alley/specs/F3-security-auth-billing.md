---
id: F3
tier: foundation
status: draft
version: "0.1"
depends-on: [F0, F1]
tags: [auth, security, billing, stripe, byok, permissions, audit]
---

# F3 — Security, Auth, Provider Credentials, and Billing

**Version:** 0.1
**Status:** Draft
**Depends on:** F0, F1

---

## Purpose

Define authentication, session management, BYOK credential storage, permission conventions, audit event conventions, and billing integration. Domain specs reference the conventions defined here — they do not redefine them.

## Non-Goals

- Data model table definitions (→ F1)
- API naming conventions (→ F5)
- Provider routing and AI generation logic (→ D1)
- Billing domain workflow (→ D7)

---

## Owned Concepts

Auth flow; JWT token model; BYOK encryption; permission key convention; audit event convention; Stripe setup; trial enforcement; `subscriptions` state machine (consumption — owned by F1).

---

## 1. Authentication

### 1.1 Library and Token Model (DEC-017)

**Library:** FastAPI Users with JWT backend.

**Access token:**
- Algorithm: HS256
- Lifetime: 15 minutes
- Payload: `{ "sub": "<user_uuid>", "aud": "fastapi-users:auth", "exp": <unix_ts> }`
- Signing secret: `SECRET_KEY` env var (min 32 random bytes, never committed)

**Refresh token:**
- Stored in the `access_tokens` table managed by FastAPI Users
- Lifetime: 30 days
- Rotates on every use (old token invalidated, new token issued)
- Cookie delivery option: httpOnly, Secure, SameSite=Lax cookie for web clients — preferred over localStorage to reduce XSS exposure

**Token delivery to frontend:**
- Login response body includes `{ "access_token": "...", "token_type": "bearer", "refresh_token": "..." }`
- Frontend stores access token in memory (React state / Zustand store) — not in localStorage
- Refresh token stored in an httpOnly Secure cookie (set by the backend on login)
- Frontend requests a token refresh automatically when the access token expires (401 response triggers silent refresh)

### 1.2 Registration and Email Verification

FastAPI Users handles registration. The flow:

1. `POST /auth/register` with `{email, password}`
2. FastAPI Users creates a `users` row with `is_verified = false`
3. A verification email is sent with a one-time token link
4. `POST /auth/verify` with the token sets `is_verified = true`
5. `is_verified = false` users can log in but see a verification banner in the UI. They cannot use AI generation (pro feature) or save more than 3 diagrams until verified.

**Email sending:** In V1, use a transactional email provider with a free tier. **Resend** (free up to 3000 emails/month) is recommended. SMTP credentials are env vars.

### 1.3 Password Reset

Standard FastAPI Users flow:
1. `POST /auth/forgot-password` with `{email}` — sends reset link
2. `POST /auth/reset-password` with `{token, password}` — updates hash, invalidates all refresh tokens for that user

### 1.4 Auth on Every Route

All API routes except the public list below require a valid access token. See F5 for the exact header convention.

**Public routes (no auth required):**
- `GET /health`
- `POST /auth/register`
- `POST /auth/login`
- `POST /auth/refresh`
- `POST /auth/forgot-password`
- `POST /auth/reset-password`
- `GET /share/{share_token}` (view-only shared diagrams — D6)

---

## 2. Permission System

### 2.1 Permission Key Convention

All permission keys follow the pattern: `domain.entity.action`

- `domain`: the product area (e.g., `diagram`, `project`, `provider`, `billing`, `admin`)
- `entity`: the noun being acted on (e.g., `diagram`, `version`, `template`, `subscription`)
- `action`: the verb (e.g., `create`, `read`, `update`, `delete`, `restore`, `export`, `import`)

Examples:
- `diagram.diagram.create`
- `diagram.version.restore`
- `provider.provider.create`
- `billing.subscription.upgrade`
- `admin.user.read`

### 2.2 V1 Role Model

V1 is single-user with no team roles. Two roles exist:

| Role | Who | Permissions |
|------|-----|------------|
| `user` | Any authenticated user | All `diagram.*`, `project.*`, `provider.*`, `billing.*` permissions on their own resources |
| `superuser` | Admin (set via CLI or DB) | All `admin.*` permissions; read access to all user data for support |

**Ownership enforcement:** All non-admin routes verify that the resource being accessed belongs to the requesting user via `project.user_id = request.user.id` or `diagram → project → user_id` chain. This is enforced in the service layer, not just the API layer.

### 2.3 Permission Table (V1)

| Permission Key | Role | Description |
|----------------|------|-------------|
| `diagram.diagram.create` | user | Create a new diagram |
| `diagram.diagram.read` | user | Read own diagrams |
| `diagram.diagram.update` | user | Update own diagram spec |
| `diagram.diagram.delete` | user | Archive or delete own diagram |
| `diagram.diagram.export` | user | Export own diagram |
| `diagram.diagram.import` | user | Import and create diagram from external format |
| `diagram.version.read` | user | Read version history |
| `diagram.version.restore` | user | Restore a previous version |
| `project.project.create` | user | Create a project |
| `project.project.read` | user | Read own projects |
| `project.project.update` | user | Rename, archive project |
| `project.project.delete` | user | Archive project |
| `provider.provider.create` | user | Add a model provider |
| `provider.provider.read` | user | Read own providers |
| `provider.provider.update` | user | Update provider config |
| `provider.provider.delete` | user | Remove a provider |
| `billing.subscription.read` | user | Read own subscription status |
| `billing.subscription.upgrade` | user | Start or upgrade subscription |
| `billing.subscription.cancel` | user | Cancel subscription |
| `admin.user.read` | superuser | Read any user's profile |
| `admin.user.update` | superuser | Update any user (support actions) |
| `admin.subscription.update` | superuser | Manually adjust a subscription |

---

## 3. Audit Events

### 3.1 Convention

All audit events are past-tense or noun-verb constants, SCREAMING_SNAKE_CASE.

Audit events are stored in a lightweight `audit_log` table (see §3.3) — no external logging service in V1.

### 3.2 V1 Audit Events

| Event | Trigger | Actor | Entity |
|-------|---------|-------|--------|
| `USER_REGISTERED` | Registration | user | users |
| `USER_VERIFIED` | Email verify | user | users |
| `USER_LOGIN` | Login | user | users |
| `USER_PASSWORD_RESET` | Password reset | user | users |
| `DIAGRAM_CREATED` | Diagram created | user | diagrams |
| `DIAGRAM_UPDATED` | Spec written | user | diagrams |
| `DIAGRAM_ARCHIVED` | Archived | user | diagrams |
| `DIAGRAM_DELETED` | Hard deleted | user | diagrams |
| `DIAGRAM_EXPORTED` | Export downloaded | user | diagrams |
| `DIAGRAM_IMPORTED` | Import created | user | diagrams |
| `VERSION_CREATED` | Version snapshot | user | diagram_versions |
| `VERSION_RESTORED` | Version restored | user | diagram_versions |
| `PROJECT_CREATED` | Project created | user | projects |
| `PROJECT_ARCHIVED` | Project archived | user | projects |
| `PROVIDER_CREATED` | Provider added | user | model_providers |
| `PROVIDER_DELETED` | Provider removed | user | model_providers |
| `SHARE_CREATED` | Share link created | user | diagram_shares |
| `SHARE_REVOKED` | Share link revoked | user | diagram_shares |
| `SHARE_ACCESSED` | Public share viewed | system (anonymous) | diagram_shares |
| `DIAGRAM_AI_GENERATED` | AI generation succeeded | user | diagrams |
| `DIAGRAM_AI_IMPROVE_PROPOSED` | AI improve proposal returned | user | diagrams |
| `SUBSCRIPTION_TRIAL_STARTED` | Registration | system | subscriptions |
| `SUBSCRIPTION_UPGRADED` | Payment succeeded | user/stripe | subscriptions |
| `SUBSCRIPTION_CANCELED` | Canceled | user/stripe | subscriptions |
| `SUBSCRIPTION_EXPIRED` | Trial ended | stripe | subscriptions |

### 3.3 `audit_log` Table

| Column | Type | Description |
|--------|------|-------------|
| `id` | UUID | PK |
| `user_id` | UUID | Actor (nullable for system events) |
| `event` | TEXT | Event constant (e.g. `DIAGRAM_CREATED`) |
| `entity_type` | TEXT | Table name (e.g. `diagrams`) |
| `entity_id` | UUID | Row ID |
| `detail` | JSONB | Event-specific context |
| `ip_address` | TEXT | Request IP (nullable) |
| `created_at` | TIMESTAMPTZ | Event time |

Audit log rows are append-only. No update or delete operations on this table.

---

## 4. BYOK Credential Storage

### 4.1 Threat Model

BYOK API keys are user-supplied cloud provider credentials. They are sensitive. The risk is a database breach exposing all stored keys. V1 mitigation: encrypt at rest with a server-side key.

### 4.2 Encryption Scheme

- **Algorithm:** AES-256-GCM (via Python `cryptography` library, `Fernet` symmetric encryption)
- **Key source:** Derived from `SECRET_KEY` env var via HKDF with a stable salt. One encryption key for all BYOK values. (Per-user keys are a V2 hardening option.)
- **Storage:** Encrypted bytes stored as base64 in `model_providers.api_key_encrypted`
- **Decryption:** Only in the server process, never sent to the client. The frontend never sees raw API keys.

### 4.3 Credential Opt-Out

Users can set `store_credentials = false` on a `model_providers` row. When this is true:
- `api_key_encrypted` is set to NULL
- The user must re-enter the API key on each session (frontend holds it in memory only for the session)
- The provider config (base_url, model name) is still stored

---

## 5. Billing

### 5.1 Stripe Setup (DEC-004)

**Stripe objects to create before launch:**

| Object | Name | Config |
|--------|------|--------|
| Stripe Customer | created on trial start | `metadata: {user_id: "<uuid>"}` |
| Stripe Product | Diagram Alley Pro | — |
| Stripe Price | Pro Monthly | Recurring, monthly, amount TBD |

**Env vars:** `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `STRIPE_PRICE_ID_PRO` (set via `fly secrets set`)

### 5.2 Trial Lifecycle (DEC-010)

On user registration, the backend:
1. Creates a Stripe Customer with the user's email
2. Creates a `subscriptions` row with `plan = 'trial'`, `status = 'trialing'`, `trial_ends_at = now() + 14 days`, `stripe_customer_id = <id>`
3. Emits `SUBSCRIPTION_TRIAL_STARTED` audit event

On trial expiry (Stripe webhook `customer.subscription.deleted` or a daily cron check against `trial_ends_at`):
1. Update `subscriptions.status = 'canceled'`
2. Emit `SUBSCRIPTION_EXPIRED` audit event
3. User loses access to AI generation features (not the editor — they keep access to manually editing/exporting existing diagrams)

**Trial access rules:**
- Full editor access: yes
- AI generation: yes (requires BYOK/Ollama — no hosted AI, DEC-010)
- Unlimited saved diagrams: yes
- All export formats: yes
- After trial expires: AI generation disabled, saved diagrams remain accessible read-only, upgrade prompt shown

### 5.3 Upgrade to Pro

1. Frontend calls `POST /billing/checkout-session`
2. Backend creates a Stripe Checkout Session for the Pro price
3. Frontend redirects to Stripe-hosted checkout
4. On success, Stripe fires `checkout.session.completed` webhook
5. Backend creates Stripe Subscription, updates `subscriptions` row: `plan = 'pro'`, `status = 'active'`
6. Emits `SUBSCRIPTION_UPGRADED`

### 5.4 Webhook Handling

All Stripe webhooks arrive at `POST /webhooks/stripe`. The backend:
1. Verifies the webhook signature with `STRIPE_WEBHOOK_SECRET`
2. Dispatches on `event.type`:
   - `checkout.session.completed` → upgrade flow
   - `invoice.payment_succeeded` → extend `current_period_end`
   - `invoice.payment_failed` → set `status = 'past_due'`
   - `customer.subscription.deleted` → set `status = 'canceled'`

Webhook endpoint is **not** auth-protected (Stripe calls it externally). It is signature-verified only.

### 5.5 Feature Gate

The service layer checks subscription status before permitting AI generation:

```python
def require_active_subscription(user: User, db: Session) -> None:
    sub = db.query(Subscription).filter_by(user_id=user.id).first()
    if not sub or sub.status not in ('trialing', 'active'):
        raise SubscriptionRequiredError("Active subscription or trial required")
```

Gated features in V1:
- AI diagram generation (`diagram.ai.generate`)
- AI diagram modification (`diagram.ai.modify`)

Non-gated features (always available to authenticated users):
- Manual editor
- All export formats
- Version history (read)
- Project management
- Provider configuration

---

## 6. Security Checklist

- [ ] `SECRET_KEY` is at least 32 random bytes, never committed
- [ ] `STRIPE_SECRET_KEY` and `STRIPE_WEBHOOK_SECRET` are Fly.io secrets, never in source
- [ ] Refresh tokens stored in httpOnly cookie, not localStorage
- [ ] Access tokens stored in memory only (React Zustand store), not localStorage
- [ ] BYOK keys encrypted at rest (AES-256-GCM)
- [ ] All routes except public list verify access token
- [ ] All resource reads check `user_id` ownership in the service layer
- [ ] Stripe webhooks are signature-verified before processing
- [ ] Email verification token is single-use and expires (FastAPI Users default: 24h)
- [ ] Password reset token is single-use and expires (FastAPI Users default: 1h)

---

## Open Questions

None. All F3 decisions are locked.

---

## Acceptance Criteria

- `POST /auth/register` creates a user with `is_verified = false` and a `subscriptions` row with `status = 'trialing'`.
- `POST /auth/login` returns an access token that expires in 15 minutes and sets a refresh cookie.
- A request with an expired access token but a valid refresh cookie receives a new access token.
- `POST /auth/refresh` with an already-used refresh token returns 401 (token rotation enforced).
- Accessing `GET /diagrams` without a token returns 401.
- Accessing a diagram owned by a different user returns 403 (ownership check in service layer).
- A user with `status = 'canceled'` subscription calling `POST /diagrams/generate` receives 403 with a clear upgrade message.
- A Stripe `checkout.session.completed` webhook with a valid signature updates the subscription to `status = 'active'`.
- `model_providers.api_key_encrypted` decrypts to the original API key value using the server's encryption key.
