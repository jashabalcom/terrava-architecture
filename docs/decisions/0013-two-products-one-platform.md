# ADR-0013: Two products on one shared platform

**Status:** Accepted
**Date:** 2025
**Scope:** Platform-wide

## Context

Terrava ships two products to two different audiences:

- **The investor app** — analysis, calculators, portfolio, community, Chrome extension. The
  user is evaluating properties.
- **The Agent CRM** — client messaging, pipeline, AI voice profile, auto-respond. The user is
  a real-estate agent serving those investors.

They overlap substantially and diverge sharply. Roughly 65% of the API surface is shared —
auth, billing, properties, market data, observability, secrets. The remaining 35% is genuinely
different: portfolios versus deal pipelines, analysis UX versus messaging UX, an advisor
persona versus an agent's own voice.

The question is where to draw the boundary.

## Options considered

### Option A — Two separate platforms

Full independence: separate accounts, stacks, deploys, databases. Rejected. It duplicates auth,
billing, observability, secrets, and the entire property data layer — twice the infrastructure
cost and twice the operational surface for one person. Worse, the two products need to talk to
each other (an agent's client *is* an investor-app user), and cross-platform integration is
harder than intra-platform separation.

### Option B — One monolithic product with a role flag

Simplest. Rejected: it collapses two genuinely different data models into one, and every
feature ends up carrying `if (isAgent)` branches. That's the pattern that makes a codebase
unmaintainable — not because it's ugly, but because the branches multiply and nobody can
reason about which combination is being tested.

### Option C — Shared platform, separated product surfaces

One AWS account and one set of infrastructure stacks. Shared concerns live in Lambda Layers
and shared handlers. Product-specific logic lives in per-product Lambda groups with their own
data models. Features gate on subscription tier through one entitlement path (ADR-0011).

## Decision

**One platform, two product surfaces**, separated by:

- **Shared Lambda Layers** — auth, database access, AI client, observability, common utilities.
  Written once, used by both.
- **Per-product Lambda groups** — the 35% that differs stays isolated in its own handlers.
- **Divergent data models** — portfolios and pipelines are different tables, not a polymorphic
  compromise.
- **One Stripe codebase across eight tiers**, with tier-gated feature flags resolving
  entitlement for both products through the same path.
- **Different AI personas on shared AI infrastructure** — the advisor persona and the agent's
  voice profile are prompt-registry entries (ADR-0016) using the same router, guardrails, cache,
  and token ledger.

## Consequences

**What got better**

- One VPC, one database cluster, one observability stack, one secrets store, one CI/CD
  pipeline. The fixed infrastructure cost is paid once and amortized across two products.
- A platform improvement lands in both products at once — the AI router's cost work, the
  guardrails, the auth latency win all applied to the CRM the day the CRM existed.
- Cross-product features are natural rather than an integration project.
- Eight tiers across two products in one billing implementation.

**What got worse**

- **Blast radius.** A bad Layer deploy or a database migration can take down both products
  simultaneously. Separate platforms would have contained it. This is the central cost of the
  decision and it's accepted knowingly.
- **Shared Layers are a coupling point.** Changing a shared utility means checking both
  products' usage, and the ways it can break are not always obvious from the diff.
- **The 65/35 boundary drifts.** Product-specific logic leaks into shared code under deadline
  pressure, and shared code accumulates product-specific branches — the exact pattern Option B
  was rejected for, arriving through the back door. This needs active policing, and it is the
  most likely way this architecture degrades.
- **Noisy-neighbour risk.** CRM traffic consumes Aurora ACU and Lambda concurrency that
  investor-app requests need, with no isolation between them.
- **Deploys couple two release cadences.** Shipping a CRM feature means deploying stacks the
  investor app also depends on.

## Revisit when

- The shared surface drops meaningfully below ~65% — at that point the products have genuinely
  diverged and sharing costs more than it saves.
- Either product's traffic profile starts degrading the other's latency (noisy neighbour
  becoming measurable rather than theoretical).
- One product needs a compliance or data-residency posture the other doesn't — that's a hard
  split, not a soft one.
- Team size grows past the point where two teams are contending on the same stacks.
