---
name: automotive-feature-classification
description: Use this skill when the user submits a request to classify the advanced features of a specific car (brand + model + year + market) against the reference parameter list in `artifacts/params.csv`, producing per-parameter Presence / Status / Classification verdicts with cited traceability blocks. Trigger phrases include "classify features for <car>", "evaluate features of <brand> <model>", "run the parameter list for <car>", or any submission that names a car and asks about features, levels, options, or capabilities.
---

# Automotive Feature Classification — Core Framework

This is the **entry point** for the lead agent. Loading this file commits the agent to the framework described below. The file does **not** repeat the business logic — it operationalises it. Every rule stated here cites a business-logic document under [`business-logic/`](../business-logic/) by ID (ARCH-*, AS-*, Q-*).

> Vocabulary note (see [`business-logic/01-glossary.md`](../business-logic/01-glossary.md)): internal artefacts use **category** (never *domain* for features) and **source site** (never *domain* for a host). Accept *domain* from the user only when the context is features, and acknowledge in replies using *category*.

---

## 1. When this skill runs

Load this file when the user's request matches **all** of:

1. Mentions a specific car — brand + model are named or clearly inferable.
2. Asks about features, parameters, options, capabilities, or levels.
3. Does not instruct the agent to bypass the classification workflow.

If *any* of the four required fields — brand, model, model year, market — cannot be extracted from the request, pause immediately per the rules in §7 and ask **only for the missing field(s)** (Q-INPUT-5, W9 in [`business-logic/13-potential-workflows.md`](../business-logic/13-potential-workflows.md)).

---

## 2. Preflight — what the lead agent does before anything else

Executed exactly once per run, before loading any workflow or sub-skill.

1. **Generate a run ID.** Format: `<YYYYMMDD>-<brand-slug>-<model-slug>-<year>-<market-slug>-<short-hash>`. The short hash disambiguates concurrent runs on the same car (AS-SCOPE-A).
2. **Create the run workspace.** Directory tree defined in [`references/workspace-layout.md`](references/workspace-layout.md).
3. **Freeze the CSV.** Copy `artifacts/params.csv` into `runs/<run-id>/params.csv` and record its hash in the main STATE.md. All later reads of parameters use this frozen copy (Q-SCOPE-2, AS-SCOPE-2).
4. **Initialise STATE.md files** from the templates — see §4.
5. **Detect the workflow to load.** Default: W1 (Mode A) if the user supplied URLs, W2 (Mode B) otherwise. Only W1 is authored in this repository today; other workflows are deferred. Record the chosen workflow in the main STATE.md.

---

## 3. Roles — who does what

See [`references/file-ownership-matrix.md`](references/file-ownership-matrix.md) for the exhaustive file-by-file matrix. The high-level split is:

| Concern                                                    | Lead agent | Subagent |
| :--------------------------------------------------------- | :--------- | :------- |
| Parse request and resolve the 4 required fields            | ✅         | —        |
| Resolve ambiguous car identity + full-option trim          | ✅         | —        |
| Create + manage the KB dataset                             | ✅         | —        |
| Source discovery (web search) + validation                 | ✅         | —        |
| Post candidate URL list to user; interpret free-text reply | ✅         | —        |
| Download + upload + `run_document` + ingestion polling     | ✅         | —        |
| Partition parameters + spawn subagents + monitor           | ✅         | —        |
| KB retrieval per parameter + read chunks + write verdicts  | —          | ✅       |
| Update per-document metadata with found parameters         | —          | ✅       |
| Consolidate subagent working files into category STATE     | ✅         | —        |
| Archive consolidated subagent files                        | ✅         | —        |
| Emit deliverable.json + deliverable.csv                    | ✅         | —        |

**Hard rule (ARCH-4):** subagents **never** use open-web search or browser rendering. Their only tools are the knowledge-base retrieval/inspection set — `retrieval`, `list_chunks`, `download_attachment`, `doc_infos`, `get_metadata_summary`, `set_doc_metadata`, `batch_update_doc_metadata`.

**Concurrency:** maximum **3 subagents running concurrently** (orchestration rule, see [`skills/lead-agent-subagent-orchestration/SKILL.md`](skills/lead-agent-subagent-orchestration/SKILL.md)).

---

## 4. STATE files — ownership and layout

The framework uses a **hybrid** state layout. The client-facing summary lives at the run-workspace root; agent-internal detail lives under `.harness/` (AS-DEL-D, adapted).

### 4.1 Client-facing

* `runs/<run-id>/STATE.md` — **main STATE**. Written by the lead agent only. Template: [`templates/STATE.main.md.tmpl`](templates/STATE.main.md.tmpl). Contains: submission date, last-run date, current `RunStage`, current `RunStatus`, current `PauseReason` (if any), resolved car identity + full-option trim, KB dataset id, CSV snapshot hash, summary counts, per-category roster, event log. Always the first file the lead reads on resume.

### 4.2 Agent-internal (under `.harness/`)

* `.harness/Category/<category>.md` — **category STATE**. Written by the lead agent only. Template: [`templates/STATE.category.md.tmpl`](templates/STATE.category.md.tmpl). Holds the consolidated per-parameter records for that category, assigned subagents, and per-category warnings promoted from subagent working files.
* `.harness/SubAgent/<agent-name>.md` — **subagent working file**. Lead pre-creates this file seeded with the contract JSON embedded in the header; the subagent writes scratch + final output into its own file. See [`templates/subagent-working-file.md.tmpl`](templates/subagent-working-file.md.tmpl).
* `.harness/Archive/<agent-name>.md` — consolidated working files that the lead has already merged into the category STATE. Moved here at consolidation time (see §6).
* `.harness/advisories/` — undefined-tier advisories (AS-HARN-B), out-of-list findings (AS-SCOPE-B). Not part of the client deliverable.

### 4.3 Writer discipline

1. **Only the lead agent writes STATE.md (main) and `.harness/Category/*.md`.** Subagents never touch these files.
2. **Subagents only write their own `.harness/SubAgent/<agent-name>.md`.** Cross-parameter warnings live in their working file; the lead promotes them into the category STATE during consolidation.
3. A subagent must be able to discover its own name — on spawn, the lead passes the absolute path of the subagent's working file as the first argument. The filename stem is the agent name.

---

## 5. Workflow + sub-skill loading

1. Once the preflight (§2) completes and the workflow is chosen, open [`workflows/W1-normal-pipeline-mode-a/WORKFLOW.md`](workflows/W1-normal-pipeline-mode-a/WORKFLOW.md) (today the only authored workflow). Execute stage by stage.
2. Each workflow stage names the sub-skill(s) that must be loaded for it. Sub-skills live under [`skills/`](skills/). Do **not** pre-load sub-skills; load them lazily at the stage that needs them.
3. Sub-skills are authored per-actor. A sub-skill whose header says `loader: lead` is never loaded by a subagent, and vice versa.

---

## 6. Consolidation + archiving protocol

When a subagent signals completion (by writing `status: consolidated-ready` into its working file and returning control):

1. Lead reads `.harness/SubAgent/<agent-name>.md`.
2. Lead validates every per-parameter record against [`templates/intermediate-parameter-record.json.tmpl`](templates/intermediate-parameter-record.json.tmpl) and the enums in [`references/enums.md`](references/enums.md).
3. Lead merges validated records into `.harness/Category/<category>.md` for every category the subagent touched.
4. Lead updates summary counts in the main `STATE.md`.
5. Lead moves `.harness/SubAgent/<agent-name>.md` to `.harness/Archive/<agent-name>.md` and sets `SubagentStatus = archived`.
6. If the subagent raised warnings or undefined-tier advisories, lead copies them into `.harness/Category/<category>.md` (warnings) or `.harness/advisories/` (advisories). Warnings are surfaced in the deliverable; advisories are internal (AS-HARN-B, AS-SCOPE-B, Q-C-15, Q-C-16).

Consolidation is strictly serial even when subagents finish concurrently — one consolidation completes before the next begins.

---

## 7. Pause / resume protocol

A pause is a deterministic state write + a message to the user. The agent **does not busy-loop**.

To pause:

1. Set in main STATE.md: `RunStatus = paused`, `PauseReason = <one of the enum values in references/enums.md>`, `PausedAt = <timestamp>`, `WorkflowStage = <current stage id>`.
2. Append an event-log line explaining what the user must do to resume.
3. Post the required client-facing artefact:
   * For `awaiting-source-approval` — the candidate URL list formatted from [`templates/source-candidate-list.md.tmpl`](templates/source-candidate-list.md.tmpl), plus explicit instructions: *"reply `approve all`, or `approve <indexes>`, or `reject <indexes>` with reasons."*
   * For `awaiting-client-resume` — a short note that ingestion has been triggered and the user should send `resume` once they are ready for ingestion verification (Q-SRC-6).
   * For `ingestion-error-client-confirmation-needed` — a list of not-yet-ingested documents with per-doc `run_status` from `doc_infos`, and the question *"wait longer or proceed without these documents?"*
   * For `missing-required-input` — a targeted prompt for the missing field(s) only (W9, Q-INPUT-5).
4. Return control to the user.

To resume:

1. Read `STATE.md`. If `RunStatus ≠ paused`, abort with an error.
2. Read the user's free-text reply. Interpret per the rules in the sub-skill named in the stage file.
3. Update `STATE.md`: `RunStatus = active`, clear `PauseReason`, set `ResumedAt`.
4. Continue from `WorkflowStage`.

---

## 8. Files every actor reads

### 8.1 Files the **lead agent** must read

At preflight:
* This file (`SKILL.md`).
* [`references/enums.md`](references/enums.md) — committed to memory; every write uses listed values only.
* [`references/workspace-layout.md`](references/workspace-layout.md) — for directory creation.
* [`references/file-ownership-matrix.md`](references/file-ownership-matrix.md).
* The frozen `runs/<run-id>/params.csv` — to enumerate parameters + schema types.

At every workflow stage:
* The current workflow stage definition.
* The sub-skill(s) named by that stage.
* The main `STATE.md` (to verify status) — every time the lead re-enters execution.

At consolidation:
* The target `.harness/SubAgent/<agent-name>.md`.
* The target `.harness/Category/<category>.md` (to merge into).
* [`templates/intermediate-parameter-record.json.tmpl`](templates/intermediate-parameter-record.json.tmpl) — validation shape.

### 8.2 Files the **subagent** must read

On spawn:
* [`skills/subagent-classification-loop/SKILL.md`](skills/subagent-classification-loop/SKILL.md).
* [`references/enums.md`](references/enums.md).
* [`references/lead-to-subagent-contract.md`](references/lead-to-subagent-contract.md) and [`references/subagent-to-lead-contract.md`](references/subagent-to-lead-contract.md).
* Its own `.harness/SubAgent/<agent-name>.md` — the contract is embedded at the top of that file.
* [`templates/traceability-block.json.tmpl`](templates/traceability-block.json.tmpl).
* The relevant category row(s) from `runs/<run-id>/params.csv` — parameter name + description + schema type.

Subagents **never** read or write:
* The main `STATE.md`.
* Any `.harness/Category/*.md` file.
* Any other subagent's working file.
* Any tool manifest under [`tools/`](../../tools/) beyond what `subagent-classification-loop` explicitly lists.

---

## 9. Tool allow-list

Per-actor allow-list. Binding to concrete tools is done in sub-skill files; vendor names are confined to [`tools/`](../../tools/).

| Capability            | Concrete tools             | Lead | Subagent |
| :-------------------- | :------------------------- | :--- | :------- |
| Open-web search       | `google_search`, `google_search_news`, `google_search_images`, `google_search_videos`, `google_search_autocomplete` | ✅ | ❌ |
| Browser rendering     | `browser_render_markdown`, `browser_render_pdf`, `browser_render_content`, `browser_render_scrape`, `browser_render_links`, `browser_crawl_create`, `browser_crawl_status`, `browser_crawl_cancel` | ✅ | ❌ |
| Dataset management    | `create_dataset`, `list_datasets`, `get_dataset_detail`, `update_dataset`, `delete_datasets` | ✅ | ❌ |
| Document management   | `upload_with_metadata`, `list_docs`, `doc_infos`, `delete_docs`, `rename_doc`, `set_doc_metadata`, `batch_update_doc_metadata` | ✅ (write) | ✅ (metadata only) |
| Document ingestion    | `run_document`             | ✅ | ❌ |
| Metadata summary      | `get_metadata_summary`     | ✅ | ✅ (read-only) |
| Chunk inspection      | `list_chunks`, `download_attachment` | ✅ (rare) | ✅ |
| Retrieval             | `retrieval`, `retrieve_knowledge_graph` | ✅ (rare) | ✅ |

ARCH-4 is enforced by this table.

---

## 10. Traceability to business logic

This framework operationalises the following business-logic IDs. If any cited ID changes, this file and the workflow must be reviewed — see [`framework-maintenance/impact-checklist.md`](../framework-maintenance/impact-checklist.md).

ARCH-1 (platform agnostic), ARCH-2 (parallel subagents), ARCH-3 (normal pipeline order), ARCH-4 (hard phase separation), AS-DEL-A (header), AS-DEL-C (footer), AS-DEL-D (workspace + `.harness/`), AS-HARN-D (partition ≤15), AS-SCOPE-2 (frozen CSV), Q-SRC-6 (explicit resume), Q-KB-4 (download retries), Q-KB-5 (60-min ingestion ceiling), Q-HARN-9 + Q-HARN-10 + AS-HARN-A (NIF is a Presence value only), Q-HARN-13 (category partition), Q-C-14 (silent-all output shape), Q-DEL-1 (JSON + CSV deliverable).

---

## 11. Failure modes

| Failure                                       | Response                                                                                           |
| :-------------------------------------------- | :------------------------------------------------------------------------------------------------- |
| Missing required input                        | Pause with `PauseReason = missing-required-input`; ask only for missing field(s).                  |
| Download failure after 3 retries              | Drop URL; record in `.harness/Archive/sources_excluded.md`; `FailureStage = download` (Q-KB-4).    |
| Ingestion exceeds 60-min per-doc ceiling      | Drop doc; record in `sources_excluded.md`; `FailureStage = ingestion` (Q-KB-5).                    |
| Partial ingestion at resume, timeout not reached | Pause with `PauseReason = ingestion-error-client-confirmation-needed`; list not-yet-ingested docs. |
| Subagent produces invalid record (enum mismatch / missing field) | Mark subagent `failed`; re-spawn **once** with the same contract; if fails again, lead writes the records manually as `Rule-4 / No Information Found` for the subagent's parameters and continues. |
| Concurrent-run collision on same car          | Allowed; fully independent (AS-SCOPE-A). No coordination.                                          |

---

## 12. What this file is NOT

* It is not the workflow — that lives under [`workflows/`](workflows/).
* It is not a rule book for classification verdicts — that lives in [`business-logic/10-decision-rules.md`](../business-logic/10-decision-rules.md).
* It is not a tool reference — tools live under [`tools/`](../../tools/).
* It is not prose documentation — it is normative. Violations are bugs.
