---
name: stellantis-decision-rules
description: Use whenever an agent must convert a per-parameter evidence set into a Presence / Status / Classification triple. Loaded primarily by classification subagents but valid for any agent that emits parameter records.
loader: shared
stage: classification
requires:
  - stellantis-domain-context
provides:
  - decision-rule-engine
  - master-decision-matrix
  - rule-invariants
  - consensus-verification
  - inverse-retrieval-protocol
tools: []
---

## Dependencies

Hard prerequisite:
- `stellantis-domain-context` — vocabulary, three-field model, presence/status/classification enums.

Tightly paired with:
- `stellantis-output-contract` — the record shape this rule engine populates.
- `stellantis-subagent-classification-loop` — the subagent that invokes this engine.

---

# Sub-skill — Decision rules

## Overview

The single source of truth for converting an evidence set into a parameter verdict. Five rules, six valid output shapes, eight invariants. No skill — including the subagent classification loop — may implement a rule that contradicts this file. Updates are governed by `framework-maintenance/impact-checklist.md`.

## When this runs

Every time a parameter record is emitted. In W1 the subagent invokes the engine once per parameter in its partition. In gap-fill cycles the rules are re-applied with the enlarged evidence set. The engine is also consulted by the lead when manually authoring fallback Rule-4 records (e.g. after a subagent double-failure).

## The five decision rules

The rule engine takes one input — the **per-parameter evidence set**, with each piece of evidence labelled *clear active*, *vague active*, or *silent* — and emits one `(Presence, Status, Classification)` triple plus traceability blocks.

### Rule 1 — Success (present)

All clear active sources support the same valid level. Vague sources alongside clear sources do not block agreement; they coexist. A single uncontradicted clear active source is sufficient.

| Field | Value |
| :--- | :--- |
| `Presence` | `Yes` |
| `Status` | `Success` |
| `Classification` | the agreed level (must belong to the parameter's applicable level set) |
| Traceability | ≥ 1 clear-source block (vague blocks may also appear) |

### Rule 2a — Level conflict

Two or more clear active sources all confirm the feature is present, but disagree on **which level** the implementation meets.

| Field | Value |
| :--- | :--- |
| `Presence` | `Yes` |
| `Status` | `Conflict` |
| `Classification` | `Empty` |
| Traceability | ≥ 2 clear-source blocks |

### Rule 2b — Presence conflict

Clear active sources disagree on **whether the feature exists at all** — one says present, another says absent.

| Field | Value |
| :--- | :--- |
| `Presence` | `Disputed` |
| `Status` | `Conflict` |
| `Classification` | `Empty` |
| Traceability | ≥ 2 clear-source blocks (mixed stances) |

`Disputed` is reserved exclusively for this rule. It must appear nowhere else.

### Rule 3 — Unable to Decide

There are active sources for the parameter, but **no clear** active source — every active source is vague.

| Field | Value |
| :--- | :--- |
| `Presence` | `Yes` |
| `Status` | `Unable to Decide` |
| `Classification` | `Empty` |
| Traceability | ≥ 1 vague-source block |

### Rule 4 — Silent-all

No active sources at all — every reviewed source is silent on the parameter.

| Field | Value |
| :--- | :--- |
| `Presence` | `No Information Found` |
| `Status` | `Unable to Decide` |
| `Classification` | `Empty` |
| Traceability | none |

`No Information Found` does **not** assert physical absence. It asserts only that the approved, ingested source set contains no mention.

### Rule 5 — Success (absent)

All clear active sources agree the feature is **explicitly absent** (negative statement, not silence). Vague sources may coexist.

| Field | Value |
| :--- | :--- |
| `Presence` | `No` |
| `Status` | `Success` |
| `Classification` | `Empty` |
| Traceability | ≥ 1 clear-source block |

## Master decision matrix

Every emitted record must match exactly one row.

| Rule | Evidence situation | Presence | Status | Classification |
| :--- | :--- | :--- | :--- | :--- |
| Rule 1  | All clear active sources agree on the same valid level (vague may coexist) | `Yes` | `Success` | the agreed level |
| Rule 5  | All clear active sources agree feature is absent (vague may coexist) | `No` | `Success` | `Empty` |
| Rule 2a | ≥ 2 clear active sources agree present but disagree on level | `Yes` | `Conflict` | `Empty` |
| Rule 2b | ≥ 2 clear active sources disagree on presence vs absence | `Disputed` | `Conflict` | `Empty` |
| Rule 3  | ≥ 1 active source, but no clear active source (all active are vague) | `Yes` | `Unable to Decide` | `Empty` |
| Rule 4  | No active sources at all (every source silent for this parameter) | `No Information Found` | `Unable to Decide` | `Empty` |

## Source-type categories and consensus verification

A clear source is not just clear — it has a **source-type** that determines what kind of claim it can authoritatively make. Treating all clear sources as interchangeable was the mistake behind a class of false-positives observed in early runs: an owner's-manual mention of "how Night Vision works" was being snapped to `Presence = Yes` even when the manual covered the entire model line, including trims where the feature was optional or absent.

The fix is to require **consensus across independent source-types** for the strongest verdict, and to flag single-source verdicts as lower-confidence so the deliverable carries an honest signal.

### Source-type categories (independent groups)

Each ingested document is assigned exactly one `source_type` during source validation. The categories are defined as follows. Two clear sources count as **independent** only when they belong to different categories.

| `source_type` | What it is | Authority for |
| :--- | :--- | :--- |
| `manufacturer_official_spec` | OEM build-and-price page, configurator, OEM brochure, OEM press kit. | Trim availability, optional / standard split, package contents. |
| `manufacturer_owner_manual` | Owner's manual, user guide. | How a feature operates; rarely authoritative for trim-level availability ("if equipped"). |
| `dealer_fleet_guide` | Fleet-buyer guide, dealer order guide, salesperson reference. | Standard-vs-optional matrix per trim — the canonical Standard / Optional / — grid. |
| `third_party_press_long_form` | Long-form review (TheVerge, MotorTrend, Car and Driver, Top Gear, etc.). | Operational behaviour, real-world performance claims. |
| `third_party_aggregator` | Edmunds, KBB, Cars.com feature lists. | Trim availability matrices; cross-source for press claims. |
| `manufacturer_video` | OEM YouTube channel walkthroughs. | How a feature operates; rarely trim-specific. |
| `forum_or_ugc` | Owner forums, Reddit, dealer forums. | Soft signal only; never sole authority. |

### `confidence` annotation on every record

Every emitted record now carries a `confidence` field alongside `(Presence, Status, Classification)`. The decision-rule engine assigns it deterministically:

| Rule | Independent clear-source-types backing the verdict | `confidence` |
| :--- | :--- | :--- |
| Rule 1 / Rule 5 | ≥ 2 distinct `source_type` categories all clear and aligned | `consensus` |
| Rule 1 / Rule 5 | exactly 1 distinct `source_type` category (whether 1 source or many from the same category) | `single-source` |
| Rule 1 / Rule 5 | the single source is `forum_or_ugc` | downgrade to Rule 3 (`Unable to Decide`); UGC alone is never sufficient for Success |
| Rule 2a / Rule 2b | mandatorily ≥ 2 clear-source blocks; if both are the same `source_type`, mark `confidence = single-source` and add an advisory `kind = same-source-type-conflict` | `consensus` (different types) or `single-source` (same type) |
| Rule 3 | vague-only evidence | `confidence = vague-only` |
| Rule 4 | silent-all (after inverse retrieval — see below) | `confidence = silent-all` |

The `confidence` field is **not** a status value and never changes the matrix row. It is a parallel annotation that surfaces in the deliverable (and can be used by humans to triage). Subagent emitters and the lead consolidator must populate it on every record.

### When `single-source` Rule 1 is the right call

`single-source` is **not** a failure mode — there are parameters (especially how-it-works descriptions of niche features) where only one canonical source ever discusses them. The annotation exists so the deliverable distinguishes "two independent sources agree" from "one source said so". Lower-confidence verdicts are still useful; they are just labelled.

### Promotion rule

If a parameter starts a cycle at `Rule 1, confidence = single-source` and a later cycle (gap-fill, deep-dive doc-read, retrieval expansion) surfaces a clear corroborating source from a different `source_type`, the engine re-emits the parameter at `Rule 1, confidence = consensus`. The promotion is unconditional — no human gate.

### Demotion rule

If the only clear source backing a Rule 1 verdict is `forum_or_ugc`, the engine **demotes** the verdict to Rule 3 (`Yes, Unable to Decide, Empty`). Forum claims alone do not meet the bar for `Success` because there is no editorial accountability.

## Inverse retrieval — the prerequisite for Rule 4

Standard retrieval is biased toward presence: a vector search for "Night Vision" surfaces docs that mention the term, not docs that explicitly state it is unavailable. The result is that physically absent features are routinely emitted as Rule 4 (`No Information Found`) when they should be emitted as Rule 5 (`No, Success, Empty`). Rule 5 carries far more value to the deliverable consumer because it is a confirmed absence; Rule 4 is an admission that the run is silent.

### The inverse-retrieval pass

Before any parameter can be **finalised at Rule 4**, the subagent must execute an inverse-retrieval pass. The pass is bounded — one or two queries per parameter — and runs only over the same KB dataset:

1. Build queries that target the language of negation, exclusion, or removal:
   - `"<feature_name>" "not available"`
   - `"<feature_name>" "deleted"`
   - `"<feature_name>" "no longer offered"`
   - `"<feature_name>" "<top_trim>" "exclusion"`
   - `"features not available on <top_trim>"`
   - `"<top_trim>" "deletes" OR "omits" OR "lacks"`
2. Inspect the top chunks. If any chunk explicitly states the feature is absent on the resolved trim — promote the verdict to **Rule 5** (`No, Success, Empty`) with the chunk as a clear-source traceability block.
3. If the inverse pass produces no negative evidence either, the verdict stays at Rule 4. Record `inverse_retrieval_attempted = true` in the per-parameter record so the deliverable can show that absence-of-evidence was tested, not assumed.

### Where this slot in the loop

The inverse-retrieval pass is **part of the decision rule engine's contract**, not an optional optimisation. Subagent loops (search-mode and doc-deep-dive alike) must run it before they emit a Rule 4 record. The lead consolidator must reject any Rule 4 record where `inverse_retrieval_attempted` is missing or `false`, and re-spawn the subagent for that parameter.

## Edge cases

- **Single clear + silent others.** Silent sources don't count; the lone clear source carries Rule 1 (or Rule 5 if its statement is "absent").
- **Mixed conflict — valid level vs undefined tier.** A clear source describing a level that exists in the parameter's applicable set, alongside a clear source describing a tier the schema does not define: the undefined-tier source is **disregarded** for classification. Apply Rule 1 (or Rule 2a if multiple valid-level sources still conflict). Record an internal advisory noting that out-of-schema evidence was observed.
- **All active sources support an undefined tier only.** No valid-level evidence exists. Apply Rule 3 (`Unable to Decide` with `Presence = Yes`). Record the same internal advisory.
- **Gap-fill re-evaluation.** After a gap-fill cycle, a parameter that was Rule 4 in cycle 1 may become Rule 1, 3, or 5 with the enlarged evidence set. The engine re-runs from scratch — it does not patch the previous record.

## Invariants (validators must enforce all 11)

1. Silent sources are never counted for Rule 1, 2, 3, or 5.
2. Vague sources alone never produce `Success` or `Conflict`. They can only produce `Unable to Decide` (Rule 3 with `Presence = Yes`).
3. A single clear active source is sufficient for `Success` provided no other clear active source contradicts it. The verdict carries `confidence = single-source` (see invariant 9).
4. `Classification` is non-empty **only** under Rule 1. It is `Empty` under all other rules.
5. `Unable to Decide` corresponds to either Rule 3 (`Presence = Yes`) or Rule 4 (`Presence = No Information Found`). It is never a catch-all for "uncertain".
6. Every record matches exactly one matrix row.
7. `Classification` must belong to the parameter's applicable level set. Out-of-schema evidence triggers Rule 3, never a snap.
8. `Disputed` is exclusive to Rule 2b. `No Information Found` is exclusive to Rule 4 and is a `Presence` value only — never a `Status` value.
9. **Consensus.** Every Rule 1 / Rule 5 record carries a `confidence` value of `consensus` or `single-source`. `consensus` is set only when ≥ 2 clear sources from **distinct** `source_type` categories agree. A Rule 1 / Rule 5 record whose only clear-source traceability blocks share a single `source_type` category is `single-source`.
10. **No UGC-only Success.** A Rule 1 or Rule 5 record whose only clear source is `source_type = forum_or_ugc` is invalid. The engine demotes it to Rule 3 with `confidence = vague-only`.
11. **Inverse retrieval before Rule 4.** Every Rule 4 record carries `inverse_retrieval_attempted = true` with at least one inverse-style query logged. A Rule 4 record without this flag is rejected by the consolidator and re-emitted by the originating loop.

## Tips & common pitfalls

- **`Status = Success` does not imply `Presence = Yes`.** Rule 5 emits `Success` with `Presence = No`. Both are valid `Success` shapes.
- **`Conflict` never produces a level.** If you find yourself wanting to "pick the most credible source", stop — that is exactly the behaviour the rules forbid. Emit `Conflict` and let the human adjudicate.
- **Don't infer absence from silence.** Rule 4 is not a degraded form of Rule 5; they are distinct.
- **A vague source plus a clear source is not a conflict.** Clear sources alone determine Success / Conflict. Vague sources contribute only when no clear source exists.
- **No nearest-level snap.** Schema-undefined evidence is filtered, not rounded.
- **One uncontradicted clear source = Success.** Do not require corroboration.

## What this skill is NOT

- Not a retrieval strategy. Retrieval lives in the subagent classification loop.
- Not a source taxonomy classifier. The agent decides clear / vague / silent operationally as a function of the evidence's presence and classification mappability — there is no separate algorithm.
- Not a deliverable schema (see `stellantis-output-contract`).
- Not a workflow.

## Maintenance

Adding, removing, or changing a rule requires:
1. Update `business-logic/10-decision-rules.md` first.
2. Update this skill, the master matrix, and any invariants affected.
3. Apply every follow-up listed in `framework-maintenance/impact-checklist.md` under "Change a decision rule in business-logic".
