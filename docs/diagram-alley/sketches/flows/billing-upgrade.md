---
phase: 4
artifact: flow — billing and upgrade
status: complete
depends-on: [D7, F3, F1, personas]
decisions: [DEC-001, DEC-004, DEC-010, DEC-022, DEC-025]
---

# Flow: Trial Start, Limit Hit, Upgrade, and Diagram Unlock

Narrative covering the full billing lifecycle from free trial signup through limit enforcement, upgrade to Pro, and post-upgrade access restoration. Cast: **Priya** (P1, Solo Developer) signing up for the first time, running through her trial, and converting to Pro.

---

## Cast and Context

| | |
|-|-|
| **User** | Priya, P1 — Solo Developer |
| **Arc** | Signs up → uses the product → trial expires → hits the edit block → upgrades to Pro → continues working |
| **Billing provider** | Stripe (DEC-004) |
| **Trial terms** | 14 days, unlimited diagrams, no hosted AI credits, full editor/export access (DEC-010) |

---

## Phase 1: Signup and Trial Start

### 1. Registration

Priya visits `https://diagramalley.com` and clicks **Start free trial** on the landing page. She is directed to `/register`.

She fills in:
- Email: `priya@devmail.io`
- Password: `[chosen]`

She clicks **Create account**. The backend (F3 §1.2):

1. Creates a `users` row with `is_verified = false`.
2. Creates a `subscriptions` row: `plan = 'trial'`, `status = 'trialing'`, `current_period_end = now() + 14 days`.
3. Sends a verification email via Resend.

She receives the verification email and clicks the link. `is_verified = true`. She is redirected to `/dashboard`.

### 2. Onboarding: Provider Setup

The dashboard shows a "Get started" banner: "Set up a model provider to use AI generation." Priya clicks it and is directed to `/settings/providers`.

She adds her BYOK OpenAI-compatible key. Connection test passes. She sets it as default.

She returns to the dashboard and starts generating diagrams (see AI generation flow).

### 3. Trial in Progress

During the first 13 days, Priya has full access. She creates 7 diagrams across 3 projects. The header bar shows no trial banner until day 10.

**Day 10:** A trial countdown chip appears in the header bar: "4 days left in trial — Upgrade to Pro." The chip links to `/settings/billing`. Priya dismisses it mentally and keeps working.

**Day 13:** A more prominent banner appears at the top of the workspace: "Your trial expires tomorrow. Upgrade to Pro to keep editing your diagrams."

---

## Phase 2: Trial Expires — Access Change

### 4. Trial Expiry

On day 14 at midnight (UTC), the Stripe trial period ends. Stripe sends a `customer.subscription.trial_will_end` event 3 days before (already shown in banner). On expiry, Stripe fires `customer.subscription.updated` with `status = 'canceled'` (since Priya has no payment method, the trial ends without converting to paid).

The backend Stripe webhook handler (D7, F3):

1. Receives `customer.subscription.updated`.
2. Updates `subscriptions` row: `status = 'canceled'`, `access_ends_at = now()` (per DEC-025 — the `cancel_at_period_end` model; access ends at period end).
3. The feature gate (F3) now evaluates `subscription.status == 'canceled'` for Priya's requests.

### 5. Priya Opens the App After Expiry

Next morning Priya opens `/dashboard`. She sees:

- A full-width upgrade banner at the top: "Your free trial has ended. Upgrade to Pro to continue editing your diagrams."
- Her 7 diagrams are listed — she can see them. Cards show "View" and "Export" actions. The "Open" action opens the workspace in **read-only mode** (DEC-022).

### 6. Blocked from Editing

Priya clicks one of her diagrams. The workspace opens at `/workspace/da_7f3a19c`. The visual grid editor and prompt panel are rendered but **dimmed**, with an overlay:

> "Your trial has ended.
> Your diagrams are safe. Upgrade to Pro to continue editing."
> [Upgrade to Pro button]

She cannot:
- Edit `spec_json` via any editor mode (D2 PATCH endpoint returns 403 with `billing.trial_expired`)
- Use AI generation or modification (D1 — blocked at feature gate)
- Create new diagrams

She can:
- View the current diagram (rendered ASCII and visual in read-only mode)
- Export it (copy ASCII, download any format — DEC-022 allows export)
- Manage share links (DEC-022)

---

## Phase 3: Upgrade to Pro

### 7. Initiate Upgrade

Priya clicks the **Upgrade to Pro** button in the workspace overlay. She is directed to `/settings/billing`.

The billing page shows:
- Current plan: "Free trial — expired"
- Pro plan: $X/month (price defined in D7; exact price not part of this flow)
- [Subscribe to Pro] button

### 8. Stripe Checkout

She clicks **Subscribe to Pro**. The backend creates a Stripe Checkout Session (D7) and redirects her to Stripe's hosted checkout page.

On the Stripe checkout page:
- She enters her card details.
- She clicks **Subscribe**.
- Stripe processes the payment.
- She is redirected back to `/settings/billing?session_id=cs_...` (Stripe success URL).

### 9. Subscription Activated

While Priya was on Stripe's page, the backend received a `checkout.session.completed` webhook (D7):

1. Creates a Stripe Customer and Subscription (or links to existing Customer if she had one from the trial).
2. Updates `subscriptions` row: `plan = 'pro'`, `status = 'active'`, `access_ends_at = null`, `current_period_end = now() + 1 month`.
3. The feature gate now evaluates `subscription.status == 'active'` for Priya.

The billing page shows: "Pro — Active. Your next billing date is [date]."

### 10. Diagram Unlock

Priya navigates back to her diagram. The workspace loads without the overlay — the editor is fully active. Her 7 diagrams are immediately accessible for editing. No data was lost during the trial expiry period.

A success toast appears: "Welcome to Pro! All features are unlocked."

---

## Phase 4: Cancellation (DEC-025)

Later, Priya decides to cancel her Pro subscription.

She goes to `/settings/billing` and clicks **Cancel subscription**. A confirmation dialog explains that she will keep access until the current billing period ends.

She confirms. The backend sends `subscription.cancel_at_period_end = true` to Stripe via the Billing Portal delegation (D7). The `subscriptions` row updates:

```
cancel_at_period_end = true
access_ends_at = current_period_end (e.g., 2026-07-01)
status = 'active'  (still active until the period ends — DEC-025)
```

The billing page shows: "Pro — Active until July 1, 2026. Your subscription will not renew."

On July 1, Stripe fires `customer.subscription.deleted`. The backend sets `status = 'canceled'`. Priya's access returns to the expired-trial state (view and export only).

---

## Billing State Machine Summary

| State | `status` | `access_ends_at` | What Priya can do |
|-------|----------|------------------|-------------------|
| Trial active | `trialing` | `trial_end` | Full access |
| Trial expired (no payment) | `canceled` | past | View + export only |
| Pro active | `active` | null | Full access |
| Pro canceled, period active | `active` | future date | Full access until `access_ends_at` |
| Pro period ended | `canceled` | past | View + export only |

---

## Stripe Webhook Events Handled (D7 reference)

| Event | Action |
|-------|--------|
| `checkout.session.completed` | Activate Pro subscription |
| `customer.subscription.updated` | Sync status and `current_period_end` |
| `customer.subscription.deleted` | Set `status = 'canceled'` |
| `invoice.payment_failed` | Set `status = 'past_due'`; show payment banner |
| `customer.subscription.trial_will_end` | Trigger 3-day warning banner |

---

## Spec References

| Behavior | Owner |
|----------|-------|
| Trial terms (14 days, unlimited diagrams, no hosted AI) | F3, DEC-010 |
| `subscriptions` table and state machine | F1 |
| Feature gate logic (plan check on routes) | F3 |
| Stripe Checkout Session and webhook handling | D7 |
| Trial-expired access rules (view + export only) | F3, DEC-022 |
| Cancellation model (`cancel_at_period_end`) | D7, DEC-025 |
| Billing portal delegation to Stripe | D7 |
| Upgrade banner and workspace overlay UX | D7 |
