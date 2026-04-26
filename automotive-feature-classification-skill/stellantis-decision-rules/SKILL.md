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

## Edge cases

- **Single clear + silent others.** Silent sources don't count; the lone clear source carries Rule 1 (or Rule 5 if its statement is "absent").
- **Mixed conflict — valid level vs undefined tier.** A clear source describing a level that exists in the parameter's applicable set, alongside a clear source describing a tier the schema does not define: the undefined-tier source is **disregarded** for classification. Apply Rule 1 (or Rule 2a if multiple valid-level sources still conflict). Record an internal advisory noting that out-of-schema evidence was observed.
- **All active sources support an undefined tier only.** No valid-level evidence exists. Apply Rule 3 (`Unable to Decide` with `Presence = Yes`). Record the same internal advisory.
- **Gap-fill re-evaluation.** After a gap-fill cycle, a parameter that was Rule 4 in cycle 1 may become Rule 1, 3, or 5 with the enlarged evidence set. The engine re-runs from scratch — it does not patch the previous record.

## Invariants (validators must enforce all 8)

1. Silent sources are never counted for Rule 1, 2, 3, or 5.
2. Vague sources alone never produce `Success` or `Conflict`. They can only produce `Unable to Decide` (Rule 3 with `Presence = Yes`).
3. A single clear active source is sufficient for `Success` provided no other clear active source contradicts it.
4. `Classification` is non-empty **only** under Rule 1. It is `Empty` under all other rules.
5. `Unable to Decide` corresponds to either Rule 3 (`Presence = Yes`) or Rule 4 (`Presence = No Information Found`). It is never a catch-all for "uncertain".
6. Every record matches exactly one matrix row.
7. `Classification` must belong to the parameter's applicable level set. Out-of-schema evidence triggers Rule 3, never a snap.
8. `Disputed` is exclusive to Rule 2b. `No Information Found` is exclusive to Rule 4 and is a `Presence` value only — never a `Status` value.

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
