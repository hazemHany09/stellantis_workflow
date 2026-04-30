---
name: stellantis-lead-agent-subagent-orchestration
description: Use when ingestion verification has passed and parameter classification must begin — lead agent, W1 stage 6. Lead context only.
loader: lead
stage: W1-stage-6, W1-stage-6b
requires:
  - stellantis-domain-context
  - stellantis-decision-rules
  - stellantis-output-contract
  - stellantis-failure-handling
  - stellantis-run-workspace
  - stellantis-subagent-classification-loop
  - stellantis-subagent-doc-deep-dive
provides:
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
- `stellantis-subagent-classification-loop` — Round-1 search-mode subagent, spawned per partition in stage 6a.
- `stellantis-subagent-doc-deep-dive` — Round-2 read-only-on-single-document subagent, spawned per (target-doc × gap-parameters) tuple in stage 6b.

Receives KB dataset from `stellantis-source-download-and-ingest`; orchestrates subagent partitions across two rounds.

---

# Sub-skill — Lead orchestration of subagents (two-round flow)

## Overview

Drives the two-round subagent architecture that produces classification records for the deliverable.

**Round 1 — search-mode (broad recall).** Partition the frozen `params.csv` into per-category batches; spawn a Round-1 search-mode subagent per partition (`stellantis-subagent-classification-loop`); each runs per-parameter KB retrieval, applies the decision rules, and reports records + a `document_promise_scores[]` table + a partition summary.

**Lead consolidation pass A.** Consolidate all Round-1 envelopes. Aggregate `document_promise_scores[]` across partitions into a run-level promise board. Identify gap parameters: anything finalised at Rule 3, Rule 4, or Rule 1 / Rule 5 with `confidence = single-source` where a corroborating source-type may exist in the KB. Map each gap parameter to the highest-promise document(s) most likely to resolve it.

**Round 2 — doc-deep-dive (high precision).** For each chosen (target document × gap-parameters) tuple, spawn a Round-2 deep-dive subagent (`stellantis-subagent-doc-deep-dive`). Each Round-2 subagent reads exactly that one Markdown document end-to-end, no search, and emits records constrained to its evidence.

**Lead consolidation pass B.** Merge Round-2 records back with Round-1 records. Promote single-source verdicts to consensus where the merge produces ≥ 2 distinct `source_type` categories agreeing. Apply consensus invariants from `stellantis-decision-rules`. Finalise category STATE files and the deliverable.

Concurrency cap (max 3 concurrent subagents) applies separately within each round; the rounds themselves are serial — Round 2 cannot start before Round 1 has fully consolidated.

## When this runs

W1 stage 6 (and 6a, 6b). After ingestion verification has passed.

## Inputs

| Input                            | Source                         |
| :------------------------------- | :----------------------------- |
| Frozen `params.csv`              | `runs/<run-id>/params.csv`     |
| `kb_dataset_id`                  | Main STATE.md                  |
| `car_identity` + `resolved_trim` | Main STATE.md                  |

## Tools allowed (for the lead during this phase)

* The host platform's subagent-spawn primitive (whatever the runtime exposes for launching child agents). The skill is platform-agnostic; substitute the equivalent primitive of the deployment system.
* File read/write on the run workspace.
* `get_metadata_summary` (read-only, for sanity check after consolidation).
* `batch_update_doc_metadata` (applied on staged patches from subagent envelopes).

## Round 1 — search-mode partitioning

The frozen `.harness/params.csv` is the authoritative source for which parameters to classify. It may contain all categories from the original reference sheet, or only a filtered subset if the client provided `requested_categories`. Do **not** assume a fixed number of parameters (e.g. 160) — use whatever the frozen CSV actually contains.

1. Enumerate categories from the frozen CSV — the distinct non-empty values of `Domain / Category` rows that are **not** category banners or `Category Overview` rows.
2. For each category, count parameters.
3. If count ≤ 15, make one partition named `<category-slug>-1`.
4. If count > 15, split into consecutive groups of ≤ 15 parameters each (preserve CSV row order). Name partitions `<category-slug>-1`, `<category-slug>-2`, ….
5. The complete partition list is the subagent roster. Record it in the main STATE.md roster table.

## Contract construction

For each partition, construct a contract per the lead-to-subagent-contract specification and validate against the subagent-contract schema.

1. Pre-create the working file at `.harness/SubAgent/<agent-name>.md` using the standard template.
2. Embed the contract JSON in the `<!-- AUTOGENERATED BY LEAD -->` block at the top of the file.
3. Pass the absolute path of the working file to the subagent on spawn as the first argument. The subagent's name is the filename stem.

## Concurrency

* **Max 3 subagents running concurrently.** Queue the rest.
* A subagent slot is freed once the lead has consolidated its envelope (not merely when the subagent has returned control). This keeps consolidation serial.
* If a subagent takes longer than a configurable per-partition ceiling (default 30 minutes), the lead may intervene: inspect the working file; if no progress in the last 10 minutes, mark `SubagentStatus = failed` and re-spawn once.

## Monitoring

The lead periodically (not busy-loop) inspects each running subagent's working file for:

* Presence of the `<!-- RESULT ENVELOPE -->` block → ready for consolidation.
* Monotonically increasing content size → still making progress.
* Presence of `"status": "failed"` inside the envelope → subagent surrendered; re-spawn once.

## Round-1 consolidation (serial)

For each Round-1 subagent that signals ready:

1. Read the working file; extract the result envelope.
2. Validate each record against the intermediate-parameter-record schema and each traceability block against the traceability-block schema.
3. Validate every enum value against the enums reference. Validate that every Rule 4 record carries `inverse_retrieval_attempted = true` (per invariant 11 in `stellantis-decision-rules`); reject and re-spawn if the flag is missing.
4. Apply `docs_metadata_updates[]` by calling `batch_update_doc_metadata` with the staged patches.
5. Merge records into `.harness/Category/<category>.md` — full per-parameter detail rendered via templates. Round-1 records are tagged `round = 1` so the merge with Round-2 records is unambiguous.
6. Promote `warnings[]` into the category STATE's warnings section.
7. Promote `advisories[]` into `.harness/advisories/undefined-tier.md` or `.harness/advisories/out-of-list-findings.md` based on `kind`.
8. Append the subagent's `document_promise_scores[]` to `.harness/document-promise-board.md`, deduplicated by `doc_id`. When the same `doc_id` appears in multiple partitions, **sum** `parameters_evidenced[]` and `parameters_likely_to_be_evidenced[]` (deduped), and take the **maximum** `promise_score` (with the rationale of the highest scorer).
9. Append the subagent's `partition_summary` to `.harness/partition-summaries.md`.
10. Move the working file to `.harness/Archive/<agent-name>.md`. Update `SubagentStatus = archived`.
11. Update main STATE.md summary counts and subagent roster.

## Round-1 → Round-2 hand-off (lead consolidation pass A)

After every Round-1 subagent has finished and consolidated:

**Before spawning any Round-2 subagent, write the following to STATE.md:**
1. Write the empty **Round 2 subagent roster** section (header + empty table). This must exist before the first R2 spawn.
2. Write the **Gap parameters + R2 target assignments** JSON block (see below). This is write-once — never edit after this point.

### Identify gap parameters

Walk every Round-1 record. Mark a parameter as a **gap parameter** when any of the following hold:

* The verdict is **Rule 4** (`No Information Found, Unable to Decide, Empty`). Even with `inverse_retrieval_attempted = true`, a Rule 4 may still be recoverable if a high-promise document discusses the feature in detail outside what vector retrieval surfaced.
* The verdict is **Rule 3** (`Yes, Unable to Decide, Empty`). The feature is mentioned but no clear source emerged; a deep-dive may surface the level.
* The verdict is **Rule 1 or Rule 5 with `confidence = single-source`**. A second `source_type` corroboration from a different document in the KB would promote the record to `consensus`.

Do **not** mark Rule 2a / Rule 2b records as gap parameters — conflicts are resolved by human adjudication, not by another retrieval pass.

### Pick target documents

For each gap parameter, walk the document-promise board and select the highest-`promise_score` documents whose `parameters_likely_to_be_evidenced[]` includes that parameter. Greedy set-cover:

1. Pick the document with the highest `promise_score`. Assign all gap parameters listed in its `parameters_likely_to_be_evidenced[]` to that document's deep-dive.
2. Subtract those parameters from the gap set. Repeat with the next-highest-promise document for the remaining gaps.
3. Cap the round at **at most 6 deep-dive subagents per run** (configurable). Once the cap is reached, remaining gap parameters are accepted at their Round-1 verdict; record this in the run-level warnings.

When a gap parameter appears in `parameters_likely_to_be_evidenced[]` of zero documents, no deep-dive is spawned for it — Round 1's verdict stands. This is the right outcome: vector retrieval already saw nothing in the KB, and no Round-1 subagent thought any document was promising for it. Spending a deep-dive there would not help.

### Pre-create Round-2 contracts

Write the **Gap parameters + R2 target assignments** JSON block into STATE.md now (if not already written above). The block must include all `gap_parameters[]`, all `r2_target_assignments[]`, any `gap_params_without_r2_target[]`, `r2_cap_reached`, and `generated_at`.

For each (target_doc × gap_parameters) tuple, pre-create a working file at `.harness/DeepDiveAgent/<agent-name>.md`. Embed the deep-dive contract per `stellantis-subagent-doc-deep-dive`. The contract carries the `target_doc.local_path` (which must already exist in `.harness/downloads/`; if not, the lead calls `ragflow_download` once before spawning), the gap parameter list with their Round-1 verdicts, and the trim package map.

## Round 2 — doc-deep-dive concurrency

Same concurrency rules as Round 1 — max 3 deep-dive subagents at once. Subagents only have `read_file` access, so failure modes are simpler (no retrieval timeouts, no upload failures). The per-partition ceiling is shorter (default 15 minutes); a deep-dive that produces no envelope in that window is failed and re-spawned once.

## Round-2 consolidation (lead consolidation pass B)

For each Round-2 subagent that signals ready:

1. Read and validate the deep-dive envelope as in Round 1, but only the records for `gap_parameters` are emitted — there is no full-partition coverage from this subagent.
2. **Merge with Round-1 records.** For each gap parameter in the deep-dive's records, locate the Round-1 record for that parameter in `.harness/Category/<category>.md`. Apply the merge logic below.
3. Promote `warnings[]` and `advisories[]` as in Round 1.
4. Move the working file to `.harness/Archive/<agent-name>.md`.

### Merge logic — Round 1 + Round 2 per parameter

The lead is the only agent permitted to merge across rounds. The merge is deterministic:

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

## Round-2 termination criteria

Round 2 ends when every spawned deep-dive subagent has finished consolidating. The lead does **not** start a Round 3 in the normal pipeline — additional cycles belong to gap-fill workflows, which are scoped separately. Persisting unresolved parameters surface as warnings in the deliverable footer.

## Consolidation (legacy single-round path — deprecated)

For each subagent that signals ready:

1. Read the working file; extract the result envelope.
2. Validate each record against the intermediate-parameter-record schema and each traceability block against the traceability-block schema.
3. Validate every enum value against the enums reference.
4. Apply `docs_metadata_updates[]` by calling `batch_update_doc_metadata` with the staged patches.
5. Merge records into `.harness/Category/<category>.md` — full per-parameter detail rendered via templates.
6. Promote `warnings[]` into the category STATE's warnings section.
7. Promote `advisories[]` into `.harness/advisories/undefined-tier.md` or `.harness/advisories/out-of-list-findings.md` based on `kind`.
8. Move the working file to `.harness/Archive/<agent-name>.md`. Update `SubagentStatus = archived`.
9. Update main STATE.md summary counts and subagent roster.

This single-round path is preserved as a reference for older runs; new runs follow the two-round flow above.

## Failure handling

| Failure                                            | Response                                                                 |
| :------------------------------------------------- | :----------------------------------------------------------------------- |
| Subagent envelope fails validation                 | Mark `failed`, re-spawn **once** with the same contract.                 |
| Second spawn also fails                            | Lead writes Rule-4 records (`No Information Found, Unable to Decide, Empty`) for every parameter in the partition; add a run-level warning in the deliverable footer. |
| Subagent touches files outside its allowlist       | Mark `failed`; re-spawn once; log a framework-invariant violation in `.harness/event-log.md`. |
| Subagent exceeds per-partition ceiling             | Intervene as described in *Monitoring* above.                            |

## Success criteria

* Every parameter in the frozen CSV has a consolidated record in the corresponding `.harness/Category/<category>.md`.
* Every Round-1 subagent ends in `SubagentStatus = archived` or `SubagentStatus = failed` (with lead-written Rule-4 fallback records).
* Every Round-2 deep-dive subagent ends in `SubagentStatus = archived` or `SubagentStatus = failed`. Failed deep-dives leave the Round-1 verdict untouched.
* Every Rule 4 record in the consolidated output carries `inverse_retrieval_attempted = true`.
* Every Rule 1 / Rule 5 record carries `confidence ∈ {consensus, single-source}`. The mix of consensus vs single-source is visible in the run summary.
* `.harness/document-promise-board.md` exists and was used to drive the Round-2 doc selection. The chosen target documents are recorded with their promise rationale for audit.
* Main STATE.md counts sum consistently across categories.

## Final STATE.md update (mandatory)

After all subagents are archived and all records consolidated (both rounds), the lead **must** write a final update to `.harness/STATE.md` before proceeding to deliverable emission:

1. Set `RunStage = classification-complete`.
2. **[T1] Summary counts** — final total parameters, breakdown by `Presence` value.
3. **[T2] Confidence distribution** — final breakdown: consensus count, single-source count, Round 2 upgrades (single-source → consensus), Round 2 verdict changes (rule promoted), Rule 4 records remaining after Round 2.
4. **[T2] Warnings and advisories summary** — final counts: run-level warnings, category-level warnings, Rule 2a conflicts, Rule 2b conflicts, undefined-tier advisories, out-of-list findings, R2 cap reached count.
5. Record `classification_completed_at` timestamp in the Header.
6. Set `completed_at` in Stage timing telemetry for `classification-round-2` (or `classification-round-1` if no R2 ran).
7. Append event-log line: `"All subagents consolidated — N parameters classified (R1: N, R2 upgrades: N). Proceeding to deliverable emission."`.

This step is **not optional**. Skipping it leaves STATE.md in a stale intermediate state and makes the run non-auditable. The lead must not proceed to Stage 7 (emit deliverable) until this update is written.
