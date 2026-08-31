# ADR-0010: Three-tier VPC with a data tier that has no internet route

**Status:** Superseded by [ADR-0017](0017-retire-vpc-aurora-elasticache.md) (2026-07)
**Date:** 2025
**Scope:** VpcStack (retired), SecurityStack, FrontendStack

> **The VPC described here no longer exists.** It was retired in July 2026 once the data tier
> it protected was gone — every remaining egress target is a public endpoint, so the VPC
> granted access to nothing private while adding a NAT single point of failure. The
> isolated-tier reasoning below stands for any future VPC-resident datastore. The WAF, KMS,
> Secrets Manager, CloudTrail, and GuardDuty controls described here are unaffected and remain
> in place. See [ADR-0017](0017-retire-vpc-aurora-elasticache.md).

## Context

The platform holds financial profiles, portfolio positions, and identity documents for
investors across 11 markets. The relevant threat isn't only "can an attacker get in" — it's
"if an attacker gets code execution somewhere, can they get data out."

Exfiltration needs an egress path. Most breaches escalate from a foothold in application code
to data leaving the network. If the database has no route to the internet, that escalation
path doesn't exist, regardless of what the attacker achieved at the application layer.

## Options considered

### Option A — Public subnets with security groups

Simple, no NAT costs. Rejected: security groups are the only thing between the database and
the internet. One misconfigured rule — and rules get changed under pressure during incidents —
is a directly reachable database. Defense that depends on a single control being permanently
correct is not defense in depth.

### Option B — Two tiers (public + private with NAT)

Better: nothing is directly reachable and everything egresses through NAT. Rejected as
insufficient for the data tier. Lambda legitimately needs egress (Bedrock, Stripe, market data
APIs), but **Aurora and Redis never do**. Giving them a NAT route grants an exfiltration path
that has no legitimate use.

### Option C — Three tiers across three AZs

- **Public** — NAT gateways and load balancers only.
- **Private** — Lambda functions. Egress to the internet via NAT, no inbound from it.
- **Isolated** — Aurora and ElastiCache. **No internet route at all**, in either direction.

VPC endpoints provide access to AWS services (S3, Secrets Manager, DynamoDB, Bedrock) without
traversing the internet, which also removes NAT data-processing charges for that traffic.

## Decision

**Three-tier VPC across three Availability Zones, with the data tier in isolated subnets that
have no internet route**, plus VPC endpoints for AWS service access.

Layered with WAFv2 at CloudFront and API Gateway, KMS customer-managed keys for data at rest,
Secrets Manager for all credentials, and CloudTrail feeding GuardDuty.

## Consequences

**What got better**

- Data-tier exfiltration requires first compromising a Lambda in the private tier *and* using
  it as a relay. Two steps instead of one, with the second producing anomalous traffic patterns
  that GuardDuty is positioned to catch.
- Three AZs give real availability: an AZ failure loses capacity, not the service.
- VPC endpoints keep AWS service traffic off the public internet entirely, which is both a
  security win and a NAT cost reduction.
- It maps cleanly onto what security reviewers and enterprise customers expect to see, which
  makes due-diligence conversations short.

**What got worse**

- **NAT gateways are expensive** — an hourly charge per AZ plus per-GB processing. Across three
  AZs this is one of the larger fixed infrastructure costs, and it's paid whether or not
  traffic flows.
- **VPC-attached Lambda cold starts** carry ENI setup. Hyperplane ENIs have largely fixed this,
  but it still shows in p99.
- **Debugging is harder.** No SSH to a bastion in the data tier by default; reaching the
  database for a migration or an investigation needs a deliberate, audited path (Session
  Manager through a private-tier host), which is correct but slower under incident pressure.
- **Every new AWS service integration needs an endpoint decision** — add a VPC endpoint, or
  route through NAT and pay for it. Forgetting this produces a confusing timeout rather than a
  clear error.
- More subnets, route tables, and security groups to reason about; the network is genuinely
  more complex than a flat one.

## Revisit when

- NAT gateway cost becomes a top-three line item — consolidating to fewer AZs for NAT (trading
  some availability) or adding more VPC endpoints becomes the lever.
- A workload needs the data tier to reach an external service, which should be treated as a
  design smell and solved with a private-tier proxy rather than by opening the isolated tier.
