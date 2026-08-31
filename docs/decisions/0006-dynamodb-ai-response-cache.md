# ADR-0006: DynamoDB AI response cache with 24-hour TTL

**Status:** Accepted
**Date:** 2025
**Scope:** AIInfraStack

## Context

AI requests in this domain repeat heavily. Many users ask about the same neighbourhoods, run
the same calculators against the same well-known buildings, and ask identical regulatory
questions ("what's the Golden Visa threshold?"). Each repeat is a full-price inference call
producing a near-identical answer.

The cheapest inference call is the one that doesn't happen.

## Options considered

### Option A — No cache

Every answer freshly generated, always current. Rejected: it's paying repeatedly for the same
computation, and it's also *slower* — a cache hit returns in milliseconds where an inference
call takes seconds.

### Option B — ElastiCache Redis for AI responses

Redis is already in the stack (ADR-0012) and is faster than DynamoDB. Rejected for this
specific use: AI responses are large, long-lived, and low-frequency per key. Holding them in
memory means paying memory prices for cold data, and Redis Serverless bills on stored data.
Redis is kept for hot, short-lived, high-frequency work — rate-limit counters and API response
caching.

### Option C — DynamoDB with native TTL

Pay-per-request pricing, so cost tracks actual usage. Native TTL expires items automatically
with no sweeper job. Single-digit-millisecond reads, which is far below inference latency and
therefore fast enough. Durable, so the cache survives a deployment.

## Decision

**DynamoDB response cache, keyed on a hash of prompt plus context, with a 24-hour TTL.**

Measured 40–60% cache hit rate on production traffic.

The key is a hash of the normalized prompt *and* the relevant context — user tier, market,
model, prompt-registry version. Context must be in the key: the same question from a Free user
in Dubai and a Pro user in London are different questions with different correct answers.

The 24-hour TTL is a deliberate freshness/cost trade. Property market data, regulations, and
fee structures do move, but not hourly. A day-old answer to "what's the average yield in
Downtown Dubai?" is materially correct; a day-old answer is not acceptable for anything
priced live, so live-priced paths bypass the cache entirely.

## Consequences

**What got better**

- 40–60% of AI requests cost nothing in inference. Combined with routing (ADR-0005), this is
  the other half of the ~60% cost reduction.
- Cached requests return in milliseconds instead of seconds — the cache is a latency
  improvement as much as a cost one.
- Cache hits don't consume Bedrock throughput, so they don't contribute to throttling during
  traffic spikes.
- Hit rate is a first-class metric; a sudden drop is an early signal that prompts or context
  keys changed unintentionally.

**What got worse**

- **Staleness is real.** Up to 24 hours. Mitigated by keeping live-priced and
  portfolio-specific paths uncached, but it's a permanent correctness caveat that has to be
  reasoned about per feature.
- **Cache key design is subtle and unforgiving.** Too coarse and users get answers meant for
  someone else — a genuine data-leak class of bug in a multi-tenant product. Too fine and the
  hit rate collapses. The prompt-registry version must be part of the key, or a prompt update
  silently serves answers from the old prompt for a day.
- **A poisoned entry persists.** A bad generation that lands in the cache is served for up to
  24 hours to everyone who matches the key. This needs a manual invalidation path.
- **Storage cost grows with prompt diversity**, and unbounded diversity means an unbounded
  table — TTL is what keeps it bounded.

## Revisit when

- Hit rate drops below ~30%, which would mean prompt diversity has outgrown the caching
  strategy and per-user personalization is defeating the key.
- A feature needs sub-hour freshness broadly, rather than as a per-path exception.
- Inference prices fall far enough that cache complexity and staleness risk cost more than the
  calls they avoid.
