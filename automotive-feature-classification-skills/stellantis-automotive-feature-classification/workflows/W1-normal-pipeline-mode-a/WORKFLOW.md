# W1 — Normal Pipeline

Authoritative stage-by-stage definition of the W1 workflow

Trigger: the user submits a classification request. URLs and domains are optional — when the client provides them, they are used directly; when absent, the agent performs its own web search.

## Table of stages

| #  | Stage id                 | Actor    | Pauses?          | Sub-skill(s)                                                                                                                                                          |
| :- | :----------------------- | :------- | :--------------- | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1  | `parse-request`          | Lead     | if missing field | — (inline logic)                                                                                                                                                      |
| 1b | `resolve-trim`           | Lead     | no               | [`car-trim-resolution`](../../skills/car-trim-resolution/SKILL.md)                                                                                                    |
| 2  | `create-dataset`         | Lead     | no               | — (direct tool call)                                                                                                                                                  |
| 3  | `source-discovery`       | Lead     | no               | [`source-discovery-with-client-domains`](../../skills/source-discovery-with-client-domains/SKILL.md) or [`source-discovery-without-client-domains`](../../skills/source-discovery-without-client-domains/SKILL.md) → [`source-validation`](../../skills/source-validation/SKILL.md) |
| 3b | `post-candidate-list`    | Lead     | **yes**          | — (writes the list and pauses)                                                                                                                                        |
| 3c | `parse-approval-reply`   | Lead     | no               | — (inline logic; resumes from 3b)                                                                                                                                     |
| 4  | `download-and-upload`    | Lead     | no               | [`source-download-and-ingest`](../../skills/source-download-and-ingest/SKILL.md)                                                                                      |
| 4b | `await-ingestion-resume` | Lead     | **yes**          | —                                                                                                                                                                     |
| 5  | `verify-ingestion`       | Lead     | maybe            | [`source-download-and-ingest`](../../skills/source-download-and-ingest/SKILL.md) (resume path)                                                                        |
| 6  | `partition-and-spawn`    | Lead     | no               | [`lead-agent-subagent-orchestration`](../../skills/lead-agent-subagent-orchestration/SKILL.md)                                                                        |
| 6a | `subagent-classify`      | Subagent | no               | [`subagent-classification-loop`](../../skills/subagent-classification-loop/SKILL.md)                                                                                  |
| 6b | `consolidate`            | Lead     | no               | [`lead-agent-subagent-orchestration`](../../skills/lead-agent-subagent-orchestration/SKILL.md) (consolidation path)                                                   |
| 7  | `emit-deliverable`       | Lead     | no               | — (inline; uses [`deliverable.schema.json`](../../templates/deliverable.schema.json) + [`deliverable.csv.columns.md`](../../templates/deliverable.csv.columns.md))    |
| 8  | `mark-complete`          | Lead     | no               | —                                                                                                                                                                     |

***

## Stage 1 — Parse request

**Inputs.** Raw user message.

**Actions.**

1. Extract `brand`, `model`, `model_year`, `market_input`. Normalise `market_input` to `market_canonical` (ISO alpha-2 for countries; zone codes for zones such as `EU`, `MEA`, `GCC`, `NAFTA`).
2. Extract `client_urls[]` and `client_domains[]` if present. A token with a scheme (`http://`, `https://`) is a URL; a bare host (`bmw.com`) is a domain. Both fields default to empty — their absence is valid and handled in Stage 3.
3. Extract `client_categories[]` if the client supplies parameter category names (e.g. "only Safety, Comfort, and Infotainment" or an explicit list of category names). If present, record them in STATE.md as `requested_categories`. If absent, `requested_categories` is empty — meaning all categories from `params.csv` apply.
4. If any of the four required fields is missing — **pause** with `PauseReason = missing-required-input`. Ask only for the missing field. On resume, repeat this stage.

**Outputs.** Preflight can create the run workspace (see [`../../SKILL.md`](../../SKILL.md) §2). Main STATE.md is initialised from [`../../templates/STATE.main.md.tmpl`](../../templates/STATE.main.md.tmpl).

**Success.** Main STATE.md exists with the four fields populated, RunStage = `parsing`, RunStatus = `active`.

***

## Stage 1b — Resolve full-option trim

**Inputs.** `car_identity` from STATE.md.

**Actions.** Load [`car-trim-resolution`](../../skills/car-trim-resolution/SKILL.md). Run. Persist `resolved_trim` and `variant_note` into STATE.md. Update RunStage = `car-identity-resolution`.

**Success.** `resolved_trim` is non-empty.

***

## Stage 2 — Create KB dataset

**Inputs.** `car_identity`, `run_id`.

**Actions.**

1. Call `create_dataset` with name `afc-<run-id>` and description that embeds the car tuple + resolved trim.
2. Record the returned dataset id into STATE.md as `kb_dataset_id`.
3. Update RunStage = `dataset-creation` → on completion, move to `source-discovery`.

***

## Stage 3 — Source discovery

**Inputs.** `client_urls[]`, `client_domains[]` (may be empty), frozen `params.csv`.

**Actions.**

1. **Branch on whether the client supplied any URLs or domains:**
   * **Client-supplied (≥ 1 URL or domain present):** Load [`source-discovery-with-client-domains`](../../skills/source-discovery-with-client-domains/SKILL.md); run. No URL cap. Tag each URL `origin = client`.
   * **Agent-discovered (no URLs or domains supplied):** Load [`source-discovery-without-client-domains`](../../skills/source-discovery-without-client-domains/SKILL.md); run. Cap list at 20 URLs (or client-specified cap) before validation.
2. Load [`source-validation`](../../skills/source-validation/SKILL.md); run on the candidate list produced by whichever branch executed.

**Success.** `runs/<run-id>/source-candidate-list.md` exists with ≥ 1 row.

**Failure.** Per the two sub-skills. If zero candidates survive and the user holds credentials, pause and ask; else abort with `RunStatus = failed`.

***

## Stage 3b — Post candidate list + pause

**Actions.**

1. Write `source-candidate-list.md` using [`../../templates/source-candidate-list.md.tmpl`](../../templates/source-candidate-list.md.tmpl).
2. Post the markdown content as a message to the user.
3. Set RunStatus = `paused`, PauseReason = `awaiting-source-approval`, RunStage = `awaiting-approval`. Append event log line.
4. Return control.

***

## Stage 3c — Parse approval reply

**Trigger.** User replies with free-text approval.

**Parsing rules.**

Accepted patterns (case-insensitive, tolerant of surrounding prose):

* `approve all` → every candidate `Approved`.
* `approve <n>[, <m>, …]` → listed indexes `Approved`; unlisted `Rejected`.
* `reject <n>[, <m>, …]` → listed indexes `Rejected`; unlisted `Approved`.
* Combinations: user may send multiple lines mixing approve + reject. Intersection rules:
  * Explicit `approve X` + explicit `reject Y` ⇒ X approved, Y rejected, all others **approved** (user was specific).
  * `approve all` overrides prior rejections in the same message — last-wins.
* Reasons (free text after `—`, `:`, `,`) are captured and attached to the rejected record.

**Actions.**

1. Write `runs/<run-id>/source-approved.md` — the parsed result: index, canonical URL, decision (`Approved`/`Rejected`), reason (if any).
2. Update SourceLifecycle per URL.
3. Set RunStatus = `active`, clear PauseReason. RunStage = `download-and-ingest`.
4. If **zero URLs approved**, pause again with a gentle re-prompt. Do not proceed with zero approvals.

***

## Stage 4 — Download, upload, ingestion submission

**Inputs.** Approved URL set from `source-approved.md`.

**Actions.** Load [`source-download-and-ingest`](../../skills/source-download-and-ingest/SKILL.md). Run through the download → upload sequence; ingestion is queued by the knowledge base on upload. Set `ingestion_started_at` in STATE.md.

**Pause boundary.** End of stage 4 = stage 4b pause.

***

## Stage 4b — Await client resume for ingestion verification

**Actions.**

1. Post to user: *"Ingestion has been queued for N documents. Send* *`resume`* *once you are ready; I will verify ingestion before classification."*
2. Set RunStatus = `paused`, PauseReason = `awaiting-client-resume`, RunStage = `awaiting-ingestion`.

***

## Stage 5 — Verify ingestion

**Trigger.** User replies `resume` (or any affirmative continuation).

**Actions.** Re-enter [`source-download-and-ingest`](../../skills/source-download-and-ingest/SKILL.md) at the "on resume" section:

1. Poll `doc_infos` for every uploaded doc.
2. Categorise each doc: Ingested / Failed / Running.
3. Apply the 60-minute per-document ceiling:
   * Failed + timed-out docs are dropped, recorded in `.harness/Archive/sources_excluded.md`.
   * Still-running docs within budget → pause again (`awaiting-ingestion-completion`) and ask the user to wait, with an ETA sketch.
   * Still-running docs past budget → drop with timeout reason.
4. If every doc is failed/timed out and none ingested → pause with `ingestion-error-client-confirmation-needed`.
5. On success (≥ 1 doc ingested, all others resolved), set `ingestion_completed_at`, RunStage = `classification`, RunStatus = `active`.

**Success.** At least one Ingested doc AND all other docs resolved.

***

## Stage 6 — Partition + spawn

**Inputs.** Frozen `params.csv`, `kb_dataset_id`, `car_identity`, resolved trim.

**Actions.** Load [`lead-agent-subagent-orchestration`](../../skills/lead-agent-subagent-orchestration/SKILL.md).

1. Partition parameters into batches of ≤15 parameters each (preserving CSV row order).
2. Write the subagent roster into STATE.md.
3. Pre-seed each `.harness/SubAgent/<agent-name>.md` with the contract.
4. Spawn up to 3 subagents concurrently; queue the rest.

**Sub-actors (stage 6a).** Each spawned subagent runs [`subagent-classification-loop`](../../skills/subagent-classification-loop/SKILL.md) independently.

***

## Stage 6b — Consolidate

Loop invariant: one subagent consolidated at a time even if multiple finish simultaneously.

Per finished subagent (per orchestration skill):

1. Validate envelope.
2. Apply `docs_metadata_updates[]`.
3. Merge records into `.harness/Category/<category>.md`.
4. Promote warnings + advisories.
5. Move working file to Archive. SubagentStatus = archived.
6. Update STATE.md counts.
7. Spawn the next queued subagent (if any) to refill the concurrency slot.

Exit stage when all subagents are archived or failed (with lead-written Rule-4 fallback records).

***

## Stage 7 — Emit deliverable

**Inputs.** All `.harness/Category/*.md` consolidated records + STATE.md header + `sources_excluded.md`.

**Actions.**

1. Assemble `deliverable.json` matching [`../../templates/deliverable.schema.json`](../../templates/deliverable.schema.json):
   * Header (run id, timestamps, car identity + variant note, resolved trim, CSV hash, KB dataset id, source counts, telemetry timestamps, subagents spawned, non-determinism disclosure).
   * Legend for SchemaType, Presence, Status, Classification enums.
   * Summary (counts + distributions + lists of Conflict / Unable-to-Decide / No Information Found parameter ids).
   * Telemetry block.
   * `records[]` in CSV row order, one per parameter in `params.csv`.
   * Footer `sources_excluded`; plus run-level warnings.
2. **Produce `deliverable.csv` via Python script** — write `.harness/scripts/json_to_csv.py` that reads the JSON deliverable and converts it to CSV using Python's `csv.writer` (not string concatenation). Execute the script. Verify row count equals `records[]` length. See `stellantis-output-contract` for the full procedure and column specification.
3. Write both to the run workspace root.
4. Set RunStage = `deliverable`.

**Invariants.** Every record in `deliverable.json` validates against the schema. Number of `records` == number of parameters in frozen CSV.

***

## Stage 8 — Mark complete

**Actions.**

1. Write the final STATE.md update (if not already done after classification — see `lead-agent-subagent-orchestration` "Final STATE.md update" section). This is mandatory.
2. Set RunStatus = `complete`, RunStage = `complete`, `completed_at` timestamp.
3. Append a final event-log line pointing at the two deliverable files.
4. Post a summary message to the user with counts and file paths.

**The lead must not skip step 1.** Agents sometimes proceed from deliverable emission directly to the user-facing summary without ever writing the final run state. If STATE.md still shows `RunStage = classification` or any intermediate stage at this point, fix it now.

***

## Error overlay

The error situations from the workflow error overlay apply at every stage. The two dominant ones are:

* **Download failure:** handled in stage 4 via 3-retry policy.
* **Ingestion timeout:** handled in stage 5 via 60-minute per-doc ceiling.

Both produce entries in `.harness/Archive/sources_excluded.md` and the deliverable footer. Neither aborts the run unless *every* approved URL is affected.
