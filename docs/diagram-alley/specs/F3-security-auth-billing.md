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

Auth flow; JWT token model; BYOK encryption; permission key convention; audit event convention and constants; Stripe setup; free tier and Pro feature gate enforcement; `subscriptions` state machine consumption (table and states owned by F1).

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
5. `is_verified = false` users can log in but see a persistent verification banner in the UI. AI generation and new diagram creation are available once verified; unverified users cannot create diagrams or use AI generation until they verify.

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
- `POST /auth/verify`
- `GET /api/v1/share/{token}` (view-only shared diagram API — D6)

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

Audit events are stored in the lightweight `audit_log` table owned by F1 §4.10 — no external logging service in V1.

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
| `SUBSCRIPTION_UPGRADED` | Payment succeeded; plan upgraded to Pro | user/stripe | subscriptions |
| `SUBSCRIPTION_CANCELED` | Pro canceled (reverts to free tier at period end) | user/stripe | subscriptions |
| `USER_FREE_TIER_LIMIT_REACHED` | Free user attempted to create a 6th private diagram | user | diagrams |

### 3.3 Audit Log Storage

F1 §4.10 owns the `audit_log` table shape. F3 owns the event constants above and the rule that audit log rows are append-only. No update or delete operations are exposed in V1.

---

## 4. BYOK Credential Storage

### 4.0 Supported AI Providers at Launch (DEC-042)

V1 supports three provider types:

| Provider type | `provider_type` value | Notes |
|---|---|---|
| OpenAI and OpenAI-compatible endpoints | `openai_compatible` | Includes Azure OpenAI, OpenRouter, local OpenAI-compatible servers |
| Anthropic (Claude) | `anthropic` | Direct Anthropic API |
| Ollama / llama.cpp | `ollama` | Local models; no API key required |

Product copy should say "Supports OpenAI, Anthropic, and Ollama at launch" — not "any AI provider." A provider adapter system for additional integrations is planned for V1.x.

**Security statement for product copy (DEC-048):** OpenAI and Anthropic API keys are encrypted server-side before storage. Requests to those providers route through the Diagram Alley backend — your keys are never sent to the browser. Ollama uses a local connection for localhost instances and does not require storing an API key. Diagram Alley does not resell model usage or maintain AI credits on your behalf.

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

### 5.1 Stripe Setup (DEC-004, DEC-037)

**Free tier users do not require a Stripe Customer.** A Stripe Customer is created lazily the first time a user initiates a checkout session (upgrade to Pro).

**Stripe objects to create before launch:**

| Object | Name | Config |
|--------|------|--------|
| Stripe Customer | created on first checkout | `metadata: {user_id: "<uuid>"}` |
| Stripe Product | Diagram Alley Pro | — |
| Stripe Price | Pro Monthly | Recurring, monthly, **$9.00 USD** (DEC-037) |

**Env vars:** `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `STRIPE_PRICE_ID_PRO` (set via `fly secrets set`)

### 5.2 Free Tier Lifecycle (DEC-034, replaces DEC-010)

On user registration, the backend:
1. Creates a `subscriptions` row with `plan = 'free'`, `status = 'active'` — no Stripe interaction required at registration
2. Emits `USER_REGISTERED` audit event (no separate subscription event)

There is no trial expiry job. Free tier is permanent. The daily job that previously checked `trial_ends_at` is removed.

**Free tier access rules:**
- Full editor access: yes
- AI generation: yes (requires BYOK/Ollama — no hosted AI, DEC-010)
- Private diagrams: up to 5 (hard cap enforced by feature gate in §5.5)
- Public diagrams: unlimited
- Export formats: Mermaid, PNG, SVG downloads; ASCII/Markdown copy with attribution comment (DEC-038, DEC-049); JSON/YAML, ASCII/Markdown file download, and Documentation Bundle are Pro-only
- GitHub commit: not available (Pro only, DEC-039)
- Version history: 14-day rolling window; versions older than 14 days are pruned (DEC-040)
- After 5 private diagrams: diagram creation blocked with `403 DIAGRAM_LIMIT_REACHED`; existing diagrams remain fully editable

### 5.3 Upgrade to Pro

1. Frontend calls `POST /billing/checkout-session`
2. Backend creates (or retrieves) a Stripe Customer for the user, storing `stripe_customer_id` in the `subscriptions` row
3. Backend creates a Stripe Checkout Session for the Pro price with the customer ID
4. Frontend redirects to Stripe-hosted checkout
5. On success, Stripe fires `checkout.session.completed` webhook
6. Backend updates `subscriptions` row: `plan = 'pro'`, `status = 'active'`, `stripe_subscription_id = <id>`
7. Emits `SUBSCRIPTION_UPGRADED`

### 5.4 Webhook Handling

All Stripe webhooks arrive at `POST /webhooks/stripe`. The backend:
1. Verifies the webhook signature with `STRIPE_WEBHOOK_SECRET`
2. Dispatches on `event.type`:
   - `checkout.session.completed` → upgrade flow
   - `invoice.payment_succeeded` → extend `current_period_end`
   - `invoice.payment_failed` → set `status = 'past_due'`
   - `customer.subscription.updated` with `cancel_at_period_end=true` → keep `status = 'active'`, set `cancel_at_period_end = true`, set `access_ends_at = current_period_end`
   - `customer.subscription.deleted` → revert `plan = 'free'`, `status = 'active'`, clear `stripe_subscription_id`, set `canceled_at = now()` (DEC-034: downgrade to free, not hard lock)

Webhook endpoint is **not** auth-protected (Stripe calls it externally). It is signature-verified only.

### 5.5 Feature Gate (DEC-034)

The service layer enforces plan-based limits. Two gate functions:

```python
def require_verified(user: User) -> None:
    """Blocks diagram creation and AI generation for unverified users."""
    if not user.is_verified:
        raise UnverifiedError("Email verification required")

def require_diagram_quota(user: User, db: Session) -> None:
    """Blocks private diagram creation when free user hits 5-diagram cap."""
    sub = db.query(Subscription).filter_by(user_id=user.id).first()
    if sub and sub.plan == 'free':
        count = db.query(func.count(Diagram.id)).filter_by(
            project__user_id=user.id, is_archived=False
        ).scalar()
        if count >= 5:
            emit_audit(user.id, 'USER_FREE_TIER_LIMIT_REACHED')
            raise DiagramLimitError("Free plan limit reached: upgrade to Pro for unlimited diagrams")

def require_pro(user: User, db: Session) -> None:
    """Blocks access to Pro-only features (blueprint exports, GitHub commit)."""
    sub = db.query(Subscription).filter_by(user_id=user.id).first()
    if not sub or sub.plan != 'pro':
        raise PlanRequiredError("This feature requires a Pro plan")

def require_active_for_ai(user: User, db: Session) -> None:
    """AI generation requires verified email. Available on free and Pro plans."""
    require_verified(user)
    sub = db.query(Subscription).filter_by(user_id=user.id).first()
    if not sub or sub.status not in ('active',):
        raise SubscriptionError("Active subscription required")
    if sub.access_ends_at and sub.access_ends_at <= now():
        raise SubscriptionError("Subscription access has ended")
```

**Feature availability by plan:**

| Feature | Unverified | Free (verified) | Pro |
|---------|-----------|-----------------|-----|
| Diagram creation (private) | Blocked | Up to 5 | Unlimited |
| Diagram creation (public) | Blocked | Unlimited | Unlimited |
| Manual editor and spec updates | Blocked | Yes | Yes |
| AI generation / modification | Blocked | Yes (BYOK required) | Yes (BYOK required) |
| Basic exports (Mermaid, PNG, SVG) | Yes | Yes | Yes |
| ASCII/Markdown copy (with attribution) | No | Yes | Yes (clean) |
| ASCII/Markdown file download | No | No | Yes |
| Blueprint exports (JSON, YAML) | No | No | Yes |
| Documentation Bundle (ZIP) | No | No | Yes |
| GitHub commit | No | No | Yes |
| Version history window | — | 14 days | Unlimited |
| Viewing existing diagrams | Yes | Yes | Yes |
| Share link management | Yes | Yes | Yes |
| Public link branding | — | Diagram Alley branded | Unbranded |
| Provider configuration | Yes | Yes | Yes |
| Billing upgrade flow | Yes | Yes | Yes |

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

- `POST /auth/register` creates a user with `is_verified = false` and a `subscriptions` row with `plan = 'free'`, `status = 'active'` (no `trial_ends_at`).
- `POST /auth/login` returns an access token that expires in 15 minutes and sets a refresh cookie.
- A request with an expired access token but a valid refresh cookie receives a new access token.
- `POST /auth/refresh` with an already-used refresh token returns 401 (token rotation enforced).
- Accessing `GET /diagrams` without a token returns 401.
- Accessing a diagram owned by a different user returns 403 (ownership check in service layer).
- A free user who already has 5 private diagrams calling `POST /diagrams` receives 403 `DIAGRAM_LIMIT_REACHED`.
- An unverified user calling `POST /diagrams` receives 403 `EMAIL_VERIFICATION_REQUIRED`.
- A Stripe `checkout.session.completed` webhook with a valid signature updates the subscription to `plan = 'pro'`, `status = 'active'`.
- A `customer.subscription.deleted` webhook reverts the subscription to `plan = 'free'`, `status = 'active'`.
- `model_providers.api_key_encrypted` decrypts to the original API key value using the server's encryption key.
