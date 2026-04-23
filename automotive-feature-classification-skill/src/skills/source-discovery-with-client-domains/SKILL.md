---
name: source-discovery-with-client-domains
description: Mode A source discovery. Invoked by the lead agent when the user has provided one or more domains or URLs. Produces a candidate URL list restricted to those domains + any direct URLs the user supplied. No more than 5 URLs per client-supplied domain.
loader: lead
---

# Sub-skill — Source discovery with client-supplied domains (Mode A)

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

## Rules

1. **Direct URLs are kept as-is.** Every `client_urls[]` entry is added to the candidate list with `origin = client` and **skips the cap** (AS-SRC-E). Canonicalise per AS-SRC-A before adding.
2. **Per-domain search.** For each `client_domains[]` entry, run at least two queries:
   * `<brand> <model> <model_year> <market> site:<domain>`
   * `<brand> <model> <model_year> features specifications site:<domain>`
3. **Optional per-category enrichment.** For large categories (EV, ADAS, INFOT.), add one sub-query that names the category keyword alongside the car tuple. Keep it bounded — at most 3 sub-queries per domain.
4. **Best 5 per domain.** After deduplication and paywall filtering, keep the top 5 URLs per domain ranked by:
   * Manufacturer-official page > technical brochure > detailed press review > summary > listing.
   * Closer `model_year` match > older year of same model.
   * Longer, feature-rich pages > short blurbs.
5. **Canonicalisation + dedupe** per AS-SRC-A: lowercase host, strip fragment, strip tracking params, collapse trailing slash, sort query parameters. Two URLs canonicalising to the same form count as one.
6. **Paywall filter** per Q-SRC-4. If the snippet indicates paywall (`"Subscribe to read"`, `"Sign in to continue"`, `"Register for free"`), drop silently and move on.
7. **No cap on client-supplied URLs** (AS-SRC-E). Agent-discovered URLs under client-supplied domains ARE counted against the run cap, default 20 (AS-SRC-E, AS-INPUT-D).
8. **Origin tagging** (AS-SRC-B): if a client URL matches a canonicalised agent-discovered URL, the merged record takes `origin = client`.

## Outputs

Write `runs/<run-id>/source-candidate-list.md` using the template at [`../../templates/source-candidate-list.md.tmpl`](../../templates/source-candidate-list.md.tmpl). Each row carries: index, canonical URL, title, origin, why-selected (one sentence), excerpt (≤ 240 chars).

## Success criteria

* At least one candidate per client-supplied domain (unless every candidate is paywalled or dead — note in the "why selected" fallback).
* All client-supplied URLs are included.
* Candidate list is non-empty OR the agent explicitly flags that Mode-B discovery is needed as a supplement (require user confirmation before switching).

## Failure modes

| Failure                                          | Response                                                                                         |
| :----------------------------------------------- | :----------------------------------------------------------------------------------------------- |
| Client domain returns zero usable results        | Note in "why selected" fallback column: `no viable page found; consider removing this domain`. Keep the domain in the roster so the user can decide. |
| Every client URL is paywalled                    | Surface paywall status explicitly (override Q-SRC-4's silent drop for client-supplied URLs) — the user may hold credentials. |
| No client input resolves to a valid candidate    | Pause with `PauseReason = missing-required-input` and ask the user whether to switch to Mode B.  |

## Business-logic trace

AS-SRC-A, AS-SRC-B, AS-SRC-E, AS-SRC-G, AS-INPUT-D, Q-SRC-4, Q-C-10.
