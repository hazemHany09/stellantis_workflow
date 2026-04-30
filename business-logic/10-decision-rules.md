# 10 — Decision Rules

This document defines, per parameter, **how the harness turns retrieved evidence into Presence / Status / Classification**. It is the business rule-set the harness must implement. The *how* (prompting, retrieval strategy) remains out of scope — only the outcome contract is fixed here.

It introduces a **source taxonomy** — the categories into which each source falls when evaluated against one specific parameter — and then states the decision rules as a function of that taxonomy.

## Source taxonomy (per parameter)

The same source document can fall into different taxonomy buckets for different parameters. The taxonomy is always evaluated **per parameter**, not globally.

| Taxonomy label          | Meaning                                                                                                                                                                                                                 |
| :---------------------- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Silent source**       | Does not mention the parameter at all for the target car. It contributes nothing to the decision.                                                                                                                       |
| **Vague active source** | Mentions the parameter for the target car but does not provide enough detail to map the evidence to any of the parameter's valid levels or to confirm a presence verdict.                                               |
| **Clear active source** | Mentions the parameter for the target car *and* provides usable information — either a level that maps to the parameter's schema (Basic / Medium / High as applicable) or an explicit statement of presence or absence. |

Derived term: an **active source** is any source that is not silent (i.e. clear or vague).

## Decision rules

The four rules below govern the output triple (Presence, Status, Classification). Wording is aligned with the client's specification.

### Rule 1 — Success

> If **all clear active sources** support the same valid level, Status = `Success` and Classification = the agreed level.
>
> Vague sources alongside clear sources **do not block agreement**. Only the clear sources determine the outcome.
> Agreement can come from **one** clear active source (if no other clear source contradicts it) or from multiple aligned sources.

Resulting output:

* `presence = Yes`
* `status = Success`
* `classification = <the agreed level>` (must be a member of the parameter's Schema type)

### Rule 2 — Conflict

A **conflict** exists when two or more clear active sources support **different conclusions** for the same parameter. Vague sources are ignored in conflict detection. Silent sources must not reduce or suppress a conflict.

Conflict splits into two sub-cases, each with a different `presence` output.

#### Rule 2a — Level conflict

> Two or more clear active sources all confirm the feature is present, but disagree on **which level** the implementation meets (e.g. one source supports `High`, another supports `Basic`).

Resulting output:

* `presence = Yes`
* `status = Conflict`
* `classification = Empty`

#### Rule 2b — Presence conflict

> Clear active sources disagree on **whether the feature exists at all** — one source says the feature is present, another says it is absent.

Resulting output:

* `presence = Disputed`
* `status = Conflict`
* `classification = Empty`

`Disputed` is a dedicated Presence value used only under Rule 2b. It signals that no further classification analysis is meaningful because the sources do not even agree that the feature exists.

### Rule 3 — Unable to Decide

> If there are **active sources** for the parameter but **none provide enough information** to confirm any of the valid levels (High, Medium, or Basic as applicable), Status = `Unable to Decide` and Classification = Empty.
>
> This status is **not** triggered by vague sources alongside clear ones — in that case the clear sources determine the outcome (Success or Conflict).
> Unable to Decide means the parameter is mentioned but the evidence is collectively insufficient to classify it.

Resulting output:

* `presence = Yes`
* `status = Unable to Decide`
* `classification = Empty`

### Rule 4 — Silent-all (Absent by silence)

> If there are **no active sources** for the parameter at all — every reviewed source is silent — Status = `Unable to Decide`, Presence = `No Information Found`, Classification = `Empty`.
>
> This distinguishes the "no evidence found" case from the "evidence explicitly says absent" case (Rule 5). `No Information Found` does **not** assert the feature is physically absent from the car; it asserts only that the approved, ingested source set contains no mention of the parameter.
>
> **Revised 2026-04-23.** Reinstates `No Information Found` as a `presence` value for the silent-all case. The former `Success / No` output for this case (decided 2026-04-22) is superseded. See `11-assumptions.md` (Q-HARN-9, Q-HARN-10).

Resulting output:

* `presence = No Information Found`
* `status = Unable to Decide`
* `classification = Empty`

### Rule 5 — Absent by agreement

If all clear active sources agree that the feature is **absent** (explicit negative statement, not silence):

* `presence = No`
* `status = Success`
* `classification = Empty`

This is the "Success" shape for the absent case. It mirrors Rule 1 but applies to a negative presence conclusion rather than a level. (Q-HARN-7 closed: client confirmed `Status = Success` with `Presence = No`.)

### Single-clear + silent others

A frequent pattern: one source provides a clear statement (e.g. *feature is present and at High*) and all other sources are silent on the parameter. Per the source taxonomy, silent sources do not count, so the single clear source is alone in the decision and Rule 1 applies — `Status = Success`, `Presence = Yes`, `Classification = High`. No conflict arises.

## Three-field model — why Presence, Status, Classification are all distinct

The three fields answer **three different questions** and must not be collapsed:

| Field              | Question it answers                                                             |
| :----------------- | :------------------------------------------------------------------------------ |
| **Presence**       | What does the evidence say about whether this feature exists on the car?        |
| **Status**         | What is the outcome of the *assignment process* for this parameter?             |
| **Classification** | If the assignment process succeeded, at which level is the feature implemented? |

Example combinations:

* Presence = `Yes`, Status = `Conflict`, Classification = `Empty` — feature is present, but clear sources disagree on level (Rule 2a).
* Presence = `Disputed`, Status = `Conflict`, Classification = `Empty` — clear sources disagree on whether the feature even exists (Rule 2b).
* Presence = `Yes`, Status = `Unable to Decide`, Classification = `Empty` — feature is mentioned but only by vague sources (Rule 3).
* Presence = `No Information Found`, Status = `Unable to Decide`, Classification = `Empty` — no active sources at all (Rule 4 silent-all; distinguishable from Rule 5 by both the `presence` value and the empty traceability-block set).
* Presence = `No`, Status = `Success`, Classification = `Empty` — clear sources agree the feature is absent (Rule 5).

## Master decision matrix

Every output record must match exactly one row of this table.

| Rule    | Evidence situation (per parameter)                                                                          | Presence               | Status             | Classification   |
| :------ | :---------------------------------------------------------------------------------------------------------- | :--------------------- | :----------------- | :--------------- |
| Rule 1  | All clear active sources agree feature is present and on the same valid level (≥1 clear; vague may coexist) | `Yes`                  | `Success`          | the agreed level |
| Rule 5  | All clear active sources agree feature is absent (explicit; ≥1 clear; vague may coexist)                    | `No`                   | `Success`          | `Empty`          |
| Rule 2a | ≥2 clear active sources agree feature is present but disagree on level                                      | `Yes`                  | `Conflict`         | `Empty`          |
| Rule 2b | ≥2 clear active sources disagree on feature being present vs absent                                         | `Disputed`             | `Conflict`         | `Empty`          |
| Rule 3  | ≥1 active source, but **no clear active source** (all active are vague)                                     | `Yes`                  | `Unable to Decide` | `Empty`          |
| Rule 4  | No active sources at all (every source is silent on this parameter)                                         | `No Information Found` | `Unable to Decide` | `Empty`          |

Presence values in use: `Yes`, `No`, `Disputed`, `No Information Found`. `No Information Found` as a **status** value remains retired.
Status values in use: `Success`, `Conflict`, `Unable to Decide`.

Rule 4 and Rule 5 produce distinct output triples. Rule 4 emits `(No Information Found, Unable to Decide, Empty)` with no traceability blocks. Rule 5 emits `(No, Success, Empty)` with at least one clear-source block. The two cases are directly distinguishable by `presence` and `status` without inspecting traceability.

## Source-type categories and consensus

Not every clear source is interchangeable. Each source document is assigned a **source-type** that describes the kind of claim it can authoritatively make. The deliverable distinguishes verdicts backed by **multiple independent source-types** from verdicts backed by a single category.

### Source-type categories

Each ingested document is assigned exactly one source-type. Two clear sources count as **independent** only when they belong to different categories.

| Source-type                    | What it is                                                                  | Authority for                                                                      |
| :----------------------------- | :-------------------------------------------------------------------------- | :--------------------------------------------------------------------------------- |
| `manufacturer_official_spec`   | OEM build-and-price page, configurator, OEM brochure, OEM press kit.        | Trim availability, optional / standard split, package contents.                    |
| `manufacturer_owner_manual`    | Owner's manual, user guide.                                                 | How a feature operates; rarely authoritative for trim-level availability.          |
| `dealer_fleet_guide`           | Fleet-buyer guide, dealer order guide, salesperson reference.               | Standard / Optional matrix per trim — the canonical Standard / Optional / — grid.  |
| `third_party_press_long_form`  | Long-form review (TheVerge, MotorTrend, Car and Driver, Top Gear, etc.).    | Operational behaviour, real-world performance claims.                              |
| `third_party_aggregator`       | Edmunds, KBB, Cars.com feature lists.                                       | Trim availability matrices; cross-source for press claims.                         |
| `manufacturer_video`           | OEM video walkthroughs.                                                     | How a feature operates; rarely trim-specific.                                      |
| `forum_or_ugc`                 | Owner forums, community discussion threads, dealer forums.                  | Soft signal only; never sole authority.                                            |

### Confidence annotation

Every emitted record carries a `confidence` value, assigned deterministically alongside `(Presence, Status, Classification)`:

| Rule           | Independent source-types backing the verdict                                                | `confidence`                  |
| :------------- | :------------------------------------------------------------------------------------------ | :---------------------------- |
| Rule 1 / Rule 5 | ≥ 2 distinct source-type categories all clear and aligned                                  | `consensus`                   |
| Rule 1 / Rule 5 | exactly 1 distinct category (whether 1 source or many from the same category)              | `single-source`               |
| Rule 2a / Rule 2b | mandatorily ≥ 2 clear-source blocks; same category on both sides                          | `single-source` (with advisory `same-source-type-conflict`) |
| Rule 2a / Rule 2b | different categories on each side                                                          | `consensus`                   |
| Rule 3         | vague-only evidence                                                                         | `vague-only`                  |
| Rule 4         | silent-all (only after the inverse-retrieval test below)                                    | `silent-all`                  |

Confidence is not a status. It does not change which row of the master matrix a record matches. It is a parallel signal so deliverable consumers can distinguish "two independent kinds of source agree" from "one kind said so".

### Promotion and demotion

* **Promotion.** If a parameter is emitted at `Rule 1, single-source` and a later evidence pass surfaces a clear corroborating source from a different category, the record is re-emitted at `Rule 1, consensus`.
* **UGC demotion.** A Rule 1 or Rule 5 record whose only clear source is `forum_or_ugc` is invalid. The verdict is demoted to Rule 3 (`Yes, Unable to Decide, Empty`, `confidence = vague-only`). Forum claims alone never suffice for `Success`.

## Inverse retrieval — prerequisite for Rule 4

Standard searching is biased toward presence: looking for a feature surfaces documents that mention the feature, not documents that explicitly state it is unavailable. The result is that explicitly-absent features are routinely classified as Rule 4 (`No Information Found`) when they should have been classified as Rule 5 (`No, Success`). Rule 5 carries far more value to the consumer because it is a confirmed absence.

Before a parameter can be **finalised at Rule 4**, an inverse-retrieval test must run. The test re-queries the same approved-and-ingested source set using the language of negation:

* `"<feature_name>" "not available"`
* `"<feature_name>" "deleted"`
* `"<feature_name>" "no longer offered"`
* `"<feature_name>" "<top_trim>" "exclusion"`
* `"features not available on <top_trim>"`
* `"<top_trim>" "deletes" OR "omits" OR "lacks"`

Outcomes:

* If the inverse pass surfaces any clear statement that the feature is absent on the resolved trim, the verdict is **promoted to Rule 5** (`No, Success, Empty`) with that statement as a clear-source traceability block (stance = absent).
* If the inverse pass produces no negative evidence, the verdict stays at Rule 4. The record carries `inverse_retrieval_attempted = true` so the deliverable shows that absence-of-evidence was tested, not assumed.

A Rule 4 record without `inverse_retrieval_attempted = true` is invalid.

## Invariants (for validators)

1. Silent sources are never counted in Rule 1, Rule 2, Rule 3, or Rule 5.
2. Vague sources alone never trigger `Success` or `Conflict`. They can only trigger `Unable to Decide` (Rule 3, `presence = Yes`), and only when no clear active source exists. Rule 4 (`presence = No Information Found`) is triggered by total silence, not by vague sources.
3. A single clear active source is sufficient to decide `Success` — provided no other clear active source contradicts it.
4. `Classification` is filled **only** under Rule 1 (agreed level). It is always `Empty` under Rules 2a, 2b, 3, 4, and 5.
5. `Unable to Decide` must never be used as a catch-all for "the model was unsure". It corresponds to two concrete source-taxonomy situations: (a) Rule 3 — active sources exist but none are clear (`presence = Yes`); (b) Rule 4 — no active sources exist at all (`presence = No Information Found`). `No Information Found` as a *status* value remains retired and must not be emitted.
6. Every output record must map to exactly one row of the master decision matrix.
7. **A parameter's** **`Classification`** **must belong to the parameter's Schema type** — the non-empty level columns in `artifacts/params.csv`. If the CSV leaves a level cell empty, that level is **not** a valid target for Classification, regardless of what any source describes. Evidence of an undefined tier triggers Rule 3 (`Unable to Decide`), never a nearest-level snap. This enforces the client's guidance: *unless a level is explicitly specified in* *`params.csv`, the parameter cannot be assigned that level.*
8. `Disputed` is the only Presence value permitted under Rule 2b and cannot appear anywhere else.
9. **Consensus.** Every Rule 1 / Rule 5 record carries `confidence ∈ {consensus, single-source}`. `consensus` requires ≥ 2 clear sources from **distinct** source-type categories.
10. **No UGC-only Success.** A Rule 1 or Rule 5 record whose only clear-source traceability block is `source_type = forum_or_ugc` is invalid; the verdict is demoted to Rule 3.
11. **Inverse retrieval before Rule 4.** Every Rule 4 record carries `inverse_retrieval_attempted = true`. A Rule 4 record without this flag is invalid.

