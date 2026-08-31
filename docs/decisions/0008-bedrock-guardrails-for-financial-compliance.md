# ADR-0008: Bedrock Guardrails as a hard compliance gate

**Status:** Accepted
**Date:** 2025
**Scope:** All AI paths

## Context

Terrava operates adjacent to regulated financial advice. The author holds Series 7/66 and has
worked inside a broker-dealer, so the regulatory line here is not theoretical: language
implying **guaranteed returns** on an investment is a specific, well-known compliance problem.
An LLM writing investment commentary will produce that language unprompted, because it appears
constantly in the marketing copy it was trained on.

There are three separate risks on every AI path:

1. **Compliance language** — "guaranteed," "risk-free," "you will earn" in an investment
   context.
2. **PII leakage** — users paste passport numbers, Emirates IDs, and bank details into chat
   without thinking, and that flows to a model provider and into logs.
3. **Prompt injection** — the Chrome extension sends *third-party page content* to the model.
   Any listing site could embed instructions in page text. This is the highest-risk surface in
   the platform, because the input is genuinely untrusted.

Without a control for all three, the platform cannot responsibly operate in this domain.

## Options considered

### Option A — Prompt instructions only ("never say guaranteed")

Free, immediate. Rejected as a sole control: system-prompt instructions are guidance, not
enforcement. They degrade over long conversations, and prompt injection specifically targets
them. A control that the attack is designed to bypass is not a control.

### Option B — Application-side regex and deny-lists

Deterministic and auditable. Rejected as a sole control — it catches "guaranteed returns" and
misses "you're certain to see 8%." Natural language has unbounded paraphrase. Still useful as
one layer, but not the layer.

### Option C — A second LLM as a judge on every output

Flexible and semantically aware. Rejected as the primary gate: it doubles inference cost per
request (directly against ADR-0005), adds latency, and the judge is itself a model that can be
talked out of its job.

### Option D — Bedrock Guardrails, applied as a mandatory gate

A managed policy layer enforced by the platform rather than requested in a prompt. Covers
denied topics, PII detection and anonymization, word filters, content filters, and
prompt-attack detection. Applied on input *and* output, independent of the model. Guardrail
violations are logged as events.

## Decision

**Bedrock Guardrails on every AI path, applied to both input and output, as a hard gate.**

Configured to: block guaranteed-return and similar compliance language; detect and anonymize
PII; refuse off-topic prompts; and harden against prompt injection — which specifically covers
the extension's third-party page content.

Guardrail and prompt configuration is versioned in DynamoDB (ADR-0016) so policy can roll
forward without a code deploy. A compliance fix should be a config change measured in minutes,
not a deploy cycle.

Layered with Option B's deterministic checks: Guardrails is the semantic layer, deny-lists
catch the known-exact strings, and neither is trusted alone.

## Consequences

**What got better**

- Compliance enforcement is structural, not aspirational. It holds regardless of what the
  prompt says or how the conversation drifts.
- PII is anonymized before it reaches the model provider and before it lands in logs, which
  shrinks the data-handling surface considerably.
- The extension's untrusted-input path has a real defense rather than a hopeful prompt.
- Blocked requests are logged as policy events — an audit trail that shows the control is
  working, which is what a compliance reviewer actually asks for.
- Policy changes ship as config, so a newly identified phrase is blocked platform-wide in
  minutes.

**What got worse**

- **False positives.** Legitimate analysis gets blocked — discussing a property's historical
  yield can trip return-related filters. Every false positive is a user seeing a refusal
  instead of the product working, and tuning thresholds is a permanent balancing act between
  compliance safety and usability.
- **Latency and cost on every call**, including the ones that were never going to be a problem.
- **Not a complete defense.** Guardrails reduces prompt-injection risk; it doesn't eliminate
  it. The extension path still needs least-privilege design so a successful injection can't
  reach anything valuable.
- **A single vendor control.** Adding Gemini Flash as a fallback (ADR-0005) means that path
  needs equivalent protection applied separately — a fallback that skips the gate would be
  a compliance hole hidden inside a resilience feature.
- Aggressive PII anonymization can strip context the model needed to answer well.

## Revisit when

- False-positive rate in sampled review reaches the point where users route around the AI
  features.
- A prompt-injection attempt succeeds against the extension path in testing — that's a signal
  the layer needs supplementing, not tuning.
- A regulatory change requires controls Guardrails can't express, which would mean adding a
  dedicated policy service.
