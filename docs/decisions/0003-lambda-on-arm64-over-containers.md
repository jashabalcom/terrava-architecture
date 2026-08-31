# ADR-0003: Lambda on ARM64/Graviton2 instead of containers

**Status:** Accepted
**Date:** 2025
**Scope:** All 77 functions

## Context

The platform's traffic is spiky and unevenly distributed: interactive calculator and AI
requests during market hours, scheduled sync and digest jobs overnight, near-zero at other
times. There is no operations team — the same person writing features is the one paged at 3am.

Two questions had to be answered together: what compute primitive, and what CPU architecture.

## Options considered

### Option A — ECS Fargate or EKS

Long-running containers, no cold starts, no 15-minute execution ceiling, straightforward for
persistent connections. Rejected on cost shape and operational load: containers bill for idle
time, and with traffic that goes to near-zero for hours, most of the bill would be for
capacity doing nothing. EKS additionally means owning cluster upgrades, node patching, and
autoscaler tuning — real work for a team of one.

### Option B — Lambda on x86_64

The default. Rejected once ARM64 was benchmarked: same performance for this workload at
roughly 20% lower cost, with the only migration cost being native-dependency compatibility.

### Option C — Lambda on ARM64/Graviton2

~20% cheaper per GB-second than x86_64. Node.js 20 runs natively on ARM. The workload is
I/O-bound — database queries, HTTP calls to Bedrock and third-party APIs — so it doesn't lean
on x86-specific instruction sets or vectorized native code.

## Decision

**Lambda on ARM64/Graviton2 across all 77 functions, with shared Lambda Layers for common
dependencies.** ~20% compute reduction with no measured performance regression, and no servers
to patch.

## Consequences

**What got better**

- ~20% lower compute cost than the x86 equivalent, fleet-wide.
- Scale-to-zero: overnight and weekend idle costs nothing.
- No OS patching, no node pools, no cluster upgrades — the operational surface a solo builder
  can actually sustain.
- Shared Layers keep the AWS SDK, database client, and common utilities in one place instead of
  bundled 77 times, cutting both deploy artifact size and cold-start unpack time.

**What got worse**

- **Cold starts.** VPC-attached Lambdas that talk to Aurora carry ENI setup on a cold start.
  Hyperplane ENIs have made this far less painful than it once was, but the tail latency is
  real and shows up in p99, not p50.
- **Native dependencies must be ARM-compatible.** Anything with a compiled binary needs an ARM
  build or a Docker-based bundling step. This is the tax that gets paid every time a new
  dependency is added.
- **The 15-minute ceiling** is a hard architectural constraint. Long jobs must be decomposed
  and driven by EventBridge and SQS rather than written as a single script (see ADR-0012's
  neighbor concerns and the async design in `ARCHITECTURE.md`).
- **Layers are a versioning trap.** A Layer update that breaks a rarely-invoked function may
  not surface until that function next runs — potentially days later.
- Per-function concurrency limits and account-level concurrency become a capacity-planning
  concern that containers wouldn't have.

## Revisit when

- Sustained baseline traffic gets high enough that provisioned container capacity is cheaper
  than per-invocation billing — for this workload shape, roughly when utilization stops
  dropping to near-zero for large parts of the day.
- A workload appears that genuinely needs >15 minutes or persistent connections (a streaming
  ingest pipeline, a WebSocket fanout) and decomposition stops being natural.
- Cold-start p99 on interactive paths becomes a user-visible complaint that provisioned
  concurrency can't economically fix.
