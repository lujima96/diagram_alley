---
phase: 11
slice: D7
spec: specs/D7-billing-subscription.md
status: planned
---

# Slice D7 — Billing and Subscription

## Pre-conditions

- [ ] Slice F1 complete — `subscriptions` table exists.
- [ ] Slice F3 complete — Stripe webhook handler, trial expiry job, subscription feature gate, `on_after_register` hook (creates subscriptions row + Stripe customer) all implemented.
- [ ] Slice F5 complete — endpoint stubs for billing routes registered.

Note: F3 owns the core billing mechanics (webhook handling, trial lifecycle, feature gate). D7 owns the UI workflows and the checkout/portal session endpoints that build on top of F3.

---

## Build Sequence

1. **Billing subscription status endpoint** — Implement `GET /api/v1/billing/subscription`. Returns the full subscription state for the billing page per D7 §4: plan, status, trial_ends_at, current_period_start/end, cancel_at_period_end, access_ends_at, canceled_at, days_remaining (computed server-side from `trial_ends_at - now()` for trialing; null otherwise). (→ D7 §4)

2. **Checkout session endpoint** — Implement `POST /api/v1/billing/checkout-session`. Load subscription; if `status='active'` return `400 ALREADY_SUBSCRIBED`. Create a Stripe Checkout Session (`stripe.checkout.Session.create`) with the Pro price, the user's `stripe_customer_id`, success_url, and cancel_url. Return the `checkout_url`. (→ D7 §3.2)

3. **Billing portal session endpoint** — Implement `POST /api/v1/billing/portal-session`. Check user has `stripe_customer_id`. Create a Stripe Billing Portal session with `return_url` pointing back to `/settings/billing`. Return the `portal_url`. (→ D7 §5)

4. **Trial countdown UI** — Wire the trial badge in the app shell header (from F4 §3.3 stub) to the `GET /api/v1/billing/subscription` response. Show badge text and colour per D7 §2.2 thresholds: >7 days neutral, 4–7 days orange, 1–3 days red, expired red. Badge links to `/settings/billing`. (→ D7 §2.2)

5. **Billing settings page** — Implement `<BillingSettingsPage>` at `/settings/billing`. Show the four states from D7 §9: trialing, active Pro, cancel-at-period-end, expired/canceled. Trialing state: plan info, days remaining, Upgrade to Pro button → calls checkout session endpoint → redirects to Stripe. Active Pro state: renewal date, Manage Billing button → calls portal session → redirects to Stripe. Cancel-at-period-end state: "ends on <date>" message. Expired state: "Your trial has ended. Your diagrams are safe." + Upgrade button. Past-due state: "Your payment failed" banner + Manage Billing button. (→ D7 §9)

6. **Checkout success/cancel handling** — On `/settings/billing?session_id=...` (Stripe success redirect), call `GET /api/v1/billing/subscription` to confirm fresh `status='active'`, then show a success message: "You're now on Pro!" On `/settings/billing?checkout=canceled` (Stripe cancel redirect), show a dismissable "Checkout was canceled" banner. (→ D7 §3.3–3.4)

7. **Expired trial overlay** — When `diagramStore` loads a diagram and the subscription status is `status='canceled'` (expired trial or canceled Pro with ended access), show a workspace overlay per D7 §2.3: "Your trial has expired. Upgrade to continue editing." with an Upgrade button. Editor controls are disabled. View, export, and share management remain accessible. The overlay is not a replacement for the API gate — it is a UX layer only. (→ D7 §2.3, DEC-022, GP-04)

---

## Done Criteria

- [ ] `GET /api/v1/billing/subscription` returns correct plan, status, and `days_remaining` for a trialing user. (→ D7 §4)
- [ ] `POST /api/v1/billing/checkout-session` for an already-active user returns `400 ALREADY_SUBSCRIBED`. (→ D7 §3.2)
- [ ] A Stripe `checkout.session.completed` webhook updates the subscription to `status='active'` (wired in F3 — verified here via end-to-end test). (→ F3 §5.3)
- [ ] Trial badge shows correct text and colour for each threshold in D7 §2.2. (→ D7 §2.2)
- [ ] Billing page shows the upgrade button for trialing users and the Manage Billing button for active Pro users. (→ D7 §9)
- [ ] An expired user can still view existing diagrams and manage share links, but the workspace editor shows the upgrade overlay. (→ DEC-022, GP-04)
- [ ] `past_due` subscription shows the payment failure banner on the billing page. (→ D7 §7, FM-04)
- [ ] `cancel_at_period_end=true` subscription shows "ends on <date>" instead of the renewal date. (→ D7 §9.2, DEC-025)
- [ ] A canceled user can resubscribe via the checkout flow; after successful payment the workspace editor unlocks. (→ D7 §8, GP-04)
