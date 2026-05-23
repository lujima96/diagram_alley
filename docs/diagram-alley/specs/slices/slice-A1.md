---
phase: 11
slice: A1
spec: specs/A1-admin-console.md
status: planned
---

# Slice A1 — Admin Console

## Pre-conditions

- [ ] Slice F1 complete — `users`, `subscriptions`, `audit_log` tables exist.
- [ ] Slice F3 complete — superuser guard (`require_superuser` dependency) implemented; audit event helper available.
- [ ] Slice F5 complete — pagination helper and endpoint stubs for `/api/v1/admin/*` registered.

Note: A1 can be built independently of D1–D7. It only reads and overrides data; it does not depend on any domain workflow.

---

## Build Sequence

1. **Superuser CLI command** — Create `app/cli.py` with a `make-superuser <email>` Click command that looks up the user by email and sets `is_superuser=true`. This is the only way to create a superuser in V1 — there is no in-app promotion flow. (→ A1 §1)

2. **User list endpoint** — Implement `GET /api/v1/admin/users`. All filters from A1 §2.1: search (email substring), is_verified, is_active, plan, status. Default `page_size=50`. Join `subscriptions` to include plan/status/trial_ends_at inline. Guard with `require_superuser`. (→ A1 §2.1)

3. **User get endpoint** — Implement `GET /api/v1/admin/users/{id}`. Returns full user profile + subscription detail + `diagram_count` (COUNT query on `diagrams`) + `provider_count` (COUNT on `model_providers`). Guard with `require_superuser`. (→ A1 §2.2)

4. **User update endpoint** — Implement `PATCH /api/v1/admin/users/{id}`. Allows `is_active`, `is_verified`, `name` updates. Must reject attempts to change `email`, `hashed_password`, or `is_superuser` (return `403 FORBIDDEN` for those fields). Guard with `require_superuser`. (→ A1 §2.3)

5. **Subscription override endpoint** — Implement `PATCH /api/v1/admin/users/{id}/subscription`. Allows `plan`, `status`, `trial_ends_at`, `cancel_at_period_end`, `access_ends_at` overrides per A1 §2.4. Setting `status='active'` without a `stripe_subscription_id` is valid (manual comp). All changes are written directly to the `subscriptions` row and emit an audit detail entry with `admin.subscription.update`. Guard with `require_superuser`. (→ A1 §2.4)

6. **Audit log endpoints** — Implement `GET /api/v1/admin/audit-log` and `GET /api/v1/admin/audit-log/{id}`. List supports all filters from A1 §3.1: `user_id`, `event`, `entity_type`, `entity_id`, `from`, `to`. Default `page_size=100`, `sort_dir=desc`. Single-event endpoint returns the same shape. Guard with `require_superuser`. (→ A1 §3)

7. **Bulk user export** — Implement `GET /api/v1/admin/users/export`. Returns a CSV stream with headers: id, email, name, is_verified, is_active, plan, status, created_at. Set `Content-Type: text/csv` and `Content-Disposition: attachment; filename="users-export-<date>.csv"`. No pagination — returns all users. Guard with `require_superuser`. (→ A1 §4.1)

8. **Admin dashboard stats** — Add a `GET /api/v1/admin/stats` endpoint (or include in the dashboard page fetch): total users, active trials count, active Pro count, trials expiring in 7 days, last 10 audit events. Used by `<AdminDashboardPage>`. (→ A1 §5.2)

9. **Admin UI pages** — Implement the four admin pages (replacing F4 stubs) guarded by `<SuperuserGuard>` that redirects non-superusers to `/dashboard`:
   - `<AdminDashboardPage>` — quick stats + recent audit events from the stats endpoint.
   - `<AdminUsersPage>` — filterable/searchable user table with row actions (View, Extend Trial modal, Activate Pro modal, Deactivate).
   - `<AdminUserDetailPage>` — full profile, subscription fields, per-user audit log (filtered by `user_id`), action buttons (Extend Trial, Activate Pro, Verify Email, Deactivate).
   - `<AdminAuditLogPage>` — filterable table with detail JSON on row click. (→ A1 §5)

---

## Done Criteria

- [ ] `GET /api/v1/admin/users` by a non-superuser returns `403 FORBIDDEN`. (→ A1 Acceptance Criteria)
- [ ] `GET /api/v1/admin/users` by a superuser returns all users with subscription info. (→ A1 Acceptance Criteria)
- [ ] `PATCH /api/v1/admin/users/{id}/subscription` with an updated `trial_ends_at` is reflected in `GET /api/v1/billing/subscription` for that user. (→ A1 Acceptance Criteria)
- [ ] `GET /api/v1/admin/audit-log?event=DIAGRAM_CREATED` returns only `DIAGRAM_CREATED` events. (→ A1 Acceptance Criteria)
- [ ] `GET /api/v1/admin/users/export` returns a valid CSV file with the correct headers. (→ A1 Acceptance Criteria)
- [ ] `/admin/*` routes redirect non-superuser authenticated users to `/dashboard`. (→ A1 Acceptance Criteria)
- [ ] Deactivating a user (`is_active=false`) causes subsequent logins for that user to return `401`. (→ A1 Acceptance Criteria)
- [ ] `make-superuser <email>` CLI command sets `is_superuser=true` on the target user. (→ A1 §1)
- [ ] Attempting to set `email` or `is_superuser` via the PATCH endpoint returns `403 FORBIDDEN`. (→ A1 §2.3)
