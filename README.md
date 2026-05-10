# Terrava — Enterprise Wealth Intelligence Platform

> Production multi-tenant SaaS on AWS. Investment intelligence for global real-estate investors, plus an AI Agent CRM for the agents who serve them. 14 CDK stacks, 77 Lambda functions, multi-model Bedrock with RAG, live across 11 markets.

[![Live](https://img.shields.io/badge/Live-dubairealestateinvestor.com-00C853?style=for-the-badge)](https://dubairealestateinvestor.com)
[![AWS](https://img.shields.io/badge/AWS-14_CDK_Stacks-FF9900?style=for-the-badge&logo=amazonaws)](https://aws.amazon.com)
[![Lambda](https://img.shields.io/badge/Lambda-77_Functions_·_Graviton2-FF9900?style=for-the-badge&logo=awslambda)](https://aws.amazon.com/lambda/)
[![Bedrock](https://img.shields.io/badge/Bedrock-Multi--Model_Router-8B5CF6?style=for-the-badge&logo=amazonaws)](https://aws.amazon.com/bedrock/)
[![Markets](https://img.shields.io/badge/Markets-11_Countries-blue?style=for-the-badge)](.)
[![Extension](https://img.shields.io/badge/Chrome_Extension-40%2B_Sites-4285F4?style=for-the-badge&logo=googlechrome)](.)

This repository holds the architecture documentation, screenshots, and design records for Terrava. The production codebase is private.

---

## At a glance

| Area | What's in production |
|---|---|
| Infrastructure-as-code | 14 CDK stacks (TypeScript), reproducible across dev / staging / prod with automated drift detection |
| Compute | 77 Lambda functions on ARM64 / Graviton2 with shared Lambda Layers |
| Data | Aurora PostgreSQL Serverless v2 with RDS Proxy multiplexing 1,000 Lambda connections down to 20–50 DB connections, plus ElastiCache Redis Serverless |
| AI | Multi-model Bedrock router (Sonnet / Haiku / Gemini) with circuit-breaker fallback, RAG over Knowledge Bases + OpenSearch Serverless, Bedrock Guardrails for financial-compliance filtering |
| Security | Three-tier VPC across three Availability Zones (data tier has no internet route), WAFv2 on edge and API Gateway, KMS customer-managed keys, Secrets Manager, CloudTrail, GuardDuty, custom ES256 JWT authorizer |
| Observability | CloudWatch Synthetics canaries every five minutes, X-Ray tracing, custom token-economics dashboards, Budgets anomaly alerts |
| Async | EventBridge Scheduler, SQS with dead-letter queues for guaranteed delivery |
| Commercials | Stripe across eight subscription tiers, webhook as source of truth, daily usage metering, tier-gated feature flags |

---

## Outcomes from specific decisions

| Lever | Mechanism | Outcome |
|---|---|---|
| AI infrastructure cost | Multi-model router: Sonnet for deep analysis, Haiku for fast Q&A, Gemini Flash as fallback | $400+/mo on a single-model setup dropped to roughly $70–130/mo (about 60% lower) at current scale |
| Bedrock call reduction | DynamoDB response cache with 24-hour TTL, keyed on hashed prompt + context | 40–60% cache hit rate on production traffic |
| API cost | API Gateway HTTP v2 instead of REST | 71% cheaper at request volume ($1/M vs $3.50/M) |
| Compute cost | Lambda on ARM64 / Graviton2 across the fleet | About 20% compute reduction with no performance regression |
| Database cost | Aurora Serverless v2 (0.5–16 ACU auto-scale) plus RDS Proxy connection multiplexing | About 60% DB cost reduction vs always-on provisioned |
| Database load | ElastiCache Redis Serverless for response cache plus sliding-window rate limiting | DB load reduced about 73% during peak hours |
| Email cost | Moved to Amazon SES v2 with DKIM/SPF/DMARC | About 95% email-infra cost reduction; 99.2% deliverability |
| Auth latency | Custom ES256 JWT authorizer using `webcrypto.subtle` | About 40 ms shaved per request; no third-party dependency |
| Time-to-deploy | 14 CDK stacks; full IaC; GitHub Actions OIDC | New environment from `cdk deploy` in under 30 minutes; zero static AWS credentials |

---

## The Product

| Homepage | Properties | Property Detail |
|:---:|:---:|:---:|
| ![Homepage](docs/screenshots/app/01-homepage-hero.png) | ![Properties](docs/screenshots/app/02-properties-listings.png) | ![Property Detail](docs/screenshots/app/03-property-detail.png) |

| Investment Calculator | Pricing & Billing |
|:---:|:---:|
| ![Calculator](docs/screenshots/app/04-investment-calculator.png) | ![Pricing](docs/screenshots/app/05-pricing-stripe-tiers.png) |

| Community Hub | Neighborhoods Guide | Golden Visa Calculator |
|:---:|:---:|:---:|
| ![Community](docs/screenshots/app/07-community-hub.png) | ![Neighborhoods](docs/screenshots/app/08-neighborhoods-guide.png) | ![Golden Visa](docs/screenshots/app/09-golden-visa-calculator.png) |

**1,122 live property listings** · **11 global markets** · **13 investment calculators** · **AI wealth advisor** · **Chrome Extension on 40+ listing sites** · **AI Agent CRM** · **8 subscription tiers**

---

## Engineering notes

A few of the harder problems and how they were solved.

**Token economics as a product constraint.** Every AI call costs real money against a thin SaaS margin. The platform tracks `model`, `input_tokens`, `output_tokens`, `latency`, `cache_hit`, and `user_tier` per request to a CloudWatch custom-metrics namespace and a DynamoDB ledger. The AI router uses that signal to pick the cheapest model that meets the task SLO. Cost decisions happen at runtime, not in a quarterly review.

**Two products on one platform.** The investor app and the Agent CRM share auth, billing, observability, secrets, and roughly 65% of the API surface, but they diverge on data model (portfolios vs pipeline), UX (analysis vs messaging), and AI persona (advisor vs voice profile). Solved with shared Lambda Layers, per-product Lambda groups, and tier-gated features in a single Stripe codebase across eight tiers.

**Compliance-grade AI in a regulated domain.** Bedrock Guardrails block "guaranteed returns" language, anonymize PII, refuse off-topic prompts, and harden against prompt injection. Without these, the platform cannot operate in financial services. Guardrail and prompt configs are versioned in DynamoDB so policy can roll forward without code deploys.

**A Chrome Extension that runs anywhere with one config entry per market.** 40+ listing sites across 11 countries with different currencies, mortgage terms, expense ratios, and vacancy rates — solved with a single `market-defaults.js` config map and a content-script price-extraction layer that targets DOM patterns rather than specific sites. Adding a new country is one config entry.

---

## Investment Calculator Suite — 13 Specialized Tools

13 calculators with premium UX, AI-powered deal scoring, and real fee accuracy across multiple markets.

| ROI Calculator — Deal Score & KPI Cards | Property Analysis Chain |
|:---:|:---:|
| ![Calculator V2](docs/screenshots/app/21-calculator-advanced-fees.png) | ![Property Chain](docs/screenshots/app/23-property-chain-flow.png) |
| *Deal Score engine rates every property 1-10. KPI cards show ROI, net yield, breakeven at a glance.* | *Property → ROI → Mortgage → Total Cost analysis chain. Click "Analyze" on any listing and data pre-fills across all calculators.* |

**Key features:** Deal Score Engine (AI 1–10) · Cash/Mortgage toggle (defaults adapt per market) · Advanced fee inputs (97%+ accuracy) · Dual currency display · Area presets · Property Chain pre-fill across calculators.

**13 calculators:** ROI · Mortgage · Total Cost · Airbnb Yield · Rent vs Buy · STR vs LTR · Off-Plan · Cap Rate · DSCR · Commercial Lease · Free Zone · Scenario Comparison · Golden Visa.

---

## Community Platform

| Community Feed — Trending Ticker & Deal Cards | Saved Strategies Library |
|:---:|:---:|
| ![Community V2](docs/screenshots/app/24-community-v2-feed.png) | ![Saved Strategies](docs/screenshots/app/29-saved-strategies.png) |

Trending ticker · structured deal cards · follow + quote repost · area tags with sentiment · who-to-follow · saved strategies (Pro+).

---

## Chrome Extension — 40+ Property Sites Worldwide

Works on listing sites across 11 markets — instant on-page AI investment analysis.

**Supported sites include:** Zillow, Redfin, Realtor.com, Trulia (US) · Bayut, PropertyFinder, Dubizzle (UAE) · Rightmove, Zoopla, OnTheMarket (UK) · Domain, RealEstate.com.au (AU) · PropertyGuru, 99.co (SG) · Idealista (ES/PT/IT) · plus Canada, Turkey, Japan, Thailand, Germany, Netherlands, France.

**Tiers:** Free (3 preview analyses) · Investor (3/day) · Pro (unlimited) · Portfolio (unlimited + team sharing).

---

## AI Agent CRM

The AI learns each agent's communication style and handles 80% of client messaging automatically.

| Agent Dashboard | Agent Messages — 3-Panel CRM |
|:---:|:---:|
| ![Agent Dashboard](docs/screenshots/app/30-agent-dashboard.png) | ![Agent Messages](docs/screenshots/app/31-agent-messages-crm.png) |

| Voice Profile — AI Writes in Your Voice | Agent Pipeline — Deal Stages |
|:---:|:---:|
| ![Voice Profile](docs/screenshots/app/33-agent-voice-profile.png) | ![Pipeline](docs/screenshots/app/35-agent-pipeline.png) |

**Features:** AI Voice Profile (4-step wizard) · AI Draft Replies (voice + sales psychology) · AI Auto-Respond (offline mode) · Contact Labels with cadence-aware AI follow-ups · Smart Notifications (behavioral triggers) · Pipeline Management with conversion tracking · Future-ready unified inbox for WhatsApp + SMS.

---

## Architecture Overview

Every component deployed via AWS CDK (TypeScript). Zero console clicks. Full infrastructure-as-code.

```mermaid
graph TB
    subgraph "Edge Layer"
        R53[Route 53] --> CF[CloudFront CDN]
        CF --> WAF[WAFv2]
        CF --> S3[S3 - React SPA]
    end

    subgraph "API Layer"
        CF --> APIGW[API Gateway v2<br/>HTTP API - 77 routes]
        APIGW --> AUTH[Lambda JWT Authorizer<br/>ES256 webcrypto.subtle]
        AUTH --> FN[77 Lambda Functions<br/>ARM64 / Graviton2 + Layers]
    end

    subgraph "Data Layer - Isolated Subnets (no internet)"
        FN --> PROXY[RDS Proxy<br/>1000 → 20-50 conns]
        PROXY --> AURORA[Aurora PostgreSQL<br/>Serverless v2]
        AURORA --> READER[Read Replica]
        FN --> REDIS[ElastiCache Redis<br/>Serverless]
    end

    subgraph "AI Layer — Multi-Model Router"
        FN --> ROUTER[AI Service Router<br/>Model Selection + Cache]
        ROUTER --> SONNET[Claude Sonnet<br/>Deep Analysis]
        ROUTER --> HAIKU[Claude Haiku<br/>Fast Queries]
        ROUTER --> GEMINI[Gemini Flash<br/>Fallback]
        HAIKU --> KB[Knowledge Base<br/>RAG Pipeline]
        KB --> OS[OpenSearch Serverless<br/>Vector Store]
        ROUTER --> GUARD[Bedrock Guardrails<br/>Financial Compliance]
        ROUTER --> PROMPT[DynamoDB<br/>Prompt Registry]
        ROUTER --> TRACK[DynamoDB<br/>Token Tracking]
    end

    subgraph "Async & Messaging"
        EB[EventBridge Scheduler] --> FN
        FN --> SES[SES v2 Email<br/>DKIM/SPF/DMARC]
        FN --> DDB[DynamoDB]
        EB --> DLQ[SQS Dead Letter Queues]
    end

    subgraph "Observability"
        SYNTH[Synthetics Canaries<br/>API + Frontend / 5 min] --> CW[CloudWatch Alarms]
        FN --> XRAY[X-Ray Distributed Tracing]
        CW --> SNS[SNS Notifications]
        BUDGET[AWS Budgets<br/>Anomaly Detection] --> SNS
    end

    subgraph "Security & Identity"
        KMS[KMS CMK] --> AURORA
        SM[Secrets Manager] --> FN
        CT[CloudTrail] --> GD[GuardDuty]
        COG[Cognito User Pool<br/>MFA + JWT]
    end

    subgraph "CI/CD"
        GH[GitHub Actions<br/>OIDC Federation] -.->|"zero static creds"| FN
    end
```

### VPC — 3-Tier Network Isolation

```
+------------------------------------------------------------------+
|  VPC (10.0.0.0/16) — 3 Availability Zones                         |
|                                                                    |
|  Public Subnets     — NAT Gateways, load balancers                 |
|  Private Subnets    — Lambda functions (internet via NAT only)     |
|  Isolated Subnets   — Aurora + Redis (NO internet route at all)    |
+------------------------------------------------------------------+
```

![VPC Subnets](docs/screenshots/aws/12-vpc-subnets-3tier.png)

---

## Decision tree — how the AI Router picks a model

```mermaid
flowchart TD
    REQ[User AI request<br/>+tier +task +context] --> CACHE{In response<br/>cache?}
    CACHE -- yes --> HIT[Return cached<br/>40-60% hit rate<br/>$0 cost]
    CACHE -- no --> TIER{User tier?}

    TIER -- "Pro / Portfolio<br/>+ deep analysis task" --> SONNET[Claude Sonnet 4.5<br/>$3 / M tokens<br/>multi-step reasoning]
    TIER -- "Investor / Free<br/>+ Q&A or summarize" --> HAIKU[Claude Haiku 4.5<br/>$0.80 / M tokens<br/>fast structured]
    TIER -- "Agent CRM<br/>draft reply" --> HAIKU2[Haiku + Voice Profile<br/>+ sales-psych cadence]
    TIER -- "Regulatory Q&A" --> RAG[Haiku + KB Retrieve<br/>OpenSearch Serverless<br/>cite sources]

    SONNET --> GUARD[Bedrock Guardrails<br/>PII / topic / prompt-inject]
    HAIKU --> GUARD
    HAIKU2 --> GUARD
    RAG --> GUARD

    GUARD -- pass --> RESP[Return response]
    GUARD -- block --> POLICY[Policy refusal<br/>logged]

    RESP --> WRITE[Write cache<br/>+ token ledger<br/>+ CW custom metric]

    SONNET -.fail.-> CB{Circuit<br/>breaker open?}
    CB -- no --> HAIKU
    CB -- yes --> GEMINI[Gemini Flash fallback<br/>$0.075 / M tokens]

    classDef expensive fill:#FFB000,stroke:#D97706,color:#000
    classDef cheap fill:#10B981,stroke:#047857,color:#fff
    classDef gate fill:#8B5CF6,stroke:#6D28D9,color:#fff

    class SONNET expensive
    class HAIKU,HAIKU2,GEMINI,HIT cheap
    class GUARD,CB gate
```

This tree is the reason AI cost dropped about 60%.

---

## AWS Console — Deployed Infrastructure

### All CDK Stacks
![CloudFormation Stacks](docs/screenshots/cloudformation-all-stacks.png)
*Every stack in `CREATE_COMPLETE`*

### Database — Aurora Serverless v2 + RDS Proxy
| Aurora Cluster (Writer + Reader) | RDS Proxy Configuration |
|:---:|:---:|
| ![Aurora Cluster](docs/screenshots/aws/02-aurora-cluster-writer-reader.png) | ![RDS Proxy](docs/screenshots/aws/03-rds-proxy-config.png) |

- Aurora PostgreSQL Serverless v2 auto-scales **0.5–16 ACU**
- RDS Proxy multiplexes **1,000 Lambda connections → 20–50 real DB connections**
- Read replica in separate AZ for failover

### Caching — ElastiCache Redis Serverless
![ElastiCache](docs/screenshots/aws/04-elasticache-serverless.png)

### AI — Multi-Model Bedrock with RAG
| AI Investment Advisor (Bedrock) | Bedrock Knowledge Base (RAG) |
|:---:|:---:|
| ![AI Response](docs/screenshots/ai-assistant-bedrock-response.png) | ![Bedrock KB](docs/screenshots/bedrock-knowledge-base.png) |

| Bedrock Guardrails | CloudWatch AI Metrics |
|:---:|:---:|
| ![Guardrails](docs/screenshots/bedrock-guardrails.png) | ![AI Metrics](docs/screenshots/cloudwatch-ai-metrics.png) |

| DynamoDB Prompt Registry | Token Usage Tracking |
|:---:|:---:|
| ![Prompts](docs/screenshots/dynamodb-prompt-registry-items.png) | ![Tokens](docs/screenshots/dynamodb-token-tracking.png) |

| DynamoDB AI Tables | OpenSearch Vector Store |
|:---:|:---:|
| ![Tables](docs/screenshots/dynamodb-ai-tables.png) | ![OpenSearch](docs/screenshots/aws/07-opensearch-collection.png) |

**RAG pipeline**: User asks regulatory question → Titan Embeddings v2 vectorizes the query → OpenSearch Serverless finds the 5 most relevant chunks → Claude generates a grounded answer **with citations from the source documents**.

### Compute — 77 Lambda Functions on ARM64
![Lambda List](docs/screenshots/aws/08-lambda-functions-77.png)
*All ARM64/Graviton2, Node.js 20*

### API — Gateway HTTP v2
![API Gateway](docs/screenshots/aws/10-api-gateway-routes.png)
- HTTP API v2 — **71% cheaper** than REST API
- JWT authorizer validates tokens at the edge
- Built-in throttling + per-route configuration

### Monitoring — Synthetics Canaries
![Synthetics Canaries](docs/screenshots/aws/11-synthetics-canaries.png)
*Canaries running every 5 minutes — API health + frontend heartbeat*

### Security — IAM + Secrets Manager + WAF
| IAM Roles (Least Privilege) | Secrets Manager |
|:---:|:---:|
| ![IAM Roles](docs/screenshots/aws/13-iam-roles-least-privilege.png) | ![Secrets Manager](docs/screenshots/aws/14-secrets-manager.png) |

| WAF Rules | IAM Lambda Permissions |
|:---:|:---:|
| ![WAF Rules](docs/screenshots/aws/15-waf-rules.png) | ![IAM](docs/screenshots/iam-lambda-role-permissions.png) |

### Auth — Cognito User Pool
![Cognito](docs/screenshots/aws/16-cognito-user-pool.png)

### Async — EventBridge + SQS Dead Letter Queues
![EventBridge Schedules](docs/screenshots/aws/17-eventbridge-schedules.png)

### Cost — Real AWS Spend
![Cost Explorer](docs/screenshots/aws/18-cost-explorer-before.png)

---

## 14 CDK Stacks (+2 supporting)

Every stack is TypeScript CDK. Dependency order managed automatically.

| # | Stack | AWS Services | Purpose |
|---|-------|-------------|---------|
| 1 | **IamStack** | IAM, OIDC Provider | CI/CD with zero stored credentials |
| 2 | **SecurityStack** | KMS, CloudTrail, GuardDuty | Encryption keys, audit trail, threat detection |
| 3 | **DomainStack** | Route 53, ACM | Custom domain + TLS certificates |
| 4 | **EmailStack** | SES v2, DKIM | Transactional email with domain verification |
| 5 | **FrontendStack** | S3, CloudFront, WAFv2 | React SPA hosting with CDN + DDoS protection |
| 6 | **VpcStack** | VPC, Subnets, NAT, Endpoints | 3-tier network (public/private/isolated) × 3 AZs |
| 7 | **DatabaseStack** | Aurora Serverless v2, RDS Proxy | PostgreSQL with auto-scaling + connection pooling |
| 8 | **CacheStack** | ElastiCache Redis Serverless | API response caching + rate limiting |
| 9 | **ApiStack** | API Gateway v2, 77 Lambdas | HTTP API + ARM64 functions + JWT authorizer |
| 10 | **MessagingStack** | EventBridge, SQS DLQs | Async job scheduling with failure capture |
| 11 | **CognitoStack** | Cognito User Pool | Multi-tenant auth with MFA |
| 12 | **ObservabilityStack** | CloudWatch, Budgets, SNS | Alarms, dashboards, cost anomaly detection |
| 13 | **BedrockKBStack** | Bedrock KB, OpenSearch Serverless | RAG pipeline over regulatory docs |
| 14 | **SyntheticsStack** | CloudWatch Synthetics | Canary uptime monitoring |
| 15 | **DynamoDBStack** | DynamoDB | High-throughput event and session storage |
| 16 | **AIInfraStack** | DynamoDB (×3), IAM, SSM | AI prompt registry, token tracking, response cache |

---

## 77 Lambda Functions

| Group | Count | What They Do |
|-------|-------|-------------|
| **AI & Analysis** | 7 | Property analysis, calculator insights, support agent, daily digest, regulatory KB query |
| **Properties** | 3 | Market data sync, scheduled sync, connection testing |
| **Email** | 8 | Welcome flow, drip campaigns, trial-ending, payment-failed, weekly digest |
| **Stripe Billing** | 6 | Checkout, webhooks, subscriptions, customer portal |
| **Admin** | 3 | Revenue stats, snapshot metrics, alert handler |
| **Neighborhoods** | 4 | Geocoding, batch-geocode, POI fetching |
| **News** | 3 | RSS sync, article processing |
| **Affiliate** | 2 | Commission tracking, referrals |
| **Data/Auth/Core** | 41 | CRUD, JWT authorizer, health checks, migrations |

---

## Chrome Extension Architecture

```
+--------------------------------------------------+
|  Chrome Extension (Manifest V3)                    |
|                                                    |
|  Content Script (injected into 40+ listing sites) |
|  - Detect listing page (URL pattern match)        |
|  - Extract price, size, location from DOM         |
|  - Calculate quick score (no API, instant, free)  |
|  - Inject overlay badge on listing                |
|                                                    |
|  Popup (detailed analysis view)                   |
|  - Send page text to API Gateway → Bedrock        |
|  - Display: yield, cap rate, mortgage, score      |
|  - Market-specific calculations per currency      |
|                                                    |
|  Shared Config (market-defaults.js)               |
|  - 11 markets with per-market:                    |
|    mortgage terms, rent ratios, expense ratios,   |
|    vacancy rates, price/sqft benchmarks           |
|  - Adding a new country = 1 config entry          |
+--------------------------------------------------+
```

---

## Security Architecture

| Layer | Implementation |
|-------|---------------|
| **Network** | 3-tier VPC across 3 AZs. Database + cache in **isolated subnets — no internet route**. |
| **Edge** | WAFv2 with rate limiting, SQL injection, XSS rules on CloudFront + API Gateway. |
| **Auth** | JWT authorizer + Cognito User Pool with MFA. Custom ES256 with `webcrypto.subtle`. |
| **Encryption** | KMS customer-managed key for data at rest. TLS 1.2+ in transit. ACM auto-renewed. |
| **Secrets** | All credentials in AWS Secrets Manager. **Zero hardcoded secrets in code.** |
| **Audit** | CloudTrail logs every API call. GuardDuty analyzes CloudTrail + VPC Flow Logs + DNS. |
| **Monitoring** | Synthetics canaries every 5 min. CloudWatch alarms → SNS → email. |
| **CI/CD** | GitHub Actions OIDC federation. **No static AWS credentials stored anywhere.** |
| **AI compliance** | Bedrock Guardrails: PII anonymization, topic blocking, prompt-injection defense. |

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| **Frontend** | React 18, TypeScript, Tailwind CSS, Vite, Framer Motion, Recharts, Mapbox GL, shadcn/ui |
| **API** | API Gateway HTTP v2, Lambda (Node.js 20, ARM64/Graviton2), Lambda Layers |
| **Database** | Aurora PostgreSQL Serverless v2, DynamoDB, ElastiCache Redis Serverless |
| **AI/ML** | Amazon Bedrock (Claude Sonnet 4.5 / Haiku 4.5), Knowledge Bases, Guardrails, Titan Embeddings v2, OpenSearch Serverless |
| **Extension** | Chrome Manifest V3, vanilla JS, 40+ supported sites, 11-market calculation engine |
| **Auth** | Cognito User Pool with MFA + custom ES256 JWT authorizer |
| **Payments** | Stripe (Checkout, Webhooks, Customer Portal, 8 tiers) |
| **Email** | Amazon SES v2 with DKIM/SPF/DMARC |
| **IaC** | AWS CDK (TypeScript, 14 stacks) |
| **CI/CD** | GitHub Actions with OIDC federation |
| **Monitoring** | CloudWatch Synthetics, X-Ray, Alarms, Budgets |
| **Security** | WAFv2, KMS, Secrets Manager, CloudTrail, GuardDuty |

---

## Related project — Swing Institute

A second production SaaS by the same author. [Swing Institute](https://github.com/jashabalcom/swing-institute-architecture) is an athlete-development platform shipping web, iOS App Store, and PWA from one TypeScript codebase, with an AWS Bedrock vision pipeline (Claude Sonnet 4) for biomechanical swing analysis and Stripe Connect Express coach payouts. Terrava is the AWS-native reference architecture (CDK, Lambda, Aurora). Swing Institute is the leaner ship-first architecture (Supabase + AWS edge) with a documented migration path to the same AWS-native pattern. Each one is the right answer for its team size and compliance posture.

---

## About

**Jasha Balcom** — Solutions Architect · AI Platform Engineer · Global Real Estate Advisor

[![LinkedIn](https://img.shields.io/badge/LinkedIn-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white)](https://linkedin.com/in/jashabalcom)
[![Email](https://img.shields.io/badge/Email-EA4335?style=for-the-badge&logo=gmail&logoColor=white)](mailto:jashabalcom@gmail.com)

**Sotheby's International Realty** (Global Real Estate Advisor) · **Merrill Lynch** (Series 7/66 Financial Services) · Former MLB Draft Pick (Chicago Cubs)

---

*Architecture documentation for Terrava. Production codebase is private.*

Copyright (c) 2025-2026 Jasha Balcom. All rights reserved.
