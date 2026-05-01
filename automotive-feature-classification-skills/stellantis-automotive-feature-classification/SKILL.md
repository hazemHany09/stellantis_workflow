---
name: stellantis-automotive-feature-classification
description: Use when user names a specific car (brand, model, year, market) and asks about features, parameters, options, capabilities, or levels.
stage: W1-entry
requires:
  - stellantis-domain-context
  - stellantis-decision-rules
  - stellantis-output-contract
  - stellantis-workflow-modes
  - stellantis-failure-handling
  - stellantis-knowledge-base-protocol
  - stellantis-run-workspace
  - stellantis-car-trim-resolution
  - stellantis-source-discovery-with-client-domains
  - stellantis-source-discovery-without-client-domains
  - stellantis-source-validation
  - stellantis-approval-gate
  - stellantis-source-download-and-ingest
  - stellantis-lead-agent-subagent-orchestration
  - stellantis-subagent-classification-loop
provides:
  - run-orchestration
  - deliverable.json
  - deliverable.csv
tools: []
---

## Dependencies

This skill orchestrates two layers of sub-skills, loaded lazily at the workflow stage that needs them.

### Foundational skills (load at preflight, used by every stage)

| Foundational skill | Role | Purpose |
|---|---|---|
| `stellantis-domain-context` | Lead + Subagent | Mission, vocabulary, master invariants, glossary, tips |
| `stellantis-decision-rules` | Lead + Subagent | Five decision rules + master matrix + 8 invariants |
| `stellantis-output-contract` | Lead + Subagent | Per-parameter record shape, deliverable format, run header, footer |
| `stellantis-workflow-modes` | Lead | Identify workflow at preflight; enforce composition rules |
| `stellantis-failure-handling` | Lead + Subagent | Retries, drops, timeouts, `sources_excluded` policy |
| `stellantis-knowledge-base-protocol` | Lead + Subagent | KB capability contract; ingestion wait; retrieval discipline |
| `stellantis-run-workspace` | Lead + Subagent | Workspace layout, writer ownership, pause/resume contract |

### Operational sub-skills (loaded per stage)

| Sub-skill | Stage(s) | Role | Purpose |
|-----------|----------|------|---------|
| `stellantis-car-trim-resolution` | W1-stage-1b | Lead (may spawn search subagent) | Resolve trim to manufacturer's published top trim |
| `stellantis-source-discovery-with-client-domains` | W1-stage-3-modeA | Lead solo | Search within user-supplied domains/URLs |
| `stellantis-source-discovery-without-client-domains` | W1-stage-3-modeB | Lead solo | Open-web search (Mode B) |
| `stellantis-source-validation` | W1-stage-3b | Lead solo | Validate & filter candidate source list |
| `stellantis-approval-gate` | W1-stage-3c | Lead solo | Flag URLs, yield, resume on explicit signal |
| `stellantis-source-download-and-ingest` | W1-stage-4, W1-stage-5 | Lead + download subagents | Download URLs, upload to KB, wait for ingestion |
| `stellantis-lead-agent-subagent-orchestration` | W1-stage-6 | Lead + Dispatch Subagent | Lead spawns ONE dispatch subagent; dispatch subagent reads params.csv + STATE.md, partitions, spawns R1/R2; lead consolidates returned envelope |
| `stellantis-subagent-classification-loop` | W1-stage-6a | Classification Subagent | Classify parameters using KB retrieval |

**Key reference:** See `REFERENCES.md` for template and schema IDs (`state-main-template`, `enums-reference`, etc.).

---

# Automotive Feature Classification

## Overview

Entry point for lead agent. Operationalizes the automotive feature classification framework — a structured workflow for mapping specific car parameters against a reference CSV, with defined roles (lead agent vs. subagent), state management, and deliverable formats.

**Vocabulary:** Use **category** (not domain) for features; **source site** (not domain) for hosts. Accept user's "domain" when context is features; acknowledge using "category".

## When to Use

Load this skill when the user's request matches ALL of:

1. Names a specific car (brand + model are named or clearly inferable)
2. Asks about features, parameters, options, capabilities, or levels
3. Does not instruct the agent to bypass the classification workflow

If any of the four required fields — brand, model, model year, market — cannot be extracted, pause immediately and ask **only for missing field(s)**.


## Core Pattern: Lead Agent Lifecycle

### Lead Spawning Constraint (Hard Rule)

The lead agent runs **solo** from preflight through ingestion completion. It does **not** spawn any classification-related subagents until:

1. The `.harness/` structure is fully created and initialised
2. Sources have been approved by the user (approval gate closed)
3. All approved sources are ingested and ingestion verified (stage 5 complete)

The only permitted subagent spawn before this point is the **trim-resolution search** at stage 1b — a single-purpose web-search agent that returns `resolved_trim` and `trim_package_map`. This is the sole exception.

After stage 5 completes, the lead spawns exactly **ONE classification dispatch subagent**. The dispatch subagent reads `params.csv` and `STATE.md`, partitions all parameters, spawns and monitors R1/R2 classification subagents, and returns a consolidated envelope. The lead never directly spawns R1/R2 classification agents — that is the dispatch subagent's job.

### Preflight

Executed exactly once per run, before loading any workflow or sub-skill:

1. **Create .harness folder structure IMMEDIATELY.** Before any other processing:
   - Create `.harness/` folder at project root
   - Create subdirectories: `downloads/`, `Category/`, `SubAgent/`, `Archive/`, `advisories/`
   - All agent-internal work lives here; no nesting under `runs/<run-id>/`
2. **Load foundational skills.** `stellantis-domain-context`, `stellantis-decision-rules`, `stellantis-output-contract`, `stellantis-workflow-modes`, `stellantis-failure-handling`, `stellantis-knowledge-base-protocol`, `stellantis-run-workspace`. These stay in context for the rest of the run.
3. **Parse request and resolve 4 required fields.** Brand, model, model_year, market. Do not default any.
4. **Identify workflow.** Use `stellantis-workflow-modes` to classify the request. Default to W1; pick Mode A if the user supplied URLs or domains, otherwise Mode B.
5. **Generate run ID:** `<YYYYMMDD>-<brand-slug>-<model-slug>-<year>-<market-slug>-<short-hash>`. Record in `.harness/STATE.md`.
6. **Initialize .harness/STATE.md** with:
   - Run metadata (run_id, submitted_at, workflow, mode, model)
   - Car identity (brand, model, year, market)
   - KB dataset reference (to be created next)
   - Empty source roster
   - Event log (first entry: "Preflight complete, .harness created")
7. **Resolve trim** — Call `stellantis-car-trim-resolution` sub-skill, record result in STATE.md.
8. **Create KB dataset** per `stellantis-knowledge-base-protocol`. Record dataset_id in STATE.md.
9. **Freeze the reference list:** copy `artifacts/params.csv` → `.harness/params.csv`, record hash in STATE.md.
   - If `requested_categories` is non-empty, filter `.harness/params.csv` to rows whose `Domain / Category` value matches one of the requested category names (case-insensitive). Write the filtered result back to `.harness/params.csv`. Record both the full-list hash and the filtered-list hash in STATE.md, and note which categories were requested.
   - If `requested_categories` is empty, all categories from the CSV are in scope. Do **not** assume a fixed count such as 160 — the actual number is whatever the frozen CSV contains after any filtering.
10. **Update STATE.md** event log: "Preflight complete, ready for source discovery."

### Roles & Responsibilities

The lead is a **pure orchestrator**. It acquires sources, manages the KB, and consolidates classification results. It does **not** classify parameters itself and does **not** partition or dispatch classification work — those belong to the dispatch subagent.

The dispatch subagent is a **classification dispatcher**. It reads `params.csv` and `STATE.md`, creates partitions, spawns and monitors R1/R2 classification subagents, and returns a consolidated envelope to the lead. It does **not** touch STATE.md or any `.harness/Category/` file.

The classification subagent is a **pure worker**. It retrieves evidence from the KB, applies decision rules, and writes records into its working file. It does **not** discover sources, upload documents, spawn other agents, or touch any file outside its own working file.

When in doubt about who does something: web/KB dataset/STATE.md writes → lead. Partitioning/R1/R2 spawning → dispatch subagent. KB retrieval for classification → classification subagent.

| Concern | Lead | Dispatch Subagent | Classification Subagent |
| :--- | :--- | :--- | :--- |
| Parse request; resolve 4 required fields | ✅ | — | — |
| Resolve car identity + full-option trim | ✅ (may spawn search agent) | — | — |
| Create + manage KB dataset | ✅ | — | — |
| Create `.harness/` folder structure | ✅ | — | — |
| Source discovery (web search) + validation | ✅ | — | — |
| Post candidate URL list; interpret approval reply | ✅ | — | — |
| Download sources → `.harness/downloads/` | ✅ (via download subagents) | — | — |
| Upload to KB; run ingestion; deduplicate docs | ✅ | — | — |
| Spawn ONE classification dispatch subagent (stage 6) | ✅ | — | — |
| Read `params.csv`; partition parameters | — | ✅ | — |
| Pre-seed R1 SubAgent working files; spawn R1 subagents | — | ✅ | — |
| Monitor R1 progress; detect timeouts; re-spawn on failure | — | ✅ | — |
| Identify gap parameters; build promise board; select R2 targets | — | ✅ | — |
| Pre-seed R2 DeepDiveAgent working files; spawn R2 subagents | — | ✅ | — |
| Monitor R2 progress; merge R1+R2 records; build dispatch envelope | — | ✅ | — |
| KB retrieval per parameter; read chunks; apply decision rules | — | — | ✅ |
| Stage metadata patches in result envelope | — | — | ✅ |
| Consolidate dispatch envelope → write `.harness/Category/*.md` | ✅ | — | — |
| Apply metadata patches via `batch_update_doc_metadata` | ✅ | — | — |
| Promote warnings; write advisories | ✅ | — | ✅ (in envelope) |
| Archive subagent working files | ✅ (dispatch + classification) | ✅ (R1/R2 after collection) | — |
| Write final STATE.md update | ✅ | — | — |
| Emit `deliverable.json` + `deliverable.csv` | ✅ | — | — |

**Hard rule:** Classification subagents **never** use open-web search, browser rendering, `fetch_url`, or `fetch_webpage`. Their only tools are the knowledge-base retrieval/inspection set — `retrieval`, `retrieve_knowledge_graph`, `list_chunks`, `ragflow_download`, `doc_infos`, `get_metadata_summary`.

**Hard rule:** The dispatch subagent **never** writes to `.harness/STATE.md` or `.harness/Category/*.md`. It writes only to its own `.harness/DispatchAgent/dispatch.md` working file (and seeds SubAgent / DeepDiveAgent contract files).

**Concurrency:** maximum **3 subagents running concurrently** (enforced by `lead-agent-subagent-orchestration` skill).

## Quick Reference

### STATE Files

The framework uses a **hybrid** state layout. All files live under `.harness/`, organized by concern:

**Client-facing (still readable in .harness/):**
- `.harness/STATE.md` (lead-write only) — tiered [T1]/[T2] sections: run metadata, car identity, KB dataset ID, CSV hash, per-doc ingestion status, category + subagent rosters (R1 + R2), gap parameters + R2 target assignments, summary counts, event log, and audit/telemetry sections

**Agent-internal (`.harness/`):**
- `.harness/downloads/` — Pre-upload markdown & binary files (staging area before KB ingest)
- `.harness/document-promise-board.md` — R1 document promise scores aggregated across partitions; drives R2 target selection
- `.harness/partition-summaries.md` — per-partition summary from each R1 subagent
- `.harness/Category/<category>.md` (lead-write only) — consolidated per-parameter records, assigned subagents, category warnings
- `.harness/DispatchAgent/dispatch.md` (lead seeds contract; dispatch subagent writes scratch + envelope) — classification dispatch subagent working file
- `.harness/SubAgent/<agent-name>.md` (dispatch subagent seeds contract; R1 subagent writes) — Round 1 classification subagent working file
- `.harness/DeepDiveAgent/<agent-name>.md` (dispatch subagent seeds contract; R2 subagent writes) — Round 2 deep-dive subagent working file
- `.harness/Archive/<agent-name>.md` — consolidated files moved after merge (dispatch + R1 + R2)
- `.harness/advisories/` — undefined-tier advisories + out-of-list findings (internal only)
- `.harness/params.csv` — frozen reference snapshot
- `.harness/source-candidates.md` — validated candidate list before approval
- `.harness/source-approved.md` — parsed approved URL set
- `.harness/sources_excluded.md` — download/ingestion drops

**Deliverables (at project root, alongside .harness/):**
- `<brand>-<model>-<year>-<market>-<run-id>.json` — Primary deliverable (full traceability)
- `<brand>-<model>-<year>-<market>-<run-id>.csv` — Secondary deliverable (flattened)

**Writer discipline:**
- Lead writes `.harness/STATE.md`, `.harness/Category/*.md`, seeds `.harness/DispatchAgent/dispatch.md` contract.
- Dispatch subagent writes `.harness/DispatchAgent/dispatch.md` (scratch + envelope), seeds `.harness/SubAgent/<agent-name>.md` and `.harness/DeepDiveAgent/<agent-name>.md` contracts.
- R1/R2 classification subagents write only their own working file.
- Each agent discovers its name from the file path stem (passed on spawn).

### Workflow & Sub-Skill Loading

1. After preflight, execute the W1 pipeline stage by stage
2. Load sub-skills lazily (not pre-loaded) at the stage that needs them. See **Dependencies** section above for the full list, stages, and purposes.
3. Sub-skill header specifies `loader: lead` or `loader: subagent` — enforce isolation

### Consolidation & Archiving

On subagent `status: consolidated-ready`:

1. Validate every per-parameter record against the intermediate parameter record schema and `enums-reference`
2. Merge validated records into `.harness/Category/<category>.md`
3. Update summary counts in `.harness/STATE.md`
4. Archive: move `.harness/SubAgent/<agent-name>.md` → `.harness/Archive/<agent-name>.md`
5. Promote warnings → `.harness/Category/<category>.md` (surfaced in deliverable); advisories → `.harness/advisories/` (internal only)

**Serial constraint:** One consolidation must complete before next begins (even if subagents finish concurrently).

### Pause/Resume Protocol

**To pause** (deterministic state write, no busy-loop):
1. Set in `.harness/STATE.md`: `RunStatus = paused`, `PauseReason`, `PausedAt`, `WorkflowStage`
2. Append event-log line explaining user action required
3. Post required artifact:
   - `awaiting-source-approval`: URL list + instructions: approve all / approve indexes / reject indexes
   - `awaiting-client-resume`: short note, user sends `resume`
   - `ingestion-error-client-confirmation-needed`: list not-yet-ingested docs + "wait longer or proceed?"
   - `missing-required-input`: prompt for missing field(s) only
4. Return control

**To resume:**
1. Verify `.harness/STATE.md` has `RunStatus = paused`
2. Interpret user's free-text reply per rules in stage's sub-skill
3. Update `.harness/STATE.md`: `RunStatus = active`, clear `PauseReason`, set `ResumedAt`
4. Continue from `WorkflowStage`

## Implementation: File Reading Maps

**Lead agent reads:**
- Preflight: this skill, `enums-reference`, frozen `.harness/params.csv`
- Each stage: current stage definition (see `W1-workflow-definition`), named sub-skill(s), main `.harness/STATE.md` (status check)
- Stage 6 consolidation: `.harness/DispatchAgent/dispatch.md` (dispatch envelope), then writes `.harness/Category/<category>.md`

**Dispatch subagent reads (on spawn):**
- `stellantis-lead-agent-subagent-orchestration` skill (REQUIRED)
- `enums-reference`
- Own `.harness/DispatchAgent/dispatch.md` (contract embedded at top)
- `.harness/params.csv` (authoritative parameter list)
- `.harness/STATE.md` (read-only — for `kb_dataset_id`, `car_identity`, `resolved_trim`, `trim_package_map`)

**Dispatch subagent never touches:**
- `.harness/STATE.md` (write)
- Any `.harness/Category/*.md` file
- Other subagent's working files (except seeding new SubAgent/DeepDiveAgent contracts)

**Classification subagent reads (on spawn):**
- `stellantis-subagent-classification-loop` skill (REQUIRED)
- `enums-reference`
- `lead-to-subagent-contract` and `subagent-to-lead-contract`
- Own `.harness/SubAgent/<agent-name>.md` (contract embedded at top)

**Classification subagent never touches:**
- `.harness/STATE.md`
- Any `.harness/Category/*.md` file
- `.harness/DispatchAgent/dispatch.md`
- Other subagent's working file

## Tool Allow-List

Per-actor permissions (tool bindings in sub-skill files):

| Capability          | Tools                                                                                                                       | Lead      | Dispatch Subagent | Classification Subagent |
|:---                 |:---                                                                                                                         |:---       |:---               |:---                     |
| Open-web search     | `google_search`, `google_search_news`, `google_search_images`, `google_search_videos`, `google_search_autocomplete`         | ✅        | ❌                | ❌                      |
| Web content fetch   | `fetch_url` — saves webpage (→ Markdown) or binary (PDF/DOCX/XLSX/PPTX) to `.harness/downloads/`. No video/YouTube.       | ✅        | ❌                | ❌                      |
| Browser (advanced)  | `browser_render_content`, `browser_render_scrape`, `browser_render_links` (use `fetch_url` for download use-cases)         | ✅ rare   | ❌                | ❌                      |
| Crawl workflow      | `browser_crawl_create`, `browser_crawl_status`, `browser_crawl_cancel`                                                      | ✅        | ❌                | ❌                      |
| Dataset mgmt        | `create_dataset`, `list_datasets`, `get_dataset_detail`, `update_dataset`, `delete_datasets`                                | ✅        | ❌                | ❌                      |
| Document mgmt       | `ragflow_upload`, `list_docs`, `doc_infos`, `delete_docs`, `rename_doc`, `set_doc_metadata`, `batch_update_doc_metadata`    | ✅ write  | ❌                | ❌                      |
| Metadata summary    | `get_metadata_summary`                                                                                                      | ✅        | ❌                | ✅ read-only            |
| Chunk inspection    | `list_chunks`, `ragflow_download` (saves to `.harness/downloads/`)                                                          | ✅ rare   | ❌                | ✅                      |
| Retrieval           | `retrieval`, `retrieve_knowledge_graph`                                                                                     | ✅ rare   | ❌                | ✅                      |
| Tag operations      | `list_tags`, `set_tags`                                                                                                     | ✅        | ❌                | ❌                      |
| File read           | `.harness/params.csv`, `.harness/STATE.md` (read-only), own working file                                                   | ✅        | ✅ read-only      | ✅ own file only        |
| File write          | STATE.md, Category files, DispatchAgent contract, deliverables                                                              | ✅        | ❌ (read STATE.md only) | ❌             |
| Spawn subagents     | dispatch subagent (stage 6), download subagents (stage 4), trim search (stage 1b)                                           | ✅        | ✅ (R1/R2 only)  | ❌                      |

Web search and browser rendering may be used only in source discovery and download phases (lead only). After ingestion completes, classification reads evidence exclusively from the knowledge base.

## Common Mistakes

- **Don't pre-load operational sub-skills** — load them lazily at the stage that needs them. Foundational skills (domain-context, decision-rules, output-contract, workflow-modes, failure-handling, kb-protocol, run-workspace) are the exception: they load at preflight and stay.
- **Don't delay .harness creation** — it must be the first thing in preflight, before foundational skills are even loaded.
- **Don't spawn the classification dispatch subagent before ingestion completes.** The harness must be fully set up and sources approved and ingested first. Violating this leaves the dispatch subagent with an empty KB.
- **Don't let the lead directly spawn R1/R2 classification subagents.** That is the dispatch subagent's job. The lead spawns ONE dispatch subagent at stage 6 and waits for its envelope.
- **Don't let the dispatch subagent write STATE.md or Category files** — it collects results in its own working file and the lead does all final writes.
- **Don't let classification subagents use web search or browser tools** — they read evidence from the knowledge base only.
- **Don't let classification subagents write STATE.md or category files** — violations corrupt consolidation.
- **Don't batch lead consolidations** — serialize even if subagents finish concurrently.
- **Don't write records before validating** — always check against template schema and enums first.
- **Don't default required inputs.** If `brand`, `model`, `model_year`, or `market` is missing, pause and ask for that field only.
- **Don't proceed past source download until ALL uploads complete** — the upload completion gate enforces this hard boundary.
- **Don't re-search the web after the approval gate closes.** Coverage gaps surface as Rule 4 records; gap-fill (when authored) is the dedicated remedy.
- **Don't silently chain workflows.** A re-run is a new run. A gap-fill is a separate workflow. Each is initiated by an explicit user action.
- **Don't skip the foundational skills' tips sections.** They encode lessons learned across many runs.

## References & Maintenance

Review this skill and related workflows when operational rules change.

## Failure Modes

| Failure                              | Response                                                                              |
|:---                                  |:---                                                                                  |
| Missing required input               | Pause with `PauseReason = missing-required-input`; ask only for missing field(s)     |
| Source download fails after 3 retries| Drop URL; record in `.harness/sources_excluded.md`; `FailureStage = download`       |
| Source upload fails after 3 retries or 5-min timeout | Drop URL; record in `.harness/sources_excluded.md`; `FailureStage = upload` |
| Ingestion exceeds 60-min per-doc     | Drop doc; record in `.harness/sources_excluded.md`; `FailureStage = ingestion`       |
| Partial upload, but completion gate requires all | Pause and ask user whether to retry or abort |
| Subagent invalid record              | Mark failed; re-spawn once with same contract; if fails again, lead writes manually  |
| Concurrent-run collision             | Allowed; fully independent. No coordination required.                               |

## What This File Is NOT

- Not the workflow
- Not classification verdict rules
- Not a tool reference
- Not prose — it is normative. Violations are bugs.

