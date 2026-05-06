# Automotive Feature Classification — End-to-End Workflow Description

## Overview

This workflow automates a manual research task performed by automotive analysts. For one car identified by `(brand, model, model_year, market)`, the system classifies each of the ~160 reference parameters against the manufacturer's full-option trim. For each parameter, it answers two questions:

1. **Presence** — Is the feature present on the car? (`Yes` / `No` / `Disputed` / `No Information Found`)
2. **Level** — If present, at which implementation tier does it operate? (`Basic` / `Medium` / `High`)

Every verdict is backed by a citation to an approved, ingested source document.

The workflow consists of **12 stages** spanning three broad phases: **Setup**, **Source Pipeline**, and **Classification**. Three mandatory client interaction gates separate the phases (two source-related pauses plus a configuration prompt that opens the classification stage). The pipeline covers one car per run; multiple cars are handled as independent parallel runs.

---

## Agents Reference

| Agent | Type | Role |
|-------|------|------|
| **Lead Agent** | Orchestrator | Executes all setup, source discovery, orchestration, aggregation, and inter-stage transitions. The central conductor of the entire run. |
| **Document Downloader Agent** | Specialized | Fetches a batch of up to 5 URLs and ingests them into the knowledge base. Multiple instances run in parallel. |
| **Parameter Classifier Agent** | Specialized | Classifies one or more parameters sequentially using the knowledge base. Many instances run in parallel; the number of parameters per agent is configured by the client at the start of Stage 11. |

> The `parameter-searching-agent` and `deep-document-analyzer` agent definitions remain in the framework but are not invoked by the current workflow; they are reserved for future fallback / deep-analysis stages.

---

## Tools Reference

| Tool Category | Tools | Used In Stages |
|---------------|-------|----------------|
| **Web Search** | `google_search`, `google_search_news`, `google_search_autocomplete`, `webpage_scrape` (via Serper) | 4, 7 |
| **Web Content Fetching** | `fetch_url`, `fetch_webpage` | 9 |
| **Knowledge Base (RAGFlow)** | `ragflow_create_dataset`, `ragflow_upload`, `ragflow_run_document`, `ragflow_retrieval`, `ragflow_list_docs`, `ragflow_delete_docs` | 2, 9, 9b, 10, 11 |
| **Code Execution** | Inline Python scripts | 5, 11, 13 |

---

## Skills Reference

| Skill | Loaded By | Purpose |
|-------|-----------|---------|
| `automotive-workflow-parameter-extraction` | Lead Agent | Master workflow skill — orchestrates the entire pipeline. Invoked once at run start. |
| `serper-search` | Lead Agent | Web search operator patterns, query construction |
| `deep-research` | Lead Agent | Source evaluation methodology used during source discovery |
| `task-contracts` | Lead Agent | Contract file template and rendering helpers used to generate per-parameter task files |
| `retrieval-queries` | Parameter Classifier Agent | Multi-query strategy for knowledge base retrieval |
| `classification-rules` | Parameter Classifier Agent | Decision rules converting evidence into verdicts and the `## Result` JSON schema |

---

## Stage-by-Stage Breakdown

---

### Stage 1 — Parse Request

**Worker:** Lead Agent

**Stage Description:**
The workflow begins when the client submits a natural-language request. The lead agent parses the message to extract the four required identifying fields: `brand`, `model`, `model_year`, and `market`. It also detects any optional inputs: an explicit list of parameters to classify (otherwise all ~160 from `params.csv` are used), a pre-supplied domain/URL list to scope web search, and any of the four optional car identity fields (`resolved_trim`, `body_type`, `powertrain`, `transmission`). Any optional car identity field the client supplies is used directly and **will not be re-resolved** in Stage 4. If any of the four required fields are missing, the agent halts and requests only the missing fields — it never guesses or defaults.

**Inputs:**

| Input | Source | Required |
|-------|--------|----------|
| `brand` | Client | **Required** |
| `model` | Client | **Required** |
| `model_year` | Client | **Required** |
| `market` | Client | **Required** |
| `parameters` | Client | Optional — defaults to full `params.csv` |
| `domains` / `urls` | Client | Optional — scopes web search if provided |
| `resolved_trim` | Client | Optional — if omitted, resolved in Stage 4 |
| `body_type` | Client | Optional — if omitted, resolved in Stage 4 |
| `powertrain` | Client | Optional — if omitted, resolved in Stage 4 |
| `transmission` | Client | Optional — if omitted, resolved in Stage 4 |

**Output:**
Validated run configuration object with all four required fields confirmed. Optional fields captured or marked absent. Run is cleared to proceed.

**Agents:** 1 (Lead Agent)

---

### Stage 2 — Create RAGFlow Dataset (Knowledge Base)

**Worker:** Lead Agent

**Stage Description:**
A fresh, isolated RAGFlow dataset is created to serve as the knowledge base (KB) for this run. The dataset is scoped exclusively to the one car being analysed — no other run shares it. The returned `dataset_id` is saved and used in every subsequent stage that touches the knowledge base.

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
The reference parameter list is locked for this run. If the client supplied an explicit parameter subset in Stage 1, that list is used as-is. Otherwise, the full `params.csv` is copied to `classification-tasks/params-frozen.csv`, which becomes the immutable reference for the run. This file is never modified after this point — it defines the exact set of parameters that will be classified. The CSV has no ID column; the lead agent generates a sequential `parameter_id` of the form `P{row_index:03d}` (1-indexed) per row when contract files are created in Stage 5.

**Key CSV columns used:**

| Column | Purpose |
|--------|---------|
| `Parameter` | Parameter name |
| `Domain / Category` | Category grouping |
| `Category Description` | Short category description |
| `Category Overview` | Full category overview / context |
| `Parameter Description` | Semantic description for retrieval |
| `Basic Criterion` / `Medium Criterion` / `High Criterion` | Level tier definitions (empty = tier not applicable) |
| `Optimised Parameter Name` | Externally-reported name used in queries and the deliverable |

**Inputs:**

| Input | Source | Required |
|-------|--------|----------|
| `params.csv` (workspace artifact) | Agent reads from workspace | **Required** |
| Explicit parameter list | Client (Stage 1) | Optional |

**Output:**
`classification-tasks/params-frozen.csv` — the locked parameter list for the run.

**Agents:** 1 (Lead Agent)

---

### Stage 4 — Resolve Car Details

**Worker:** Lead Agent

**Stage Description:**
For any of the four optional car identity fields not supplied by the client (`resolved_trim`, `body_type`, `powertrain`, `transmission`), the lead agent resolves them now in a **single research pass** — there is no separate stage per field. The resolution target is the most premium (highest-spec, most expensive) commercially available configuration of `(brand, model, model_year)` in `market`.

The agent runs Serper queries against the manufacturer's official site and authoritative automotive sources, identifies the highest-spec trim, and from that trim records:

- `resolved_trim` — the official top trim name (e.g. `"Platinum Reserve"`, `"Lounge"`, `"M Sport Ultimate Package"`)
- `body_type` — body style (Sedan, SUV, Van, Pickup, Coupe, …)
- `powertrain` — engine/energy type (Gasoline, Diesel, Hybrid, Plug-in Hybrid, Electric)
- `transmission` — gearbox type (Automatic, Manual, CVT, Dual-Clutch)

If the top trim is offered in multiple body styles or powertrains, the most premium variant (highest price or output) is chosen. Fields the client already supplied in Stage 1 are used directly and never overridden. All four fields must be known before Stage 5.

**Inputs:**

| Input | Source | Required |
|-------|--------|----------|
| `brand`, `model`, `model_year`, `market` | Stage 1 | **Required** |
| `resolved_trim`, `body_type`, `powertrain`, `transmission` | Client (Stage 1) | Optional — any field provided skips its resolution step |

**Output:**
All four car identity fields confirmed. Used in all contract files, `STATE.md`, and the final deliverable.

**Skills used:** `serper-search`

**Agents:** 1 (Lead Agent)

---

### Stage 5 — Create Classification Task Files (Contract Files)

**Worker:** Lead Agent

**Stage Description:**
The lead agent generates one contract file per parameter in a single inline Python script pass — no subagent is spawned for this step. Each file represents a task unit for a classifier agent and is placed in `classification-tasks/`. The script reads `params-frozen.csv`, assigns a sequential `parameter_id` (`P001`, `P002`, … `P160`), and renders each file using the `TEMPLATE` and `level_rows` helper provided by the `task-contracts` skill. Every file is fully pre-populated with the parameter definition, the classification levels applicable to that parameter (empty level columns are omitted), and the full car identity including `resolved_trim`, `body_type`, `powertrain`, `transmission`, and `dataset_id`. The `## Result` block is left as an empty JSON placeholder — it will be overwritten by the classifier agent in Stage 11.

**Contract file contents:**
- Parameter name, internal ID, category, category description and overview, parameter description, optimised parameter name
- Classification levels table (only the non-empty tiers for that parameter)
- `## Task` section with car identity (including body type, powertrain, transmission), trim, dataset ID, and instructions for the classifier agent
- Empty `## Result` JSON block (placeholder)

**Inputs:**

| Input | Source | Required |
|-------|--------|----------|
| `params-frozen.csv` | Stage 3 output | **Required** |
| `brand`, `model`, `model_year`, `market`, `resolved_trim`, `body_type`, `powertrain`, `transmission` | Stages 1 / 4 | **Required** |
| `dataset_id` | Stage 2 output | **Required** |

**Output:**
`classification-tasks/<parameter_id>-<optimised_name_slug>.md` — one file per parameter (~160 total). These files serve as the task definition, evidence workspace, and result container for every classifier agent.

**Skills used:** `task-contracts`

**Agents:** 1 (Lead Agent)

---

### Stage 6 — Write STATE.md

**Worker:** Lead Agent

**Stage Description:**
The central run state file is created. `STATE.md` is the shared coordination document for the entire run. It records the car identity (including body type, powertrain, transmission), full-option trim, run date, `dataset_id`, total parameter count, and the path to contract files. It also contains a `## Sources` table that will be incrementally updated as sources move through the lifecycle. All subsequent stages read and update `STATE.md` as the single source of truth for run state. The initial status is set to `source-discovery`.

**STATE.md skeleton:**

```markdown
# Run State
- **Car:** <brand> <model> <model_year> <market>
- **Body Type:** <body_type>
- **Powertrain:** <powertrain>
- **Transmission:** <transmission>
- **Full-option trim:** <resolved_trim>
- **Date:** <YYYY-MM-DD>
- **Dataset ID:** <ragflow_dataset_id>
- **Parameters:** <N> total
- **Classification tasks:** `classification-tasks/`
- **Status:** source-discovery

## Sources
| # | URL | source_type | doc_type | description | doc_id | status |
|---|-----|-------------|----------|-------------|--------|--------|
```

**Inputs:**

| Input | Source | Required |
|-------|--------|----------|
| `brand`, `model`, `model_year`, `market`, `resolved_trim`, `body_type`, `powertrain`, `transmission` | Stages 1, 4 | **Required** |
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

- **Mode A (client-supplied domains/URLs):** The client provided a list of domains or specific URLs in Stage 1. The agent runs all queries below but restricts each search to the provided domains using `site:` operators. Per-page discovery is still performed within the scoped domains.
- **Mode B (agent-discovered):** No domains provided. The agent runs open web searches across the full web using at minimum five query patterns:
  - Official spec page: `"<brand> <model> <model_year>" "<resolved_trim>" specifications site:<brand>.com`
  - Owner manual: `"<brand> <model> <model_year>" owner manual filetype:pdf`
  - Press reviews: `"<brand> <model> <model_year>" "<market>" review (site:motortrend.com OR site:caranddriver.com OR site:thecarguide.com)`
  - Aggregators: `"<brand> <model> <model_year>" features standard optional trim (site:edmunds.com OR site:kbb.com)`
  - Dealer order guide: `"<brand> <model> <model_year>" "<market>" dealer order guide`

The agent compiles a deduplicated candidate URL list, removes paywalled pages and forum-only sources (keeping at least one forum source for a category if nothing else is available), and assigns each candidate a `source_type` from the canonical taxonomy. Each candidate is appended to the `## Sources` table in `STATE.md` with status `pending`.

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
The lead agent groups all approved sources into batches of up to **5 files each** and spawns one `document-downloader-agent` per batch. Each agent receives its batch as a list of file descriptors and is responsible for fetching each URL, uploading to the RAGFlow dataset, and triggering ingestion. Batches run in parallel up to the concurrent subagent limit; the lead waits for one round of batches to complete before spawning the next.

**Per-file descriptor passed to the agent:**
- `url` — source URL
- `dataset_id` — RAGFlow dataset for this run
- `source_type` — canonical source type from the `## Sources` table in `STATE.md`
- `doc_type` — document format (`pdf`, `webpage`, `docx`, …)
- All other metadata about the source

**Download logic per file:**
1. Attempts `fetch_url` first (handles PDFs, DOCX, XLSX, PPTX, and webpages)
2. On access-denied or rate-limit errors: falls back to `fetch_webpage` for HTML pages; marks binary files as `excluded`
3. Validates metadata (`source_type`, `url`, `doc_type`) before uploading — rejects upload if any required field is missing
4. Uploads via `ragflow_upload` with full metadata
5. Triggers ingestion via `ragflow_run_document` (does **not** wait for ingestion to finish)
6. Returns `status: "success"` with `doc_id`, or `status: "excluded"` with reason

**Excluded reasons:** `access-denied`, `rate-limited`, `failed-to-download`, `upload-failed`, `missing-metadata`.

**Lead agent state management per source:**
- Before spawning: set status → `downloading`
- On agent success: set status → `ingesting`, record `doc_id`
- On agent exclusion: set status → `excluded: <reason>`, no `doc_id`

The agent returns a JSON array with one result object per file in the batch.

**Inputs:**

| Input | Source | Required |
|-------|--------|----------|
| Approved source URLs | Stage 8 | **Required** |
| `dataset_id` | Stage 2 | **Required** |
| `source_type` per URL | Stage 7/8 STATE.md | **Required** |
| `doc_type` per URL | Stage 7/8 STATE.md | **Required** |

**Output:**
All approved sources either `ingesting` (with `doc_id`) or `excluded` (with reason) in `STATE.md`.

**Agents:** 1 Lead Agent + N Document Downloader Agents (one per batch of ≤5 files)

---

### Stage 9b — Deduplicate Ingested Documents

**Worker:** Lead Agent

**Stage Description:**
After all download batches complete, the lead agent calls `ragflow_list_docs` to retrieve the full document list from the dataset. Documents sharing the same filename are identified as duplicates. For each duplicate group: keep the document that has been ingested, or — if both have not been ingested — keep the one with the earliest `create_time`. All other duplicates are deleted via `ragflow_delete_docs`, and their rows are removed from `STATE.md`. If no duplicates are found, this step passes silently.

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
The lead agent queries RAGFlow for the current ingestion status of every document in the dataset and updates `STATE.md` accordingly:

- RAGFlow confirms parsing complete → status `ingested`
- Still processing → status remains `ingesting`
- RAGFlow reports an error → status `failed: <reason>`

A status summary table is presented to the client showing all sources with URL, source type, doc type, doc ID, status, and notes. The client decides whether to proceed with the currently ingested set or return to Stage 7 to find additional sources.

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
`STATE.md` with all sources at final status (`ingested` / `ingesting` / `failed`). Classification stage cleared to begin.

**Agents:** 1 (Lead Agent)

---

### Stage 11 — Classify Parameters

**Worker:** Lead Agent + Parameter Classifier Agents (specialized)

**Stage Description:**
This is the core classification stage. The lead agent dispatches `parameter-classifier-agent` instances in parallel batches. Each agent classifies one or more parameters **sequentially** — finishing one parameter completely (read → query KB → write `## Result`) before moving to the next. The number of parameters per agent is configured by the client at the start of the stage.

> **Non-negotiable:** Stage 11 must run to full completion — every parameter classified — before stopping or presenting any summary. The lead does not pause, stop, or yield mid-stage.

**Step 0 — PAUSE 3: Configure `params_per_agent` *(Client Interaction)*:**
Before running any code or dispatching any agents, the lead asks:

```
How many parameters should each classifier agent handle?
- Enter a number (e.g. 3 → one agent classifies 3 parameters sequentially)
- Or press Enter / type "1" for the default (one agent per parameter)
```

The lead **stops** and waits for the client reply. The value is recorded as `params_per_agent` (integer, minimum 1; default 1).

**Step 1 — Find unclassified parameters:**
An inline Python script scans all `P*.md` contract files in `classification-tasks/`. A file is considered unclassified if its `## Result` block is missing, unparseable, or empty (no `parameter_id` and no `parameter_name`). Only unclassified files are dispatched.

**Step 2 — Group into chunks:**
The unclassified list is split into consecutive chunks of size `params_per_agent`:
- `params_per_agent = 1` → one agent per parameter
- `params_per_agent = 3` → one agent per three parameters

**Step 3 — Batch dispatch:**
Agents are spawned in batches of up to the maximum concurrent subagent limit (default 7, max 15). The lead waits for all agents in a batch to complete before spawning the next batch. Each chunk's task prompt lists every contract file path in order, and instructs the agent to classify each parameter sequentially with a fresh 10-query retrieval budget per parameter.

**Step 4 — Verify and retry:**
After each batch completes, the lead checks every contract file the batch was responsible for. Any file whose `## Result` block is still empty is re-dispatched as a single-parameter agent (effectively `params_per_agent = 1` for retries) before continuing to the next batch.

**Per-parameter classification protocol (Parameter Classifier Agent):**

1. **Read the contract file** — extracts parameter definition, classification levels, full car identity (`brand`, `model`, `model_year`, `market`, `resolved_trim`, `body_type`, `powertrain`, `transmission`), and `dataset_id`
2. **Load STATE.md once at session start** — builds a `doc_id → {source_type, url}` map used to assign source authority to retrieved chunks. This map is the agent's authority register. Source type is **never** inferred from URL or filename.
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
   - `presence`, `classification`
   - `confidence` (`consensus` / `single-source` / `vague-only` / `silent-all`)
   - `decision_rule` (Rule-1 through Rule-5)
   - `inverse_retrieval_attempted` (mandatory `true` on Rule 4)
   - `traceability` blocks with `source_name`, `source_link`, `source_type`, optional `stance`, `source_justification`, `classification_justification`
8. **Verify** the written Result block is non-empty and valid JSON, then move to the next assigned parameter (or terminate if it was the last)

**Inputs:**

| Input | Source | Required |
|-------|--------|----------|
| Contract file paths | Stage 5 output | **Required** |
| `STATE.md` (for `doc_id → source_type` map) | Stage 9 | **Required** |
| RAGFlow KB (via `dataset_id`) | Stage 2 | **Required** |
| `params_per_agent` | Client (Step 0) | **Required** |

**Output:**
All ~160 contract files with populated `## Result` blocks. Each file now contains the complete verdict (presence, classification, confidence, decision rule, traceability blocks).

**Skills used:** `retrieval-queries`, `classification-rules`

**Agents:** 1 Lead Agent + ⌈N / `params_per_agent`⌉ Parameter Classifier Agents (batched ≤7–15 parallel)

---

### Stage 13 — Aggregate Results and Produce Deliverable

**Worker:** Lead Agent

**Stage Description:**
The lead agent runs two inline Python scripts to produce the final deliverables.

**Script 1 — Aggregate to `results.json`:**
Reads the `## Result` JSON block from every contract file in `classification-tasks/`. Every parameter file produces exactly one record — files with an empty or unparseable Result block are included as agent-failure placeholders (with null values and `"_agent_failure": true`) rather than dropped. Metadata for failed records (`parameter_id`, `parameter_name`, `category`) is recovered from the contract file header. The output JSON contains a `run_metadata` header (car identity, trim, dataset ID, date, totals, agent-failure count) and a `records` array with the full verdict for every parameter including all traceability blocks. Output path: `/mnt/user-data/outputs/results.json`.

**Script 2 — Export to `results.csv`:**
Generates a flat CSV from `results.json` containing columns: `parameter_id`, `parameter_name`, `category`, `presence`, `classification`, `confidence`, `decision_rule`. Traceability blocks are excluded from the CSV (they remain in the JSON). `None` values are normalised to empty strings. Intended for analyst review in spreadsheet tools. Output path: `/mnt/user-data/outputs/results.csv`.

**Step 3:** The lead verifies both files exist and that row counts match, updates `STATE.md` status to `complete`, and reports final counts to the client.

**Deliverable completeness invariants:**
- Every parameter in `params-frozen.csv` appears exactly once in the output
- `classification` is non-empty only when the verdict is a successful presence determination at a defined level
- `classification` is always within the applicable level set of the parameter
- Every `presence = Yes` record carries at least one traceability block
- Every traceability block references a source that was approved and ingested in the run's KB
- Every Rule 4 record carries `inverse_retrieval_attempted = true`
- `forum_or_ugc` alone never produces a `Success` verdict

**Inputs:**

| Input | Source | Required |
|-------|--------|----------|
| All contract files | Stage 11 | **Required** |
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
Stage 4 ── Resolve Car Details ────────────── Lead Agent + Serper
            (trim + body_type + powertrain + transmission)
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
Stage 8 ── ⏸ PAUSE 1: Source Approval ───── CLIENT GATE
     │
     ▼
Stage 9 ── Download & Ingest ──────────────── Lead + N×Document Downloader Agents
     │
     ▼
Stage 9b ─ Deduplicate ────────────────────── Lead Agent
     │
     ▼
Stage 10 ─ ⏸ PAUSE 2: Classify Go-Ahead ── CLIENT GATE
     │
     ▼
Stage 11 ─ Classify Parameters ────────────── Lead + ⌈N/params_per_agent⌉ Parameter Classifier Agents
            ├─ Step 0: ⏸ PAUSE 3: ask params_per_agent  ── CLIENT GATE
            ├─ Step 1: find unclassified files
            ├─ Step 2: chunk by params_per_agent
            ├─ Step 3: batch dispatch (≤7–15 parallel)
            └─ Step 4: verify + single-parameter retries
     │
     ▼
Stage 13 ─ Aggregate & Deliverable ────────── Lead Agent (inline Python)
     │
     ▼
results.json + results.csv
```

---

## Key Business Rules

1. **One car per run.** Multiple cars are handled as independent parallel runs — no cross-car KB sharing.
2. **Full-option trim only.** Classification always targets the most premium configuration; `body_type`, `powertrain`, and `transmission` are anchored to that trim.
3. **Approval before download.** No URL is fetched until the client explicitly approves it.
4. **Full ingestion before classification.** The KB must be fully ingested before Stage 11 begins.
5. **All evidence from the KB only.** Classifier agents read only from the knowledge base via `ragflow_retrieval` — they have no web search tools.
6. **Every parameter produces exactly one verdict.** No parameter is silently dropped; agent failures are emitted as placeholder records with `_agent_failure: true`.
7. **Every `presence = Yes` verdict requires a citation.** A verdict without a traceability block is invalid.
8. **`source_type` always comes from STATE.md.** Classifier agents resolve source type via the `doc_id → source_type` map built from `STATE.md`; URL-based or filename-based inference is forbidden.
9. **`forum_or_ugc` is never sole authority.** Forum-only evidence always demotes to Rule 3 (`Unable to Decide`).
10. **Rule 4 requires an inverse query.** No parameter may be declared silent-all without having run a negative-evidence query first.
11. **Classification level must be in the applicable level set.** Evidence pointing to a level the parameter does not define is discarded — never snapped to the nearest valid level.
