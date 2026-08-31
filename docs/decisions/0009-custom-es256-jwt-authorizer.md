# ADR-0009: Custom ES256 JWT authorizer alongside Cognito

**Status:** Accepted
**Date:** 2025
**Scope:** ApiStack, CognitoStack

## Context

Every authenticated request is authorized before it reaches a handler, so authorizer latency
is added to *every* API call in the platform. Multiply a few tens of milliseconds by every
request and it becomes one of the largest single contributors to baseline API latency.

The platform also needs authorization that understands its own domain: subscription tier,
which product surface (investor app vs Agent CRM), and feature entitlements. That's not
generic authentication, it's business logic.

## Options considered

### Option A — Cognito's built-in JWT authorizer

Zero code, AWS-managed, correct by default. Kept for the paths where it fits. Rejected as the
only mechanism because it validates the token and stops there — tier and entitlement checks
would then happen in every one of 77 handlers, duplicating authorization logic 77 times. That
is how authorization bugs get shipped.

### Option B — Auth0, Clerk, or a similar third-party provider

Excellent developer experience, more features than Cognito. Rejected: a per-MAU cost that
grows with success, a third party on the critical path of every request, and another vendor
in the compliance story. Cognito already covers user pool management and MFA.

### Option C — Custom Lambda authorizer using RS256

A custom authorizer gives control over tier and entitlement logic. RS256 is the common default
for JWT signing.

### Option D — Custom Lambda authorizer using ES256 with `webcrypto.subtle`

Same control, but ECDSA P-256 instead of RSA. Two concrete advantages: ES256 verification is
meaningfully faster than RS256 at equivalent security, and ES256 tokens are smaller (a P-256
signature is 64 bytes against 256 for RSA-2048), which cuts bytes on every request.

Using the Node 20 runtime's native `webcrypto.subtle` rather than a JWT library means no
third-party dependency in the authorizer — the most security-sensitive code in the platform
has the smallest possible supply-chain surface.

## Decision

**Custom Lambda authorizer using ES256 verified via `webcrypto.subtle`, with Cognito User Pool
retained for user management, MFA, and the account lifecycle.**

Cognito owns identity. The custom authorizer owns authorization — signature verification plus
tier and entitlement resolution in a single pass, with the result cached by API Gateway.

Roughly 40 ms saved per request, with no third-party dependency on the auth path.

## Consequences

**What got better**

- ~40 ms off every authenticated request. On an interactive product, that's felt.
- Tier and entitlement checks resolve once at the edge instead of being reimplemented in 77
  handlers. Handlers receive an authorizer context they can trust.
- No third-party JWT library on the most security-critical path — nothing to audit, nothing to
  CVE-patch under time pressure.
- Smaller tokens mean fewer bytes per request, which matters on mobile and on the extension.
- API Gateway authorizer result caching means repeated calls from the same session skip the
  Lambda invocation entirely.

**What got worse**

- **This is now security code that is owned, not rented.** Signature verification, expiry
  handling, clock skew, algorithm confusion (`alg: none`, RS256/ES256 substitution), key
  rotation — all of it is the platform's responsibility. A subtle bug here is a full auth
  bypass. This is the single highest-consequence piece of custom code in the system and it
  needs the test coverage to match.
- **Key rotation is a designed process now**, not a managed feature. Rotating the signing key
  means supporting overlapping valid keys through a transition window.
- **Two auth concepts to reason about** (Cognito's tokens and the platform's), and the
  boundary between them has to stay clear or it becomes a source of confusion.
- **Authorizer caching cuts both ways.** A tier downgrade or revocation isn't effective until
  the cache TTL expires, so the cache window is a deliberate security/performance trade that
  has to be set consciously.
- Cold starts on the authorizer add latency to the first request of a burst — on the path
  where latency was the entire point.

## Revisit when

- Cognito's authorizer gains rich enough context injection to cover tier and entitlement
  resolution natively.
- An audit or a near-miss in the custom verification path suggests the risk of owning this code
  exceeds 40 ms of value.
- Federated or enterprise SSO (SAML/OIDC B2B) becomes a requirement, at which point a managed
  provider's feature set may justify its cost.
