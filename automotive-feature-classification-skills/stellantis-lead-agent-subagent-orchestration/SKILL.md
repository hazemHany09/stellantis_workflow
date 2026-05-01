---
name: stellantis-lead-agent-subagent-orchestration
description: Use when ingestion verification has passed and parameter classification must begin — W1 stage 6. Covers two distinct roles: (1) lead agent spawning and consolidating the dispatch subagent; (2) dispatch subagent reading params.csv and STATE.md, partitioning parameters, spawning and monitoring R1/R2 classification subagents, and returning a consolidated envelope to the lead.
loader: shared
stage: W1-stage-6
requires:
  - stellantis-domain-context
  - stellantis-decision-rules
  - stellantis-output-contract
  - stellantis-failure-handling
  - stellantis-run-workspace
  - stellantis-subagent-classification-loop
  - stellantis-subagent-doc-deep-dive
provides:
  - dispatch-subagent-contract
  - partition-manifests
  - consolidated-category-STATE-files
  - archived-subagent-files
  - round1-document-promise-aggregate
  - round2-deep-dive-manifests
tools:
  - batch_update_doc_metadata
  - get_metadata_summary
---

## Dependencies

Foundational:
- `stellantis-domain-context`, `stellantis-decision-rules`, `stellantis-output-contract`, `stellantis-failure-handling`, `stellantis-run-workspace`.

Hard:
- `stellantis-subagent-classification-loop` — Round-1 search-mode subagent, spawned per partition by the **dispatch subagent** in stage 6a.
- `stellantis-subagent-doc-deep-dive` — Round-2 read-only-on-single-document subagent, spawned per (target-doc × gap-parameters) tuple by the **dispatch subagent** in stage 6b.

Receives KB dataset from `stellantis-source-download-and-ingest`. The lead spawns one dispatch subagent; the dispatch subagent orchestrates all R1/R2 classification work.

---

# Sub-skill — Stage 6 Orchestration (two-round classification)

## Overview

This skill governs **two distinct actors** at stage 6:

**Lead** — spawns exactly one classification dispatch subagent, monitors until its envelope is ready, then consolidates results into `.harness/Category/<category>.md` and the final STATE.md.

**Dispatch subagent** — reads `.harness/params.csv` and `.harness/STATE.md`, partitions parameters, drives the two-round subagent architecture (Round 1 + Round 2), collects all envelopes, and reports a single consolidated dispatch envelope back to the lead.

**Round 1 — search-mode (broad recall).** Dispatch subagent partitions the frozen `params.csv` into per-category batches and spawns a Round-1 search-mode subagent per partition (`stellantis-subagent-classification-loop`). Each runs per-parameter KB retrieval, applies the decision rules, and reports records + a `document_promise_scores[]` table + a partition summary.

**Dispatch consolidation pass A.** Dispatch subagent aggregates all Round-1 envelopes internally. Builds the document-promise board. Identifies gap parameters. Maps each gap parameter to the highest-promise document(s) most likely to resolve it.

**Round 2 — doc-deep-dive (high precision).** For each (target document × gap-parameters) tuple, dispatch subagent spawns a Round-2 deep-dive subagent (`stellantis-subagent-doc-deep-dive`). Each reads exactly one Markdown document end-to-end and emits records constrained to its evidence.

**Dispatch consolidation pass B.** Dispatch subagent merges Round-2 records with Round-1 records, applies merge logic, and builds the final dispatch envelope.

**Lead consolidation.** Lead reads the dispatch envelope, writes `.harness/Category/<category>.md` files, applies metadata patches, writes final STATE.md, archives dispatch working file, and proceeds to deliverable emission.

Concurrency cap (max 3 concurrent subagents) is enforced by the dispatch subagent separately within each round. Rounds are serial — Round 2 cannot start before Round 1 fully consolidates internally.

---

# Part 1 — Lead role at stage 6 entry

## When the lead runs this

W1 stage 6. After ingestion verification passes (stage 5 complete). Before deliverable emission (stage 7).

## Lead inputs

| Input                            | Source                          |
| :------------------------------- | :------------------------------ |
| Frozen `params.csv`              | `.harness/params.csv`           |
| `kb_dataset_id`                  | `.harness/STATE.md`             |
| `car_identity` + `resolved_trim` + `trim_package_map` | `.harness/STATE.md` |

## Step 1 — Pre-create the dispatch subagent working file

Create `.harness/DispatchAgent/dispatch.md` with the dispatch contract block:

```
<!-- DISPATCH CONTRACT -->
{
  "subagent_name": "dispatch",
  "params_csv_path": ".harness/params.csv",
  "state_md_path": ".harness/STATE.md",
  "kb_dataset_id": "<kb_dataset_id>",
  "car_identity": { <brand, model, model_year, market, resolved_trim> },
  "trim_package_map": { <from STATE.md> },
  "r2_cap": 6,
  "concurrency_cap": 3,
  "tool_allowlist": [
    "retrieval", "retrieve_knowledge_graph",
    "list_chunks", "ragflow_download",
    "doc_infos", "get_metadata_summary"
  ]
}
<!-- END DISPATCH CONTRACT -->
```

Pass the absolute path of the working file to the dispatch subagent on spawn as the first argument.

## Step 2 — Spawn dispatch subagent

Spawn ONE dispatch subagent. The lead does not spawn any R1/R2 classification subagents directly — that is the dispatch subagent's sole responsibility.

Update `.harness/STATE.md`:
- Record dispatch subagent name, spawn timestamp, `SubagentStatus = running`.
- Append event-log line: `"Classification dispatch subagent spawned"`.

## Step 3 — Monitor dispatch subagent

Periodically (not busy-loop) inspect `.harness/DispatchAgent/dispatch.md` for:

- Presence of `<!-- DISPATCH RESULT ENVELOPE -->` block → ready for lead consolidation.
- Monotonically increasing content size → still making progress.
- Presence of `"status": "failed"` inside the envelope → dispatch subagent surrendered; re-spawn once with the same contract.

If the dispatch subagent exceeds 3 hours with no progress, the lead may intervene, inspect the working file, and re-spawn once.

## Step 4 — Lead consolidation

When the dispatch subagent sets `"status": "dispatch-ready"` in its result envelope:

1. Read `.harness/DispatchAgent/dispatch.md`; extract the dispatch result envelope.
2. Validate the envelope structure (all required sections present; category record groups complete).
3. For each category in `category_record_groups[]`:
   a. Write `.harness/Category/<category>.md` — full per-parameter detail rendered via templates. Include all Round-1 and Round-2 records, category-level warnings.
4. Apply `docs_metadata_updates[]` by calling `batch_update_doc_metadata` with all staged patches from the envelope.
5. Promote `warnings[]` into the relevant `.harness/Category/<category>.md` warnings section.
6. Promote `advisories[]` into `.harness/advisories/undefined-tier.md` or `.harness/advisories/out-of-list-findings.md` based on `kind`.
7. Write `.harness/document-promise-board.md` from the envelope's `document_promise_board[]`.
8. Write `.harness/partition-summaries.md` from the envelope's `partition_summaries[]`.
9. Move `.harness/DispatchAgent/dispatch.md` to `.harness/Archive/dispatch.md`. Update `SubagentStatus = archived`.
10. Update main STATE.md with final summary counts, confidence distribution, gap-parameter list, R2 target assignments (extracted from envelope).

## Step 5 — Final STATE.md update (mandatory)

After dispatch subagent is archived and all records consolidated:

1. Set `RunStage = classification-complete`.
2. **[T1] Summary counts** — final total parameters, breakdown by `Presence` value.
3. **[T2] Confidence distribution** — consensus count, single-source count, Round 2 upgrades, Rule 4 records remaining after Round 2.
4. **[T2] Warnings and advisories summary** — counts: run-level warnings, category-level warnings, Rule 2a/2b conflicts, undefined-tier advisories, out-of-list findings, R2 cap reached.
5. Record `classification_completed_at` timestamp in the Header.
6. Set `completed_at` in Stage timing telemetry for `classification-round-2` (or `classification-round-1` if no R2 ran).
7. Append event-log line: `"Dispatch subagent consolidated — N parameters classified. Proceeding to deliverable emission."`.

This step is **not optional**. Skipping it leaves STATE.md in a stale intermediate state.

## Lead failure handling

| Failure | Response |
| :--- | :--- |
| Dispatch envelope fails validation | Mark `failed`, re-spawn once with the same contract. |
| Second spawn also fails | Lead writes Rule-4 records for every parameter; add a run-level warning in the deliverable footer. |
| Dispatch subagent exceeds ceiling | Intervene as described in *Monitor*; re-spawn once. |

---

# Part 2 — Dispatch subagent role

## When the dispatch subagent runs

Spawned by the lead at W1 stage 6 entry, after ingestion verification passes. The dispatch subagent drives the full two-round classification flow and reports back to the lead.

## Identity discovery

The dispatch subagent is invoked with the absolute path of its working file as the first argument. From that path:

1. `agent_name` = filename stem (`dispatch`).
2. Open the file; read the `<!-- DISPATCH CONTRACT -->` block.
3. Parse the contract JSON. If any required field is missing, write `"status": "failed"` in the result envelope with reason `missing-contract-field` and return.

## Inputs the dispatch subagent reads

| Input | Path | Access |
| :--- | :--- | :--- |
| Parameter list | `.harness/params.csv` | Read-only |
| Run state (kb_dataset_id, car identity, trim map) | `.harness/STATE.md` | Read-only |
| Own contract | `.harness/DispatchAgent/dispatch.md` | Read + write (scratch + envelope) |

The dispatch subagent **never writes** to `.harness/STATE.md` or any `.harness/Category/*.md` file.

## Tools allowed (dispatch subagent)

The dispatch subagent does **not** perform KB retrieval itself — it only orchestrates R1/R2 classification subagents. Its tools are limited to:

- File read/write on `.harness/DispatchAgent/dispatch.md`, `.harness/params.csv`, `.harness/STATE.md` (read-only).
- Pre-create `.harness/SubAgent/<agent-name>.md` and `.harness/DeepDiveAgent/<agent-name>.md` working files.
- The host platform's subagent-spawn primitive (for R1/R2 only).

The dispatch subagent does **not** use: `google_search`, any browser tool, `fetch_url`, any dataset/document management tool, `ragflow_upload`, `batch_update_doc_metadata`, `set_doc_metadata`.

## Round 1 — search-mode partitioning

The frozen `.harness/params.csv` is the authoritative parameter list. Do **not** assume a fixed count — use whatever the frozen CSV actually contains.

1. Enumerate categories from the frozen CSV — the distinct non-empty values of `Domain / Category` rows that are **not** category banners or `Category Overview` rows.
2. For each category, count parameters.
3. If count ≤ 15, make one partition named `<category-slug>-1`.
4. If count > 15, split into consecutive groups of ≤ 15 parameters each (preserve CSV row order). Name partitions `<category-slug>-1`, `<category-slug>-2`, ….
5. Record the complete partition list in the dispatch working file's scratch section.

## Contract construction (R1)

For each partition, construct a contract per the lead-to-subagent-contract specification.

1. Pre-create the working file at `.harness/SubAgent/<agent-name>.md` using the standard template.
2. Embed the contract JSON in the `<!-- AUTOGENERATED BY LEAD -->` block at the top of the file.
3. Pass the absolute path of the working file to the subagent on spawn as the first argument. The subagent's name is the filename stem.

## Concurrency

- **Max 3 subagents running concurrently.** Queue the rest.
- A subagent slot is freed once the dispatch subagent has internally consolidated its envelope (not merely when the subagent returns control). This keeps consolidation serial within the dispatch subagent.
- If a subagent takes longer than 30 minutes with no progress, mark `SubagentStatus = failed` and re-spawn once.

## Monitoring (within dispatch subagent)

The dispatch subagent periodically (not busy-loop) inspects each running subagent's working file for:

- Presence of `<!-- RESULT ENVELOPE -->` block → ready for internal consolidation.
- Monotonically increasing content size → still making progress.
- Presence of `"status": "failed"` → subagent surrendered; re-spawn once.

## Round-1 internal consolidation (serial within dispatch subagent)

For each Round-1 subagent that signals ready:

1. Read the working file; extract the result envelope.
2. Validate each record against the intermediate-parameter-record schema and each traceability block against the traceability-block schema.
3. Validate every enum value against the enums reference. Validate that every Rule 4 record carries `inverse_retrieval_attempted = true` (per invariant 11 in `stellantis-decision-rules`); reject and re-spawn if the flag is missing.
4. Stage `docs_metadata_updates[]` into the dispatch subagent's own scratch (to be forwarded to lead in the final envelope — dispatch subagent does **not** call `batch_update_doc_metadata` itself).
5. Accumulate records per category (keyed by category name) in the dispatch subagent's internal state.
6. Stage `warnings[]` per category.
7. Stage `advisories[]` per kind.
8. Append the subagent's `document_promise_scores[]` to the dispatch subagent's internal promise board, deduplicated by `doc_id`. When the same `doc_id` appears in multiple partitions, **sum** `parameters_evidenced[]` and `parameters_likely_to_be_evidenced[]` (deduped), and take the **maximum** `promise_score`.
9. Append the subagent's `partition_summary` to the dispatch subagent's partition summaries list.
10. Move the working file to `.harness/Archive/<agent-name>.md`.

## Round-1 → Round-2 hand-off (dispatch subagent consolidation pass A)

After every Round-1 subagent has finished and internally consolidated:

**Before spawning any Round-2 subagent, record in the dispatch working file's scratch section:**
1. The **Round 2 subagent roster** (header + empty table).
2. The **Gap parameters + R2 target assignments** JSON block (write-once — never edit after this point).

### Identify gap parameters

Walk every Round-1 record. Mark a parameter as a **gap parameter** when any of the following hold:

- The verdict is **Rule 4** (`No Information Found, Unable to Decide, Empty`). Even with `inverse_retrieval_attempted = true`, a Rule 4 may still be recoverable if a high-promise document discusses the feature in detail outside what vector retrieval surfaced.
- The verdict is **Rule 3** (`Yes, Unable to Decide, Empty`). The feature is mentioned but no clear source emerged.
- The verdict is **Rule 1 or Rule 5 with `confidence = single-source`**. A corroborating source-type from a different document would promote the record to `consensus`.

Do **not** mark Rule 2a / Rule 2b records as gap parameters — conflicts are resolved by human adjudication.

### Pick target documents

For each gap parameter, walk the internal promise board and select the highest-`promise_score` documents whose `parameters_likely_to_be_evidenced[]` includes that parameter. Greedy set-cover:

1. Pick the document with the highest `promise_score`. Assign all gap parameters listed in its `parameters_likely_to_be_evidenced[]` to that document's deep-dive.
2. Subtract those parameters from the gap set. Repeat with the next-highest-promise document for the remaining gaps.
3. Cap the round at **at most 6 deep-dive subagents per run** (configurable via contract `r2_cap`). Once the cap is reached, remaining gap parameters are accepted at their Round-1 verdict; record this in warnings.

When a gap parameter appears in zero documents' `parameters_likely_to_be_evidenced[]`, no deep-dive is spawned for it — Round 1's verdict stands.

## Contract construction (R2)

For each (target_doc × gap_parameters) tuple, pre-create a working file at `.harness/DeepDiveAgent/<agent-name>.md`. Embed the deep-dive contract per `stellantis-subagent-doc-deep-dive`. The contract carries the `target_doc.local_path` (which must already exist in `.harness/downloads/`; if not, note this in the dispatch working file as an anomaly), the gap parameter list with their Round-1 verdicts, and the trim package map.

## Round 2 — doc-deep-dive concurrency

Same concurrency rules as Round 1 — max 3 deep-dive subagents at once. Per-partition ceiling: 15 minutes; a deep-dive that produces no envelope in that window is failed and re-spawned once.

## Round-2 internal consolidation (dispatch subagent consolidation pass B)

For each Round-2 subagent that signals ready:

1. Read and validate the deep-dive envelope.
2. **Merge with Round-1 records.** For each gap parameter in the deep-dive's records, apply the merge logic below against the internally accumulated Round-1 record for that parameter.
3. Stage `warnings[]` and `advisories[]`.
4. Move the working file to `.harness/Archive/<agent-name>.md`.

### Merge logic — Round 1 + Round 2 per parameter

The dispatch subagent is the only agent permitted to merge across rounds. The merge is deterministic:

| Round 1 | Round 2 | Result |
| :--- | :--- | :--- |
| Rule 4 (silent-all) | Rule 1 / Rule 5 (single-source) | Replace with Round-2 record. `confidence = single-source`. |
| Rule 4 | Rule 3 / Rule 4 | Keep Round 1. Append a note that the deep-dive also found no clear evidence. |
| Rule 3 (vague) | Rule 1 / Rule 5 (single-source) | Replace with Round-2 record. `confidence = single-source`. |
| Rule 3 | Rule 3 | Keep Round 1; append the deep-dive vague block. |
| Rule 1 (single-source) | Rule 1 (same level, different `source_type`) | Promote: Rule 1, `confidence = consensus`. Add the deep-dive's traceability block. |
| Rule 1 (single-source) | Rule 1 (different level) | Escalate: Rule 2a (`Yes, Conflict, Empty`). Both blocks attached. |
| Rule 1 (single-source) | Rule 5 (absent) | Escalate: Rule 2b (`Disputed, Conflict, Empty`). Both blocks attached. |
| Rule 5 (single-source) | Rule 1 | Escalate: Rule 2b. |
| Rule 5 (single-source) | Rule 5 (different `source_type`) | Promote: Rule 5, `confidence = consensus`. |
| Anything | Rule 4 / Rule 3 from the deep-dive | Keep the higher-strength side. |

The merge re-runs the consensus check from `stellantis-decision-rules`: any record whose merged traceability blocks span ≥ 2 distinct `source_type` categories with the same stance is `confidence = consensus`; otherwise `single-source`.

## Round-2 termination

Round 2 ends when every spawned deep-dive subagent has finished internal consolidation. The dispatch subagent does **not** start a Round 3.

## Build and emit dispatch envelope

After all R1 and R2 subagents are archived, the dispatch subagent writes the final dispatch envelope to its working file:

```
<!-- DISPATCH RESULT ENVELOPE -->
{
  "status": "dispatch-ready",
  "category_record_groups": [
    {
      "category": "<category-name>",
      "records": [ <per-parameter record objects> ],
      "warnings": [ <category-level warning strings> ]
    }
  ],
  "docs_metadata_updates": [ <all staged patches from all R1+R2 subagents> ],
  "document_promise_board": [ <aggregated doc promise entries> ],
  "partition_summaries": [ <partition summary strings> ],
  "gap_parameters": {
    "list": [ <gap param ids> ],
    "r2_target_assignments": [ <assignment objects> ],
    "gap_params_without_r2_target": [ <param ids> ],
    "r2_cap_reached": true | false,
    "generated_at": "<ISO-8601>"
  },
  "advisories": {
    "undefined_tier": [ <advisory objects> ],
    "out_of_list": [ <advisory objects> ]
  },
  "run_warnings": [ <run-level warning strings> ],
  "confidence_distribution": {
    "consensus": <n>,
    "single_source": <n>,
    "r2_upgrades": <n>,
    "rule4_remaining": <n>
  },
  "round_stats": {
    "r1_partitions": <n>,
    "r2_deep_dives": <n>,
    "r2_skipped_cap": <n>
  }
}
<!-- END DISPATCH RESULT ENVELOPE -->
```

## Dispatch subagent success criteria

- Every parameter in the frozen CSV has a consolidated record in `category_record_groups[]`.
- Every Round-1 subagent ends in `SubagentStatus = archived` or `SubagentStatus = failed` (with dispatch-written Rule-4 fallback records).
- Every Round-2 deep-dive subagent ends in `SubagentStatus = archived` or `SubagentStatus = failed`. Failed deep-dives leave the Round-1 verdict untouched.
- Every Rule 4 record carries `inverse_retrieval_attempted = true`.
- Every Rule 1 / Rule 5 record carries `confidence ∈ {consensus, single-source}`.
- `document_promise_board` exists and was used to drive Round-2 doc selection.

## Dispatch subagent failure handling

| Failure | Response |
| :--- | :--- |
| R1/R2 subagent envelope fails validation | Mark `failed`, re-spawn **once** with the same contract. |
| Second spawn also fails | Dispatch writes Rule-4 records for every parameter in the partition; add a run-level warning. |
| Subagent touches files outside its allowlist | Mark `failed`; re-spawn once; log in dispatch working file scratch. |
| Subagent exceeds per-partition ceiling | Intervene as described in *Monitoring*; re-spawn once. |
| A required contract field missing | Set `"status": "failed"` with reason in envelope; return control to lead. |
