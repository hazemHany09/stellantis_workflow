---
name: stellantis-car-trim-resolution
description: Use when car identity is established but trim is unspecified or ambiguous — lead agent preflight, stage 1b. Load before creating the KB dataset.
loader: lead
stage: W1-stage-1b
requires:
  - stellantis-domain-context
  - serper-api-mastery
provides:
  - resolved_trim
  - variant_note
  - evidence_url
  - trim_package_map
tools:
  - google_search
  - google_search_news
---

## Dependencies

- `stellantis-domain-context` — vocabulary and the "full-option trim" definition.
- `serper-api-mastery` — query operators (`site:`, `intitle:`, `filetype:pdf`, `tbs` for model-year recency) used by every search this skill issues.

Reads `enums-reference` for market validation. Outputs are written by the lead into the main state file.

---

# Sub-skill — Car trim resolution

## Overview

Resolves an unspecified or ambiguous trim to the manufacturer-official full-option variant for the target `(model_year, market_canonical)`. Runs once per preflight; records the decision in STATE.md with an evidence URL. No user confirmation gate.

## When this runs

Preflight stage 1b of W1 (or any workflow) when:

* The user did not name a trim.
* Or `(brand, model, year, market)` is ambiguous (multiple body styles, facelifts, regional rebadges).

Always runs after parsing the four required fields; always before creating the KB dataset.

## Inputs

* `car_identity` (brand, model, model_year, market_input, market_canonical).

## Tools allowed

* `google_search`, `google_search_news`.

## Policy — what "full option" means

**"Full option" = the manufacturer's published top trim for the specified `(model_year, market_canonical)`.** This definition is fixed; the agent never invents a meaning of "full option". The resolved trim name is recorded in the deliverable header so the user can see (and override via re-run) the agent's choice.

## Rules

1. **Query pattern.** Issue at least:
   * `<brand> <model> <model_year> <market_canonical> top trim`
   * `<brand> <model> <model_year> full option specifications`
   * `<brand> <model> <model_year> trim levels`
2. **Selection rule.**
   * Select the manufacturer-official top trim for `(model_year, market_canonical)`.
   * If multiple premium branches exist (sport vs luxury — e.g. `M Sport Pro` vs `Luxury Line`), choose the **most feature-rich** variant (more standard options, higher price).
   * On market-specific rebadges (same car sold under a different name in a different market), pick the variant that matches `market_canonical` and note the rebadge relationship.
3. **Auto-pick, no gate.** Do not ask the user to confirm the resolution. Record the decision and proceed. The user may request a re-run with an alternative trim if the resolved name is wrong.
4. **Documentation.** Append to the main state file's "Decisions made during this run" section:
   * `<timestamp>` — Auto-selected full-option trim: `<trim>` for `<brand> <model> <year> <market>`. Evidence URL: `<url>`.
5. **Variant note.** If multiple variants exist, record a `variant_note` (free text) to carry into the deliverable header `car_identity.variant_note`.

## Trim package map (pre-classification)

The single trim string returned above is a deliverable header value. It is **not** sufficient to anchor classification on its own — for several models the published top trim shares its name with a body-style label, a marketing tier, or a regional package. The 2024 Jeep Wrangler illustrates the failure mode: "Unlimited" denotes the four-door body style and is shared across many trims; "Rubicon 392" is the actual top-tier technical trim; "Xtreme 35 Package" and "Technology Group" are option groups that meaningfully change the standard-equipment list. A classification subagent that reads "Full-Option" and asks the KB about "Unlimited" will hit ambiguous evidence and fall through to `Unable to Decide` purely as a vocabulary mismatch — not because the data is missing.

To prevent this, the trim-resolution skill emits a **trim package map** alongside the resolved trim. Downstream subagents read this map and rewrite their queries against it.

### Step 1 — Enumerate every named tier/package/group for the target `(model_year, market_canonical)`

Run discovery queries (see `serper-api-mastery` for operator construction) that surface, at minimum:

* All published **trim levels** (the named tier line — e.g. `Sport`, `Sport S`, `Willys`, `Sahara`, `High Altitude`, `Rubicon`, `Rubicon X`, `Rubicon 392`).
* All published **body-style or wheelbase labels** (e.g. `Unlimited`, `4-Door`, `LWB`, `Crew Cab`).
* All published **option packages / groups** (e.g. `Xtreme 35 Package`, `Technology Group`, `Trailer Tow Group`, `Cold Weather Group`).
* All published **special editions** (e.g. `Final Edition`, `Anniversary Edition`).

Source-type ladder for this enumeration: manufacturer official build-and-price > published brochure (PDF) > model-year press kit > major-press long-form review. Avoid forums for enumeration — they conflate years and markets.

### Step 2 — Map each named entity to a role

For every entity discovered in Step 1, assign exactly one role:

| Role | Meaning | Affects "Full-Option" anchor? |
| :--- | :--- | :--- |
| `top_trim` | Manufacturer's published top tier for the year/market. Carries the richest standard-equipment list. | Yes — this is the canonical anchor. |
| `lower_trim` | A trim below `top_trim`. | No — but listed so subagents can recognise lower-trim mentions and discount them. |
| `body_style` | Body-style / wheelbase label, not a tier. Often shared across multiple trims. | No — never the anchor; flag explicitly so subagents do not equate it with the top tier. |
| `option_package` | Standalone option group buyable on multiple trims. | Conditional — see "Full-Option" definition below. |
| `special_edition` | Limited / market-exclusive variant. | No — per existing skill rule, special editions are not the anchor. |

### Step 3 — Compose the "Full-Option" definition

`resolved_trim` = the `top_trim`. `Full-Option` for classification purposes = `top_trim` **plus** the union of every `option_package` whose membership on `top_trim` is published as available (i.e. the maximally specced configuration of the published top trim).

Record this composition explicitly. Subagents must know which packages count toward Full-Option so they classify a feature offered only via `Technology Group` as **Yes** when the option group is part of Full-Option, and as **No** (or downgraded presence) otherwise.

### Step 4 — Emit the trim package map

Output an additional `trim_package_map` object:

```json
{
  "top_trim": "<string>",
  "full_option_definition": "<top_trim> + <option_packages[]>",
  "named_entities": [
    { "name": "<string>", "role": "top_trim | lower_trim | body_style | option_package | special_edition", "evidence_url": "<string>", "note": "<string or null>" }
  ],
  "package_membership": {
    "<top_trim>": ["<option_package_1>", "<option_package_2>", "..."]
  }
}
```

The lead writes this to the main STATE.md immediately under the trim-resolution decision block, and ships it as a header field in every subagent contract (so subagents can disambiguate trim mentions in retrieved chunks). This step replaces ad-hoc reasoning ("Unlimited probably means the 4-door, so the top trim must be Rubicon 392") with an explicit map that holds across all categories of the run.

### Step 5 — Resolve ambiguous matches against the map

When a subagent encounters a chunk like `"available on Wrangler Unlimited Rubicon"`, it must consult the map and recognise that `Unlimited` is `body_style` (not a tier) and `Rubicon` is a `lower_trim` relative to `Rubicon 392`. The chunk therefore does not evidence Full-Option (Rubicon 392) standard equipment — the evidence is for a lower trim and downgrades to vague at best. This rule lives in the subagent classification loop; the trim-resolution skill provides the map it consults.

## Outputs

Return `{ "resolved_trim": "<string>", "variant_note": "<string or null>", "evidence_url": "<string>", "trim_package_map": { ... } }`. Lead updates main STATE.md and ships `trim_package_map` into the subagent contract under `car_identity.trim_package_map`.

## Success criteria

* `resolved_trim` is a concrete string (never `unknown`).
* At least one evidence URL supports the choice — URL is not ingested here; it is retained as a decision-audit reference.
* `trim_package_map.named_entities[]` covers every published trim, body-style label, option package, and special edition for the target `(model_year, market_canonical)`. Missing an option package is acceptable only when no source enumerates it; missing the top trim or a body-style label is a failure.
* `trim_package_map.package_membership[top_trim]` is non-empty whenever option packages exist on the top trim. An empty list means "verified that the top trim ships every relevant feature standard, with no add-on packages affecting Full-Option" — record an evidence URL for that claim.

## Failure modes

| Failure                                           | Response                                                                 |
| :------------------------------------------------ | :----------------------------------------------------------------------- |
| Web search returns nothing plausible              | Set `resolved_trim = "full-option (trim name unavailable from public sources)"` and append a clear note in STATE.md's decisions block. Proceed. |
| Model appears to not exist for the year/market    | Continue. Note the gap; let the deliverable surface Rule-4 records later. |

## Tips & common pitfalls

- **Don't ask the user to pick a trim.** Auto-pick is the policy. The deliverable header makes the choice visible.
- **Don't pick a special edition** (limited run, market-exclusive package) over a regularly published top trim, even if the special edition is more feature-rich. Stay with the **published** top trim.
- **Don't return `unknown`.** If sources are silent, return `"full-option (trim name unavailable from public sources)"` so downstream stages have a non-empty string.
- **Do record the evidence URL.** Even if the URL is not ingested into the KB, it is the audit trail for this decision.
- **Don't pre-validate the year.** If the (model, year) pair is implausible, sources will be silent and the deliverable will reflect that via Rule 4.
- **Don't conflate body-style with trim.** "Unlimited" on a Wrangler is the four-door body style, not a trim — every body-style label needs explicit `role = body_style` in the package map so subagents do not silently treat it as the anchor.
- **Resolve the package map even when the user supplied a trim name.** The user's trim string still needs decomposition — what counts as "Full-Option" depends on the option packages, not just the tier name.
