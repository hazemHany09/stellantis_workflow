---
name: stellantis-source-discovery-with-client-domains
description: Use when the user's submission includes specific URLs or domain names — W1 stage 3, Mode A source discovery. Lead context only.
loader: lead
stage: W1-stage-3-modeA
requires:
  - stellantis-domain-context
  - stellantis-failure-handling
  - serper-api-mastery
provides:
  - source-candidate-list.md
tools:
  - google_search
  - google_search_news
---

## Dependencies

Foundational:
- `stellantis-domain-context` — vocabulary (source site vs domain).
- `stellantis-failure-handling` — URL canonicalisation, paywall drop.
- `serper-api-mastery` — query operator and date-filter reference. **Invoke before composing any `google_search` / `google_search_news` call** in this skill: it teaches `site:`, `inurl:`, `intitle:`, `filetype:`, `tbs` date filters, multi-domain `OR` chains, and the discovery → targeted two-step pattern that turns a vague car-tuple search into a high-signal candidate list.

Produces output consumed by `stellantis-source-validation`.

---

# Sub-skill — Source discovery with client-supplied domains (Mode A)

## Overview

Searches within each client-supplied domain for the target car tuple and preserves all client-direct URLs unconditionally (outside the candidate cap). Produces a ranked, deduplicated `source-candidate-list.md`.

## When this runs

W1 (Mode A) stage 3. The user's submission included either:

* Specific **URLs** that should be used directly, and/or
* **Domains** (hosts) the agent should search inside.

## Inputs

| Input                          | Source                                     |
| :----------------------------- | :----------------------------------------- |
| `car_identity`                 | Main STATE.md (4 fields + resolved trim)   |
| `client_urls[]`                | Parsed from the user's request             |
| `client_domains[]`             | Parsed from the user's request             |
| Category list                  | Frozen `runs/<run-id>/params.csv`          |

## Tools allowed

* `google_search` (with `site:` restriction)
* `google_search_news` (optional for recent press)
* No browser rendering yet — that happens in `source-download-and-ingest`.

**Before issuing any query, consult `serper-api-mastery`.** It is the canonical reference for query construction in this framework. The query patterns below are the *minimum* required per domain; `serper-api-mastery` shows how to escalate them — combining `intitle:`, `inurl:`, `filetype:pdf`, `tbs=qdr:y` for recency, and `autocorrect=False` so that `site:` and other operators are not silently rewritten by Google's spell-correction. When a domain returns weak results on the minimum patterns, escalate following `serper-api-mastery`'s discovery → targeted two-step pattern rather than firing more variations of the same shape.

## Rules

1. **Direct URLs are kept as-is.** Every `client_urls[]` entry is added to the candidate list with `origin = client` and **skips the cap**. Canonicalise before adding.
2. **Per-domain search.** For each `client_domains[]` entry, run at least two queries:
   * `<brand> <model> <model_year> <market> site:<domain>`
   * `<brand> <model> <model_year> features specifications site:<domain>`
3. **Optional per-category enrichment.** For large categories (EV, ADAS, INFOT.), add one sub-query that names the category keyword alongside the car tuple. Keep it bounded — at most 3 sub-queries per domain.
4. **Best 5 per domain.** After deduplication and paywall filtering, keep the top 5 URLs per domain ranked by:
   * Manufacturer-official page > technical brochure > detailed press review > summary > listing.
   * Closer `model_year` match > older year of same model.
   * Longer, feature-rich pages > short blurbs.
5. **Canonicalisation + dedupe**: lowercase host, strip fragment, strip tracking params, collapse trailing slash, sort query parameters. Two URLs canonicalising to the same form count as one.
6. **Paywall filter**. If the snippet indicates paywall (`"Subscribe to read"`, `"Sign in to continue"`, `"Register for free"`), drop silently and move on.
7. **No cap on client-supplied URLs**. Agent-discovered URLs under client-supplied domains ARE counted against the run cap, default 20.
8. **Origin tagging**: if a client URL matches a canonicalised agent-discovered URL, the merged record takes `origin = client`.

## Outputs

Write `runs/<run-id>/source-candidate-list.md` using the standard template. Each row carries: index, canonical URL, title, origin, why-selected (one sentence), excerpt (≤ 240 chars).

## Success criteria

* At least one candidate per client-supplied domain (unless every candidate is paywalled or dead — note in the "why selected" fallback).
* All client-supplied URLs are included.
* Candidate list is non-empty OR the agent explicitly flags that Mode-B discovery is needed as a supplement (require user confirmation before switching).

## Failure modes

| Failure                                          | Response                                                                                         |
| :----------------------------------------------- | :----------------------------------------------------------------------------------------------- |
| Client domain returns zero usable results        | Note in "why selected" fallback column: `no viable page found; consider removing this domain`. Keep the domain in the roster so the user can decide. |
| Every client URL is paywalled                    | Surface paywall status explicitly — the user may hold credentials. |
| No client input resolves to a valid candidate    | Pause with `PauseReason = missing-required-input` and ask the user whether to switch to Mode B.  |
