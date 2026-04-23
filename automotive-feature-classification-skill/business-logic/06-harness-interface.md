# 06 — Harness Interface

The harness is the classification stage that converts ingested evidence into parameter verdicts. Its **internal algorithm** is deliberately out of scope for this document set — it will be designed separately. This document fixes only the **interface**: what the harness consumes, what it produces, and the invariants its output must satisfy. Everything else in the skill depends on this interface.

The harness output is also the basis for the final deliverable — field-level serialisation rules live in `09-deliverable.md`.

## Inputs

The harness receives, per analysis run:

1. A reference to the analysis scope (car identity, run ID).
2. The reference parameter list, loaded from `artifacts/params.csv`, including the applicable level set per parameter.
3. A handle to the knowledge base populated for this run, already fully ingested.

## Outputs

For **every** parameter in the reference list, the harness must emit one record with the following fields. Field names mirror the client's requested deliverable vocabulary.

| Field                 | Type                                                  | Required                                               | Notes                                                                                                                                                                                                                                                                                                                                                   |
| :-------------------- | :---------------------------------------------------- | :----------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `category`            | string                                                | Always                                                 | The category name from the CSV (`Domain / Category`).                                                                                                                                                                                                                                                                                                   |
| `parameter_name`      | string                                                | Always                                                 | The optimised parameter name from the CSV (`Optimised Parameter Name`). This is the externally reported name.                                                                                                                                                                                                                                           |
| `schema_type`         | string                                                | Always                                                 | Describes which level tiers are defined for this parameter in the CSV — i.e. the applicable level set. See "Schema type values" below. **Q-DEL-4 open** on exact encoding.                                                                                                                                                                              |
| `presence`            | `Yes` \| `No` \| `Disputed` \| `No Information Found` | Always                                                 | Answers: does the evidence say the feature is on the car? `Disputed` appears only under Rule 2b (clear sources disagree on presence vs absence). `No Information Found` appears only under Rule 4 (no active sources at all — every reviewed source is silent on this parameter). Value is determined jointly with `status` per `10-decision-rules.md`. |
| `status`              | `Success` \| `Conflict` \| `Unable to Decide`         | Always                                                 | Outcome of the assignment process. The three values map to the five main decision rules (Rule 1 and Rule 5 both emit `Success`; Rule 2a / 2b both emit `Conflict`; Rule 3 and Rule 4 both emit `Unable to Decide`). `No Information Found` as a status value remains retired.                                                                           |
| `classification`      | `High` \| `Medium` \| `Basic` \| `Empty`              | Always (may be `Empty`)                                | The assigned level. Filled only when `status = Success` **and** `presence = Yes`. Must be a member of the parameter's applicable level set (see `02-parameters-and-levels.md`). `Empty` in every other case.                                                                                                                                            |
| `traceability_blocks` | list of traceability blocks (see `09-deliverable.md`) | At least one when `presence = Yes`; optional otherwise | Each block carries source name, source link, and its own justification text. Multiple blocks may support a single parameter verdict. See `09-deliverable.md`.                                                                                                                                                                                           |

### Schema type values

The `schema_type` field describes which of the three levels the CSV actually defines for the parameter. It is an informational mirror of the applicable level set. The exact string values are left open pending client confirmation (see Q-DEL-4) — candidate encodings are:

* `"B/M/H"`, `"B/H"`, `"M/H"`, `"H"`, `"B/M"`, `"B"`, `"M"` (compact).
* `"Basic+Medium+High"`, `"Basic+High"`, etc. (verbose).
* A structured list such as `["Basic", "High"]`.traceability

Pick one convention for the whole run. Do not mix.

## Invariants

These must hold for every deliverable. Violations are bugs.

1. Every parameter in `artifacts/params.csv` (excluding category-header and Category-Overview rows) appears exactly once in the output.
2. `classification` is never outside the applicable level set of its parameter.
3. `status` is always filled — there is no "null status" record.
4. `classification` is `Empty` unless `status = Success` and `presence = Yes`.
5. Every output record must match exactly one row of the master decision matrix in `10-decision-rules.md`.
6. When `presence = Yes`, at least one traceability block is attached.
7. Every traceabilityblock references a source document that is in the approved, ingested set of the run's KB.
8. `No Information Found` is a valid **`presence`** value used exclusively under Rule 4 (silent-all — no active sources at all). It signals that the search found no mention of the parameter in any reviewed source and therefore cannot assert either presence or absence. `No Information Found` as a **`status`** value remains retired and must not appear in any emitted record's `status` field. See `10-decision-rules.md` and `11-assumptions.md` (Q-HARN-9, Q-HARN-10, AS-HARN-A).
9. `Unable to Decide` is used in two distinct cases: (a) Rule 3 — active sources exist but none are clear (`presence = Yes`); (b) Rule 4 — no active sources exist at all (`presence = No Information Found`). The two cases are always distinguishable by their `presence` value.
10. `schema_type` matches the actual non-empty level columns of the parameter's CSV row — no invented levels, no missing levels.

## Presence × status × classification matrix

The master decision matrix lives in `10-decision-rules.md`. The summary truth table below is the validation copy — any row outside it is a bug.

| presence               | status             | classification          | rule    |
| :--------------------- | :----------------- | :---------------------- | :------ |
| `Yes`                  | `Success`          | level in applicable set | Rule 1  |
| `No`                   | `Success`          | `Empty`                 | Rule 5  |
| `Yes`                  | `Conflict`         | `Empty`                 | Rule 2a |
| `Disputed`             | `Conflict`         | `Empty`                 | Rule 2b |
| `Yes`                  | `Unable to Decide` | `Empty`                 | Rule 3  |
| `No Information Found` | `Unable to Decide` | `Empty`                 | Rule 4  |

## Why the interface matters before the algorithm

The rest of the skill — source selection, KB population, deliverable generation — is designed to feed and consume this interface. Locking the interface first lets those surrounding components be specified independently of how the harness actually decides. When the harness design begins, its authors can change the internal algorithm freely as long as they honour this contract.

## Out of scope for this document

* How the harness queries the KB.
* How it distinguishes `No` from `No Information Found`.
* How it distinguishes `Conflict` from `Unable to Decide`.
* How it matches prose descriptions against criterion text.
* Whether it runs one pass, multiple passes, or per-category passes.
* Whether it is powered by a single model call, a chain, a graph, or something else.

