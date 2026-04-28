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

> If there are **no active sources** for the parameter at all — every reviewed source is silent — Status = `Success`, Presence = `No`, Classification = `Empty`.
>
> This treats silence across all reviewed sources as an operational "not found in reviewed sources". It does **not** assert the feature is physically absent from the car; it asserts only that the approved, ingested source set contains no mention of the parameter.
>
> **Decided 2026-04-22.** Replaces the former `No Information Found` outcome for the silent-all case. See `11-assumptions.md` (Q-HARN-9) for the accepted trade-off.

Resulting output:

* `presence = No`
* `status = Success`
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
* Presence = `No`, Status = `Success`, Classification = `Empty` — no active sources at all (Rule 4 silent-all; distinguishable from Rule 5 only by the empty traceability-block set).
* Presence = `No`, Status = `Success`, Classification = `Empty` — clear sources agree the feature is absent (Rule 5).

## Master decision matrix

Every output record must match exactly one row of this table.

| Rule    | Evidence situation (per parameter)                                                                          | Presence   | Status             | Classification   |
| :------ | :---------------------------------------------------------------------------------------------------------- | :--------- | :----------------- | :--------------- |
| Rule 1  | All clear active sources agree feature is present and on the same valid level (≥1 clear; vague may coexist) | `Yes`      | `Success`          | the agreed level |
| Rule 5  | All clear active sources agree feature is absent (explicit; ≥1 clear; vague may coexist)                    | `No`       | `Success`          | `Empty`          |
| Rule 2a | ≥2 clear active sources agree feature is present but disagree on level                                      | `Yes`      | `Conflict`         | `Empty`          |
| Rule 2b | ≥2 clear active sources disagree on feature being present vs absent                                         | `Disputed` | `Conflict`         | `Empty`          |
| Rule 3  | ≥1 active source, but **no clear active source** (all active are vague)                                     | `Yes`      | `Unable to Decide` | `Empty`          |
| Rule 4  | No active sources at all (every source is silent on this parameter)                                         | `No`       | `Success`          | `Empty`          |

Presence values in use: `Yes`, `No`, `Disputed`. `No Information Found` is retired (see `06-harness-interface.md` Invariant 8 and `11-assumptions.md` AS-HARN-A).
Status values in use: `Success`, `Conflict`, `Unable to Decide`. `No Information Found` is retired as a Status value.

Rule 4 and Rule 5 share the output triple `(No, Success, Empty)` by design. The only visible difference in the deliverable is the traceability-block set: Rule 5 carries at least one clear-source block; Rule 4 carries none. Consumers who need to tell them apart must inspect traceability.

## Invariants (for validators)

1. Silent sources are never counted in Rule 1, Rule 2, Rule 3, or Rule 5.
2. Vague sources alone never trigger `Success`, `Conflict`, or `No Information Found`. They can only trigger `Unable to Decide`, and only when no clear active source exists.
3. A single clear active source is sufficient to decide `Success` — provided no other clear active source contradicts it.
4. `Classification` is filled **only** under Rule 1 (agreed level). It is always `Empty` under Rules 2a, 2b, 3, 4, and 5.
5. `Unable to Decide` must never be used as a catch-all for "the model was unsure". It corresponds to a concrete source-taxonomy situation: active sources exist but none are clear. `No Information Found` is retired and must not be emitted.
6. Every output record must map to exactly one row of the master decision matrix.
7. **A parameter's** **`Classification`** **must belong to the parameter's Schema type** — the non-empty level columns in `artifacts/params.csv`. If the CSV leaves a level cell empty, that level is **not** a valid target for Classification, regardless of what any source describes. Evidence of an undefined tier triggers Rule 3 (`Unable to Decide`), never a nearest-level snap. This enforces the client's guidance: *unless a level is explicitly specified in* *`params.csv`, the parameter cannot be assigned that level.*
8. `Disputed` is the only Presence value permitted under Rule 2b and cannot appear anywhere else.

