# 09 — Deliverable

The deliverable is the final artefact produced for one car. Its contents come from the harness (see `06-harness-interface.md`) plus the traceability blocks collected along the classification process.

## Per-parameter record — fields the client requested

Every parameter in the reference list produces exactly one record with the following fields:

| Field               | Cardinality | Meaning                                                                                                                                                                                                                                                                                                                                      |
| :------------------ | :---------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Category            | 1           | The category (functional area) of the parameter, from the CSV.                                                                                                                                                                                                                                                                               |
| Parameter name      | 1           | The optimised name of the parameter, from the CSV.                                                                                                                                                                                                                                                                                           |
| Schema type         | 1           | Which tiers (Basic / Medium / High) are defined by the CSV for this parameter — i.e. the applicable level set. See `06-harness-interface.md`.                                                                                                                                                                                                |
| Presence            | 1           | Whether the parameter was found in the evidence: `Yes`, `No`, `Disputed`, or `No Information Found`. Always filled. Determined jointly with Status per the rules in `10-decision-rules.md`. `Disputed` is used only under Rule 2b (presence conflict). `No Information Found` is used only under Rule 4 (no active sources at all). |
| Status              | 1           | Outcome of the assignment process: `Success`, `Conflict`, or `Unable to Decide`. Always filled. Corresponds to the decision rules in `10-decision-rules.md` (Rules 1, 5 → `Success`; Rules 2a, 2b → `Conflict`; Rules 3, 4 → `Unable to Decide`). `No Information Found` as a status value is retired.                                                          |
| Classification      | 1           | The level assigned: `High`, `Medium`, `Basic`, or `Empty`. Always filled (may be `Empty`). Non-`Empty` **only when** Status = Success and Presence = Yes. When non-`Empty`, must be within the parameter's Schema type.                                                                                                                      |
| Traceability blocks | 0..N        | Zero or more traceability blocks (definition below). At least one is required when Presence = Yes. Multiple blocks may support the same verdict.                                                                                                                                                                                             |

Presence / Status / Classification together form the decision stack. Their allowed combinations are listed in the matrix in `06-harness-interface.md`.

## Traceability block

Each traceability block is the unit of evidence attached to a parameter record. A single parameter can carry multiple blocks. Each block has exactly these fields:

| Field                        | Meaning                                                                                                                                                                                                                                                                                                                            |
| :--------------------------- | :--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Source name                  | Human-readable label of the source document (e.g. article title, brochure title).                                                                                                                                                                                                                                                  |
| Source link                  | The original URL the source was retrieved from. Must correspond to a URL approved via the source approval store and ingested into the run's KB.                                                                                                                                                                                    |
| Source justification         | Short prose. Explains why this source was used — i.e. why it is relevant, authoritative, or otherwise appropriate evidence for this parameter.                                                                                                                                                                                     |
| Classification justification | Short prose. Explains why, based on the evidence in this source, the parameter is (or is not) placed at the reported level. When the parameter's Presence is `Yes` but Status is not `Success`, this field still describes what the source contributed to the decision (e.g. which level it hinted at before conflict resolution). |

### Cardinality rules for traceability blocks

Blocks are built only from **active sources** — clear or vague — for the parameter at hand (see source taxonomy in `10-decision-rules.md`). Silent sources never produce blocks.

* **Status = Success, Presence = Yes.** At least one clear-source block. Additional clear or vague blocks may be attached when they reinforce the decision.
* **Status = Success, Presence = No.** At least one clear-source block attesting explicit absence. (See Q-HARN-7 — the client may later prefer a different Status label for this case.)
* **Status = Conflict (Rule 2a — level conflict, Presence = Yes).** At least **two** clear-source blocks must be attached — the blocks whose level conclusions created the conflict.
* **Status = Conflict (Rule 2b — presence conflict, Presence = Disputed).** At least **two** clear-source blocks must be attached — one asserting presence, one asserting absence. No classification analysis is attempted; each block's *classification justification* simply records the presence stance of that source.
* **Status = Unable to Decide.** At least one vague-source block. No clear-source block is attached (by definition — if there were a clear source, the Status would be Success or Conflict instead).
* **Status = Unable to Decide, Presence = No Information Found, silent-all case (Rule 4).** No blocks — by definition, there were no active sources for this parameter. Rule 4 and Rule 5 are distinguishable directly: Rule 4 emits `(No Information Found, Unable to Decide, Empty)` with no traceability blocks; Rule 5 emits `(No, Success, Empty)` with at least one clear-source block.

### Multiplicity is per classification, not per source

A single source document can appear in multiple traceability blocks across different parameters. What identifies a block is the combination of (parameter, source) + the justifications it carries for that parameter. Reuse of the source URL across parameters is expected and allowed.

## Serialisation format

**Resolved — Q-DEL-1 (closed 2026-04-22).** Primary deliverable: a structured JSON file named `<brand>-<model>-<year>-<market>-<run-id>.json` containing the full schema with all traceability blocks intact. Secondary deliverable: a flattened CSV export generated alongside the JSON for analyst review in spreadsheet tools. Traceability blocks are represented in CSV via concatenation or multi-row expansion as needed. Both files are written to the run workspace root.

Filename convention and exact CSV column mapping are defined in the workflow design phase.

## Resolved questions

All deliverable questions are closed. For the record:

* **Q-DEL-1** — JSON primary + CSV secondary. Closed 2026-04-22.
* **Q-DEL-2** — No intermediate review gate. Deliverable emitted directly as final. Closed 2026-04-22.
* **Q-DEL-3** — Re-runs are flexible via three workflow patterns (full-car, per-parameter flagging, gap-fill). Closed 2026-04-22.
* **Q-DEL-4** — Compact notation (`B/H`, `B/M/H`, `H`, etc.) with legend in deliverable header. Closed 2026-04-22.
* **Q-DEL-5** — Run-level summary + extended analytics included. Closed 2026-04-22.

