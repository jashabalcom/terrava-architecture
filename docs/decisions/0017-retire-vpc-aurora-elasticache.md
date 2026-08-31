# ADR-0017: Retire the VPC, NAT Gateway, Aurora, and ElastiCache tier

**Status:** Accepted — supersedes ADR-0004, ADR-0010, and ADR-0012
**Date:** 2026-07
**Scope:** VpcStack, DatabaseStack, CacheStack (all three removed)

## Context

ADR-0004, ADR-0010, and ADR-0012 describe a data tier built around Aurora Serverless v2
behind RDS Proxy, ElastiCache Redis, and a three-tier VPC with a no-internet isolated subnet.
That was the right architecture for the data plane as it existed when those decisions were
made.

The data plane then changed underneath it:

1. **The system of record moved to Supabase** — a managed Postgres SaaS reached over the
   public internet via TLS. Application Lambdas stopped opening Postgres connections to the
   in-VPC Aurora cluster. A search across the function source found **zero** direct
   `pg`/RDS connections.
2. **ElastiCache was deleted** during earlier cost work. `REDIS_ENDPOINT` was unset on the
   live functions, the Redis helper short-circuited to `null`, and the rate-limit check fell
   back to "allow." Redis was providing no functionality.
3. **Aurora was deleted** in the same cost work. `describe-db-clusters` returned an empty
   list. The DatabaseStack was a shell: two security groups, an orphaned master secret, and a
   `DatabaseCluster` construct with `deletionProtection: true` and `removalPolicy: RETAIN`
   that had no physical counterpart — CloudFormation drift in its purest form.

The result was a VPC whose remaining egress targets — Supabase, Stripe, Bedrock — were **all
public endpoints**. The functions sat in private subnets, routed every outbound call through a
single NAT Gateway, and gained access to nothing private in exchange.

That NAT then proved to be an active liability rather than a dormant cost. When it was removed
during unrelated infrastructure work, **every VPC-attached Lambda lost internet egress
simultaneously** and the AI features returned 500s across the board. A single point of failure
for a tier that was protecting nothing.

## Options considered

### Option A — Leave it in place

Zero effort, and a VPC is already there if a future workload needs one. Rejected: it pays
roughly $40–52/month for resources nothing uses, keeps a NAT single point of failure in the
egress path of every function, and preserves CloudFormation drift. "Keep it in case" is not
worth a standing monthly charge when CDK can recreate the whole tier in minutes.

### Option B — Keep the VPC and go fully private (PrivateLink for every AWS service)

The theoretically strongest network posture: Bedrock, Secrets Manager, S3, KMS, STS, and
CloudWatch all reached over interface endpoints, never the public internet.

Rejected, and this is the interesting one. **The system of record is Supabase, a public
SaaS.** The single most sensitive traffic in the platform — the database connection —
traverses the public internet over TLS no matter what happens at the VPC layer. A "nothing
touches the public internet" guarantee is architecturally unreachable while Supabase is the
system of record. This option buys the full cost and operational complexity of private
networking (~$7/month per interface endpoint, times many services) while still failing the
guarantee that justifies it. It optimizes a control that cannot be made whole here.

### Option C — Delete only the NAT Gateway, keep the VPC

Removes the largest single cost with the smallest change. Rejected as a half-measure: with no
workload in the VPC, the subnets, flow logs, endpoints, and DatabaseStack shell still cost
money and still carry drift. And a `PRIVATE_WITH_EGRESS` subnet with no NAT is a
misconfiguration trap for whoever touches it next. If nothing is in the VPC, remove the VPC.

### Option D — Detach, verify, then tear down

Two steps, deliberately separated:

1. **Detach** — remove the VPC configuration from the shared Lambda props. The `cdk diff` is
   purely subtractive: 37 functions each losing `VpcConfig`, zero resources added or removed,
   zero IAM changes, zero security-group changes. Fully reversible by redeploying the prior
   template.
2. **Tear down** — after production health is verified and AWS has reclaimed the Lambda ENIs,
   destroy DatabaseStack and then VpcStack in dependency order.

## Decision

**Detach all Lambda functions from the VPC, then retire VpcStack, DatabaseStack, and
CacheStack entirely.** Functions now run in the AWS-managed Lambda network with native
internet egress.

This follows the AWS Well-Architected **Serverless Lens** directly: do not place a function in
a VPC unless it must reach VPC-only resources.

## Consequences

**What got better**

- **The NAT single point of failure is gone from the egress path.** This is the primary win —
  it removes the exact failure mode that had already caused a full AI outage.
- Eliminates the NAT SNAT-port exhaustion ceiling under high concurrency, and the residual
  VPC cold-start ENI overhead.
- Removes roughly $40–52/month of standing spend: the NAT Gateway (~$32–45 plus $0.045/GB),
  the Secrets Manager interface endpoint (~$7), and the orphaned Aurora master secret.
- Removes CloudFormation drift — the phantom `RETAIN` cluster and two unused security groups.
- The deployed footprint finally matches the real data plane. Fewer components to patch,
  monitor, and reason about.

**What got worse**

- **Loss of a fixed egress IP.** Outbound calls now originate from AWS's shared address pool
  rather than the NAT's Elastic IP, which was released permanently. This only matters if an
  upstream IP-allowlists us — none do today, since Supabase, Stripe, and Bedrock all
  authenticate by key or IAM rather than source address. A future bank feed or government API
  that requires a static source IP would need a dedicated egress path for *that call*, not a
  return of the whole fleet to a VPC.
- **Loss of Lambda-level VPC Flow Logs.** Egress from these functions is no longer captured at
  the network layer. Application logging and CloudTrail remain, and are now the primary audit
  trail for function behaviour.
- **Rate limiting lost its shared counter.** With Redis gone, the sliding-window limiter
  described in ADR-0012 has no backing store and falls back to allow. This is a real gap, not
  a neutral consequence: per-tier quota enforcement needs a replacement before it can be
  relied on. A serverless cache with a public endpoint is the natural fit, precisely because
  it does not require reintroducing a VPC.
- **Relational data now lives with a third-party SaaS**, which moves durability, backup, and
  availability guarantees from AWS-managed infrastructure to a vendor relationship.
- Recreating an in-VPC datastore later means standing VpcStack back up — a CDK deploy, but a
  deliberate one.

## What this replaces in the earlier records

| Superseded claim | Current reality |
|---|---|
| ADR-0004: Aurora Serverless v2 behind RDS Proxy, 1,000 → 20–50 connections | Aurora and RDS Proxy are deleted. System of record is Supabase (managed Postgres) |
| ADR-0010: three-tier VPC, data tier with no internet route | No VPC. Lambdas run in the AWS-managed network with native egress |
| ADR-0012: ElastiCache Redis for caching and rate limiting | ElastiCache is deleted. Rate limiting currently falls back to allow |

ADR-0004's connection-pooling reasoning and ADR-0010's isolated-tier reasoning both remain
correct **for a VPC-resident datastore**. They are superseded because the premise changed, not
because the analysis was wrong. If a private datastore returns, those records are the starting
point.

## Revisit when

- An integration requires an IP-allowlisted static egress address — solve it for that one call
  path, not by reattaching the fleet.
- Rate limiting needs real enforcement again, which it does. The replacement should be a
  serverless cache with a public endpoint rather than a new VPC.
- A workload appears that genuinely needs a private datastore — at which point stand up
  VpcStack and attach *only* the functions that need it.
