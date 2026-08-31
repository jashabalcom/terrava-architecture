# ADR-0015: GitHub Actions OIDC federation, zero static AWS credentials

**Status:** Accepted
**Date:** 2025
**Scope:** IamStack, CI/CD

## Context

CI/CD needs permission to deploy 16 CDK stacks — which means broad, powerful AWS access:
creating IAM roles, modifying VPCs, updating databases. This is the most privileged identity
in the system.

The conventional approach is an IAM user with an access key stored in CI secrets. That key is
long-lived, and long-lived credentials leak. They leak into logs, into forked pull requests,
into a compromised CI dependency, into a screenshot. Leaked CI credentials are one of the most
common paths to a full cloud-account compromise, and the credential often stays valid for
months after the leak because nothing forces rotation.

## Options considered

### Option A — IAM user with a stored access key

Universal, trivially simple. Rejected: a permanent, highly privileged credential sitting in a
third-party system, with rotation depending on someone remembering. The blast radius of a leak
is the entire AWS account, and detection is likely to come from the bill rather than from a
control.

### Option B — Stored key with scheduled rotation

Better, but it reduces the exposure *window* without removing the exposure. It also adds a
rotation mechanism that itself needs credentials, and a rotation job that fails silently
leaves a stale key that everyone assumes was rotated.

### Option C — OIDC federation

GitHub Actions issues a short-lived OIDC token per workflow run. AWS trusts GitHub as an
identity provider and exchanges that token for temporary credentials scoped to an IAM role.
The trust policy constrains which repository, which branch, and which environment can assume
the role.

**No secret exists to leak.** The credential is minted per run and expires with it.

## Decision

**GitHub Actions OIDC federation with an IAM OIDC provider and repository/branch-scoped trust
policies. Zero static AWS credentials anywhere.**

The trust policy pins the specific repository and ref, so a fork or an unauthorized branch
cannot assume the deploy role even with a valid GitHub token.

## Consequences

**What got better**

- There is no long-lived AWS credential to steal, from CI or from anywhere else.
- Credentials are short-lived and automatically scoped per workflow run.
- Nothing to rotate, so nothing to forget to rotate.
- CloudTrail attributes every deploy action to a federated identity carrying the repo and ref,
  so "who deployed this" is answerable from the audit log.
- Trust-policy scoping means an attacker with repository write access still can't deploy from
  an arbitrary branch.

**What got worse**

- **The trust policy is now the security boundary**, and it is easy to get subtly wrong. A
  wildcard in the subject condition — matching any branch, or worse any repository — silently
  converts this from a strong control into a weaker one than a stored key. It looks correct
  either way.
- **Harder to run locally.** Reproducing a CI deploy on a workstation needs a separate
  credential path, so "works in CI, fails locally" becomes a category of problem.
- **A hard dependency on GitHub as an identity provider.** Moving CI platforms means
  re-establishing federation.
- **Debugging federation failures is unpleasant** — the errors are opaque, and the failure
  usually means no deploy at all rather than a degraded one.
- The deploy role remains highly privileged. OIDC fixes *credential leakage*, not
  *over-permission*; a compromised workflow definition still has full deploy rights.

## Revisit when

- The deploy role's permissions can be narrowed — the natural next step is separate roles per
  stack group, so a compromised workflow can't touch IAM and the database in one run.
- Deploys need to originate from somewhere other than GitHub Actions.
- Multi-account separation (dev/staging/prod in separate AWS accounts) is adopted, which would
  make cross-account role assumption the stronger pattern and shrink the blast radius further.
