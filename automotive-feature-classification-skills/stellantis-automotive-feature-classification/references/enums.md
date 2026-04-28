# Enums — Single Source of Truth

This file is the **only** place that lists allowed values for any categorical field used inside the skill. Every other file in this skill — templates, contracts, workflow, sub-skills, SKILL.md — must link to this file rather than repeat the value set. If a value is not listed here, it is not a valid value.

---

## `Presence`

Per-parameter answer to *"does the car support this feature?"*.

| Value                  | When used                                                                      |
| :--------------------- | :----------------------------------------------------------------------------- |
| `Yes`                  | Rule 1 (Success-present), Rule 2a (level conflict), Rule 3 (vague-only)        |
| `No`                   | Rule 5 (Success-absent, explicit negative agreement)                           |
| `Disputed`             | Rule 2b only — clear sources disagree on whether feature exists                |
| `No Information Found` | Rule 4 only — no active sources at all (silent-all)                            |

`No Information Found` is **retired as a `Status` value**; only valid as a `Presence` value.

---

## `Status`

Outcome of the *assignment process* per parameter.

| Value              | When used                                               |
| :----------------- | :------------------------------------------------------ |
| `Success`          | Rule 1 (present + level) or Rule 5 (absent)             |
| `Conflict`         | Rule 2a (level conflict) or Rule 2b (presence conflict) |
| `Unable to Decide` | Rule 3 (vague-only) or Rule 4 (silent-all)              |

---

## `Classification`

The assigned level.

| Value    | When used                                     |
| :------- | :-------------------------------------------- |
| `High`   | Rule 1 + `High` in applicable set             |
| `Medium` | Rule 1 + `Medium` in applicable set           |
| `Basic`  | Rule 1 + `Basic` in applicable set            |
| `Empty`  | All non-Rule-1 records; also Rule 1 is impossible when applicable set would disallow the only-evidenced tier |

Classification must always be a member of the parameter's applicable level set. No nearest-level snapping.

---

## `SchemaType`

Compact notation for the applicable level set of a parameter. Populated from the non-empty criterion columns in `artifacts/params.csv`.

| Value   | Meaning                       |
| :------ | :---------------------------- |
| `B/M/H` | Basic, Medium, and High all defined |
| `B/H`   | Basic and High only           |
| `M/H`   | Medium and High only          |
| `B/M`   | Basic and Medium only         |
| `H`     | High only                     |
| `M`     | Medium only                   |
| `B`     | Basic only                    |

---

## `DecisionRule`

The rule that produced a given record. Populated for audit + advisory purposes.

| Value     | Description                                                                                                                   |
| :-------- | :---------------------------------------------------------------------------------------------------------------------------- |
| `Rule-1`  | All clear active sources agree on a valid level → `Yes / Success / <level>`                                                   |
| `Rule-2a` | ≥2 clear active sources agree feature is present but disagree on level → `Yes / Conflict / Empty`                             |
| `Rule-2b` | Clear active sources disagree on whether feature exists → `Disputed / Conflict / Empty`                                       |
| `Rule-3`  | Active sources exist but none are clear → `Yes / Unable to Decide / Empty`                                                    |
| `Rule-4`  | No active sources at all (silent-all) → `No Information Found / Unable to Decide / Empty`                                     |
| `Rule-5`  | All clear active sources agree feature is absent → `No / Success / Empty`                                                     |

---

## `SourceLifecycle`

Per-URL state.

| Value                    | Meaning                                                               |
| :----------------------- | :-------------------------------------------------------------------- |
| `Proposed`               | Newly identified by search or supplied by client                      |
| `Canonicalised`          | URL canonicalised (lowercase host, strip fragment, strip tracking)    |
| `Flagged`                | Sent to the user for free-text approval                               |
| `PendingApproval`        | Waiting for client reply                                              |
| `Approved`               | Client approved (in reply)                                            |
| `Rejected`               | Client rejected (in reply)                                            |
| `Downloaded`             | File retrieved locally                                                |
| `Ingesting`              | Uploaded to KB; parsing in progress                                   |
| `Ingested`               | KB reports parsing complete                                           |
| `Queryable`              | Retrieval may target this document                                    |
| `Dropped_Paywall`        | Silently dropped pre-approval                                         |
| `Dropped_Duplicate`      | Canonicalised form already seen                                       |
| `Dropped_DownloadFail`   | 3 retries exhausted                                                   |
| `Dropped_IngestTimeout`  | 60-minute ingestion ceiling exceeded                                  |
| `DeletedByAgent`         | `delete_docs` called post-ingestion (duplicate content, off-target)   |

---

## `SourceOrigin`

| Value    | Meaning                                |
| :------- | :------------------------------------- |
| `client` | Mode A — user supplied this URL        |
| `agent`  | Mode B — agent discovered this URL     |

---

## `FailureStage`

Used in the deliverable footer `sources_excluded` list.

| Value        | Meaning                            |
| :----------- | :--------------------------------- |
| `download`   | Retries exhausted before upload    |
| `ingestion`  | Upload succeeded but parsing failed or timed out |

---

## `RunStage`

Coarse-grained progress indicator written into the main `STATE.md`.

| Value                        | Meaning                                                              |
| :--------------------------- | :------------------------------------------------------------------- |
| `parsing`                    | Extracting the 4 required fields from the request                    |
| `car-identity-resolution`    | Resolving ambiguous variants + full-option trim                      |
| `dataset-creation`           | Creating the per-run KB dataset                                      |
| `source-discovery`           | Running Mode-A or Mode-B source discovery                            |
| `awaiting-approval`          | Candidate list posted to user; workflow paused                       |
| `download-and-ingest`        | Downloading approved URLs, uploading to KB, awaiting ingestion completion |
| `awaiting-ingestion`         | Workflow paused until user sends resume signal for ingestion check   |
| `classification`             | Subagents running                                                    |
| `consolidation`              | Lead merging subagent outputs into category + main STATE             |
| `deliverable`                | Emitting `deliverable.json` + `deliverable.csv`                      |
| `complete`                   | Run finished; deliverable emitted                                    |
| `failed`                     | Run aborted                                                          |

---

## `RunStatus`

| Value      | Meaning                                            |
| :--------- | :------------------------------------------------- |
| `active`   | Lead or a subagent is currently executing          |
| `paused`   | Awaiting user input (see `PauseReason`)            |
| `complete` | Deliverable emitted                                |
| `failed`   | Run aborted; STATE.md carries failure details      |

---

## `PauseReason`

Populated only when `RunStatus = paused`.

| Value                                       | Meaning                                                                                   |
| :------------------------------------------ | :---------------------------------------------------------------------------------------- |
| `awaiting-source-approval`                  | Candidate list posted; waiting for free-text approve/reject reply                         |
| `awaiting-client-resume`                    | Ingestion triggered; waiting for client "resume" signal before ingestion verification     |
| `awaiting-ingestion-completion`             | On resume, some docs still ingesting and timeout not yet reached; waiting again           |
| `ingestion-error-client-confirmation-needed` | Not all docs ingested and timeout not yet reached; asked client how to proceed            |
| `missing-required-input`                    | One or more of brand/model/year/market could not be extracted                             |

---

## `SubagentStatus`

Per-subagent state tracked in the main STATE.md roster and in `.harness/SubAgent/<agent-name>.md`.

| Value          | Meaning                                                                      |
| :------------- | :--------------------------------------------------------------------------- |
| `assigned`     | Contract written; subagent not yet spawned                                   |
| `running`      | Subagent active                                                              |
| `consolidated` | Lead has merged the working file into the category STATE                    |
| `archived`     | Working file moved to `.harness/Archive/`                                   |
| `failed`       | Subagent reported failure or exceeded loop budget without producing output  |

---

## `TraceabilityBlockKind`

Internal tag used by subagents and the consolidator to route each block to the correct cardinality rule.

| Value      | When used                                                  |
| :--------- | :--------------------------------------------------------- |
| `clear`    | Source is a clear active source for the parameter          |
| `vague`    | Source is a vague active source for the parameter          |

Silent sources never produce a block.

---

## Invariants for consumers of this file

1. Any string field in any template, contract, or record that corresponds to one of the enums above **must** carry one of the exact listed values — case-sensitive, whitespace-exact.
2. Adding a value requires updating this file **and** `framework-maintenance/impact-checklist.md`.
3. Removing a value requires a migration note in `framework-maintenance/change-log.md`.
