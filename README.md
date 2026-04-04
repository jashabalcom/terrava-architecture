# Terrava — Enterprise Wealth Intelligence Platform

> Production-grade multi-tenant SaaS on AWS | AI-powered investment analysis across 11 global markets

[![AWS](https://img.shields.io/badge/AWS-14_CDK_Stacks-FF9900?logo=amazonaws)](https://aws.amazon.com)
[![Lambda](https://img.shields.io/badge/Lambda-77_Functions-FF9900?logo=awslambda)](https://aws.amazon.com/lambda/)
[![Bedrock](https://img.shields.io/badge/AI-Amazon_Bedrock-232F3E?logo=amazonaws)](https://aws.amazon.com/bedrock/)
[![Markets](https://img.shields.io/badge/Markets-11_Countries-blue)](.)
[![Extension](https://img.shields.io/badge/Chrome_Extension-40%2B_Sites-4285F4?logo=googlechrome)](.)

---

## What is Terrava?

Terrava is an enterprise wealth intelligence platform that helps real estate investors analyze properties across global markets using AI. It includes a Chrome Extension that works on 40+ property listing sites worldwide, providing instant investment analysis powered by Amazon Bedrock.

**This repository contains the architecture documentation, design decisions, and code patterns** from building Terrava. The production codebase is private.

---

## Architecture Overview

### System Architecture

```
                                    +------------------+
                                    |   CloudFront     |
                                    |   (400+ edges)   |
                                    +--------+---------+
                                             |
                         +-------------------+-------------------+
                         |                                       |
                +--------v---------+                   +---------v--------+
                |   S3 (Frontend)  |                   |  WAFv2           |
                |   React SPA      |                   |  Rate Limiting   |
                +------------------+                   |  Geo-blocking    |
                                                       +---------+--------+
                                                                 |
                                                       +---------v--------+
                                                       |  API Gateway     |
                                                       |  HTTP API v2     |
                                                       |  77 routes       |
                                                       +---------+--------+
                                                                 |
                                              +------------------+------------------+
                                              |                  |                  |
                                    +---------v-----+  +---------v-----+  +---------v-----+
                                    |  Lambda        |  |  Lambda        |  |  Lambda        |
                                    |  API Handlers  |  |  AI Pipeline   |  |  Background    |
                                    |  (48 fns)      |  |  (8 fns)       |  |  (15 fns)      |
                                    +---------+------+  +---------+------+  +---------+------+
                                              |                  |                  |
                         +--------------------+------------------+------------------+
                         |                    |                  |                  |
                +--------v----+    +----------v---+    +---------v---+    +---------v--------+
                | Aurora       |    | DynamoDB     |    | ElastiCache |    | Bedrock          |
                | Serverless   |    | (Events,     |    | Redis       |    | Claude 3 Sonnet  |
                | v2 (PG 15)   |    |  Market Data)|    | (Cache)     |    | Claude 3 Haiku   |
                +--------------+    +--------------+    +-------------+    | Knowledge Bases  |
                                                                          +------------------+
```

### Infrastructure Stacks (14 CDK Stacks)

```
Stack Dependency Chain:

  IamStack
     |
  FrontendStack (S3 + CloudFront + WAFv2)
     |
  VpcStack (3-tier: public / private / isolated)
     |
  DatabaseStack (Aurora Serverless v2 + RDS Proxy)
     |
  CacheStack (ElastiCache Redis)
     |
  ApiStack (API Gateway HTTP v2 + 77 Lambda functions)
     |
  +--- MessagingStack (EventBridge + SQS + DLQs)
  |
  +--- DomainStack (Route 53 + ACM)
  |
  +--- EmailStack (SES)
  |
  +--- CognitoStack (User Pools + MFA)
  |
  +--- ObservabilityStack (CloudWatch Dashboards + Alarms)
  |
  +--- BedrockKBStack (Knowledge Base + OpenSearch Serverless)
  |
  +--- SecurityStack (GuardDuty + Config Rules)
  |
  +--- SyntheticsStack (CloudWatch Canaries)
  |
  +--- DynamoDBStack (Event tables + Market data)
```

### VPC Architecture (3-Tier Network Isolation)

```
+------------------------------------------------------------------+
|  VPC (10.0.0.0/16)                                               |
|                                                                    |
|  +--------------------+  Public Subnets (10.0.0.0/20)            |
|  | NAT Gateway        |  - ALB (internet-facing)                  |
|  | Bastion (if needed)|  - NAT Gateways for private subnet egress |
|  +--------------------+                                           |
|                                                                    |
|  +--------------------+  Private Subnets (10.0.16.0/20)           |
|  | Lambda Functions   |  - Outbound internet via NAT              |
|  | ECS Tasks          |  - No inbound from internet               |
|  | RDS Proxy          |  - Can reach isolated subnets             |
|  +--------------------+                                           |
|                                                                    |
|  +--------------------+  Isolated Subnets (10.0.32.0/20)          |
|  | Aurora PostgreSQL  |  - NO internet access (inbound or outbound)|
|  | ElastiCache Redis  |  - Only reachable from private subnets    |
|  |                    |  - Data exfiltration impossible            |
|  +--------------------+                                           |
+------------------------------------------------------------------+
```

### AI / Bedrock Pipeline

```
User Query
    |
    v
API Gateway --> Lambda (Router)
                   |
                   +---> Is it a chat question?
                   |        |
                   |        v
                   |     Bedrock Claude 3 Sonnet
                   |     (Wealth Advisory, 3-7s)
                   |
                   +---> Is it a listing analysis?
                   |        |
                   |        v
                   |     Bedrock Claude 3 Haiku        (extraction, 0.5s)
                   |        |
                   |        v
                   |     Market Defaults Engine         (calculations, <1ms)
                   |        |
                   |        v
                   |     Bedrock Claude 3 Sonnet        (narrative, 3s)
                   |
                   +---> Is it a regulatory question?
                            |
                            v
                         Bedrock Knowledge Base
                         (RAG: S3 docs --> Titan Embeddings --> OpenSearch --> Haiku)
                            |
                            v
                         Cited answer with source documents
```

### Chrome Extension Architecture

```
+--------------------------------------------------+
|  Chrome Extension (Manifest V3)                   |
|                                                    |
|  Content Script (injected into 40+ listing sites) |
|  +----------------------------------------------+ |
|  | 1. Detect listing page (URL pattern match)    | |
|  | 2. Extract price, size, location from DOM     | |
|  | 3. Calculate quick score (no API, instant)    | |
|  | 4. Inject overlay badge on listing            | |
|  +----------------------------------------------+ |
|                                                    |
|  Popup (detailed analysis view)                   |
|  +----------------------------------------------+ |
|  | 1. Send page text to API Gateway              | |
|  | 2. Receive AI analysis from Bedrock           | |
|  | 3. Display: yield, cap rate, mortgage, score  | |
|  | 4. Market-specific calculations per currency  | |
|  +----------------------------------------------+ |
|                                                    |
|  Service Worker (background)                      |
|  +----------------------------------------------+ |
|  | 1. Auth token management (JWT refresh)        | |
|  | 2. Analysis count tracking (tier limits)      | |
|  | 3. Badge state (green dot = listing detected) | |
|  +----------------------------------------------+ |
|                                                    |
|  Shared Config (market-defaults.js)               |
|  +----------------------------------------------+ |
|  | Per-market: mortgage, rent ratio, expenses,   | |
|  | benchmarks, vacancy rates                     | |
|  | 11 markets, 1 config file, 0 hardcoded values | |
|  +----------------------------------------------+ |
+--------------------------------------------------+

Supported Sites (40+):
US:  Zillow, Redfin, Realtor.com, Trulia
UAE: Bayut, PropertyFinder, Dubizzle
UK:  Rightmove, Zoopla, OnTheMarket
AU:  Domain, RealEstate.com.au
SG:  PropertyGuru, 99.co
CA:  Realtor.ca
EU:  Idealista (ES/PT/IT), SeLoger (FR), Funda (NL), ImmoScout24 (DE)
+15 more across Turkey, Japan, Thailand, Malaysia, etc.
```

### Security Architecture

```
Internet
    |
    v
+---+---+
|  WAF  |  Layer 1: Network edge protection
|       |  - AWS Managed Rules (OWASP Top 10)
|       |  - Rate limiting (1,000 req/IP/5min)
|       |  - Geo-blocking
+---+---+
    |
    v
+---+-------+
| CloudFront |  Layer 2: CDN + TLS termination
|            |  - Origin Access Control (S3 private)
|            |  - Custom headers
+---+--------+
    |
    v
+---+---------+
| API Gateway |  Layer 3: API protection
|             |  - Route-level throttling
|             |  - Request size limits
+---+---------+
    |
    v
+---+------+
| Cognito  |  Layer 4: Identity
|          |  - JWT validation on every request
|          |  - MFA (TOTP) for admin roles
|          |  - Tenant context in custom claims
+---+------+
    |
    v
+---+------+
| Lambda   |  Layer 5: Application
|          |  - Input validation (schema-based)
|          |  - Least-privilege IAM per function
|          |  - Structured logging (no PII)
+---+------+
    |
    v
+---+--------+
| Data Layer |  Layer 6: Data protection
|            |  - KMS encryption at rest (all stores)
|            |  - TLS in transit
|            |  - Row-Level Security (tenant isolation)
|            |  - Isolated subnets (no internet route)
+------------+

Compliance: CloudTrail (full audit) + GuardDuty (threat detection)
```

### Multi-Tenant Isolation

```
Request Flow:

Client --> API Gateway --> Lambda Authorizer
                              |
                              v
                          Decode JWT
                          Extract: tenant_id, user_id, role
                              |
                              v
                          Validate tenant exists + active
                              |
                              v
                          Inject tenant context into request
                              |
                              v
                          Business Logic Lambda
                              |
                              v
                          Aurora PostgreSQL
                          +----------------------------------+
                          | Row-Level Security Policy:       |
                          | current_setting('app.tenant_id') |
                          | = row.tenant_id                  |
                          |                                  |
                          | Even if app code has a bug,      |
                          | DB enforces isolation.           |
                          +----------------------------------+
```

---

## AWS Services Used (20+)

| Category | Service | Role in Terrava |
|----------|---------|-----------------|
| **Compute** | Lambda (77 functions) | API handlers, AI pipeline, background processors |
| **API** | API Gateway HTTP v2 | 77 routes, JWT auth, CORS |
| **Database** | Aurora Serverless v2 | Users, portfolios, transactions (PostgreSQL 15) |
| **Database** | DynamoDB | Market data, events, session state |
| **Cache** | ElastiCache Redis | API caching, rate limiting, sessions |
| **Storage** | S3 | Frontend hosting, documents, audit archives |
| **CDN** | CloudFront | Global edge delivery, API caching |
| **AI** | Bedrock (Claude 3) | Wealth advisory, listing analysis, risk narratives |
| **AI** | Bedrock Knowledge Bases | Regulatory RAG (DIFC, ADGM documents) |
| **Auth** | Cognito | Multi-tenant user pools, MFA, JWT |
| **Security** | WAFv2 | DDoS protection, rate limiting, OWASP rules |
| **Security** | KMS | Encryption at rest for all data stores |
| **Security** | Secrets Manager | API keys, DB credentials, JWT secrets |
| **Networking** | VPC | 3-tier isolation (public/private/isolated) |
| **Networking** | RDS Proxy | Connection pooling (1000 Lambda → 30 DB connections) |
| **Messaging** | EventBridge | Event-driven workflows, scheduled tasks |
| **Messaging** | SQS + DLQ | Buffered processing, guaranteed delivery |
| **Monitoring** | CloudWatch | Metrics, alarms, dashboards, structured logs |
| **Monitoring** | X-Ray | Distributed tracing across Lambda/API/Bedrock |
| **Monitoring** | Synthetics | Canary tests every 5 minutes |
| **DNS** | Route 53 | Domain management, health checks |
| **Email** | SES | Transactional emails |
| **IaC** | CDK (TypeScript) | 14 stacks, ~2,400 lines of infrastructure code |

---

## Cost Analysis

### Monthly Infrastructure Cost: ~$72

| Service | Monthly Cost | Why This Low |
|---------|-------------|--------------|
| Lambda (77 functions) | $2.87 | Pay-per-invocation, ARM64/Graviton2 |
| Aurora Serverless v2 | $43.80 | Scales to 0.5 ACU at idle |
| ElastiCache Redis | $12.41 | t3.micro, single-AZ (prod would be multi-AZ) |
| API Gateway HTTP v2 | $0.04 | $1/M requests (vs $3.50 for REST API) |
| S3 + CloudFront | $1.23 | Static assets, lifecycle policies |
| DynamoDB (on-demand) | $0.47 | Pay-per-request, no provisioned capacity |
| Secrets Manager | $2.40 | $0.40/secret/month |
| Other (CloudWatch, WAF) | $8.90 | Managed services, minimal at low traffic |

### Key Cost Decisions
- **HTTP API v2** over REST API: $1/M vs $3.50/M requests (71% savings)
- **Lambda ARM64** over x86: 20% cheaper, 15% faster cold starts
- **Aurora Serverless v2** over provisioned: ~$44/mo vs $200+/mo at low traffic
- **S3 Intelligent Tiering**: Automatic cost optimization for stored documents
- **On-demand DynamoDB**: $0 at zero traffic vs minimum $25/mo for provisioned

### Projected Cost at Scale
| MAU | Estimated Monthly Cost | Cost per User |
|-----|----------------------|---------------|
| 100 | $72 | $0.72 |
| 1,000 | $180 | $0.18 |
| 10,000 | $800 | $0.08 |
| 100,000 | $4,500 | $0.045 |

---

## Architecture Decisions

### [ADR-001: Serverless-First Compute Strategy](architecture/decisions/ADR-001-serverless-first.md)
Why Lambda over ECS/EC2 for API workloads. Cost analysis, cold start mitigation, and exception cases.

### [ADR-002: Polyglot Persistence](architecture/decisions/ADR-002-polyglot-persistence.md)
Why Aurora + DynamoDB together. Which data goes where and why.

### [ADR-003: Multi-Tenant Pool Model](architecture/decisions/ADR-003-multi-tenant-pool.md)
Row-Level Security vs separate schemas vs separate databases. Tradeoff analysis.

### [ADR-004: Bedrock RAG over Fine-Tuning](architecture/decisions/ADR-004-rag-over-finetuning.md)
Why Retrieval-Augmented Generation for regulatory compliance instead of model fine-tuning.

### [ADR-005: Market-Aware Calculation Engine](architecture/decisions/ADR-005-market-aware-calculations.md)
How the Chrome Extension handles 11 markets with different mortgage terms, tax rules, and benchmarks.

---

## Code Patterns

> These are generic architectural patterns extracted from Terrava. They demonstrate design thinking without exposing proprietary business logic.

- [Circuit Breaker Pattern](examples/circuit-breaker.ts) — Resilient external API calls with fallback
- [Multi-Tenant Middleware](examples/multi-tenant-middleware.ts) — Tenant extraction and RLS setup
- [Bedrock RAG Query](examples/bedrock-rag-query.ts) — Knowledge Base retrieval and generation
- [Structured Logging](examples/structured-logging.ts) — CloudWatch-optimized JSON logging
- [CDK VPC Pattern](examples/cdk-vpc-pattern.ts) — 3-tier VPC with isolated data subnets
- [Rate Limiting with Redis](examples/rate-limiter.ts) — Token bucket algorithm for per-tenant limits
- [Health Check Endpoint](examples/health-check.ts) — Dependency-aware health reporting

---

## Well-Architected Framework Alignment

| Pillar | Score | Key Implementations |
|--------|-------|-------------------|
| **Security** | 8/10 | KMS, WAF, least-privilege IAM, Secrets Manager, MFA, RLS |
| **Reliability** | 7/10 | Multi-AZ, DLQ, circuit breakers, automated backups |
| **Performance** | 8/10 | Redis caching, CloudFront, ARM64 Lambda, streaming AI |
| **Cost Optimization** | 9/10 | Serverless-first, Aurora Serverless, lifecycle policies |
| **Operational Excellence** | 7/10 | IaC (CDK), CI/CD, structured logging, synthetic canaries |

---

## Tech Stack

**Frontend**: React 18, TypeScript, Vite, Tailwind CSS, Framer Motion, Recharts
**Backend**: Node.js, AWS Lambda (ARM64), API Gateway HTTP v2
**Database**: Aurora PostgreSQL Serverless v2, DynamoDB, ElastiCache Redis
**AI/ML**: Amazon Bedrock (Claude 3 Sonnet/Haiku), Knowledge Bases, Titan Embeddings
**Infrastructure**: AWS CDK (TypeScript), 14 stacks, ~2,400 lines
**Extension**: Chrome Manifest V3, vanilla JS, 40+ supported listing sites
**Security**: Cognito, WAFv2, KMS, Secrets Manager, CloudTrail, GuardDuty
**Observability**: CloudWatch, X-Ray, Synthetics, structured JSON logging

---

## About

Built by [Jasha Balcom](https://linkedin.com/in/jashabalcom) — a production-grade enterprise SaaS platform serving real estate investors across 11 global markets.

Currently scaling toward general availability. Architecture designed for multi-tenant enterprise workloads from day one.

---

*Architecture documentation for Terrava. Production codebase is private.*
