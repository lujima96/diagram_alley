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

Define the full billing domain: free tier UX, Pro subscription, Stripe checkout, upgrade, downgrade/revert to free, and the billing settings page. The Stripe setup and feature gate are defined in F3 §5; this spec defines the domain workflows and UI contract that build on that foundation.

## Non-Goals

- Auth and session management (→ F3)
- Stripe objects setup (→ F3 §5.1)
- Feature gate implementation (→ F3 §5.5)
- Hosted AI credits (not offered in V1 per DEC-010)

---

## Owned Concepts

Billing page UI; free tier usage indicator; upgrade flow (Stripe Checkout); downgrade to free / cancellation flow; billing portal session; subscription status display; billing-related error states.

---

## 1. Subscription States and Plans

From F1 §4.9 and F3 §5.2 (DEC-034):

| `plan` | `status` | Meaning |
|--------|----------|---------|
| `free` | `active` | Permanent free tier (default for all new users) |
| `pro` | `active` | Paid Pro subscription |
| `pro` | `active` + `cancel_at_period_end=true` | Canceled in Stripe portal; access continues until `access_ends_at`, then reverts to `free` |
| `pro` | `past_due` | Payment failed; Stripe retrying |
| `free` | `active` | Pro expired / all retries exhausted; reverted to free |
| `pro` | `incomplete` | Checkout started but not completed |

AI generation requires `status = 'active'` and a verified email (F3 §5.5). Free and Pro users both have access to AI generation with BYOK.

---

## 2. Free Tier Usage UX (DEC-034)

### 2.1 Free Tier Registration

On registration:
1. A `subscriptions` row is created with `plan='free'`, `status='active'` (F3 §5.2). No Stripe interaction.
2. No countdown badge is shown. The free tier is permanent.
3. A diagram usage indicator appears in the app shell header once the user creates their first diagram.

### 2.2 Diagram Usage Indicator

The indicator in the header shows the user's private diagram count vs. the free limit.

Thresholds (free users only; hidden for Pro users):
- 0–3 diagrams: hidden (no badge)
- 4/5 diagrams: orange badge ("4 of 5 diagrams used — upgrade for unlimited")
- 5/5 diagrams: red badge ("5 of 5 diagrams used — upgrade to create more")

The count is fetched from `GET /api/v1/billing/subscription` → `private_diagram_count` / `private_diagram_limit`. Badge links to `/settings/billing`.

### 2.3 Diagram Limit Overlay

When a free user with 5 private diagrams tries to create a new private diagram:
- API returns `403 DIAGRAM_LIMIT_REACHED`
- The "New Diagram" flow shows an inline upgrade prompt: "You've used all 5 free diagrams. Upgrade to Pro for unlimited private diagrams."
- An "Upgrade to Pro — $9/month" button opens the checkout flow.
- The overlay is a UX layer — the API gate (F3 §5.5) is authoritative.
- Existing diagrams remain fully editable; only creation of new private diagrams is blocked.

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
2. If plan = 'pro' AND status = 'active' → return 400 ALREADY_SUBSCRIBED (no-op)
3. Create or retrieve Stripe Customer (F3 §5.3)
4. Create Stripe Checkout Session:
   - price: STRIPE_PRICE_ID_PRO
   - customer: subscriptions.stripe_customer_id
   - success_url: https://<frontend>/settings/billing?session_id={CHECKOUT_SESSION_ID}
   - cancel_url: https://<frontend>/settings/billing?checkout=canceled
5. Return session URL
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

Free tier example:
```json
{
  "data": {
    "plan": "free",
    "status": "active",
    "private_diagram_count": 3,
    "private_diagram_limit": 5,
    "current_period_start": null,
    "current_period_end": null,
    "cancel_at_period_end": false,
    "access_ends_at": null,
    "canceled_at": null
  }
}
```

Pro example:
```json
{
  "data": {
    "plan": "pro",
    "status": "active",
    "private_diagram_count": 47,
    "private_diagram_limit": null,
    "current_period_start": "2026-05-20T00:00:00Z",
    "current_period_end": "2026-06-20T00:00:00Z",
    "cancel_at_period_end": false,
    "access_ends_at": null,
    "canceled_at": null
  }
}
```

`private_diagram_count`: computed server-side (COUNT of non-archived diagrams owned by user). `private_diagram_limit`: `5` for free users, `null` for Pro (unlimited).

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

- `customer.subscription.updated` with `cancel_at_period_end=true` → webhook keeps `plan='pro'`, `status='active'`, sets `cancel_at_period_end=true`, sets `access_ends_at=current_period_end`
- `customer.subscription.deleted` when access actually ends → webhook reverts to `plan='free'`, `status='active'`, clears `stripe_subscription_id`, sets `canceled_at=now()` (DEC-034)
- Emit `SUBSCRIPTION_CANCELED` audit event (F3 §3.2)

After cancel-at-period-end, the user retains full Pro access until `access_ends_at`. The frontend shows: "Your Pro subscription will end on <date>. After that, you'll return to the free tier."

On revert to free, diagrams beyond the 5-private limit become read-only. Diagram data is never deleted.

---

## 7. Payment Failure

When a renewal payment fails, Stripe retries according to its retry schedule and fires:
- `invoice.payment_failed` → webhook sets `status = 'past_due'`

The frontend shows a persistent "Your payment failed" banner on the billing page with a [Manage Billing] button (→ portal).

After all retries exhausted, Stripe fires `customer.subscription.deleted`:
- Webhook reverts to `plan='free'`, `status='active'` (DEC-034)
- User retains free tier access; diagrams beyond the 5-private limit become read-only

---

## 8. Resubscription

A user on `plan='free'` (whether they started free or were reverted from Pro) can upgrade to Pro at any time:
- `POST /api/v1/billing/checkout-session` is available to all free users (blocked only for active Pro: `plan='pro'`, `status='active'`, `cancel_at_period_end=false`).
- On successful checkout, `plan` updates to `'pro'`, `status` to `'active'`, and all Pro features are immediately re-enabled.
- Diagrams that were read-only due to the free tier limit become editable again once upgraded.

If `plan = 'pro'`, `status = 'active'`, and `cancel_at_period_end = true`, the user still has active Pro access. They use the Stripe Billing Portal to reactivate renewal before `access_ends_at`. When Stripe sends `customer.subscription.updated` with `cancel_at_period_end=false`, the webhook clears `cancel_at_period_end` and `access_ends_at`.

---

## 9. Billing Settings Page (`/settings/billing`)

### 9.1 Free Tier State (`plan='free'`)

```
+------------------------------------------------------------------+
| Subscription                                                     |
+------------------------------------------------------------------+
| Plan: Free                                                       |
| Private diagrams: 3 of 5 used                                   |
|                                                                  |
| [Upgrade to Pro — $9/month]                                     |
|                                                                  |
| What's included in Pro:                                          |
| ✓ Unlimited private diagrams                                     |
| ✓ AI generation with your own API key (BYOK)                     |
| ✓ All export formats + Documentation Bundle (.zip)               |
| ✓ Unlimited version history                                      |
| ✓ No Diagram Alley branding on public links                      |
+------------------------------------------------------------------+
```

When `private_diagram_count = 5`, the count line changes to:
```
| Private diagrams: 5 of 5 used (limit reached)                  |
```

### 9.2 Active Pro State (`plan='pro'`, `status='active'`)

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
| After this date you'll return to the free tier.                  |
```

### 9.3 Reverted to Free After Pro (`plan='free'`, `canceled_at` is set)

Shown when a user was previously Pro and was reverted to free via cancellation or payment failure:

```
+------------------------------------------------------------------+
| Subscription                                                     |
+------------------------------------------------------------------+
| Plan: Free                                                       |
| Private diagrams: 5 of 5 used                                   |
|                                                                  |
| Your Pro subscription has ended. Your diagrams are safe.        |
| Diagrams beyond the free limit are read-only.                   |
|                                                                  |
| [Upgrade to Pro — $9/month]                                     |
+------------------------------------------------------------------+
```

### 9.4 Past Due State (`plan='pro'`, `status='past_due'`)

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

- `POST /api/v1/billing/checkout-session` returns a Stripe Checkout URL for a free tier user.
- `POST /api/v1/billing/checkout-session` returns `400 ALREADY_SUBSCRIBED` for an active Pro user (`plan='pro'`, `status='active'`, `cancel_at_period_end=false`).
- A `checkout.session.completed` webhook with a valid signature updates `subscriptions` to `plan='pro'`, `status='active'`.
- A `customer.subscription.updated` webhook with `cancel_at_period_end=true` keeps `status='active'` and sets `access_ends_at=current_period_end`.
- A `customer.subscription.deleted` webhook reverts `subscriptions` to `plan='free'`, `status='active'`.
- `GET /api/v1/billing/subscription` returns `private_diagram_count=0`, `private_diagram_limit=5` for a newly registered free user.
- `GET /api/v1/billing/subscription` returns `private_diagram_limit=null` for an active Pro user.
- The diagram usage badge shows orange at 4/5 and red at 5/5 for free users; hidden for Pro users.
- The billing page shows the correct state for free, active Pro, past_due, and cancel-at-period-end subscriptions.
- After checkout cancellation, the subscriptions row is unchanged and a cancellation banner is shown.
- A free user who upgrades to Pro and then cancels sees the "returning to free tier" message while `cancel_at_period_end=true`.
