---
name: stellantis-source-validation
description: Use immediately after source discovery completes — to filter the candidate list before presenting it to the user. Lead context only.
loader: lead
stage: W1-stage-3b
requires:
  - stellantis-domain-context
  - stellantis-failure-handling
  - stellantis-source-discovery-with-client-domains
  - stellantis-source-discovery-without-client-domains
provides:
  - validated-source-candidate-list.md
  - source-type-tagging
tools:
  - browser_render_markdown
---

## Dependencies

Foundational:
- `stellantis-domain-context`, `stellantis-failure-handling` (canonicalisation rules, paywall drop policy).

Conditional:
- `stellantis-source-discovery-with-client-domains` (if Mode A), OR
- `stellantis-source-discovery-without-client-domains` (if Mode B)

Consumes candidate list from either discovery skill; produces validated list for `stellantis-approval-gate`.

---

# Sub-skill — Source validation

## Overview

Applies heuristic checks — car match, year, market, paywall, aggregator, dead link — to the in-memory candidate list from a discovery sub-skill. Returns a pruned, annotated list. One optional browser render per ambiguous URL; no deep crawl.

## When this runs

Immediately after `source-discovery-with-client-domains` or `source-discovery-without-client-domains`, before the candidate list is posted to the user.

## Inputs

| Input          | Source                                    |
| :------------- | :---------------------------------------- |
| Candidate list | In-memory, from a discovery sub-skill     |
| `car_identity` | Main STATE.md                             |

## Tools allowed

* `browser_render_markdown` — **only** for URLs whose Google snippet is too short to judge. One call per such URL; no deep crawl.

## Heuristics (free-text judgement)

For each candidate, the lead reviews the title + snippet (+ optional markdown render) and asks:

1. **Car match.** Does the page mention the target `brand`, `model`, and at least one of `model_year` or `market_canonical`? If neither, mark `weak-car-match` — keep but flag "why selected" accordingly.
2. **Year match.** Is the page clearly about a different model year (e.g. title says `2019` but we target `2025`)? If yes and no co-coverage, drop with reason `wrong-year`.
3. **Market match.** Explicit signal that the page is scoped to a different market (e.g. "US-only trim") — downgrade ranking but keep (markets often overlap).
4. **Paywall / login-wall.** Snippet suggests paywall: drop silently if `origin = agent`; surface flag if `origin = client`.
5. **Aggregator / resale.** Known aggregator or classified listing hostnames: drop unless `origin = client`.
6. **Dead link.** 404-like snippet or redirect chain: drop.
7. **Forum / UGC as primary source.** Soft downgrade. Keep only if it carries clear technical claims with links to primary sources.

## Source-type tagging

Every surviving candidate must be tagged with exactly one `source_type` value from the categories below. The tag travels with the URL into the KB document metadata at upload time, and the decision-rule engine uses it for consensus verification (Rule 1 / Rule 5 require ≥ 2 distinct `source_type` categories to emit `confidence = consensus`).

Assign the tag from URL host + page title + snippet (rendered markdown if needed). Use this matrix:

| `source_type` | Typical hosts / cues |
| :--- | :--- |
| `manufacturer_official_spec` | OEM domain (e.g. `stellantis.com`, `jeep.com`, `peugeot.com`), build-and-price page, configurator, "Specifications" page, OEM brochure download. |
| `manufacturer_owner_manual` | OEM owner-information portal, `owner.<oem>.com`, page title contains "Owner's Manual" / "User Guide" / "Operator Manual". |
| `dealer_fleet_guide` | Dealer order guides, fleet-buyer guides, salesperson references. Often `.pdf` titled "Order Guide" or "Fleet Guide". |
| `third_party_press_long_form` | TheVerge, MotorTrend, Car and Driver, Top Gear, Road and Track, Autocar, Auto Express, Top Speed, etc. — long-form review article. |
| `third_party_aggregator` | Edmunds, KBB, Cars.com, Parkers (UK), DriveSpark — feature-list / spec-table pages. |
| `manufacturer_video` | OEM-owned YouTube channels (`youtube.com/@Jeep`, `youtube.com/@Peugeot`). Verify channel ownership before tagging. |
| `forum_or_ugc` | Reddit, owner forums (e.g. `wranglerforum.com`, `peugeotforums.com`), Quora, blog comments. |

### Decision rules for ambiguous cases

* A press article hosted on an aggregator's news section → `third_party_press_long_form` (the editorial layer dominates the host).
* A brochure PDF hosted on an aggregator's CDN → `manufacturer_official_spec` (the document content dominates the host).
* A dealer website republishing an OEM brochure → `manufacturer_official_spec`.
* A YouTube video on a channel not owned by the OEM → `third_party_press_long_form` if the channel is a known auto outlet; otherwise `forum_or_ugc`.

The `source_type` is **mandatory** — every kept candidate must carry one. If unable to confidently assign one, fall back to the most conservative match (typically `third_party_press_long_form` over `manufacturer_*`, and `forum_or_ugc` over any third-party). Never invent a category.

## Outputs

Return to the discovery caller a pruned + annotated candidate list. Each surviving candidate has:

* `why_selected` — one-sentence rationale referencing the heuristic that kept it.
* `flags[]` — machine-readable flags (`weak-car-match`, `market-mismatch`, `origin-client-paywalled`, etc.).
* `source_type` — exactly one value from the categories above. Carried through into KB metadata and consumed by the consensus-verification step in `stellantis-decision-rules`.

## Success criteria

* Candidate list size ≤ input list size.
* No candidate has both a `dead-link` flag AND is kept.
* Every kept candidate has a non-empty `why_selected`.

## Failure modes

| Failure                            | Response                                                                            |
| :--------------------------------- | :---------------------------------------------------------------------------------- |
| All candidates fail validation     | Return empty list; caller must escalate (ask user to provide URLs or expand `cap`). |
| Browser rendering repeatedly fails | Continue on snippet-only judgement; note in main STATE.md event log.                |
