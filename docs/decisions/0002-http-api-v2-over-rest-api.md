# ADR-0002: API Gateway HTTP API v2 instead of REST API

**Status:** Accepted
**Date:** 2025
**Scope:** ApiStack — all 77 routes

## Context

Every API request in the platform passes through API Gateway. At SaaS margins, a per-request
cost difference compounds into a line item that shows up in the P&L, so the gateway choice is
a business decision as much as a technical one.

API Gateway offers two products that both front Lambda: REST API (v1) at $3.50 per million
requests, and HTTP API (v2) at $1.00 per million. The gap is not a discount for a weaker
product — it's a different feature set.

## Options considered

### Option A — REST API (v1)

The full-featured option: request/response transformation via VTL templates, API keys and
usage plans, request validation against models, WAF attachment, private endpoints, caching.

Rejected. The platform needs none of the exclusive features. Request validation happens in the
Lambda handlers where the business logic lives anyway; VTL transformation is a maintenance
liability nobody misses; per-client API keys aren't the auth model (JWT is). Paying 3.5x for
capability that would go unused is not a trade-off, it's just cost.

### Option B — Application Load Balancer + Lambda targets

Cheaper per request at very high sustained volume, since ALB prices on hours plus LCUs rather
than per request. Rejected: it means an always-on hourly charge regardless of traffic, which
is the wrong shape for a platform with peaky, growing traffic. It also gives up native JWT
authorizer support and the per-route configuration the platform relies on.

### Option C — HTTP API (v2)

$1.00 per million. Native JWT authorizer support, lower latency than REST, automatic CORS,
per-route throttling and configuration. Missing: VTL, usage plans, request validation.

## Decision

**API Gateway HTTP API v2.** 71% cheaper per request, and the missing features are ones the
architecture deliberately doesn't use.

## Consequences

**What got better**

- 71% lower gateway cost at every request volume — a permanent structural saving, not a
  one-time optimization.
- Lower request latency than REST API, before any application work.
- Native JWT authorizer integration, which is what ADR-0009 builds on.
- Simpler mental model: routes map to handlers, with no transformation layer in between.

**What got worse**

- No request validation at the gateway. Malformed requests reach Lambda and are billed for the
  invocation before being rejected. The handlers must validate defensively, and a burst of
  garbage traffic costs real compute.
- No usage plans or API keys, so any future partner/B2B API tier needs a different mechanism.
- WAF can attach to HTTP APIs but the integration is less mature than on REST; the platform
  leans on WAF at CloudFront as the primary edge (see ADR-0010).
- Migrating back to REST later would mean re-pointing every route and re-testing the
  authorizer — cheap now, expensive at 200 routes.

## Revisit when

- A partner-facing API needs metered API keys and usage plans.
- Invocation cost from rejected malformed requests becomes measurable against gateway savings.
- Sustained request volume gets high enough that ALB's hourly-plus-LCU pricing beats
  $1/million — the crossover is roughly in the tens of millions of requests per month.
