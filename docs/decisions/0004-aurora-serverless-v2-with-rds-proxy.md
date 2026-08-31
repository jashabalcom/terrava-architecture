# ADR-0004: Aurora Serverless v2 behind RDS Proxy

**Status:** Superseded by [ADR-0017](0017-retire-vpc-aurora-elasticache.md) (2026-07)
**Date:** 2025
**Scope:** DatabaseStack (retired)

> **This decision is no longer in effect.** Aurora and RDS Proxy were deleted in July 2026
> when the system of record moved to Supabase. The reasoning below remains valid for a
> VPC-resident relational database and is kept as the record of why that architecture was
> chosen — but it does not describe the platform as it runs today. See
> [ADR-0017](0017-retire-vpc-aurora-elasticache.md).

## Context

Lambda and relational databases have a structural conflict. Lambda scales by creating
independent execution environments, each of which wants its own database connection. Postgres
allocates roughly 10 MB of memory per connection and starts degrading well before a thousand
of them. A traffic spike that scales Lambda to 1,000 concurrent executions will exhaust
connections on a database that is otherwise nowhere near CPU or IO limits — the database falls
over from connection pressure, not load.

At the same time, the workload is peaky. Provisioning a database for peak means paying for
peak 24 hours a day, when actual utilization is a fraction of that outside market hours.

Two problems: connection multiplication, and paying for idle capacity.

## Options considered

### Option A — Provisioned RDS/Aurora sized for peak

Predictable performance, predictable bill. Rejected: it pays for peak capacity around the
clock and still has the connection problem. Solving connections would need an application-side
pooler anyway.

### Option B — DynamoDB for everything, no relational database

Scales with Lambda natively — no connection concept at all, so the whole class of problem
disappears. Rejected because the domain is relational. Portfolios, properties, users,
subscriptions, agent pipelines and their relationships need joins, aggregate queries across
entities, and transactional consistency. Modelling that in single-table DynamoDB means
designing every access pattern up front and rewriting the table when a new query appears —
exactly the wrong trade for a product still discovering its access patterns.

DynamoDB *is* used, but for what it's good at: high-throughput key-value work with a natural
TTL — the AI response cache, token ledger, prompt registry, sessions (ADR-0006, ADR-0016).

### Option C — Aurora Serverless v1

Scales to zero, which is attractive for dev. Rejected: v1 scales in coarse steps with
disruptive pauses and resumes, and the cold-resume latency is unacceptable on an interactive
path.

### Option D — Aurora Serverless v2 + RDS Proxy

v2 scales in fine-grained 0.5 ACU increments, in-place, without dropping connections. RDS
Proxy sits between Lambda and the cluster, maintaining a warm pool and multiplexing many
client connections onto few database connections. It also holds connections open across
Lambda cold starts and fails over transparently.

## Decision

**Aurora PostgreSQL Serverless v2 (0.5–16 ACU) behind RDS Proxy**, with a read replica in a
separate AZ.

RDS Proxy multiplexes ~1,000 Lambda connections down to 20–50 real database connections.
Serverless v2 auto-scaling cut database cost roughly 60% against always-on provisioned
capacity.

## Consequences

**What got better**

- The connection-exhaustion failure mode is gone. Lambda concurrency and database connection
  count are decoupled.
- ~60% database cost reduction versus always-on provisioned, by not paying for peak overnight.
- Failover is handled by the Proxy — it holds the client connection while the cluster promotes
  the replica, so a failover looks like a brief stall rather than a wave of connection errors.
- Read replica in a second AZ absorbs read-heavy analytics queries and provides failover
  capacity.
- Both live in isolated subnets with no internet route (ADR-0010).

**What got worse**

- **RDS Proxy is a fixed hourly cost per instance**, so it partly offsets the Serverless v2
  saving at low traffic. At very low volume the Proxy is the more expensive component.
- **Pinning.** Session-level state — `SET` statements, prepared statements in some drivers,
  explicit transactions — forces the Proxy to pin a client to a backend connection for the
  session's duration, which defeats multiplexing. Application code has to be written to avoid
  pinning, and that constraint is invisible until throughput quietly degrades.
- **An extra network hop** on every query, adding a small amount of latency.
- **The 0.5 ACU floor is not zero.** The database costs money every hour of every day, unlike
  the Lambda tier.
- **Scaling is fast but not instant.** A vertical traffic spike can briefly outrun ACU
  scale-up, showing as elevated query latency for tens of seconds.

## Revisit when

- Baseline load stays consistently near the 16 ACU ceiling — at that point reserved provisioned
  capacity is cheaper than serverless.
- Baseline load stays consistently at the 0.5 ACU floor with the Proxy dominating the bill —
  at that point the Proxy may not be earning its keep.
- Connection pinning shows up in Proxy metrics as a sustained share of connections, which means
  the multiplexing benefit is being lost and the application needs fixing (or a different
  pooling strategy).
