# ADR-0001: AWS CDK in TypeScript for all infrastructure

**Status:** Accepted
**Date:** 2025
**Scope:** All 16 stacks, every environment

## Context

Terrava is a solo-built platform that has to run in three environments (dev, staging, prod)
with the same topology in each. A regulated financial-services product also has to be able to
answer "who changed this, when, and why" about infrastructure, not just application code.

Console-clicked infrastructure fails both tests. It cannot be reviewed, it cannot be recreated
after an incident, and the only record of a change is CloudTrail — which tells you what
happened but not what was intended.

The application is already TypeScript end to end (React frontend, Node.js 20 Lambdas). Any
infrastructure tool that introduces a second language introduces a second set of tooling,
review habits, and mistakes.

## Options considered

### Option A — Console + documentation

Fastest to start. Rejected: environments drift within weeks, and the documentation is stale
the moment someone makes a hotfix at 2am. Not defensible in a compliance review.

### Option B — Terraform

Mature, huge provider ecosystem, strong state management, the industry default. Rejected on
two counts. First, HCL is a second language with its own idioms and no compile-time
relationship to the application's TypeScript types. Second, the higher-level constructs that
make AWS-native work fast — sane VPC defaults, Lambda bundling, IAM grants that infer the
right policy from usage — have to be hand-assembled or pulled from third-party modules.

The honest counterpoint: Terraform is the better answer for a multi-cloud future or a larger
team with existing Terraform fluency. Neither applied here.

### Option C — AWS SAM / Serverless Framework

Excellent for a Lambda-and-API-Gateway app. Rejected because Terrava isn't only that — it has
VPC, Aurora, OpenSearch Serverless, Bedrock Knowledge Bases, WAF, GuardDuty. SAM would cover
maybe half the footprint and the other half would need a second tool.

### Option D — AWS CDK in TypeScript

Same language as the rest of the stack. Constructs compose like ordinary code, so shared
patterns become shared functions instead of copy-paste. `grantRead()`-style APIs generate
least-privilege IAM from actual usage rather than from a human guessing at action names.
Synthesizes to CloudFormation, so drift detection and stack rollback come free.

## Decision

**AWS CDK in TypeScript, 16 stacks, zero console clicks.** The deciding factor was one
language across application and infrastructure, plus CDK's IAM grant model generating
least-privilege policies as a by-product of writing normal code.

## Consequences

**What got better**

- A new environment stands up from `cdk deploy` in under 30 minutes.
- Infrastructure changes go through the same PR review as application changes.
- IAM policies are narrow by construction rather than by discipline — the `grant*` APIs emit
  exactly the actions the code path needs.
- CloudFormation drift detection gives a real answer to "has anyone touched prod by hand?"
- Stack dependency ordering is resolved by the framework, not by a deploy runbook.

**What got worse**

- CloudFormation's limits are now the platform's limits: slow rollbacks, resources that can't
  be updated in place, and the occasional stack stuck in `UPDATE_ROLLBACK_FAILED` that needs
  manual intervention.
- CDK's abstractions can hide what's actually being provisioned. Reading `cdk diff` output
  carefully before every prod deploy is mandatory, not optional — a one-line construct change
  can silently replace a database.
- CDK version upgrades occasionally change synthesized output for unchanged source, producing
  diffs nobody asked for.
- Splitting into 16 stacks avoids the 500-resource CloudFormation limit but introduces
  cross-stack references, which make some resources awkward to delete or reorder later.

## Revisit when

- A second cloud provider enters the picture (then Terraform's abstraction wins).
- Stack count or cross-stack references make routine deploys fragile enough that a deploy
  takes more than one attempt on a regular basis.
- CloudFormation rollback times become the dominant term in incident recovery.
