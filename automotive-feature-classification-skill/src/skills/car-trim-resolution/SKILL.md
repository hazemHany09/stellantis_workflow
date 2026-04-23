---
name: car-trim-resolution
description: Lead-agent sub-skill that resolves an ambiguous car identity to the full-option trim when the user did not explicitly name one. Uses open-web search to identify the manufacturer's published top trim for the given (brand, model, model_year, market), records the decision in the main STATE.md, and returns the resolved trim string.
loader: lead
---

# Sub-skill — Car trim resolution

## When this runs

Preflight stage 1b of W1 (or any workflow) when:

* The user did not name a trim.
* Or `(brand, model, year, market)` is ambiguous (multiple body styles, facelifts, regional rebadges).

Always runs after parsing the four required fields; always before creating the KB dataset.

## Inputs

* `car_identity` (brand, model, model_year, market_input, market_canonical).

## Tools allowed

* `google_search`, `google_search_news`.

## Rules (AS-SCOPE-1, Q-INPUT-6, Q-C-2)

1. **Query pattern.** Issue at least:
   * `<brand> <model> <model_year> <market_canonical> top trim`
   * `<brand> <model> <model_year> full option specifications`
   * `<brand> <model> <model_year> trim levels`
2. **Selection rule.**
   * Select the manufacturer-official top trim for `(model_year, market_canonical)`.
   * If multiple premium branches exist (sport vs luxury — e.g. `M Sport Pro` vs `Luxury Line`), choose the most feature-rich variant (more standard options, higher price).
   * On market-specific rebadges (same car sold under a different name in a different market), pick the variant that matches `market_canonical` and note the rebadge relationship.
3. **Auto-pick, no gate** (Q-INPUT-6, Q-C-2 Option A). Do not ask the user to confirm the resolution. Record the decision and proceed.
4. **Documentation.** Append to main STATE.md the "Decisions made during this run" section:
   * `<timestamp>` — Auto-selected full-option trim: `<trim>` for `<brand> <model> <year> <market>`. Evidence URL: `<url>`.
5. **Variant note.** If multiple variants exist, record a `variant_note` (free text) to carry into the deliverable header `car_identity.variant_note`.

## Outputs

Return `{ "resolved_trim": "<string>", "variant_note": "<string or null>", "evidence_url": "<string>" }`. Lead updates main STATE.md.

## Success criteria

* `resolved_trim` is a concrete string (never `unknown`).
* At least one evidence URL supports the choice — URL is not ingested here; it is retained as a decision-audit reference.

## Failure modes

| Failure                                           | Response                                                                 |
| :------------------------------------------------ | :----------------------------------------------------------------------- |
| Web search returns nothing plausible              | Set `resolved_trim = "full-option (trim name unavailable from public sources)"` and append a clear note in STATE.md's decisions block. Proceed. |
| Model appears to not exist for the year/market    | Continue per AS-INPUT-C (no pre-validation). Note the gap; let the deliverable surface Rule-4 records later. |

## Business-logic trace

AS-SCOPE-1, Q-INPUT-6, Q-C-2, Q-C-6.
