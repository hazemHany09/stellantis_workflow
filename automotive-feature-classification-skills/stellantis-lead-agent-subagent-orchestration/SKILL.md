---
name: stellantis-lead-agent-subagent-orchestration
description: Use when ingestion verification has passed and parameter classification must begin — W1 stage 6. The lead is the sole spawner of subagents. The lead reads `params.csv`, partitions parameters by category (max 15 per subagent, never crossing categories), spawns Round-1 classification subagents, consolidates their envelopes, then spawns Round-2 deep-dive subagents for any gap parameters. No subagent ever spawns another subagent. No subagent ever reads `params.csv` — the lead embeds the relevant parameter slice in each subagent's contract.
loader: lead
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
  - partition-manifests
  - round1-document-promise-aggregate
  - round2-deep-dive-manifests
  - consolidated-category-STATE-files
  - archived-subagent-files
tools:
  - batch_update_doc_metadata
  - get_metadata_summary
---

## Dependencies

Foundational:
- `stellantis-domain-context`, `stellantis-decision-rules`, `stellantis-output-contract`, `stellantis-failure-handling`, `stellantis-run-workspace`.

Hard:
- `stellantis-subagent-classification-loop` — Round-1 search-mode classification subagent, spawned by the **lead** per partition in stage 6a.
- `stellantis-subagent-doc-deep-dive` — Round-2 read-only-on-single-document subagent, spawned by the **lead** per (target-doc × gap-parameters) tuple in stage 6b.

Receives the populated KB dataset from `stellantis-source-download-and-ingest`.

---

# Sub-skill — Stage 6 Orchestration (lead-only spawner, two rounds)

## Hard architectural rules

1. **The lead is the sole spawner.** No subagent at any stage may spawn another subagent. Violations are framework bugs.
2. **No subagent reads `params.csv`.** The lead reads `.harness/params.csv` and embeds the exact parameter slice each subagent needs into that subagent's contract.
3. **One subagent per category, max 15 parameters per subagent.** A subagent's partition is always drawn from a single `Domain / Category` value in `params.csv`. A category with > 15 parameters is split into multiple consecutive partitions, each ≤ 15 parameters, in CSV row order. No subagent ever holds parameters from two different categories.
4. **Two stages where the lead spawns multiple subagents:**
   - W1 stage 4 — **download + ingest subagents** (one per approved URL). Governed by `stellantis-source-download-and-ingest`.
   - W1 stage 6 — **classification subagents** (Round 1 + Round 2). Governed by this skill.
5. **Concurrency cap: max 3 classification subagents running concurrently.** Lead queues the rest. Round 2 cannot start until Round 1 fully consolidates.
6. **Stage 6 may only begin after the user has issued the classification go-ahead.** Ingestion completing is **not** sufficient. The lead must have paused with `PauseReason = awaiting-classification-go-ahead` (per `stellantis-source-download-and-ingest`), posted the ingestion + planned-partition summary, and received the user's explicit `go` reply. Stage 6 entry verifies `RunStatus = active` AND the most recent event-log entry includes `"Classification go-ahead received"`; if not, abort entry and pause again.

---

## Overview

At stage 6 the lead drives a two-round subagent architecture:

**Round 1 — search-mode (broad recall).** The lead reads the frozen `.harness/params.csv`, partitions parameters by category, and spawns a Round-1 classification subagent (`stellantis-subagent-classification-loop`) per partition. Each subagent runs per-parameter KB retrieval, applies the decision rules, and returns records + a `document_promise_scores[]` table + a partition summary. The lead consolidates each envelope into `.harness/Category/<category>.md` as it arrives (serial consolidation).

**Lead consolidation pass A.** After all Round-1 subagents are archived, the lead aggregates the document-promise board, identifies gap parameters, and maps each gap to the highest-promise document(s) most likely to resolve it.

**Round 2 — doc-deep-dive (high precision).** For each (target document × gap-parameters) tuple, the lead spawns a Round-2 deep-dive subagent (`stellantis-subagent-doc-deep-dive`). Each reads exactly one Markdown document end-to-end and emits records constrained to its evidence.

**Lead consolidation pass B.** The lead merges Round-2 records with Round-1 records, applies the merge logic, updates `.harness/Category/<category>.md`, applies metadata patches, writes the final STATE.md, and proceeds to deliverable emission.

---

# Stage 6 entry — lead actions

## Inputs the lead reads

| Input                            | Source                          |
| :------------------------------- | :------------------------------ |
| Frozen `params.csv`              | `.harness/params.csv`           |
| `kb_dataset_id`                  | `.harness/STATE.md`             |
| `car_identity` + `resolved_trim` + `trim_package_map` | `.harness/STATE.md` |

The lead is the **only** actor permitted to read `.harness/params.csv` during classification.

## Step 0 — Verify classification go-ahead received

Before any partition or spawn work, the lead checks `.harness/STATE.md`:
- `RunStage = classification`,
- `RunStatus = active`,
- the event log contains `"Classification go-ahead received — spawning R1 subagents"` more recently than any pause line.

If any check fails, do **not** proceed. Re-pause with `PauseReason = awaiting-classification-go-ahead` and re-post the ingestion + planned-partition summary per `stellantis-source-download-and-ingest`. Stage 6 spawning is forbidden until the user has explicitly said go.

## Step 1 — Partition `params.csv` by category

(If the planned partition list was already written under the **R1 subagent roster** section during the pre-classification pause, the lead re-validates it here; otherwise it computes it now.)

1. Enumerate categories from the frozen CSV — the distinct non-empty values of the `Domain / Category` column on rows that are **not** category banners or `Category Overview` rows.
2. For each category, count parameters.
3. If count ≤ 15, make one partition named `<category-slug>-1`.
4. If count > 15, split into consecutive groups of ≤ 15 parameters each (preserve CSV row order). Name partitions `<category-slug>-1`, `<category-slug>-2`, ….
5. Write the partition list into `.harness/STATE.md` under the **R1 subagent roster** section.

**Hard constraint:** every partition is single-category. A subagent's contract never lists parameters whose `Domain / Category` values differ.

## Step 2 — Pre-seed Round-1 working files

For each partition:

1. Create `.harness/SubAgent/<agent-name>.md` from the standard subagent working-file template.
2. Embed the lead-to-subagent contract (see `lead-to-subagent-contract`) at the top of the file. The contract carries the exact parameter slice (the `parameters[]` array, fully self-contained — internal name, optimised name, description, schema_type, criteria) so the subagent **never** needs to read `params.csv`.
3. The subagent's name = filename stem (`<category-slug>-<partition-index>`).
4. Pass the absolute path of the working file to the subagent on spawn as the first argument.

## Step 3 — Spawn Round-1 subagents (lead is the spawner)

1. Spawn up to 3 Round-1 subagents concurrently. Queue the rest.
2. For each spawn, append an event-log line in `.harness/STATE.md`: `"R1 subagent <agent-name> spawned for category <category>"`.
3. Update the R1 subagent roster row: `SubagentStatus = running`, `spawned_at = <ISO-8601>`.

The lead never spawns a Round-1 classification subagent that crosses two categories. The lead never asks any subagent to perform spawning.

## Step 4 — Monitor Round-1 subagents

Periodically (not busy-loop) inspect each running subagent's working file for:

- Presence of `<!-- RESULT ENVELOPE -->` block → ready for consolidation.
- Monotonically increasing content size → still making progress.
- Presence of `"status": "failed"` inside the envelope → subagent surrendered; re-spawn once with the same contract.

A subagent slot is freed once the lead has consolidated its envelope (see Step 5), not merely when the subagent returns control. This keeps consolidation **serial** even when subagents finish concurrently.

If a subagent exceeds 30 minutes with no progress, mark `SubagentStatus = failed`, archive its working file, and re-spawn once with the same contract.

## Step 5 — Round-1 consolidation (serial)

For each Round-1 subagent that signals ready:

1. Read the working file; extract the result envelope.
2. Validate every record against the intermediate-parameter-record schema and every traceability block against the traceability-block schema.
3. Validate every enum value against `enums-reference`. Validate that every Rule 4 record carries `inverse_retrieval_attempted = true` (per invariant 11 in `stellantis-decision-rules`); reject and re-spawn once if missing.
4. Apply `docs_metadata_updates[]` immediately via `batch_update_doc_metadata`.
5. Merge the records into `.harness/Category/<category>.md` (one category file is shared across partitions of that category).
6. Promote category-level `warnings[]` into the same category file.
7. Promote `advisories[]` into `.harness/advisories/undefined-tier.md` or `.harness/advisories/out-of-list-findings.md` based on `kind`.
8. Append the subagent's `document_promise_scores[]` to a running `.harness/document-promise-board.md`, deduplicated by `doc_id`. When the same `doc_id` appears in multiple partitions, **sum** `parameters_evidenced[]` and `parameters_likely_to_be_evidenced[]` (deduped), and take the **maximum** `promise_score`.
9. Append the subagent's `partition_summary` to `.harness/partition-summaries.md`.
10. Move `.harness/SubAgent/<agent-name>.md` to `.harness/Archive/<agent-name>.md`. Update the roster row to `SubagentStatus = archived`.
11. Update STATE.md summary counts.
12. Spawn the next queued Round-1 subagent (if any) to refill the concurrency slot.

**Serial constraint:** one consolidation completes before the next begins, even when multiple subagents finish concurrently.

## Step 6 — Round-1 → Round-2 hand-off (lead consolidation pass A)

After every Round-1 subagent has finished and consolidated:

**Before spawning any Round-2 subagent, record in `.harness/STATE.md`:**
1. The **Round 2 subagent roster** (header + empty table).
2. The **Gap parameters + R2 target assignments** JSON block (write-once — never edited after this point).

### Identify gap parameters

Walk every Round-1 record. Mark a parameter as a **gap parameter** when any of the following hold:

- The verdict is **Rule 4** (`No Information Found, Unable to Decide, Empty`). Even with `inverse_retrieval_attempted = true`, a Rule 4 may still be recoverable if a high-promise document discusses the feature in detail outside what vector retrieval surfaced.
- The verdict is **Rule 3** (`Yes, Unable to Decide, Empty`). The feature is mentioned but no clear source emerged.
- The verdict is **Rule 1 or Rule 5 with `confidence = single-source`**. A corroborating source-type from a different document would promote the record to `consensus`.

Do **not** mark Rule 2a / Rule 2b records as gap parameters — conflicts are resolved by human adjudication.

### Pick target documents

For each gap parameter, walk `.harness/document-promise-board.md` and select the highest-`promise_score` documents whose `parameters_likely_to_be_evidenced[]` includes that parameter. Greedy set-cover:

1. Pick the document with the highest `promise_score`. Assign all gap parameters listed in its `parameters_likely_to_be_evidenced[]` to that document's deep-dive.
2. Subtract those parameters from the gap set. Repeat with the next-highest-promise document for the remaining gaps.
3. Cap the round at **at most 6 deep-dive subagents per run** (configurable). Once the cap is reached, remaining gap parameters are accepted at their Round-1 verdict; record this as a run-level warning.

When a gap parameter appears in zero documents' `parameters_likely_to_be_evidenced[]`, no deep-dive is spawned for it — Round 1's verdict stands.

## Step 7 — Pre-seed and spawn Round-2 deep-dive subagents

For each (target_doc × gap_parameters) tuple:

1. Pre-create a working file at `.harness/DeepDiveAgent/<agent-name>.md` (e.g. `infot-deepdive-1`).
2. Embed the deep-dive contract (see `stellantis-subagent-doc-deep-dive`). The contract carries `target_doc.local_path` (which must already exist in `.harness/downloads/`; if not, log this as an anomaly in STATE.md and skip), the gap parameter list with their Round-1 verdicts, the trim package map, and `tool_allowlist: ["read_file"]`.
3. The deep-dive subagent's gap-parameter list is also single-category-bound: a deep-dive never receives parameters from two different categories. (If the same target document covers gaps from multiple categories, spawn one deep-dive per category.)
4. Spawn the subagent (lead is the spawner). Max 3 concurrent. Per-deep-dive ceiling: 15 minutes.

The lead never spawns a Round-2 deep-dive that asks the subagent to do KB retrieval, web search, or scraping. Deep-dives use `read_file` only, against a single local Markdown.

## Step 8 — Round-2 consolidation (lead consolidation pass B)

For each Round-2 subagent that signals ready:

1. Read and validate the deep-dive envelope.
2. **Merge with Round-1 records.** For each gap parameter in the deep-dive's records, apply the merge logic below against the Round-1 record already written into `.harness/Category/<category>.md`.
3. Stage `warnings[]` and `advisories[]`.
4. Move the working file to `.harness/Archive/<agent-name>.md`.

### Merge logic — Round 1 + Round 2 per parameter

The lead is the only actor permitted to merge across rounds. The merge is deterministic:

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

## Step 9 — Round-2 termination

Round 2 ends when every spawned deep-dive subagent has finished consolidation. The lead does **not** start a Round 3.

## Step 10 — Final STATE.md update (mandatory)

After every classification subagent (R1 + R2) is archived and all records consolidated:

1. Set `RunStage = classification-complete`.
2. **[T1] Summary counts** — final total parameters, breakdown by `Presence` value.
3. **[T2] Confidence distribution** — consensus count, single-source count, Round 2 upgrades, Rule 4 records remaining after Round 2.
4. **[T2] Warnings and advisories summary** — counts: run-level warnings, category-level warnings, Rule 2a/2b conflicts, undefined-tier advisories, out-of-list findings, R2 cap reached.
5. Record `classification_completed_at` timestamp in the Header.
6. Set `completed_at` in Stage timing telemetry for `classification-round-2` (or `classification-round-1` if no R2 ran).
7. Append event-log line: `"Classification complete — N parameters classified. Proceeding to deliverable emission."`.

This step is **not optional**. Skipping it leaves STATE.md in a stale intermediate state.

---

## Tools allowed (lead at stage 6)

| Tool | Purpose |
| :--- | :------ |
| File read/write on `.harness/STATE.md`, `.harness/params.csv`, `.harness/Category/*.md`, `.harness/document-promise-board.md`, `.harness/partition-summaries.md`, `.harness/advisories/*.md` | All consolidation and bookkeeping |
| Pre-create `.harness/SubAgent/<agent-name>.md` and `.harness/DeepDiveAgent/<agent-name>.md` working files | Subagent contract seeding |
| Host platform's subagent-spawn primitive | Spawning R1 + R2 subagents |
| `batch_update_doc_metadata` | Apply staged metadata patches from R1/R2 envelopes |
| `get_metadata_summary` | Sanity-check after metadata patches |

The lead does **not** call `retrieval`, `retrieve_knowledge_graph`, `list_chunks`, `ragflow_download` itself at stage 6 — KB retrieval is the classification subagents' job.

## Subagent prohibitions (enforced)

- Subagents **never** spawn other subagents.
- Subagents **never** read `params.csv`. Their parameter list comes from the contract.
- Subagents **never** read or write `.harness/STATE.md`.
- Subagents **never** read or write `.harness/Category/*.md`.
- Subagents **never** read another subagent's working file.
- Round-1 classification subagents use only KB retrieval tools (no web search, no `fetch_url`, no `read_file` on docs in `.harness/downloads/`).
- Round-2 deep-dive subagents use only `read_file` on the single `target_doc.local_path` in their contract — no KB tools, no web search.
- Download + ingest subagents (stage 4) use only `fetch_url` (or `fetch_webpage` for explicit retry mode), `ragflow_upload`, and `os.stat`-equivalent size lookup. They **never** read the content of the file they downloaded — they report only the size to the lead.

Violations of any of these rules cause the lead to mark the subagent `SubagentStatus = failed` and re-spawn once.

---

## Lead failure handling

| Failure | Response |
| :--- | :--- |
| R1/R2 subagent envelope fails validation | Mark `failed`, re-spawn once with the same contract. |
| Second spawn also fails | Lead writes Rule-4 fallback records for every parameter in the partition; add a run-level warning in the deliverable footer. |
| Subagent exceeds per-partition ceiling (R1: 30 min, R2: 15 min) | Mark `failed`, archive working file, re-spawn once. |
| A subagent attempts to spawn another subagent | Mark `failed`; write a run-level warning; re-spawn once with the same contract. |
| A subagent attempts to read `params.csv` or another subagent's working file | Mark `failed`; write a run-level warning; re-spawn once. |
| `target_doc.local_path` missing from `.harness/downloads/` for an R2 spawn | Skip the deep-dive; record an anomaly in STATE.md; the affected gaps remain at their Round-1 verdicts. |

## Success criteria

- Every parameter in the frozen CSV has exactly one consolidated record in some `.harness/Category/<category>.md` file.
- Every Round-1 subagent ends in `SubagentStatus = archived` or `SubagentStatus = failed` (with lead-written Rule-4 fallback records).
- Every Round-2 deep-dive subagent ends in `SubagentStatus = archived` or `SubagentStatus = failed`. Failed deep-dives leave the Round-1 verdict untouched.
- No subagent spawned another subagent.
- No subagent read `.harness/params.csv`.
- Every Rule 4 record carries `inverse_retrieval_attempted = true`.
- Every Rule 1 / Rule 5 record carries `confidence ∈ {consensus, single-source}`.
- `.harness/document-promise-board.md` exists and was used to drive Round-2 doc selection.
