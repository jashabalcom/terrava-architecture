# ADR-0012: ElastiCache Redis Serverless for API caching and rate limiting

**Status:** Superseded by [ADR-0017](0017-retire-vpc-aurora-elasticache.md) (2026-07)
**Date:** 2025
**Scope:** CacheStack (retired)

> **ElastiCache was deleted in July 2026.** The rate-limiting design below is the part that
> matters and the part that is currently missing: with no shared counter, the limiter falls
> back to allow. The analysis of *why* Lambda needs a shared atomic counter remains correct
> and should drive the replacement — a serverless cache with a public endpoint, not a
> reintroduced VPC. See [ADR-0017](0017-retire-vpc-aurora-elasticache.md).

## Context

Two problems shared a solution.

**Read amplification.** Property listings, neighbourhood data, and market statistics are read
constantly and change rarely. Every one of those reads was hitting Aurora, which meant paying
ACU for queries whose answers hadn't changed since the last hundred identical queries.

**Rate limiting.** Tiers have request quotas — the extension's 3/day for Investor, unlimited
for Pro — and AI endpoints need abuse protection independent of tier. Enforcing a quota
requires shared state across every concurrently executing Lambda. Lambda has no shared memory,
so per-instance counters are meaningless: 200 concurrent executions each counting to 3 lets
through 600 requests.

Both needs are the same shape: fast, shared, ephemeral state with atomic operations.

## Options considered

### Option A — Rate limit in Aurora

The database is already there. Rejected: it puts a write on every single request, against the
database this is meant to protect, and row-level contention on hot counters is a bad
throughput story. It solves read load by adding write load.

### Option B — DynamoDB for both

Serverless, scales well, atomic counters. Rejected for these two uses — sub-millisecond
latency matters when it's on every request, and a high-frequency counter with per-request
write costs is exactly the access pattern DynamoDB prices least favourably. DynamoDB is used
where it fits better: durable, TTL'd, lower-frequency AI data (ADR-0006, ADR-0016).

### Option C — API Gateway throttling

Free and built in. Kept as a coarse layer, rejected as the mechanism: it throttles per route
and per stage, not per user per tier. It can't express "this Investor user has used 3 of 3
extension analyses today."

### Option D — ElastiCache Redis Serverless

Sub-millisecond, atomic `INCR`/`EXPIRE` primitives that make sliding-window rate limiting a
few operations, and native TTL. Serverless means no node sizing, no cluster management, and
capacity that tracks usage.

## Decision

**ElastiCache Redis Serverless for API response caching and sliding-window rate limiting**,
in the isolated subnet tier alongside Aurora (ADR-0010).

Database load reduced ~73% during peak hours.

Division of labour across the three data stores is deliberate:

| Store | Used for | Why |
|---|---|---|
| **Aurora** | Relational domain data | Joins, transactions, aggregates |
| **Redis** | Hot cache, rate-limit counters | Sub-ms, atomic, ephemeral |
| **DynamoDB** | AI cache, token ledger, prompt registry, sessions | Durable, TTL, pay-per-request |

## Consequences

**What got better**

- ~73% less database load at peak, which directly reduces Aurora ACU consumption and the bill
  (ADR-0004).
- Rate limiting is actually correct across concurrent Lambdas, because the counter is shared
  and atomic.
- Sub-millisecond cache reads make cached endpoints dramatically faster, not just cheaper.
- Serverless removes node-sizing and failover management.
- Redis absorbs traffic spikes that would otherwise force Aurora to scale up, so it smooths the
  cost curve as well as the load curve.

**What got worse**

- **A third data store to reason about.** Every new feature now needs a deliberate answer to
  "where does this live," and the wrong answer is easy.
- **Cache invalidation.** The standard hard problem: stale listings after an update need an
  explicit invalidation path, and a missed invalidation is a user seeing wrong data with no
  error anywhere.
- **Redis is now on the critical path** for rate-limited endpoints. If it's unavailable, the
  choice is fail-open (no rate limiting, abuse risk) or fail-closed (users blocked). This has
  to be decided consciously per endpoint — AI endpoints fail closed because the downside is
  uncontrolled spend; read endpoints fail open to the database.
- **Redis Serverless has a minimum cost** and bills on stored data, so an unbounded cache is an
  unbounded bill — every key needs a TTL.
- Being in the isolated tier means no direct inspection from a developer machine; debugging
  cache state needs a path through the private tier.

## Revisit when

- Cache hit rate doesn't justify the minimum serverless cost.
- Fail-open/fail-closed behaviour causes a real incident, which would argue for a local
  in-Lambda fallback counter that degrades gracefully rather than binary behaviour.
- Rate-limiting needs grow into multi-dimensional quotas complex enough to want a dedicated
  quota service.
