---
id: A1
tier: domain
status: draft
version: "0.1"
depends-on: [F1, F3]
tags: [admin, users, audit-log, feature-flags, superuser, support]
---

# A1 — Admin Console

**Version:** 0.1
**Status:** Draft
**Depends on:** F1, F3

---

## Purpose

Define the admin console: user management, subscription overrides, audit log review, and support tools available to superusers. All admin routes require `is_superuser = true` (F3 §2.2).

## Non-Goals

- Feature flags infrastructure (no flag service in V1 — flags are env vars)
- Billing provider admin (Stripe dashboard is the tool for that)
- Content moderation (V1 is single-user SaaS; no public content)
- Multi-tenant admin (V2)

---

## Owned Concepts

Admin endpoint contracts; superuser access enforcement; user management actions; subscription override; audit log query interface; `admin.*` permission keys (F3 §2.3).

---

## 1. Access Control

All admin endpoints require:
1. A valid access token (F3 §1.1 / F5 §4)
2. `users.is_superuser = true` on the authenticated user

A superuser is created via a CLI command (`app/cli.py make-superuser <email>`) or directly via the database. There is no in-app superuser promotion flow in V1.

Any admin route accessed by a non-superuser returns `403 FORBIDDEN`.

Admin routes are under `/api/v1/admin/` prefix.

---

## 2. User Management

### 2.1 List Users

**`GET /api/v1/admin/users`**

Returns all users (including unverified and inactive).

**Query parameters:**
- `page`, `page_size` (F5 §7; default page_size 50)
- `search`: filter by email substring
- `is_verified`: `true | false | null` (default null = all)
- `is_active`: `true | false | null` (default `true`)
- `plan`: `trial | pro | null`
- `status`: `trialing | active | past_due | canceled | null`
- `sort_by`: `created_at | email | updated_at` (default `created_at`)
- `sort_dir`: `asc | desc` (default `desc`)

**Response (list envelope):**
```json
{
  "data": [
    {
      "id": "uuid",
      "email": "user@example.com",
      "name": "Paul",
      "is_active": true,
      "is_verified": true,
      "is_superuser": false,
      "created_at": "2026-05-20T10:00:00Z",
      "subscription": {
        "plan": "trial",
        "status": "trialing",
        "trial_ends_at": "2026-06-03T10:00:00Z"
      }
    }
  ],
  "meta": { "total": 42, ... }
}
```

### 2.2 Get User

**`GET /api/v1/admin/users/{id}`**

Returns the full user profile including subscription, provider count, and diagram count.

```json
{
  "data": {
    "id": "uuid",
    "email": "user@example.com",
    "name": "Paul",
    "is_active": true,
    "is_verified": true,
    "is_superuser": false,
    "created_at": "2026-05-20T10:00:00Z",
    "updated_at": "2026-05-20T14:00:00Z",
    "subscription": {
      "plan": "pro",
      "status": "active",
      "stripe_customer_id": "cus_...",
      "current_period_end": "2026-06-20T10:00:00Z"
    },
    "diagram_count": 12,
    "provider_count": 2
  }
}
```

### 2.3 Update User

**`PATCH /api/v1/admin/users/{id}`**

Support actions available:

```json
{
  "is_active": true,
  "is_verified": true,
  "name": "string"
}
```

Superusers **cannot** change another user's `email` or `hashed_password` via this endpoint (those require password reset flow). Superusers **cannot** promote other users to superuser via the API — only via CLI.

Returns the updated user object.

### 2.4 Subscription Override

**`PATCH /api/v1/admin/users/{id}/subscription`**

For support cases (e.g., extend a trial, manually activate a subscription for a comped user).

```json
{
  "plan": "trial | pro",
  "status": "trialing | active | canceled",
  "trial_ends_at": "2026-06-20T10:00:00Z"
}
```

Rules:
- Setting `status = 'active'` without a Stripe subscription ID records the subscription as manually activated (`stripe_subscription_id` stays null — Stripe is not involved).
- Setting `trial_ends_at` extends the trial.
- All changes emit `admin.subscription.update` audit detail.

Returns the updated subscription object.

---

## 3. Audit Log

### 3.1 List Audit Events

**`GET /api/v1/admin/audit-log`**

Returns the append-only audit log in reverse-chronological order.

**Query parameters:**
- `page`, `page_size` (default page_size 100)
- `user_id`: filter by actor
- `event`: filter by event constant (e.g. `DIAGRAM_CREATED`)
- `entity_type`: filter by entity table
- `entity_id`: filter by entity UUID
- `from`: ISO timestamp (inclusive lower bound)
- `to`: ISO timestamp (inclusive upper bound)

**Response:**
```json
{
  "data": [
    {
      "id": "uuid",
      "user_id": "uuid",
      "event": "DIAGRAM_CREATED",
      "entity_type": "diagrams",
      "entity_id": "uuid",
      "detail": { "diagram_type": "architecture" },
      "ip_address": "1.2.3.4",
      "created_at": "2026-05-20T14:32:00Z"
    }
  ],
  "meta": { "total": 1024, ... }
}
```

### 3.2 Get Audit Event

**`GET /api/v1/admin/audit-log/{id}`**

Single audit event by ID. Same shape as list items.

---

## 4. Bulk Operations

### 4.1 Bulk User Export (CSV)

**`GET /api/v1/admin/users/export`**

Returns a CSV of all users (id, email, name, is_verified, is_active, plan, status, created_at). Used for support reporting.

Content-Type: `text/csv`. The filename header is `users-export-<date>.csv`.

No pagination — returns all users. If the user list is very large (>10,000), this may be slow; acceptable for V1 given expected user count at launch.

---

## 5. Admin UI

The admin console is not a separate application — it is a section of the same frontend app, visible only to superusers.

### 5.1 Route

| Route | Page | Auth |
|-------|------|------|
| `/admin` | Admin dashboard | Required + superuser |
| `/admin/users` | User list | Required + superuser |
| `/admin/users/:id` | User detail | Required + superuser |
| `/admin/audit-log` | Audit log | Required + superuser |

Non-superuser access to `/admin/*` redirects to `/dashboard`.

### 5.2 Admin Dashboard

Quick stats visible at a glance:
- Total users
- Active trials
- Active Pro subscribers
- Trial expirations in next 7 days
- Recent audit events (last 10)

### 5.3 User List Page

Table with columns: Email, Name, Verified, Plan, Status, Trial Ends / Period End, Diagrams, Created.

Row actions:
- View user detail
- Extend trial (opens modal with date picker → calls subscription override endpoint)
- Activate Pro manually (modal confirmation → subscription override)
- Deactivate account (`is_active = false`)

### 5.4 User Detail Page

Shows all user fields, subscription details, and a filterable audit log for that user.

Action buttons: Extend Trial, Activate Pro, Verify Email (sets `is_verified = true`), Deactivate Account.

### 5.5 Audit Log Page

Filterable table. Columns: Time, User, Event, Entity Type, Entity ID, IP. Clicking a row shows the full `detail` JSON.

---

## 6. Reconciliation Notes

### 6.1 New Routes in F4/Route Map

Admin routes (`/admin`, `/admin/users`, etc.) must be added to F4 §2 route map.

**Action required:** Add admin routes to F4 §2 with `Auth: Required + superuser`.

### 6.2 New Endpoints in F5

The following endpoints must be added to F5 §10 admin section:

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/admin/users` | List all users |
| GET | `/api/v1/admin/users/{id}` | Get user |
| PATCH | `/api/v1/admin/users/{id}` | Update user |
| PATCH | `/api/v1/admin/users/{id}/subscription` | Override subscription |
| GET | `/api/v1/admin/audit-log` | List audit events |
| GET | `/api/v1/admin/audit-log/{id}` | Get audit event |
| GET | `/api/v1/admin/users/export` | Export users CSV |

The first three (`GET`, `GET`, `PATCH /api/v1/admin/users*`) are already in F5 §10. The subscription override, audit log, and CSV export endpoints are new and must be added.

---

## Open Questions

None. All A1 decisions are locked.

---

## Acceptance Criteria

- `GET /api/v1/admin/users` by a non-superuser returns `403 FORBIDDEN`.
- `GET /api/v1/admin/users` by a superuser returns all users with subscription info.
- `PATCH /api/v1/admin/users/{id}/subscription` with `trial_ends_at` updates the expiry and is reflected in `GET /api/v1/billing/subscription`.
- `GET /api/v1/admin/audit-log?event=DIAGRAM_CREATED` returns only `DIAGRAM_CREATED` events.
- `GET /api/v1/admin/users/export` returns a valid CSV file with headers.
- `/admin/*` routes redirect non-superuser authenticated users to `/dashboard`.
- Deactivating a user (`is_active = false`) causes subsequent logins for that user to return `401`.
