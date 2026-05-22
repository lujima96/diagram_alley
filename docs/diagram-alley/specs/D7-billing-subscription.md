---
id: D7
tier: domain
status: draft
version: "0.1"
depends-on: [F1, F3, F5]
tags: [billing, stripe, subscription, trial, upgrade, downgrade]
---

# D7 — Billing and Subscription

**Version:** 0.1
**Status:** Draft
**Depends on:** F1, F3, F5

---

## Purpose

Define the full billing domain: free trial lifecycle, Pro subscription, Stripe checkout, upgrade, downgrade, trial expiry, and the billing settings page. The Stripe setup and feature gate are defined in F3 §5; this spec defines the domain workflows and UI contract that build on that foundation.

## Non-Goals

- Auth and session management (→ F3)
- Stripe objects setup (→ F3 §5.1)
- Feature gate implementation (→ F3 §5.5)
- Hosted AI credits (not offered in V1 per DEC-010)

---

## Owned Concepts

Billing page UI; trial countdown and expiry UX; upgrade flow (Stripe Checkout); downgrade / cancellation flow; billing portal session; subscription status display; billing-related error states.

---

## 1. Subscription States and Plans

From F1 §4.9 and F3 §5.2:

| `plan` | `status` | Meaning |
|--------|----------|---------|
| `trial` | `trialing` | Active 14-day trial |
| `trial` | `canceled` | Trial expired, no payment |
| `pro` | `active` | Paid Pro subscription |
| `pro` | `active` + `cancel_at_period_end=true` | Canceled in Stripe portal, access continues until `access_ends_at` |
| `pro` | `past_due` | Payment failed; Stripe retrying |
| `pro` | `canceled` | Access ended after cancellation or payment retries exhausted |
| `pro` | `incomplete` | Checkout started but not completed |

AI generation is gated on `status IN ('trialing', 'active')` (F3 §5.5).

---

## 2. Trial Lifecycle UX

### 2.1 Trial Start

On registration:
1. A `subscriptions` row is created with `plan='trial'`, `status='trialing'`, `trial_ends_at = now() + 14 days` (F3 §5.2).
2. No Stripe subscription is created at trial start — Stripe Customer ID is created and stored.
3. The user sees a "14 days left in your trial" badge in the header (F4 §3.3).

### 2.2 Trial Countdown

The trial badge in the header counts down daily. Thresholds:
- > 7 days: neutral badge ("14 days left")
- 4–7 days: orange badge ("6 days left — upgrade to keep access")
- 1–3 days: red badge ("2 days left — upgrade now")
- Expired: red badge ("Trial expired — upgrade to continue")

The countdown is computed from `subscriptions.trial_ends_at` relative to the current browser time.

### 2.3 Trial Feature Restrictions

During `trialing`:
- Full editor access: ✓
- All export formats: ✓
- AI generation (with configured provider): ✓
- All other features: ✓

On trial expiry (`plan = 'trial'`, `status = 'canceled'`):
- AI generation: blocked (403 SUBSCRIPTION_REQUIRED on all AI endpoints)
- Existing saved diagrams: read-only (can view, export, share — not edit spec)
- Editor: shows a banner "Your trial has expired. Upgrade to continue editing."

This state means an expired trial, not a canceled Pro subscription. A canceled Pro subscription remains `plan = 'pro'`, `status = 'active'`, `cancel_at_period_end = true` until access ends (DEC-025).

**Read-only enforcement for expired users:** The frontend may proactively render read-only controls or an upgrade overlay, but the API gate is authoritative. The PATCH /api/v1/diagrams/{id} endpoint checks subscription status. If `status = 'canceled'`, spec updates are blocked with `403 SUBSCRIPTION_REQUIRED`. Title updates and archiving are still permitted.

---

## 3. Upgrade Flow

### 3.1 Entry Points

Users can initiate upgrade from:
- Trial badge in header (→ click → `/settings/billing`)
- `SUBSCRIPTION_REQUIRED` error banner (→ "Upgrade" button)
- `/settings/billing` page directly

### 3.2 Stripe Checkout Session

**`POST /api/v1/billing/checkout-session`** (requires auth)

No request body.

Service steps:
```
1. Load user's subscriptions row
2. If status = 'active' → return 400 ALREADY_SUBSCRIBED (no-op)
3. Create Stripe Checkout Session:
   - price: STRIPE_PRICE_ID_PRO
   - customer: subscriptions.stripe_customer_id
   - success_url: https://<frontend>/settings/billing?session_id={CHECKOUT_SESSION_ID}
   - cancel_url: https://<frontend>/settings/billing?checkout=canceled
4. Return session URL
```

Response:
```json
{
  "data": {
    "checkout_url": "https://checkout.stripe.com/pay/cs_..."
  }
}
```

The frontend redirects to `checkout_url`. Stripe hosts the checkout page.

### 3.3 Checkout Success

On Stripe checkout completion, Stripe fires `checkout.session.completed` webhook to `POST /webhooks/stripe`:
```
1. Verify webhook signature (STRIPE_WEBHOOK_SECRET)
2. Extract customer_id and subscription_id from event
3. Update subscriptions: plan='pro', status='active', stripe_subscription_id=..., current_period_start/end from Stripe
4. Emit SUBSCRIPTION_UPGRADED audit event (F3 §3.2)
```

The frontend's success_url shows a success message on `/settings/billing?session_id=...`. The frontend fetches `GET /api/v1/billing/subscription` to get the fresh status and updates the UI.

### 3.4 Checkout Cancellation

If the user cancels at Stripe checkout, they are redirected to `/settings/billing?checkout=canceled`. The subscriptions row is unchanged. A dismissable "Checkout was canceled" banner is shown.

---

## 4. Billing Status Endpoint

**`GET /api/v1/billing/subscription`** (requires auth)

Returns the current subscription state for the billing settings page.

```json
{
  "data": {
    "plan": "trial",
    "status": "trialing",
    "trial_ends_at": "2026-06-03T10:00:00Z",
    "current_period_start": null,
    "current_period_end": null,
    "cancel_at_period_end": false,
    "access_ends_at": "2026-06-03T10:00:00Z",
    "canceled_at": null,
    "days_remaining": 14
  }
}
```

`days_remaining`: computed server-side from `trial_ends_at - now()` for trialing; null for non-trial plans.

---

## 5. Billing Portal (Manage Subscription)

For Pro subscribers, subscription management (cancel, update payment method, view invoices) is delegated to the Stripe Billing Portal.

**`POST /api/v1/billing/portal-session`** (requires auth)

Service steps:
```
1. Check user has stripe_customer_id
2. Create Stripe Billing Portal session
   - customer: subscriptions.stripe_customer_id
   - return_url: https://<frontend>/settings/billing
3. Return portal URL
```

Response:
```json
{
  "data": {
    "portal_url": "https://billing.stripe.com/session/..."
  }
}
```

Frontend redirects to `portal_url`. Stripe hosts the portal page.

---

## 6. Downgrade / Cancellation

Cancellation is handled via the Stripe Billing Portal (§5). The user navigates to the portal, cancels the subscription, and Stripe fires:

- `customer.subscription.updated` with `cancel_at_period_end=true` → webhook keeps `subscriptions.status = 'active'`, sets `cancel_at_period_end = true`, and sets `access_ends_at = current_period_end`
- `customer.subscription.deleted` when access actually ends → webhook updates `subscriptions.status = 'canceled'`, sets `canceled_at = now()`
- Emit `SUBSCRIPTION_CANCELED` audit event (F3 §3.2)

After cancel-at-period-end, the user retains active access until `access_ends_at` (Stripe's standard behavior). The frontend shows: "Your Pro subscription will end on <date>. After that, you'll lose editing and AI access."

Diagram data is never deleted on cancellation.

---

## 7. Payment Failure

When a renewal payment fails, Stripe retries according to its retry schedule and fires:
- `invoice.payment_failed` → webhook sets `status = 'past_due'`

The frontend shows a persistent "Your payment failed" banner on the billing page with a [Manage Billing] button (→ portal).

After all retries exhausted, Stripe fires `customer.subscription.deleted`:
- Sets `status = 'canceled'`
- Same access restrictions as trial expiry apply

---

## 8. Resubscription

A user with `status = 'canceled'` can resubscribe:
- The `POST /api/v1/billing/checkout-session` endpoint works for canceled users (only blocked for `status = 'active'`).
- On successful checkout, `status` updates to `active` and AI features are immediately re-enabled.

If `plan = 'pro'`, `status = 'active'`, and `cancel_at_period_end = true`, the user still has active access and does not use the checkout-session resubscribe flow. They use Stripe Billing Portal to resume/reactivate renewal before `access_ends_at`. When Stripe sends `customer.subscription.updated` with `cancel_at_period_end=false`, the webhook clears `cancel_at_period_end` and `access_ends_at`.

---

## 9. Billing Settings Page (`/settings/billing`)

### 9.1 Trialing State

```
+------------------------------------------------------------------+
| Subscription                                                     |
+------------------------------------------------------------------+
| Plan: Free Trial                                                 |
| Status: Trialing                                                 |
| Trial ends: June 3, 2026 (14 days remaining)                    |
|                                                                  |
| [Upgrade to Pro — $X/month]                                     |
|                                                                  |
| What's included in Pro:                                          |
| ✓ AI generation with your own API key (BYOK)                     |
| ✓ Unlimited diagrams                                             |
| ✓ All export formats                                             |
| ✓ Version history                                                |
+------------------------------------------------------------------+
```

### 9.2 Active Pro State

```
+------------------------------------------------------------------+
| Subscription                                                     |
+------------------------------------------------------------------+
| Plan: Pro                                                        |
| Status: Active                                                   |
| Renewal: June 20, 2026                                           |
|                                                                  |
| [Manage Billing →]    (opens Stripe Portal)                      |
+------------------------------------------------------------------+
```

If `cancel_at_period_end = true`, replace the renewal line with:

```
| Ends: June 20, 2026                                              |
| Your Pro access remains active until this date.                  |
```

### 9.3 Expired / Canceled State

```
+------------------------------------------------------------------+
| Subscription                                                     |
+------------------------------------------------------------------+
| Plan: Free Trial                                                 |
| Status: Expired                                                  |
|                                                                  |
| Your trial has ended. Upgrade to continue editing and using AI. |
| Your diagrams are safe and will remain accessible.              |
|                                                                  |
| [Upgrade to Pro — $X/month]                                     |
+------------------------------------------------------------------+
```

### 9.4 Past Due State

```
+------------------------------------------------------------------+
| Subscription                                                     |
+------------------------------------------------------------------+
| Plan: Pro                                                        |
| Status: Payment Failed                                           |
|                                                                  |
| ⚠ Your last payment failed. Please update your payment method.  |
|                                                                  |
| [Update Payment Method →]   (opens Stripe Portal)               |
+------------------------------------------------------------------+
```

---

## Open Questions

None. All D7 decisions are locked.

---

## Acceptance Criteria

- `POST /api/v1/billing/checkout-session` returns a Stripe Checkout URL for a trialing user.
- `POST /api/v1/billing/checkout-session` returns `400 ALREADY_SUBSCRIBED` for an active Pro user.
- A `checkout.session.completed` webhook with a valid signature updates `subscriptions.status` to `active`.
- A `customer.subscription.updated` webhook with `cancel_at_period_end=true` keeps `status='active'` and sets `access_ends_at = current_period_end`.
- A `customer.subscription.deleted` webhook updates `subscriptions.status` to `canceled`.
- `GET /api/v1/billing/subscription` returns `days_remaining = 14` for a newly created trial user.
- `PATCH /api/v1/diagrams/{id}` with spec changes by a `status='canceled'` user returns `403 SUBSCRIPTION_REQUIRED`.
- The billing page shows the correct state for trialing, active, past_due, and canceled subscriptions.
- After checkout cancellation, the subscriptions row is unchanged and a cancellation banner is shown.
