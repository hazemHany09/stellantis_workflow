# Automotive Feature Classification — End-to-End Workflow Description

## Overview

This workflow automates a manual research task performed by automotive analysts. For one car identified by `(brand, model, model_year, market)`, the system classifies each of the ~160 reference parameters against the manufacturer's full-option trim. For each parameter, it answers two questions:

1. **Presence** — Is the feature present on the car? (`Yes` / `No` / `Disputed` / `No Information Found`)
2. **Level** — If present, at which implementation tier does it operate? (`Basic` / `Medium` / `High`)

Every verdict is backed by a citation to an approved, ingested source document.

The workflow consists of **13 stages** spanning three broad phases: **Setup**, **Source Pipeline**, and **Classification**. Two mandatory client approval gates separate the phases. The pipeline covers one car per run; multiple cars are handled as independent parallel runs.

---

## Agents Reference

| Agent | Type | Role |
|-------|------|------|
| **Lead Agent** | Orchestrator | Executes all setup, source discovery, orchestration, aggregation, and inter-stage transitions. The central conductor of the entire run. |
| **Document Downloader Agent** | Specialized | Fetches one URL and ingests it into the knowledge base. Multiple instances run in parallel. |
| **Parameter Classifier Agent** | Specialized | Classifies one parameter using the knowledge base. One instance per parameter, many run in parallel. |
| **Parameter Searching Agent** | Specialized | Classifies one parameter using live web search as fallback. One instance per parameter, runs only on no-information parameters. |
| **Deep Document Analyzer Agent** | Specialized (planned) | Reads full documents from the KB to answer structured questions. Not yet implemented. |

---

## Tools Reference

| Tool Category | Tools | Used In Stages |
|---------------|-------|----------------|
| **Web Search** | `google_search`, `google_search_news`, `google_search_autocomplete`, `webpage_scrape` (via Serper) | 4, 7, 12 |
| **Web Content Fetching** | `fetch_url`, `fetch_webpage` | 9 |
| **Knowledge Base (RAGFlow)** | `ragflow_create_dataset`, `ragflow_upload`, `ragflow_run_document`, `ragflow_retrieval`, `ragflow_list_docs`, `ragflow_delete_docs` | 2, 9, 9b, 10, 11 |
| **Source Approval Store** | `flag_for_approval`, `retrieve_approved` (planned) | 8 |
| **Code Execution** | Inline Python scripts | 5, 11, 11b, 12, 13 |

---

## Skills Reference

| Skill | Loaded By | Purpose |
|-------|-----------|---------|
| `serper-search` | Lead Agent, Parameter Searching Agent | Web search operator patterns, query construction |
| `retrieval-queries` | Parameter Classifier Agent | Multi-query strategy for knowledge base retrieval |
| `classification-rules` | All classifier agents, Lead Agent | Decision rules converting evidence into verdicts |
| `task-contracts` | Lead Agent | Contract file template and spawning protocol |

---

## Stage-by-Stage Breakdown

---

### Stage 1 — Parse Request

**Worker:** Lead Agent

**Stage Description:**
The workflow begins when the client submits a natural-language request. The lead agent parses the message to extract the four required identifying fields: `brand`, `model`, `model_year`, and `market`. It also detects any optional inputs: an explicit list of parameters to classify (otherwise all ~160 from params.csv are used), pre-supplied domain/URL list to scope web search, or a pre-known `resolved_trim`. If any of the four required fields are missing, the agent halts and requests only the missing fields — it never guesses or defaults.

**Inputs:**

| Input | Source | Required |
|-------|--------|----------|
| `brand` | Client | **Required** |
| `model` | Client | **Required** |
| `model_year` | Client | **Required** |
| `market` | Client | **Required** |
| `parameters` | Client | Optional — defaults to full params.csv |
| `domains` / `urls` | Client | Optional — scopes web search if provided |
| `resolved_trim` | Client | Optional — if omitted, resolved in Stage 4 |

**Output:**
Validated run configuration object with all four required fields confirmed. Optional fields captured or marked absent. Run is cleared to proceed.

**Agents:** 1 (Lead Agent)

---

### Stage 2 — Create RAGFlow Dataset (Knowledge Base)

**Worker:** Lead Agent

**Stage Description:**
A fresh, isolated RAGFlow dataset is created to serve as the knowledge base (KB) for this run. The dataset is scoped exclusively to the one car being analysed — no other run shares it. The dataset is created with car identity metadata (`brand`, `model`, `model_year`, `market`, `trim`) and a unique run ID. The returned `dataset_id` is saved and used in every subsequent stage that touches the knowledge base.

**Inputs:**

| Input | Source | Required |
|-------|--------|----------|
| `brand`, `model`, `model_year`, `market` | Parsed in Stage 1 | **Required** |
| Run ID | Agent-generated | **Required** |

**Output:**
`dataset_id` — the unique identifier for the run's isolated knowledge base. Saved to `STATE.md` in Stage 6.

**Agents:** 1 (Lead Agent)

---

### Stage 3 — Freeze Parameters

**Worker:** Lead Agent

**Stage Description:**
The reference parameter list is locked for this run. If the client supplied an explicit parameter subset in Stage 1, that list is used as-is. Otherwise, the full `params.csv` is copied to `classification-tasks/params-frozen.csv`, which becomes the immutable reference for the run. This file is never modified after this point — it defines the exact set of parameters that will be classified. The lead agent reads the CSV and recognises category header rows and overview rows, skipping them during parameter enumeration while preserving their context for later use.

**Key CSV columns used:**

| Column | Purpose |
|--------|---------|
| `Parameter` | Parameter name |
| `Domain / Category` | Category grouping |
| `Parameter Description` | Semantic description for retrieval |
| `Basic Criterion` / `Medium Criterion` / `High Criterion` | Level tier definitions (empty = tier not applicable) |
| `Optimised Parameter Name` | Externally-reported name used in the deliverable |

**Inputs:**

| Input | Source | Required |
|-------|--------|----------|
| `params.csv` (workspace artifact) | Agent reads from workspace | **Required** |
| Explicit parameter list | Client (Stage 1) | Optional |

**Output:**
`classification-tasks/params-frozen.csv` — the locked parameter list for the run. Defines the exact ~160 parameters the workflow will classify.

**Agents:** 1 (Lead Agent)

---

### Stage 4 — Resolve Trim

**Worker:** Lead Agent

**Stage Description:**
The full-option trim name for the target car must be confirmed before any classification or source searching begins. If the client provided `resolved_trim` in Stage 1, it is accepted directly. Otherwise, the lead agent uses Serper web search to locate the manufacturer's published full-option trim for `(brand, model, model_year, market)`. At least two queries are run: one targeting the manufacturer's official site and one targeting general automotive sources. The trim name is confirmed from an official source and recorded as `resolved_trim`. This single value anchors all subsequent evidence evaluation — only sources confirming fitment on this specific trim are considered valid.

**Inputs:**

| Input | Source | Required |
|-------|--------|----------|
| `brand`, `model`, `model_year`, `market` | Stage 1 | **Required** |
| `resolved_trim` | Client (Stage 1) | Optional — skips search if provided |

**Output:**
`resolved_trim` — the confirmed full-option trim name (e.g. `"M Sport Ultimate Package"`, `"Allure Ultimate"`, `"Performance Dual Motor"`). Used in all contract files, STATE.md, and the final deliverable.

**Skills used:** `serper-search`

**Agents:** 1 (Lead Agent)

---

### Stage 5 — Create Classification Task Files (Contract Files)

**Worker:** Lead Agent

**Stage Description:**
The lead agent generates one contract file per parameter in a single inline Python script pass — no subagent is spawned for this step. Each file represents a task unit for a classifier agent and is placed in `classification-tasks/`. The script reads `params-frozen.csv`, assigns a sequential `parameter_id` (`P001`, `P002`, … `P160`), and renders each file from the contract template. Every file is fully pre-populated with the parameter definition, classification levels applicable to that parameter (empty level columns are omitted), and the full car identity including `resolved_trim` and `dataset_id`. The `## Result` block is left as an empty JSON placeholder — it will be overwritten by the classifier agent in Stage 11. No further modification to these files by the lead agent occurs after this stage.

**Contract file contents:**
- Parameter name, internal ID, category, description
- Classification levels table (only the non-empty tiers for that parameter)
- `## Task` section with car identity, trim, dataset ID, and instructions for the classifier agent
- Empty `## Result` JSON block (placeholder)

**Inputs:**

| Input | Source | Required |
|-------|--------|----------|
| `params-frozen.csv` | Stage 3 output | **Required** |
| `brand`, `model`, `model_year`, `market`, `resolved_trim` | Stage 1 / Stage 4 | **Required** |
| `dataset_id` | Stage 2 output | **Required** |

**Output:**
`classification-tasks/<parameter_id>-<optimised_name_slug>.md` — one file per parameter (~160 total). These files serve as the task definition, evidence workspace, and result container for every classifier agent.

**Skills used:** `task-contracts`

**Agents:** 1 (Lead Agent)

---

### Stage 6 — Write STATE.md

**Worker:** Lead Agent

**Stage Description:**
The central run state file is created. `STATE.md` is the shared coordination document for the entire run. It records the car identity, run date, `dataset_id`, total parameter count, and the path to contract files. It also contains a `## Sources` table that will be incrementally updated as sources move through the lifecycle. All subsequent stages read and update `STATE.md` as the single source of truth for run state. The initial status is set to `source-discovery`.

**Inputs:**

| Input | Source | Required |
|-------|--------|----------|
| `brand`, `model`, `model_year`, `market`, `resolved_trim` | Stages 1, 4 | **Required** |
| `dataset_id` | Stage 2 | **Required** |
| Parameter count | Stage 3 | **Required** |

**Output:**
`STATE.md` — the run state file with car identity, dataset ID, and an empty `## Sources` table ready to be populated.

**Agents:** 1 (Lead Agent)

---

### Stage 7 — Source Discovery

**Worker:** Lead Agent

**Stage Description:**
The lead agent discovers candidate source documents to populate the knowledge base. Before running any searches, it loads the `serper-search` skill (for query construction patterns) and the `deep-research` skill (for source evaluation methodology).

**Two modes based on client input:**

- **Mode A (client-supplied domains/URLs):** The client provided a list of domains or specific URLs in Stage 1. The agent runs all queries below but restricts each search to the provided domains using `site:` operators. The web search is not skipped — per-page discovery is still required within the scoped domains.
- **Mode B (agent-discovered):** No domains provided. The agent runs open web searches across the full web using at minimum five query patterns:
  - Official spec page: `"<brand> <model> <model_year>" "<resolved_trim>" specifications site:<brand>.com`
  - Owner manual: `"<brand> <model> <model_year>" owner manual filetype:pdf`
  - Press reviews: `"<brand> <model> <model_year>" "<market>" review site:motortrend.com OR site:caranddriver.com`
  - Aggregators: `"<brand> <model> <model_year>" features standard optional trim site:edmunds.com OR site:kbb.com`
  - Dealer order guide: `"<brand> <model> <model_year>" "<market>" dealer order guide`

The agent compiles a deduplicated candidate URL list (soft cap of 20 for Mode B, uncapped for client-supplied URLs), removes paywalled pages, and assigns each candidate a `source_type` from the canonical taxonomy. Each candidate is appended to the `## Sources` table in `STATE.md` with status `pending`.

**Source type taxonomy used:**

| `source_type` | Reliability Tier |
|---------------|-----------------|
| `manufacturer_official_spec` | Tier 1 — Definitive for trim availability |
| `dealer_fleet_guide` | Tier 2 — Standard vs optional per trim |
| `third_party_press_long_form` | Tier 3 — Real-world behaviour |
| `third_party_aggregator` | Tier 4 — Trim availability matrices |
| `manufacturer_owner_manual` | Tier 5 — Feature behaviour, not availability |
| `manufacturer_video` | Tier 5 — Feature walkthroughs |
| `forum_or_ugc` | Tier 6 — Soft signal, never sole authority |

**Inputs:**

| Input | Source | Required |
|-------|--------|----------|
| `brand`, `model`, `model_year`, `market`, `resolved_trim` | Stages 1, 4 | **Required** |
| `domains` / `urls` | Client (Stage 1) | Optional |
| `STATE.md` | Stage 6 | **Required** |

**Output:**
`STATE.md` `## Sources` table updated with all candidate URLs tagged with `source_type`, `doc_type`, reliability tier, description, and coverage notes. All sources at status `pending`.

**Skills used:** `serper-search`, `deep-research`

**Agents:** 1 (Lead Agent)

---

### Stage 8 — PAUSE 1: Source Approval Gate *(Client Interaction)*

**Worker:** Lead Agent + Client

**Stage Description:**
The lead agent presents the full candidate source table to the client for review before any download or ingestion occurs. This is a hard gate — nothing proceeds until the client responds. The presentation includes a structured table with: URL, `source_type`, `doc_type`, reliability tier, description, estimated coverage, and caveats.

The client replies with one of:
- `"approved"` — accept all sources as listed
- `"approved — remove #3, #5"` — accept all except the specified rows
- `"approved — add <url>"` — accept all and add extra URLs
- Any combination of removals and additions

The approval is **per-URL** — there is no site-level cascade. The agent parses the reply, updates `STATE.md`: approved sources are set to status `approved`, rejected sources are removed from the table, and any client-added URLs are inserted with status `approved`.

> **Asynchronous model (future):** The source approval store (planned capability) will decouple the agent from the client UI. The agent will register each candidate via `flag_for_approval` and later poll via `retrieve_approved`. The approval gate remains a hard stop regardless of implementation mode.

**Inputs:**

| Input | Source | Required |
|-------|--------|----------|
| Candidate source table from Stage 7 | Agent | **Required** |
| Client approval response | **Client** | **Required** |

**Output:**
`STATE.md` `## Sources` table with final approved set, all sources at status `approved`. All rejected URLs permanently excluded from the run.

**Agents:** 1 (Lead Agent)

---

### Stage 9 — Download and Ingest

**Worker:** Lead Agent + Document Downloader Agents (specialized)

**Stage Description:**
The lead agent spawns one `document-downloader-agent` per approved source, dispatched in parallel batches of up to 5. Each downloader agent is responsible for exactly one URL: fetch the content, upload to the RAGFlow dataset, and trigger ingestion.

**Download logic per agent:**
1. Attempts `fetch_url` first (handles PDFs, DOCX, XLSX, PPTX, and webpages)
2. On access-denied or rate-limit errors: falls back to `fetch_webpage` for HTML pages; marks binary files as `excluded`
3. Validates metadata (source_type, url, file_type) before uploading — rejects upload if any required field is missing
4. Uploads via `ragflow_upload` with full metadata: `source_type`, `url`, `file_type`, title, date, description
5. Triggers ingestion via `ragflow_run_document`
6. Returns `status: "success"` with `doc_id`, or `status: "excluded"` with reason

**Excluded reasons:** `access-denied`, `rate-limited`, `failed-to-download`, `upload-failed`, `missing-metadata`, `unsupported-type` (video/streaming URLs are blocked at this stage)

**Lead agent state management per source:**
- Before spawning: set status → `downloading`
- On agent success: set status → `ingesting`, record `doc_id`
- On agent exclusion: set status → `excluded: <reason>`, no `doc_id`

**Inputs:**

| Input | Source | Required |
|-------|--------|----------|
| Approved source URLs | Stage 8 | **Required** |
| `dataset_id` | Stage 2 | **Required** |
| `source_type` per URL | Stage 7/8 STATE.md | **Required** |
| `doc_type` per URL | Stage 7/8 STATE.md | **Required** |

**Output:**
All approved sources either `ingesting` (with `doc_id`) or `excluded` (with reason) in `STATE.md`. Downloaded files stored in `downloads/` workspace directory.

**Agents:** 1 Lead Agent + N Document Downloader Agents (one per approved source, batched ≤5 parallel)

---

### Stage 9b — Deduplicate Ingested Documents

**Worker:** Lead Agent

**Stage Description:**
After all download batches complete, the lead agent calls `ragflow_list_docs` to retrieve the full document list from the dataset. Documents sharing the same filename are identified as duplicates. For each duplicate group, the most recently ingested copy is kept; all others are deleted via `ragflow_delete_docs`. Deleted document rows are removed from `STATE.md`. This prevents the same source content from being double-counted during classification. If no duplicates are found, this step passes silently.

**Inputs:**

| Input | Source | Required |
|-------|--------|----------|
| `dataset_id` | Stage 2 | **Required** |
| `STATE.md` current source table | Stage 9 | **Required** |

**Output:**
Deduplicated `STATE.md` source table. Duplicate RAGFlow documents deleted from the dataset.

**Agents:** 1 (Lead Agent)

---

### Stage 10 — PAUSE 2: Classification Go-Ahead *(Client Interaction)*

**Worker:** Lead Agent + Client

**Stage Description:**
The lead agent queries RAGFlow for the current ingestion status of every document in the dataset and updates `STATE.md` accordingly. A status summary table is presented to the client showing all sources with their final status: `ingested`, `ingesting` (still processing), or `failed: <reason>`. The client decides whether to proceed with the currently ingested set or return to Stage 7 to find additional sources.

**Classification cannot start until this gate is passed.** Starting classification against a partially-ingested knowledge base is a hard violation of the business rules.

**Client options:**
- `"start classification"` — proceed to Stage 11 with documents currently ingested
- `"go back"` — return to Stage 7 to discover and add more sources

If documents are still `ingesting` when the client chooses to start, those documents are treated as excluded for the classification run (their evidence will not be available).

**Inputs:**

| Input | Source | Required |
|-------|--------|----------|
| `dataset_id` | Stage 2 | **Required** |
| `STATE.md` source table | Stage 9b | **Required** |
| Client go-ahead response | **Client** | **Required** |

**Output:**
`STATE.md` with all sources at final status (`ingested` / `failed`). Classification stage cleared to begin.

**Agents:** 1 (Lead Agent)

---

### Stage 11 — Classify Parameters

**Worker:** Lead Agent + Parameter Classifier Agents (specialized)

**Stage Description:**
This is the core classification stage. The lead agent dispatches one `parameter-classifier-agent` per parameter in parallel batches. Each agent classifies exactly one parameter — multiple parameters are never passed to a single agent.

**Pre-dispatch (Lead Agent):**
An inline Python script scans all contract files in `classification-tasks/` to identify unclassified parameters (those with an empty `## Result` block). Only unclassified files are dispatched.

**Per-parameter classification (Parameter Classifier Agent):**
Each agent receives the absolute path to its contract file and follows this protocol:

1. **Read the contract file** — extracts parameter definition, classification levels, car identity, and `dataset_id`
2. **Load STATE.md** — builds a `doc_id → {source_type, url}` map used to assign source authority to retrieved chunks
3. **Load skills** — `retrieval-queries` and `classification-rules`
4. **Query the KB** — up to 10 queries per parameter using a multi-query strategy:
   - Primary: optimised parameter name anchored to the resolved trim
   - Alias/paraphrase: alternate industry terms for the same feature
   - Description terms: key phrases from the parameter description
   - Level criteria terms: exact language from the classification level definitions
   - Inverse/negative: explicit absence evidence (mandatory before emitting Rule 4)
5. **Classify each retrieved chunk** as Clear, Vague, or Silent per source
6. **Apply decision rules** to determine the verdict:

| Rule | Evidence Situation | Presence | Status | Classification |
|------|--------------------|----------|--------|----------------|
| Rule 1 | All clear sources agree on same valid level | `Yes` | `Success` | the agreed level |
| Rule 5 | All clear sources agree feature is absent | `No` | `Success` | `Empty` |
| Rule 2a | Clear sources present, disagree on level | `Yes` | `Conflict` | `Empty` |
| Rule 2b | Clear sources disagree on presence vs absence | `Disputed` | `Conflict` | `Empty` |
| Rule 3 | Active sources exist, none are clear | `Yes` | `Unable to Decide` | `Empty` |
| Rule 4 | No active sources at all (all silent) | `No Information Found` | `Unable to Decide` | `Empty` |

7. **Write the `## Result` block** into the contract file with the complete verdict JSON:
   - `parameter_id`, `parameter_name`, `category`
   - `presence`, `status`, `classification`
   - `confidence` (`consensus` / `single-source` / `vague-only` / `silent-all`)
   - `decision_rule` (Rule-1 through Rule-5)
   - `inverse_retrieval_attempted` (mandatory `true` on Rule 4)
   - `traceability` blocks with `source_name`, `source_link`, `source_type`, optional `stance`, `source_justification`, `classification_justification`

8. **Verify** the written Result block is non-empty and valid JSON, then terminate

**Lead agent batch management:**
- Batches of up to 7 agents (max 15) run in parallel
- After each batch completes, any agent with a still-empty Result block is re-dispatched once
- Stage runs to full completion before presenting any summary — no mid-stage pauses

**Inputs:**

| Input | Source | Required |
|-------|--------|----------|
| Contract file path | Stage 5 output | **Required** |
| `STATE.md` (for source_type map) | Stage 9 | **Required** |
| RAGFlow KB (via `dataset_id`) | Stage 2 | **Required** |

**Output:**
All ~160 contract files with populated `## Result` blocks. Each file now contains the complete verdict (presence, status, classification, confidence, decision rule, traceability blocks).

**Skills used:** `retrieval-queries`, `classification-rules`

**Agents:** 1 Lead Agent + ~160 Parameter Classifier Agents (one per parameter, batched ≤15 parallel)

---

### Stage 11b — PAUSE 3: Post-Classification Summary *(Client Interaction)*

**Worker:** Lead Agent + Client

**Stage Description:**
After all classifier batches complete, the lead agent runs an inline Python script to parse all contract files and compute summary statistics. The results are bucketed into:

- **Done** — Result block present with any valid `presence` value
- **No Information Found** — Result block present with `presence == "No Information Found"` (agent found no evidence)
- **Failed** — Result block missing or empty (agent malfunction, not a classification outcome)

A structured summary table is presented to the client with counts by presence value and classification level, plus lists of no-information and failed parameters.

**Client options:**
- `"web fallback"` — proceed to Stage 12 to attempt web search for no-information and failed parameters
- `"skip to results"` — proceed directly to Stage 13 and aggregate current results (failed parameters included with empty values)

**Inputs:**

| Input | Source | Required |
|-------|--------|----------|
| All contract files | Stage 11 | **Required** |
| Client choice | **Client** | **Required** |

**Output:**
Summary statistics object. Client decision on whether to run web fallback.

**Agents:** 1 (Lead Agent)

---

### Stage 12 — Web Search Fallback *(Optional — only if client chose "web fallback")*

**Worker:** Lead Agent + Parameter Searching Agents (specialized)

**Stage Description:**
For every parameter that returned `No Information Found` from the knowledge base, plus any parameters where the classifier agent failed entirely, the lead agent dispatches one `parameter-searching-agent` per parameter. The web fallback uses live web search rather than the knowledge base — these parameters had no usable evidence in the ingested sources, so the agent goes directly to the open web.

**Per-parameter web search (Parameter Searching Agent):**
Each agent receives the absolute path to its contract file and follows this protocol:

1. **Read the contract file** — extracts parameter definition and car identity
2. **Load skills** — `serper-search` and `classification-rules`
3. **Build search queries** — maximum 5 queries per parameter:
   - Query 1: Direct feature presence (`"<brand> <model> <model_year>" "<resolved_trim>" "<feature_name>"`)
   - Query 2: Specification source (restricted to manufacturer's official site)
   - Query 3: Review evidence (MotorTrend, Car and Driver, Edmunds)
   - Query 4: Market-specific availability
   - Query 5: Negative evidence (`"not available"`, `"not included"`, `"deleted"`)
4. **Assess snippets** as Clear, Vague, or Silent — same taxonomy as Stage 11
5. **Apply the same five decision rules** — `source_type` is inferred from the website domain
6. **Overwrite the `## Result` block** with web-search findings
7. **Append a `## Web Sources` section** listing contributing URLs with `source_type` and evidence descriptions

**Inputs:**

| Input | Source | Required |
|-------|--------|----------|
| Contract file path (no-info and failed only) | Stage 11b | **Required** |
| `brand`, `model`, `model_year`, `market`, `resolved_trim` | Stages 1, 4 | **Required** |

**Output:**
Updated contract files for no-information parameters with web-search-based verdicts. Parameters that remain without evidence after web search emit Rule 4 (`No Information Found`).

**Skills used:** `serper-search`, `classification-rules`

**Agents:** 1 Lead Agent + M Parameter Searching Agents (one per no-info / failed parameter, batched ≤15 parallel)

---

### Stage 13 — Aggregate Results and Produce Deliverable

**Worker:** Lead Agent

**Stage Description:**
The lead agent runs two inline Python scripts to produce the final deliverables.

**Script 1 — Aggregate to `results.json`:**
Reads the `## Result` JSON block from every contract file in `classification-tasks/`. Every parameter file produces exactly one record — files with an empty or unparseable Result block are included as agent-failure placeholders (with null values and `"_agent_failure": true`) rather than dropped. The output JSON contains a `run_metadata` header (car identity, trim, dataset ID, date, totals) and a `records` array with the full verdict for every parameter including all traceability blocks.

**Script 2 — Export to `results.csv`:**
Generates a flat CSV from `results.json` containing columns: `parameter_id`, `parameter_name`, `category`, `presence`, `status`, `classification`, `confidence`, `decision_rule`. Traceability blocks are excluded from the CSV (they remain in the JSON). Intended for analyst review in spreadsheet tools.

**Output file naming convention:** `<brand>-<model>-<year>-<market>-<run-id>.json` / `.csv`

**Deliverable completeness invariants:**
- Every parameter in `params-frozen.csv` appears exactly once in the output
- `classification` is non-empty only when `status = Success` AND `presence = Yes`
- `classification` is always within the applicable level set of the parameter
- Every `presence = Yes` record carries at least one traceability block
- Every traceability block references a source that was approved and ingested in the run's KB
- Every Rule 4 record carries `inverse_retrieval_attempted = true`
- `forum_or_ugc` alone never produces a `Success` verdict

**Inputs:**

| Input | Source | Required |
|-------|--------|----------|
| All contract files | Stages 11, 12 | **Required** |
| `run_metadata` (car identity, dataset ID, date) | STATE.md | **Required** |

**Output:**
- `results.json` — primary structured deliverable with full traceability
- `results.csv` — flat secondary export for analyst review

**Agents:** 1 (Lead Agent)

---

## End-to-End Flow Summary

```
Client Request
     │
     ▼
Stage 1 ── Parse Request ──────────────────── Lead Agent
     │
     ▼
Stage 2 ── Create RAGFlow Dataset ─────────── Lead Agent
     │
     ▼
Stage 3 ── Freeze Parameters ──────────────── Lead Agent
     │
     ▼
Stage 4 ── Resolve Trim ───────────────────── Lead Agent + Serper
     │
     ▼
Stage 5 ── Create Contract Files (~160) ───── Lead Agent (inline Python)
     │
     ▼
Stage 6 ── Write STATE.md ─────────────────── Lead Agent
     │
     ▼
Stage 7 ── Source Discovery ───────────────── Lead Agent + Serper
     │
     ▼
Stage 8 ── ⏸ PAUSE 1: Source Approval ──── CLIENT GATE
     │
     ▼
Stage 9 ── Download & Ingest ──────────────── Lead + N×Document Downloader Agents
     │
     ▼
Stage 9b ─ Deduplicate ────────────────────── Lead Agent
     │
     ▼
Stage 10 ─ ⏸ PAUSE 2: Classify Go-Ahead ─ CLIENT GATE
     │
     ▼
Stage 11 ─ Classify Parameters ────────────── Lead + ~160×Parameter Classifier Agents
     │
     ▼
Stage 11b ─ ⏸ PAUSE 3: Summary ──────────── CLIENT GATE
     │
     ├─── [web fallback chosen]
     │         │
     │         ▼
     │    Stage 12 ─ Web Search Fallback ─── Lead + M×Parameter Searching Agents
     │         │
     └─────────┤
               ▼
          Stage 13 ─ Aggregate & Deliverable ─ Lead Agent (inline Python)
               │
               ▼
          results.json + results.csv
```

---

## Key Business Rules

1. **One car per run.** Multiple cars are handled as independent parallel runs — no cross-car KB sharing.
2. **Full-option trim only.** Classification always targets the highest equipment level for the model.
3. **Approval before download.** No URL is fetched until the client explicitly approves it.
4. **Full ingestion before classification.** The KB must be fully ingested before Stage 11 begins.
5. **All evidence from the KB only.** Classifier agents read only from the knowledge base — never from the live web (that is Stage 12's exclusive domain and only as fallback).
6. **Every parameter produces exactly one verdict.** No parameter is silently dropped.
7. **Every `presence = Yes` verdict requires a citation.** A verdict without a traceability block is invalid.
8. **`forum_or_ugc` is never sole authority.** Forum-only evidence always demotes to Rule 3 (`Unable to Decide`).
9. **Rule 4 requires an inverse query.** No parameter may be declared silent-all without having run a negative-evidence query first.
10. **Classification level must be in the applicable level set.** Evidence pointing to a level the parameter does not define is discarded — never snapped to the nearest valid level.
