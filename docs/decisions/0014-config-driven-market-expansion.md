# ADR-0014: Config-driven market expansion in the Chrome extension

**Status:** Accepted
**Date:** 2025
**Scope:** Chrome Extension (Manifest V3)

## Context

The extension runs on 40+ property listing sites across 11 countries and injects investment
analysis directly onto the listing page. Two dimensions of variation make this hard:

**Sites differ.** Zillow, Bayut, Rightmove, Idealista and Domain share no markup conventions.
Each has its own DOM structure — and each redesigns without warning.

**Markets differ more.** Currency, typical mortgage terms and rates, transaction fees, expense
ratios, vacancy rates, price-per-square-metre benchmarks, and rental yield conventions all vary
by country. A UK buy-to-let calculation and a Dubai freehold calculation are different
calculations, not the same calculation in a different currency.

The naive implementation is 40 site adapters times 11 market rule sets. That's a combinatorial
maintenance problem that guarantees the extension is permanently broken somewhere.

## Options considered

### Option A — Per-site adapter modules

A dedicated module per site with hand-written selectors and market rules. Rejected: 40+ modules
to maintain, each breaking independently on every site redesign, and adding a country means
writing new modules rather than configuring existing behaviour. Maintenance cost grows with
the product's reach, which is backwards.

### Option B — Server-side scraping

Send the URL to the backend and fetch/parse there. Rejected on several counts: it needs the
page to be publicly fetchable (many listing pages are JS-rendered or gated), it's a
bot-detection fight the platform would keep losing, and it puts scraping infrastructure and
its legal exposure on the critical path. The user's browser is already authenticated and
already rendering the page.

### Option C — Separate DOM extraction from market economics

Two independent layers:

- **A content-script extraction layer** that targets *DOM patterns* — price-shaped strings,
  area-shaped strings, structured data (schema.org / JSON-LD), meta tags — rather than
  site-specific selectors. Property listings converge on similar structures because they're
  all describing the same thing to the same search engines.
- **A single `market-defaults.js` config map** holding per-market economics: mortgage terms,
  rent ratios, expense ratios, vacancy rates, price benchmarks, currency.

Site coverage and market coverage then scale independently.

## Decision

**Pattern-based DOM extraction plus a single per-market config map.**

Adding a new country is **one config entry**. Adding a new site is often zero work, because
pattern-based extraction generalizes — and where it doesn't, it's a pattern refinement that
benefits every site, not a new module.

The extension also computes a **quick score locally with no API call** — instant, free, and
works for unauthenticated users. Only the detailed analysis calls the backend, which keeps
inference cost tied to engaged users rather than to page views (ADR-0005).

## Consequences

**What got better**

- New market = one config entry. Expansion is a product decision, not an engineering project.
- New sites frequently work without code changes.
- The local quick score means the extension feels instant and costs nothing on the common path.
- Market economics live in one auditable file, so a wrong Dubai transfer fee is fixed in one
  place rather than hunted across modules.
- Free-tier preview analyses are genuinely free to serve, which makes the funnel viable.

**What got worse**

- **Pattern extraction is fuzzier than selectors.** It generalizes well but is wrong more often
  on edge cases — a price that's actually a monthly rent, a total-development price rather than
  a unit price. Wrong input produces a confident wrong analysis, and the user has no way to
  tell.
- **Silent degradation.** When a site redesign breaks extraction, nothing errors — the score
  is just wrong or missing. This needs synthetic monitoring per site to detect, which is real
  ongoing work the naive approach would at least have failed loudly on.
- **Config values are estimates that age.** Mortgage rates and vacancy rates move; a stale
  config is a quietly wrong calculation across a whole market. These need a review cadence.
- **Manifest V3 constraints** — service-worker lifecycle, no persistent background page — make
  state management in the extension more awkward than MV2 would have.
- **Third-party page content flows to the AI** on detailed analysis, which is the platform's
  highest-risk prompt-injection surface (ADR-0008).
- Each new site host needs a manifest permission, so site coverage still requires an extension
  release even when no code changes.

## Revisit when

- Extraction accuracy on a major site drops enough that a targeted adapter is worth the
  maintenance — the architecture should allow per-site overrides as exceptions without
  abandoning the general path.
- Enough sites publish reliable structured data (JSON-LD) that extraction can lean on it as a
  primary source rather than a hint.
- Market config drift causes a user-facing accuracy complaint, which argues for serving config
  from the backend so it updates without an extension release.
