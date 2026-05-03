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
| `stellantis-source-download-and-ingest` | W1-stage-4, W1-stage-5 | Lead + download/ingest subagents | Lead spawns one download+ingest subagent per approved URL; subagents fetch + upload (content-blind, size-only feedback) |
| `stellantis-lead-agent-subagent-orchestration` | W1-stage-6 | Lead | Lead reads `params.csv`, partitions by category (≤ 15 params, single-category), spawns R1 classification subagents directly, consolidates, then spawns R2 deep-dive subagents directly. No dispatch layer. |
| `stellantis-subagent-classification-loop` | W1-stage-6a | R1 Classification Subagent | Classify parameters from contract slice using KB retrieval |
| `stellantis-subagent-doc-deep-dive` | W1-stage-6b | R2 Deep-Dive Subagent | Read a single local Markdown to resolve gap parameters |

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

### Lead is the sole spawner (Hard Rule)

The lead agent is the **only actor** that may spawn subagents at any stage. No subagent at any stage may spawn another subagent. The lead spawns subagents in two distinct stages:

1. **W1 stage 4 — download + ingest subagents.** One per approved URL. Each fetches its URL via `fetch_url` (or, in retry mode, `fetch_webpage`) and uploads via `ragflow_upload`. Subagents are **content-blind**: they never open, read, or scan the file they downloaded — they report only the byte size of the saved file to the lead, and the lead does all block-type classification.
2. **W1 stage 6 — classification subagents.** The lead reads `.harness/params.csv` directly, partitions parameters **by category, max 15 parameters per subagent, never crossing categories**, and embeds the exact parameter slice in each subagent's contract. Subagents at stage 6 never read `params.csv`. Round 1 uses KB retrieval (`stellantis-subagent-classification-loop`); Round 2 uses local Markdown reads on a single file (`stellantis-subagent-doc-deep-dive`).

Before stage 4 the lead runs **solo** — source discovery, validation, approval gate, and `.harness/` initialisation are all lead-only. The only permitted subagent spawn before stage 4 is the **trim-resolution search** at stage 1b, a single-purpose web-search agent that returns `resolved_trim` and `trim_package_map`.

### Lead-only spawning constraint

No subagent of any kind — download, ingest, R1 classification, R2 deep-dive, trim search — spawns another subagent. This is enforced. Any apparent need for nested spawning is satisfied by the lead doing the spawn itself after consolidating the relevant envelope.

### Preflight

Executed exactly once per run, before loading any workflow or sub-skill:

1. **Create .harness folder structure IMMEDIATELY.** Before any other processing:
   - The agent runs inside a fixed three-path sandbox: `/mnt/user-data/workspace/` (read+write), `/mnt/user-data/outputs/` (read+write, deliverables only), `/mnt/user-data/uploads/` (read-only inputs). See `stellantis-run-workspace` for the full contract.
   - Create `/mnt/user-data/workspace/.harness/`. **All agent-internal run state lives under this folder — nothing else is written to the workspace root.**
   - Create subdirectories: `downloads/`, `Category/`, `SubAgent/`, `DeepDiveAgent/`, `DownloadAgent/`, `Archive/`, `advisories/`, `scripts/`.
   - No `runs/<run-id>/` nesting. One workspace = one run.
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
9. **Freeze the reference list:** copy `/mnt/user-data/uploads/params.csv` (if the user supplied one; otherwise the bundled reference list located at `stellantis-automotive-feature-classification/assets/params.csv` inside the skill repository — read it with `read_file` using its absolute path) → `/mnt/user-data/workspace/.harness/params.csv`, record sha256 hash to `.harness/params.csv.hash` and in STATE.md.
   - `/mnt/user-data/uploads/` is **read-only** — never written back.
   - If `requested_categories` is non-empty, filter `.harness/params.csv` to rows whose `Domain / Category` value matches one of the requested category names (case-insensitive). Write the filtered result back to `.harness/params.csv`. Record both the full-list hash and the filtered-list hash in STATE.md, and note which categories were requested.
   - If `requested_categories` is empty, all categories from the CSV are in scope. Do **not** assume a fixed count such as 160 — the actual number is whatever the frozen CSV contains after any filtering.
10. **Update STATE.md** event log: "Preflight complete, ready for source discovery."

### Roles & Responsibilities

The lead is a **pure orchestrator AND the sole spawner**. It acquires sources, manages the KB, partitions `params.csv`, spawns every subagent (download + R1 + R2), and consolidates classification results. It does **not** classify parameters itself. Subagents do narrow, well-scoped work and never spawn anything.

The download + ingest subagent is a **content-blind I/O worker**. It calls `fetch_url` (or `fetch_webpage` in retry mode) and `ragflow_upload`, captures the saved file's byte size, and writes a result envelope. It never reads the file's contents and never reads `params.csv`, `STATE.md`, or any other subagent's working file.

The R1 classification subagent is a **pure KB-retrieval worker**. It reads its contract (which contains the full parameter slice for its single category), retrieves evidence from the KB, applies decision rules, and writes records into its working file. It never discovers sources, never uploads documents, never spawns other agents, and never touches files outside its own working file. It does **not** read `params.csv`.

The R2 deep-dive subagent is a **single-file reader**. It reads exactly one Markdown file from `.harness/downloads/` (the one named in its contract), applies the decision rules, and writes records. It performs no KB retrieval, no web search, no spawning, no `params.csv` access.

When in doubt about who does something: spawning, web/KB dataset/STATE.md writes, partitioning of `params.csv` → lead. KB retrieval for classification → R1 subagent. Single-doc deep read → R2 subagent. Network fetch + upload → download/ingest subagent.

| Concern | Lead | Download/Ingest Subagent | R1 Classification Subagent | R2 Deep-Dive Subagent |
| :--- | :--- | :--- | :--- | :--- |
| Parse request; resolve 4 required fields | ✅ | — | — | — |
| Resolve car identity + full-option trim | ✅ (may spawn search agent) | — | — | — |
| Create + manage KB dataset | ✅ | — | — | — |
| Create `.harness/` folder structure | ✅ | — | — | — |
| Source discovery (web search) + validation | ✅ | — | — | — |
| Post candidate URL list; interpret approval reply | ✅ | — | — | — |
| Spawn one download/ingest subagent per approved URL | ✅ | — | — | — |
| `fetch_url` / `fetch_webpage` + `ragflow_upload` | — | ✅ | — | — |
| Capture saved file size; report to lead | — | ✅ | — | — |
| Read content of downloaded file | ❌ | ❌ | ❌ | ✅ (own contract's `target_doc.local_path` only) |
| Lead-side `block_type` classification (size-driven) | ✅ | — | — | — |
| Run ingestion; deduplicate docs | ✅ | — | — | — |
| Read `.harness/params.csv` | ✅ | ❌ | ❌ | ❌ |
| Partition params.csv by category (≤ 15 / partition, single-category) | ✅ | — | — | — |
| Pre-seed R1 SubAgent working files; embed contract with parameter slice | ✅ | — | — | — |
| Spawn R1 classification subagents | ✅ | — | — | — |
| Spawn ANY subagent (download / R1 / R2 / trim) | ✅ | ❌ | ❌ | ❌ |
| Monitor R1 progress; detect timeouts; re-spawn on failure | ✅ | — | — | — |
| Identify gap parameters; build promise board; select R2 targets | ✅ | — | — | — |
| Pre-seed R2 DeepDiveAgent working files; spawn R2 subagents | ✅ | — | — | — |
| Monitor R2 progress; merge R1+R2 records | ✅ | — | — | — |
| KB retrieval per parameter; read chunks; apply decision rules | — | — | ✅ | — |
| Read a single local Markdown end-to-end and apply decision rules | — | — | — | ✅ |
| Stage metadata patches in result envelope | — | — | ✅ | ✅ |
| Consolidate envelopes → write `.harness/Category/*.md` | ✅ | — | — | — |
| Apply metadata patches via `batch_update_doc_metadata` | ✅ | — | — | — |
| Promote warnings; write advisories | ✅ | — | ✅ (in envelope) | ✅ (in envelope) |
| Archive subagent working files | ✅ | — | — | — |
| Write final STATE.md update | ✅ | — | — | — |
| Emit `deliverable.json` + `deliverable.csv` | ✅ | — | — | — |

**Hard rule:** Classification subagents (R1) **never** use open-web search, browser rendering, `fetch_url`, `fetch_webpage`, or `read_file` against `.harness/downloads/`. Their only tools are the knowledge-base retrieval/inspection set — `retrieval`, `retrieve_knowledge_graph`, `list_chunks`, `ragflow_download`, `doc_infos`, `get_metadata_summary`. R2 deep-dive subagents use **only** `read_file` against the single `target_doc.local_path` in their contract.

**Hard rule:** Download/ingest subagents **never** read the file they downloaded. They observe only its byte size from the `fetch_url` return value or `os.stat` and report that to the lead.

**Hard rule:** No subagent of any kind reads `.harness/params.csv`, `.harness/STATE.md`, `.harness/Category/*.md`, or another subagent's working file. The lead is the sole reader/writer of those files.

**Hard rule:** No subagent spawns another subagent. Any nested-spawn pattern must be refactored into the lead spawning the second subagent itself after consolidating the first's envelope.

**Concurrency:** maximum **3 classification subagents running concurrently** (R1 or R2, enforced by `lead-agent-subagent-orchestration` skill). Download/ingest subagents have their own concurrency cap (max 5 concurrent) per `stellantis-source-download-and-ingest`.

## Quick Reference

### STATE Files

The framework uses a **hybrid** state layout. All files live under `.harness/`, organized by concern:

**Client-facing (still readable in .harness/):**
- `.harness/STATE.md` (lead-write only) — tiered [T1]/[T2] sections: run metadata, car identity, KB dataset ID, CSV hash, per-doc ingestion status, category + subagent rosters (R1 + R2), gap parameters + R2 target assignments, summary counts, event log, and audit/telemetry sections

**Agent-internal (`.harness/`):**
- `.harness/downloads/` — Pre-upload markdown & binary files (staging area before KB ingest)
- `.harness/DownloadAgent/<slug>-dl.md` — download/ingest subagent working files (lead seeds contract; subagent writes scratch + content-blind result envelope)
- `.harness/document-promise-board.md` — R1 document promise scores aggregated across partitions; drives R2 target selection
- `.harness/partition-summaries.md` — per-partition summary from each R1 subagent
- `.harness/Category/<category>.md` (lead-write only) — consolidated per-parameter records, assigned subagents, category warnings
- `.harness/SubAgent/<agent-name>.md` (lead seeds contract; R1 subagent writes) — Round 1 classification subagent working file
- `.harness/DeepDiveAgent/<agent-name>.md` (lead seeds contract; R2 subagent writes) — Round 2 deep-dive subagent working file
- `.harness/Archive/<agent-name>.md` — consolidated files moved after merge (download + R1 + R2)
- `.harness/advisories/` — undefined-tier advisories + out-of-list findings (internal only)
- `.harness/params.csv` — frozen reference snapshot (lead-only reader during classification)
- `.harness/source-candidates.md` — validated candidate list before approval
- `.harness/source-approved.md` — parsed approved URL set
- `.harness/sources_excluded.md` — download/ingestion drops

**Deliverables (in `/mnt/user-data/outputs/` — separate path, not the workspace):**
- `/mnt/user-data/outputs/<brand>-<model>-<year>-<market>-<run-id>.json` — Primary deliverable (full traceability)
- `/mnt/user-data/outputs/<brand>-<model>-<year>-<market>-<run-id>.csv` — Secondary deliverable (flattened)

**Writer discipline:**
- Lead writes `.harness/STATE.md`, `.harness/Category/*.md`, `.harness/document-promise-board.md`, `.harness/partition-summaries.md`, advisories, and seeds every subagent's contract block.
- Download/ingest subagents write only `.harness/DownloadAgent/<slug>-dl.md` (scratch + result envelope) and the file they fetch in `.harness/downloads/`.
- R1 / R2 classification subagents write only their own working file.
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
6. Spawn the next queued subagent (lead is the spawner) to refill the concurrency slot.

**Serial constraint:** One consolidation must complete before next begins (even if subagents finish concurrently).

### Pause/Resume Protocol

**To pause** (deterministic state write, no busy-loop):
1. Set in `.harness/STATE.md`: `RunStatus = paused`, `PauseReason`, `PausedAt`, `WorkflowStage`
2. Append event-log line explaining user action required
3. Post required artifact:
   - `awaiting-source-approval`: URL list + instructions: approve all / approve indexes / reject indexes
   - `awaiting-client-resume`: short note, user sends `resume`
   - `awaiting-classification-go-ahead`: **mandatory pause after ingestion is verified complete and before any R1 classification subagent is spawned.** Post a summary of: ingested documents (count + brief titles), the frozen reference-list size, the planned R1 category partitions (category × parameter count × subagent name), and any sources dropped during ingestion. Ask the user to reply `go` (or any affirmative) to begin classification, or to amend scope first.
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
- Stage 4 consolidation: each `.harness/DownloadAgent/<slug>-dl.md` envelope (lead derives `block_type` from reported size)
- Stage 6 partitioning: `.harness/params.csv` (only the lead reads this during classification)
- Stage 6 consolidation: each `.harness/SubAgent/<agent-name>.md` and `.harness/DeepDiveAgent/<agent-name>.md` envelope, then writes `.harness/Category/<category>.md`

**Download/ingest subagent reads (on spawn):**
- `stellantis-source-download-and-ingest` skill (REQUIRED)
- Own `.harness/DownloadAgent/<slug>-dl.md` (contract embedded at top)

**Download/ingest subagent never touches:**
- `.harness/STATE.md` (read or write)
- `.harness/params.csv`
- The content of the file it downloaded (size only, via `os.stat` or fetch tool return value)
- Any other subagent's working file
- `.harness/Category/*.md`

**R1 classification subagent reads (on spawn):**
- `stellantis-subagent-classification-loop` skill (REQUIRED)
- `enums-reference`
- `lead-to-subagent-contract` and `subagent-to-lead-contract`
- Own `.harness/SubAgent/<agent-name>.md` (contract embedded at top, with full parameter slice)

**R1 classification subagent never touches:**
- `.harness/STATE.md`
- `.harness/params.csv` (parameter slice is in the contract)
- Any `.harness/Category/*.md` file
- Other subagent's working file
- `.harness/downloads/*` (KB retrieval only — content arrives via `retrieval` / `list_chunks` / `ragflow_download`)

**R2 deep-dive subagent reads (on spawn):**
- `stellantis-subagent-doc-deep-dive` skill (REQUIRED)
- `enums-reference`
- Own `.harness/DeepDiveAgent/<agent-name>.md` (contract embedded at top, with gap-parameter slice)
- The single file at `target_doc.local_path` (and only that file)

**R2 deep-dive subagent never touches:**
- `.harness/STATE.md`
- `.harness/params.csv`
- Any `.harness/Category/*.md` file
- Any other file in `.harness/downloads/` apart from its assigned `target_doc.local_path`
- Other subagent's working file

## Tool Allow-List

Per-actor permissions (tool bindings in sub-skill files):

| Capability          | Tools                                                                                                                       | Lead      | Download/Ingest Subagent | R1 Classification Subagent | R2 Deep-Dive Subagent |
|:---                 |:---                                                                                                                         |:---       |:---                      |:---                        |:---                   |
| Open-web search     | `google_search`, `google_search_news`, `google_search_images`, `google_search_videos`, `google_search_autocomplete`         | ✅        | ❌                       | ❌                         | ❌                    |
| Web content fetch   | `fetch_url` — saves to `.harness/downloads/`. Used by lead during discovery; used by download/ingest subagent at stage 4. | ✅        | ✅ (initial mode)        | ❌                         | ❌                    |
| Backup web fetch    | `fetch_webpage` — retry-only, lead-spawned retry subagent.                                                                  | ✅ rare   | ✅ (retry-webpage mode)  | ❌                         | ❌                    |
| Browser (advanced)  | `browser_render_content`, `browser_render_scrape`, `browser_render_links`                                                   | ✅ rare   | ❌                       | ❌                         | ❌                    |
| Crawl workflow      | `browser_crawl_create`, `browser_crawl_status`, `browser_crawl_cancel`                                                      | ✅        | ❌                       | ❌                         | ❌                    |
| Dataset mgmt        | `create_dataset`, `list_datasets`, `get_dataset_detail`, `update_dataset`, `delete_datasets`                                | ✅        | ❌                       | ❌                         | ❌                    |
| Document mgmt       | `ragflow_upload`, `list_docs`, `doc_infos`, `delete_docs`, `rename_doc`, `set_doc_metadata`, `batch_update_doc_metadata`    | ✅ write  | ✅ `ragflow_upload` only | ❌                         | ❌                    |
| Metadata summary    | `get_metadata_summary`                                                                                                      | ✅        | ❌                       | ✅ read-only               | ❌                    |
| Chunk inspection    | `list_chunks`, `ragflow_download` (saves to `.harness/downloads/`)                                                          | ✅ rare   | ❌                       | ✅                         | ❌                    |
| Retrieval           | `retrieval`, `retrieve_knowledge_graph`                                                                                     | ✅ rare   | ❌                       | ✅                         | ❌                    |
| Local file read     | `read_file`                                                                                                                 | ✅        | ❌ (size-only via stat) | ❌ (KB only)               | ✅ own `target_doc.local_path` only |
| Tag operations      | `list_tags`, `set_tags`                                                                                                     | ✅        | ❌                       | ❌                         | ❌                    |
| File read           | `.harness/params.csv` (lead only), own working file                                                                          | ✅        | own working file only    | own working file only      | own working file only |
| File write          | STATE.md, Category files, deliverables                                                                                       | ✅        | ❌                       | ❌                         | ❌                    |
| Spawn subagents     | download/ingest (stage 4), retry-download (stage 4), R1 classification (stage 6a), R2 deep-dive (stage 6b), trim search (stage 1b) | ✅ (sole spawner) | ❌                       | ❌                         | ❌                    |

Web search and browser rendering may be used only in source discovery and download phases. After ingestion completes, classification reads evidence exclusively from the knowledge base (R1) or a single local Markdown (R2).

## Common Mistakes

- **Don't pre-load operational sub-skills** — load them lazily at the stage that needs them. Foundational skills (domain-context, decision-rules, output-contract, workflow-modes, failure-handling, kb-protocol, run-workspace) are the exception: they load at preflight and stay.
- **Don't delay .harness creation** — it must be the first thing in preflight, before foundational skills are even loaded.
- **Don't spawn classification subagents before ingestion completes AND the user has issued the classification go-ahead.** After verifying ingestion the lead must pause with `PauseReason = awaiting-classification-go-ahead`, post the ingestion + partition summary, and wait for the user's explicit `go` reply. Spawning R1 subagents on the silent-completion of ingestion is a framework bug.
- **Don't let any subagent spawn another subagent.** The lead is the sole spawner at every stage. If a flow seems to need nested spawning, consolidate the first envelope back to the lead and let the lead spawn the second.
- **Don't let any subagent read `params.csv`.** The lead reads it once, partitions by category, and embeds the parameter slice (≤ 15 params, single-category) in each subagent's contract.
- **Don't let download/ingest subagents read the file content** — they capture size only and report it. The lead does block-type classification.
- **Don't let R1 classification subagents use web search, browser tools, or `fetch_url`** — they read evidence from the knowledge base only.
- **Don't let R2 deep-dive subagents do KB retrieval or web search** — they read exactly one local Markdown.
- **Don't let any subagent write STATE.md or Category files** — violations corrupt consolidation.
- **Don't cross categories in a single partition.** Each R1 partition is a single-category slice of ≤ 15 parameters; large categories split into consecutive partitions.
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

