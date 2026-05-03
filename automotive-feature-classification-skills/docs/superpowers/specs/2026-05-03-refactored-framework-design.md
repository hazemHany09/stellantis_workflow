# Refactored Framework Design
**Date:** 2026-05-03  
**Status:** Approved

---

## Problem

Current framework loads 16 skills per session. Lead agent context fills with skill content it doesn't need. Framework is dense: `.harness/` state machine, large STATE.md, tight subagent contracts as JSON files, two-round R1/R2 classification. This confuses the agent and inflates token usage.

**Goal:** Decompose into focused subagents, each with only the skills and tools it needs. Reduce total skills from 16 to 5. Single workflow entry point. Minimal state files.

---

## Directory Structure

```
refactored-framework/
│
├── skills/
│   ├── workflow.md              # lead agent entry point — full 12-stage workflow + business domain summary
│   ├── task-contracts.md        # how lead creates contract files + spawns subagents via task_tool
│   ├── classification-rules.md  # decision rules, verdict logic, output contract per parameter
│   ├── retrieval-queries.md     # ragflow retrieval best practices, multi-query strategy
│   └── serper.md                # web search skill (copy of serper-api-mastery)
│
└── agents/
    ├── parameter-classifier-agent/
    │   ├── system-prompt.md
    │   ├── description.md
    │   └── manifest.md          # skills: [classification-rules, retrieval-queries] | tools: [ragflow_retrieval]
    │
    ├── document-downloader-agent/
    │   ├── system-prompt.md     # all download/retry/exclusion logic inline
    │   ├── description.md
    │   └── manifest.md          # skills: [] | tools: [fetch_url, fetch_webpage, ragflow_upload, ragflow_run_document]
    │
    ├── parameter-searching-agent/
    │   ├── system-prompt.md
    │   ├── description.md
    │   └── manifest.md          # skills: [serper, classification-rules] | tools: [serper_google_search]
    │
    └── deep-document-analyzer/
        ├── system-prompt.md     # PLACEHOLDER — not yet implemented
        ├── description.md
        └── manifest.md          # planned: skills: [classification-rules] | tools: [ragflow_download]
```

**Lead agent loads:** `workflow.md` + `task-contracts.md` + `classification-rules.md` + `serper.md`

---

## Agents

### `parameter-classifier-agent`

**Responsibility:** Classify a single parameter using RAGFlow KB retrieval.

**Input:** Path to parameter contract `.md` file + `dataset_id`  
**Output:** Writes verdict into `## Result` JSON block in the contract file  
**Tools:** `ragflow_retrieval` only  
**Skills:** `classification-rules`, `retrieval-queries`  
**Max turns:** Low — focused single-parameter task, no exploration

**Behavior:**
1. Read contract file (parameter name, category, classification levels)
2. Load skills
3. Run multi-query retrieval (3–5 queries, varying phrasing, negative queries)
4. Apply decision rules from `classification-rules`
5. Write verdict to `## Result` block in contract file
6. Exit

---

### `document-downloader-agent`

**Responsibility:** Fetch a URL, upload to RAGFlow dataset, trigger ingestion.

**Input:** URL + `dataset_id`  
**Output:** Reports success (with `doc_id`) or exclusion (with reason)  
**Tools:** `fetch_url`, `fetch_webpage`, `ragflow_upload`, `ragflow_run_document`  
**Skills:** None — all logic embedded in system prompt  

**Retry logic (in system prompt):**
- 3 attempts with exponential backoff on fetch failure
- Access-denied → mark excluded immediately, report reason
- Max retries exceeded → mark excluded, report reason
- Upload failure → retry once, then mark excluded

---

### `parameter-searching-agent`

**Responsibility:** Classify a parameter with no KB evidence via web search.

**Input:** Path to parameter contract `.md` file  
**Output:** Writes verdict into `## Result` block; appends `## Web Sources` section  
**Tools:** `serper_google_search` only  
**Skills:** `serper`, `classification-rules`  

**Behavior:**
1. Read contract file (parameter name, category, levels)
2. Generate targeted search queries using `serper` skill guidance
3. Fetch and assess results
4. Apply `classification-rules` to web evidence
5. Write verdict + sources to contract file
6. Exit

---

### `deep-document-analyzer` *(placeholder)*

**Status:** Not yet implemented.  
System prompt returns an error message if invoked.  
Manifest notes planned tools (`ragflow_download`) and skills (`classification-rules`) for future build.

---

## Skills

### `workflow.md`

Main entry point for lead agent. Contains:

**Business Domain Summary** — condensed Stellantis/automotive context. Covers: what automotive feature classification means, parameter categories (Safety, Connectivity, Comfort, etc.), the significance of Basic/Medium/High classification levels, common edge cases (trim-specific features, market-specific availability, optional vs standard). Lead uses this section to reason through anomalies without spawning a subagent.

**12-Stage Workflow:**

| Stage | Action |
|-------|--------|
| 1 | Parse request: brand, model, year, market, optional param list, optional domains/URLs |
| 2 | Create RAGFlow dataset → save `dataset_id` to STATE.md |
| 3 | Freeze `params.csv` if no params specified; read classification levels from it |
| 4 | Create `classification-tasks/` directory; spawn one general-purpose subagent that writes ALL contract `.md` files in a single pass |
| 5 | Resolve trim if not provided (inline logic in workflow) |
| 6 | Write minimal STATE.md |
| 7 | Source discovery: use provided domains/URLs or run Serper search for best sources |
| 8 | **PAUSE 1** — present candidate URLs to user; await source approval |
| 9 | Spawn one `document-downloader-agent` per approved URL (max 10 concurrent) |
| 10 | **PAUSE 2** — await user signal to start classifying |
| 11 | Spawn one `parameter-classifier-agent` per parameter (max 10 concurrent per batch) |
| 12 | Collect results; identify `no-information` contracts |
| 13 | Spawn one `parameter-searching-agent` per no-info parameter (max 10 concurrent) |
| 14 | Run Python aggregator → `results.json` (with traceability) + `results.csv` (clean, no traceability) |

**Max concurrency:** 10 agents per batch. Lead manages batching — spawn next batch only after current batch completes.

---

### `task-contracts.md`

How lead creates contract files and communicates with subagents. Covers:

- Contract file schema (see below)
- How to call `task_tool`: `description`, `prompt`, `subagent_type`
- Prompt must include: contract file path, `dataset_id` (for classifier), explicit expected output format
- Lead must never parse agent output directly — truth is always the contract file on disk
- Lead injects `## Task` section into contract file before spawning (dataset_id, directions, expected behavior)

---

### `classification-rules.md`

Condensed from current `stellantis-decision-rules` + `stellantis-output-contract`. Covers:

- Presence verdict (Present / Not Present / Unknown)
- Status verdict (Standard / Optional / Not Available)
- Classification verdict (Basic / Medium / High / N/A)
- Confidence levels (High / Medium / Low)
- The `## Result` JSON block schema (exact field names and types)
- Caveats: trim-specific features, market-specific availability, ambiguous evidence handling

Shared by: lead agent, `parameter-classifier-agent`, `parameter-searching-agent`.

---

### `retrieval-queries.md`

RAGFlow retrieval best practices:

- Generate 3–5 queries per parameter, varying phrasing and abstraction level
- Include negative queries (what the feature is NOT, to reduce false positives)
- How to interpret chunk relevance scores
- When multiple low-confidence results justify escalating to `no-information`
- Query retry strategy if first pass returns insufficient evidence

Used by: `parameter-classifier-agent` only.

---

### `serper.md`

Direct copy of existing `serper-api-mastery` skill. No modifications.

Used by: `parameter-searching-agent`, lead agent (source discovery stage 7).

---

## Contract File Schema

**File:** `classification-tasks/<param-id>-<param-name>.md`

```markdown
# Parameter: <name>
**Category:** <category>
**Internal ID:** <id>

## Classification Levels
| Level | Label | Description |
|-------|-------|-------------|
| 1 | Basic | ... |
| 2 | Medium | ... |
| 3 | High | ... |

<working_directory>
You have access to the same sandbox environment as the parent agent:
- User uploads: `/mnt/user-data/uploads`
- User workspace: `/mnt/user-data/workspace`
- Output files: `/mnt/user-data/outputs`
- Treat `/mnt/user-data/workspace` as the default working directory
</working_directory>

## Task
<injected by lead when spawning agent — includes dataset_id, directions, expected output>

## Result
```json
{
  "parameter_id": "",
  "parameter_name": "",
  "presence": "",
  "status": "",
  "classification": null,
  "confidence": "",
  "evidence": "",
  "sources": []
}
```
```

**Rule:** Lead writes everything above `## Task` at stage 4. Lead injects `## Task` content immediately before spawning. Agent writes only the `## Result` block. Lead never parses agent turn output — reads the file.

---

## STATE.md Schema

```markdown
# Run State
- **Car:** <brand> <model> <year> <market> <resolved_trim>
- **Date:** <YYYY-MM-DD>
- **Dataset ID:** <ragflow_dataset_id>
- **Parameters:** <N> total
- **Classification tasks:** `classification-tasks/`
- **Status:** <active stage name>
```

Updated by lead at each stage transition. Minimal — no subagent tracking, no per-parameter status.

---

## Output Files

### `results.json`
Full traceability record. Generated by Python aggregator reading all `## Result` blocks from `classification-tasks/*.md`.

```json
[
  {
    "parameter_id": "",
    "parameter_name": "",
    "category": "",
    "presence": "",
    "status": "",
    "classification": null,
    "confidence": "",
    "evidence": "",
    "sources": []
  }
]
```

Saved to `/mnt/user-data/outputs/results.json`.

### `results.csv`
Clean deliverable. Same data as `results.json` minus `evidence` and `sources` fields. Generated by second Python script reading `results.json`.

Columns: `parameter_id`, `parameter_name`, `category`, `presence`, `status`, `classification`, `confidence`

Saved to `/mnt/user-data/outputs/results.csv`.

Both scripts generated and run inline by lead agent at stage 14 — no external script files.

---

## Data Flow

```
params.csv ──► classification-tasks/*.md (contracts)
                         │
         ┌───────────────┼───────────────┐
         ▼               ▼               ▼
    classifier       classifier      classifier    ◄── RAGFlow KB
         │               │               │
         └───────────────┼───────────────┘
                         ▼
                 no-info contracts
                         │
                  searcher agents                  ◄── Serper web
                         │
                         ▼
         Python aggregator scans all *.md
                         │
              ┌──────────┴──────────┐
              ▼                     ▼
         results.json          results.csv
     (with traceability)    (clean deliverable)
```

---

## What Was Removed vs Current Framework

| Removed | Replaced by |
|---------|-------------|
| 16 skills | 5 skills |
| `.harness/` state machine | `classification-tasks/` directory + minimal STATE.md |
| R1/R2 two-round classification | Single classifier round + web search fallback for no-info |
| Category-partitioned batching (≤15 params) | 1 agent per parameter, lead manages concurrency cap (10) |
| JSON subagent contracts in `.harness/SubAgent/` | Markdown contract files in `classification-tasks/` |
| Separate dispatch layer | Lead is sole spawner via `task_tool` |
| Large STATE.md with full run tracking | 7-line minimal STATE.md |
