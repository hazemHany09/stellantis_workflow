# Enums — Single Source of Truth

This file is the **only** place that lists allowed values for any categorical field used inside the skill. Every other file in `src/` — templates, contracts, workflow, sub-skills, SKILL.md — must link to this file rather than repeat the value set. If a value is not listed here, it is not a valid value.

Each section cites the business-logic document(s) that justify its membership.

---

## `Presence`

Per-parameter answer to *"does the car support this feature?"*.

| Value                  | When used                                                                      | Source                |
| :--------------------- | :----------------------------------------------------------------------------- | :-------------------- |
| `Yes`                  | Rule 1 (Success-present), Rule 2a (level conflict), Rule 3 (vague-only)        | decision rules |
| `No`                   | Rule 5 (Success-absent, explicit negative agreement)                           | decision rules |
| `Disputed`             | Rule 2b only — clear sources disagree on whether feature exists                | Q-HARN-6 |
| `No Information Found` | Rule 4 only — no active sources at all (silent-all)                            | Q-HARN-9, Q-HARN-10, AS-HARN-A |

`No Information Found` is **retired as a `Status` value**; only valid as a `Presence` value.

---

## `Status`

Outcome of the *assignment process* per parameter.

| Value              | When used                            | Source                |
| :----------------- | :----------------------------------- | :-------------------- |
| `Success`          | Rule 1 (present + level) or Rule 5 (absent) | decision rules |
| `Conflict`         | Rule 2a (level conflict) or Rule 2b (presence conflict) | Q-HARN-3-v2 |
| `Unable to Decide` | Rule 3 (vague-only) or Rule 4 (silent-all) | decision rules |

---

## `Classification`

The assigned level.

| Value    | When used                                     | Source                |
| :------- | :-------------------------------------------- | :-------------------- |
| `High`   | Rule 1 + `High` in applicable set             | parameter definitions |
| `Medium` | Rule 1 + `Medium` in applicable set           | parameter definitions |
| `Basic`  | Rule 1 + `Basic` in applicable set            | parameter definitions |
| `Empty`  | All non-Rule-1 records; also Rule 1 is impossible when applicable set would disallow the only-evidenced tier (see Q-HARN-4, Invariant 7) | harness interface |

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

Source: Q-DEL-4.

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

Source: decision rules.

---

## `SourceLifecycle`

Per-URL state. See `04-sources.md` and `12-workflow-diagram.md` section B.

| Value                    | Meaning                                                               |
| :----------------------- | :-------------------------------------------------------------------- |
| `Proposed`               | Newly identified by search or supplied by client                      |
| `Canonicalised`          | URL canonicalised (lowercase host, strip fragment, strip tracking)    |
| `Flagged`                | Sent to the user for free-text approval                               |
| `PendingApproval`        | Waiting for client reply                                              |
| `Approved`               | Client approved (in reply)                                            |
| `Rejected`               | Client rejected (in reply)                                            |
| `Downloaded`             | File retrieved locally                                                |
| `Ingesting`              | Uploaded to KB; `run_document` triggered; parsing in progress         |
| `Ingested`               | KB reports parsing complete                                           |
| `Queryable`              | Retrieval may target this document                                    |
| `Dropped_Paywall`        | Silently dropped pre-approval (Q-SRC-4)                               |
| `Dropped_Duplicate`      | Canonicalised form already seen                                       |
| `Dropped_DownloadFail`   | 3 retries exhausted (Q-KB-4)                                          |
| `Dropped_IngestTimeout`  | 60-minute ingestion ceiling exceeded (Q-KB-5)                         |
| `DeletedByAgent`         | `delete_docs` called post-ingestion (duplicate content, off-target)   |

---

## `SourceOrigin`

| Value    | Meaning                                |
| :------- | :------------------------------------- |
| `client` | Mode A — user supplied this URL        |
| `agent`  | Mode B — agent discovered this URL     |

Source: AS-SRC-B.

---

## `FailureStage`

Used in the deliverable footer `sources_excluded` list.

| Value        | Meaning                            |
| :----------- | :--------------------------------- |
| `download`   | Retries exhausted before upload    |
| `ingestion`  | Upload succeeded but parsing failed or timed out |

Source: AS-DEL-C.

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
| `download-and-ingest`        | Downloading approved URLs, uploading to KB, triggering `run_document` |
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
2. Adding a value requires updating this file **and** the referenced business-logic document **and** `framework-maintenance/impact-checklist.md`.
3. Removing a value requires a migration note in `framework-maintenance/change-log.md`.
