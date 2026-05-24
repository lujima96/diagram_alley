---
phase: 11
slice: D7
spec: specs/D7-billing-subscription.md
status: planned
---

# Slice D7 — Billing and Subscription

## Pre-conditions

- [ ] Slice F1 complete — `subscriptions` table exists.
- [ ] Slice F3 complete — Stripe webhook handler, free tier lifecycle, diagram quota gate, `on_after_register` hook (creates free-tier subscriptions row) all implemented.
- [ ] Slice F5 complete — endpoint stubs for billing routes registered.

Note: F3 owns the core billing mechanics (webhook handling, free tier lifecycle, feature gate). D7 owns the UI workflows and the checkout/portal session endpoints that build on top of F3.

---

## Build Sequence

1. **Billing subscription status endpoint** — Implement `GET /api/v1/billing/subscription`. Returns the full subscription state per D7 §4: plan, status, `private_diagram_count` (computed from live diagram count), `private_diagram_limit` (5 for free, null for pro), current_period_start/end, cancel_at_period_end, access_ends_at, canceled_at. (→ D7 §4, DEC-034)

2. **Checkout session endpoint** — Implement `POST /api/v1/billing/checkout-session`. Load subscription; if `plan='pro'` and `status='active'` and `cancel_at_period_end=false` → return `400 ALREADY_SUBSCRIBED`. Create/retrieve Stripe Customer (F3 §5.3), create Stripe Checkout Session with the Pro price, success_url, and cancel_url. Return the `checkout_url`. (→ D7 §3.2)

3. **Billing portal session endpoint** — Implement `POST /api/v1/billing/portal-session`. Check user has `stripe_customer_id`. Create a Stripe Billing Portal session with `return_url` pointing back to `/settings/billing`. Return the `portal_url`. (→ D7 §5)

4. **Diagram usage indicator** — Wire the usage badge in the app shell header (from F4 §3.3 stub) to the `GET /api/v1/billing/subscription` response. Show badge for free users only (hidden for Pro): neutral/hidden at 0–3 diagrams, orange at 4/5, red at 5/5. Badge links to `/settings/billing`. (→ D7 §2.2, DEC-034)

5. **Billing settings page** — Implement `<BillingSettingsPage>` at `/settings/billing`. Show the four states from D7 §9: free tier, active Pro, cancel-at-period-end, past_due. Free state: diagram count, Upgrade button → checkout. Active Pro: renewal date, Manage Billing → portal. Cancel-at-period-end: "ends on <date>, returning to free tier." Past-due: "Your payment failed" + Manage Billing. (→ D7 §9, DEC-034)

6. **Checkout success/cancel handling** — On `/settings/billing?session_id=...` (Stripe success redirect), call `GET /api/v1/billing/subscription` to confirm fresh `plan='pro'`, then show "You're now on Pro!" On `/settings/billing?checkout=canceled`, show a dismissable "Checkout was canceled" banner. (→ D7 §3.3–3.4)

7. **Diagram limit overlay** — When `diagramStore` detects the user is on `plan='free'` and `private_diagram_count >= 5` and tries to create a new diagram, show an inline upgrade prompt: "You've used all 5 free diagrams. Upgrade to Pro for unlimited private diagrams." + "Upgrade to Pro — $9/month" button. Not a full-screen overlay — shown at the "New Diagram" trigger point only. The API gate (F3 §5.5) is authoritative. (→ D7 §2.3, DEC-034)

---

## Done Criteria

- [ ] `GET /api/v1/billing/subscription` returns `private_diagram_count` and `private_diagram_limit=5` for a free user; `private_diagram_limit=null` for a Pro user. (→ D7 §4, DEC-034)
- [ ] `POST /api/v1/billing/checkout-session` for a free user returns a Stripe checkout URL. (→ D7 §3.2)
- [ ] `POST /api/v1/billing/checkout-session` for an active Pro user returns `400 ALREADY_SUBSCRIBED`. (→ D7 §3.2)
- [ ] A Stripe `checkout.session.completed` webhook updates the subscription to `plan='pro'`, `status='active'` (wired in F3 — verified here via end-to-end test). (→ F3 §5.3, DEC-034)
- [ ] Diagram usage badge shows orange at 4/5 and red at 5/5 for free users; hidden for Pro users. (→ D7 §2.2, DEC-034)
- [ ] Billing page shows the correct state for free, active Pro, past_due, and cancel-at-period-end subscriptions. (→ D7 §9)
- [ ] `past_due` subscription shows the payment failure banner on the billing page. (→ D7 §7, FM-04)
- [ ] `cancel_at_period_end=true` subscription shows "returning to free tier on <date>." (→ D7 §9.2, DEC-025, DEC-034)
- [ ] A free user (previously Pro, now reverted) can resubscribe via the checkout flow; after successful payment diagrams become editable again. (→ D7 §8, GP-04)
