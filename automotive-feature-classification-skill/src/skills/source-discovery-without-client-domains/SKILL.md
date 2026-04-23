---
name: source-discovery-without-client-domains
description: Mode B source discovery. Invoked by the lead agent when the user did not supply any URLs or domains. Uses the car tuple + categories from params.csv to build targeted queries against the open web. Caps the candidate list at 20 URLs unless the user set a different cap.
loader: lead
---

# Sub-skill — Source discovery without client-supplied domains (Mode B)

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

1. **Base query** (AS-SRC-H): `<brand> <model> <model_year> <market_canonical>`.
2. **Feature-oriented expansions**, one per broad category that `params.csv` declares. Examples:
   * `+ "ADAS"` `+ "driver assistance"`
   * `+ "EV"` `+ "electric range" OR "charging"`
   * `+ "infotainment"` `+ "wireless carplay" OR "android auto"`
   * `+ "cockpit"` `+ "digital cluster"`
3. **Market-zone expansion** (Q-INPUT-7): if `market_canonical` is a zone (`EU`, `MEA`, `GCC`, `NAFTA`), run two or three queries with prominent member countries as tie-breakers (e.g. for `EU`: `Germany`, `France`, `Italy`).
4. **Source-type ladder.** Prefer manufacturer official > brochure / PDF > long-form press > specialist sites > generic aggregators. Explicitly de-prioritise forums and classified/resale listings (see `04-sources.md`).
5. **Canonicalisation + dedupe** per AS-SRC-A. Paywall filter per Q-SRC-4 (silent drop).
6. **Cap.** After dedupe, keep the top `cap` candidates ranked by:
   * Source-type ladder position.
   * Exact `(model, year, market)` match.
   * Recency (newer page for same model wins over older).
   * Length / depth of coverage (longer feature-rich pages beat summaries).
7. **Language.** All languages accepted (Q-SRC-5). Non-English pages are kept without translation at this stage.
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

## Business-logic trace

AS-SRC-A, AS-SRC-E, AS-SRC-H, AS-INPUT-D, Q-SRC-2, Q-SRC-4, Q-SRC-5, Q-INPUT-7, Q-C-10.
