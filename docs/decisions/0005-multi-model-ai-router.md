# ADR-0005: Multi-model AI router instead of a single model

**Status:** Accepted
**Date:** 2025
**Scope:** AI Service Router, AIInfraStack

## Context

This is the decision that determines whether the business works.

Terrava sells subscriptions at fixed monthly prices. Every AI feature — property analysis,
calculator insights, the wealth advisor chat, agent draft replies, the daily digest — costs
variable money per call. If average AI cost per user exceeds the margin on their tier, growth
makes the company lose money faster. AI cost is not an infrastructure line item; it's a unit
economics constraint that has to be enforced at runtime.

Running everything on the strongest available model produced roughly $400+/month at the
platform's scale, and that number grows linearly with usage while subscription revenue grows
in steps.

The insight that unlocks the fix: **the AI workload is not homogeneous.** "Analyse this
property against my portfolio and explain the risk" is genuine multi-step reasoning. "What's
the transfer fee in Dubai?" is retrieval and formatting. Charging the same compute cost for
both is waste, not quality.

## Options considered

### Option A — Single frontier model for everything

Simplest to build, best worst-case quality, one prompt style to maintain. Rejected on cost:
~$400+/month at current scale with no lever to pull other than using less AI, which means
degrading the product.

### Option B — Single cheap model for everything

Cheapest. Rejected on quality: deep portfolio analysis on a small fast model produces
confident, shallow output. In a financial-advice-adjacent product, that's not a cost saving,
it's a liability.

### Option C — Fine-tune one model to cover everything

Rejected — see ADR-0007. Fine-tuning bakes in knowledge that changes (regulations, fees,
market data) and needs a retraining cycle every time it does.

### Option D — Route per request by task type and user tier

Classify each request on task complexity and the caller's subscription tier, then dispatch to
the cheapest model that meets that request's quality bar:

- **Claude Sonnet** — deep multi-step analysis for Pro/Portfolio tiers
- **Claude Haiku** — fast structured Q&A, summarization, agent draft replies
- **Gemini Flash** — fallback when Bedrock is degraded or a circuit breaker is open

With a response cache in front (ADR-0006) and a token ledger behind, so the routing decision
is measurable rather than assumed.

## Decision

**A multi-model router that selects per request on task type and user tier, with
circuit-breaker fallback to a third-party model.**

Result: roughly $70–130/month against $400+/month on a single-model setup — about a 60%
reduction — while keeping frontier-model quality on the requests that need it.

## Consequences

**What got better**

- ~60% lower AI cost, achieved by matching model capability to task rather than by using less
  AI. The product got no worse.
- Cost scales with *value delivered*: Pro-tier users, who pay more, are the ones routed to
  expensive models.
- The circuit breaker means a Bedrock regional degradation is a quality dip, not an outage.
  Cross-provider fallback removes single-vendor dependency on the critical path.
- Every routing decision is logged with model, token counts, latency, cache hit, and tier —
  making cost a metric on a dashboard rather than a surprise on an invoice.
- New models can be evaluated by adding a route and comparing ledger data, without rewriting
  feature code.

**What got worse**

- **Multiple prompt surfaces to maintain.** A prompt tuned for Sonnet does not transfer
  cleanly to Haiku; each route needs its own tuning and its own regression checks.
- **Inconsistent output shape across models**, which forces defensive parsing and schema
  validation on every response.
- **Routing itself can be wrong.** A request misclassified as simple gets a shallow answer.
  This is the failure mode that hurts users most and is hardest to detect automatically — it
  needs sampled human review of routed outputs.
- **A second vendor** (Google) in the compliance and data-handling story, with its own terms,
  region behaviour, and review burden.
- **Evaluation cost.** Comparing models honestly means maintaining a golden-set eval, which is
  ongoing work that produces no visible feature.

## Revisit when

- Frontier-model pricing drops enough that routing complexity costs more in engineering time
  than it saves in inference — this has happened repeatedly in this market and should be
  re-checked on every major price change.
- Misrouting shows up in sampled quality review above an acceptable rate.
- A single model appears that is genuinely good enough at the cheap tier's price, at which
  point collapsing back to one model is a simplification worth taking.
