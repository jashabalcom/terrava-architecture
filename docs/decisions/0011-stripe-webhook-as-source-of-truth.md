# ADR-0011: Stripe webhooks as the billing source of truth

**Status:** Accepted
**Date:** 2025
**Scope:** Stripe billing Lambdas, entitlement resolution

## Context

Eight subscription tiers gate real features across two products. Getting entitlement wrong
fails in both directions: a paying customer locked out of what they bought, or a free user
consuming Sonnet-tier inference that nobody paid for. Given ADR-0005, the second one costs
money per request.

The hard part is that subscription state changes **outside the application**. Cards expire.
Payments fail and retry. Customers cancel, upgrade, or downgrade from Stripe's customer
portal. Disputes and refunds happen. None of these originate from a user clicking something in
Terrava, so nothing in the application knows they happened.

## Options considered

### Option A — Write entitlement at checkout completion

Update the database when the checkout success callback fires. Rejected — it's the common bug.
The success redirect can be missed (the user closes the tab), replayed, or forged, and it says
nothing about anything that happens after: a failed renewal three months later leaves the user
permanently entitled.

### Option B — Poll the Stripe API for subscription state

Correct-ish, and self-healing. Rejected as the primary mechanism: polling every user's
subscription is expensive in API calls and rate limits, and it's stale by the polling interval.
A user who upgrades waits for the next poll to get what they paid for.

### Option C — Webhooks as the source of truth

Stripe emits an event for every state change. The platform treats those events — not its own
UI flow — as the authoritative signal, and derives local entitlement from them.

## Decision

**Stripe webhooks are the source of truth for subscription state.** The local database holds a
*projection* of Stripe's state, never an independent record of it.

Three properties make this safe:

1. **Signature verification on every webhook.** The endpoint is public; an unverified webhook
   handler is an open API for granting yourself a subscription.
2. **Idempotent handling.** Stripe retries on non-2xx and can deliver duplicates. Every handler
   is keyed on the Stripe event ID so reprocessing is a no-op.
3. **Order independence.** Webhooks can arrive out of order. Handlers reconcile against the
   subscription object's current state rather than assuming the event sequence is ordered.

The checkout flow still exists for UX — it just doesn't grant entitlement. It creates the
Stripe session and waits for the webhook to confirm.

Daily usage metering runs against this projection, and tier-gated feature flags read from it,
so one resolution path serves both products (ADR-0013).

## Consequences

**What got better**

- Entitlement stays correct through the events the application never sees: failed renewals,
  portal-initiated cancellations, disputes, involuntary churn.
- The failure mode is *delay*, not *incorrectness* — a late webhook means a short lag, not a
  wrong grant.
- Reprocessing is safe, so replaying events to recover from a bug is a routine operation
  rather than a risk.
- One entitlement source feeds both products, billing emails, and usage metering.

**What got worse**

- **Eventual consistency at the worst possible moment.** A user who just paid may briefly not
  have access, in the seconds between checkout and webhook. The UI has to handle a "confirming
  your subscription" state gracefully, which is extra work on the most emotionally sensitive
  path in the product.
- **A public endpoint that mutates entitlement.** Signature verification is load-bearing —
  a bug there is a free-subscription vulnerability.
- **Local state can drift** if webhooks are missed during an outage. This needs a periodic
  reconciliation job against the Stripe API — Option B, retained as a repair mechanism rather
  than a primary path.
- **Webhook handling is hard to test properly.** Duplicate delivery, out-of-order delivery, and
  partial failure all need explicit test coverage, and the Stripe CLI's event fixtures only go
  so far.
- Stripe is now a hard dependency of the authorization path, not just the payment path.

## Revisit when

- Reconciliation regularly finds drift, which means webhook delivery or handling is unreliable
  and needs investigation rather than compensation.
- Entitlement latency after checkout becomes a support burden — the fix is optimistic local
  grant with webhook confirmation, accepting a small, bounded over-grant risk.
- Tier count grows past what a flat projection models cleanly, at which point entitlement wants
  its own service rather than a derived table.
