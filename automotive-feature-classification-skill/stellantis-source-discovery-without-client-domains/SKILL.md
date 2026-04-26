---
name: stellantis-source-discovery-without-client-domains
description: Use when no URLs or domains were supplied by the user — W2 stage 3 or W1 Mode-A fallback. Lead context only.
loader: lead
stage: W1-stage-3-modeB, W2-stage-3
requires: []
provides:
  - source-candidate-list.md
tools:
  - google_search
  - google_search_news
  - google_search_images
  - google_search_videos
  - google_search_autocomplete
---

## Dependencies

None. Standalone skill (Mode B fallback). Produces output consumed by `stellantis-source-validation`.

---

# Sub-skill — Source discovery without client-supplied domains (Mode B)

## Overview

Runs open-web queries for the target car tuple across feature categories, applies the source-type ladder and candidate cap, and produces a ranked `source-candidate-list.md` with `origin = agent`.

## When this runs

W2 stage 3 (not authored yet as a full workflow, but this sub-skill is ready). Also usable mid-W1 if Mode A yields nothing and the user authorises fallback.

## Inputs

| Input                    | Source                                       |
| :----------------------- | :------------------------------------------- |
| `car_identity`           | Main STATE.md                                |
| Categories + parameters  | Frozen `runs/<run-id>/params.csv`            |
| `cap`                    | Main STATE.md (default 20, AS-INPUT-D)       |

## Tools allowed

* `google_search`, `google_search_news`, `google_search_images`, `google_search_videos`, `google_search_autocomplete`.

## Rules

1. **Base query**: `<brand> <model> <model_year> <market_canonical>`.
2. **Feature-oriented expansions**, one per broad category that `params.csv` declares. Examples:
   * `+ "ADAS"` `+ "driver assistance"`
   * `+ "EV"` `+ "electric range" OR "charging"`
   * `+ "infotainment"` `+ "wireless carplay" OR "android auto"`
   * `+ "cockpit"` `+ "digital cluster"`
3. **Market-zone expansion**: if `market_canonical` is a zone (`EU`, `MEA`, `GCC`, `NAFTA`), run two or three queries with prominent member countries as tie-breakers (e.g. for `EU`: `Germany`, `France`, `Italy`).
4. **Source-type ladder.** Prefer manufacturer official > brochure / PDF > long-form press > specialist sites > generic aggregators. Explicitly de-prioritise forums and classified/resale listings.
5. **Canonicalisation + dedupe**. Paywall filter (silent drop).
6. **Cap.** After dedupe, keep the top `cap` candidates ranked by:
   * Source-type ladder position.
   * Exact `(model, year, market)` match.
   * Recency (newer page for same model wins over older).
   * Length / depth of coverage (longer feature-rich pages beat summaries).
7. **Language.** All languages accepted. Non-English pages are kept without translation at this stage.
8. **No side effects.** This sub-skill neither downloads nor ingests. Those happen in `source-download-and-ingest` after the approval gate.

## Outputs

Write `runs/<run-id>/source-candidate-list.md` using the template. All entries carry `origin = agent`.

## Success criteria

* Candidate list ≥ 1 row and ≤ `cap` rows.
* At least one candidate covers each of the largest categories in `params.csv` — EV, ADAS, INFOT. — or the lead records a gap note in the main STATE.md before posting.

## Failure modes

| Failure                                   | Response                                                                   |
| :---------------------------------------- | :------------------------------------------------------------------------- |
| Zero viable candidates after `cap` filter | Surface to user; ask whether to expand `cap`, switch car tuple, or abort. |
| Only listings / resale aggregators remain | Pause and surface the situation to the user — do not ingest aggregators.   |
