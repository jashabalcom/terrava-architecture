# Architecture Decision Records

Every significant architectural decision in Terrava, written down with the context that
forced it, the options that were rejected, and the cost that was accepted.

## Why these exist

A diagram tells you what the system *is*. It doesn't tell you why it isn't something else.
The interesting engineering is in the rejected options — and that's the part that gets lost
six months later, when someone (often the author) asks "why on earth did we do it this way?"

Each record answers five questions:

1. **Context** — what forced a decision
2. **Options** — what was on the table, and why each alternative lost
3. **Decision** — what was chosen
4. **Consequences** — what got better, and what got worse
5. **Revisit when** — the concrete signal that would make this decision wrong

The fifth is the one most teams skip. A decision without a trigger for revisiting it becomes
a permanent constraint by accident.

## The records

| # | Decision | Status | Primary driver |
|---|---|---|---|
| [0001](0001-cdk-typescript-for-all-infrastructure.md) | AWS CDK in TypeScript for all infrastructure | Accepted | Reproducibility |
| [0002](0002-http-api-v2-over-rest-api.md) | API Gateway HTTP API v2 instead of REST API | Accepted | Cost |
| [0003](0003-lambda-on-arm64-over-containers.md) | Lambda on ARM64/Graviton2 instead of containers | Accepted | Cost + operations |
| [0004](0004-aurora-serverless-v2-with-rds-proxy.md) | Aurora Serverless v2 behind RDS Proxy | **Superseded by 0017** | Cost + connection limits |
| [0005](0005-multi-model-ai-router.md) | Multi-model AI router instead of a single model | Accepted | Unit economics |
| [0006](0006-dynamodb-ai-response-cache.md) | DynamoDB response cache with 24h TTL | Accepted | Unit economics |
| [0007](0007-rag-over-fine-tuning.md) | RAG over Knowledge Bases instead of fine-tuning | Accepted | Correctness + cost |
| [0008](0008-bedrock-guardrails-for-financial-compliance.md) | Bedrock Guardrails as a hard compliance gate | Accepted | Regulatory |
| [0009](0009-custom-es256-jwt-authorizer.md) | Custom ES256 JWT authorizer alongside Cognito | Accepted | Latency |
| [0010](0010-three-tier-vpc-isolation.md) | Three-tier VPC with a no-internet data tier | **Superseded by 0017** | Security |
| [0011](0011-stripe-webhook-as-source-of-truth.md) | Stripe webhooks as the billing source of truth | Accepted | Correctness |
| [0012](0012-redis-serverless-for-cache-and-rate-limiting.md) | Redis Serverless for caching and rate limiting | **Superseded by 0017** | Database load |
| [0013](0013-two-products-one-platform.md) | Two products on one shared platform | Accepted | Leverage |
| [0014](0014-config-driven-market-expansion.md) | Config-driven market expansion in the extension | Accepted | Speed of expansion |
| [0015](0015-github-actions-oidc-no-static-credentials.md) | GitHub Actions OIDC, zero static credentials | Accepted | Security |
| [0016](0016-prompt-registry-in-dynamodb.md) | Prompt and guardrail config in DynamoDB, not code | Accepted | Iteration speed |
| [0017](0017-retire-vpc-aurora-elasticache.md) | Retire the VPC, NAT, Aurora, and ElastiCache tier | Accepted | Reliability + cost |

**Reading the superseded records.** ADR-0004, ADR-0010, and ADR-0012 describe a data tier that
was retired in July 2026. They are kept, not deleted — the reasoning in them is still correct
for the premise they were written under, and the trail from "build the isolated tier" to
"remove it once it protected nothing" is more informative than either decision alone. Start
with [ADR-0017](0017-retire-vpc-aurora-elasticache.md) for what runs today.

## Format

New records use [`0000-template.md`](0000-template.md). Number them sequentially, never
renumber, and never delete one — a superseded decision gets its status changed to
`Superseded by ADR-NNNN` and stays in place. The trail of reversals is as informative as
the decisions themselves.

## A note on the numbers

Cost and performance figures cited in these records reflect the platform at its documented
scale and are point-in-time measurements, not guarantees. Percentages are measured against
the alternative that was actually running before the change, not against a theoretical
baseline. Where a figure is an estimate rather than a measurement, it says so.
