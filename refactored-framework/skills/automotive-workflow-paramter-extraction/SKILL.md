---
name: automotive-workflow-parameter-extraction
description: this skill is invoked by the lead agent to have a multi stages workflow for the pipeline used by the clients to extract parameters. this is the only skill you need to invoke to orchasterate the parameter extraction and classifications
---

# Automotive Feature Classification — Workflow

## Business Domain Summary

**What this is:** You are automating a manual research task performed by an automotive analyst. For one car — identified by `(brand, model, model_year, market)` — you classify feature parameters against the manufacturer's full-option trim. For each parameter you answer: is the feature present, and if so, at which implementation level (Basic / Medium / High)?

**Parameters and categories:** Parameters are grouped into functional categories (e.g. ADAS, Infotainment, Comfort, EV, Safety). Each parameter has up to three classification levels; the criteria for each level are defined in the reference list (params.csv). Levels are cumulative — High implies Medium implies Basic. the params.csv is in the same directory as this skill under the name of 'automotive-workflow-parameter-extraction-skill'

**Evidence Source Types — canonical `source_type` values:**

| `source_type` | What it is | Authority for |
|----------------|------------|---------------|
| `manufacturer_official_spec` | OEM build-and-price, configurator, brochure, press kit | Trim availability, standard vs optional |
| `manufacturer_owner_manual` | Owner's manual, user guide | How the feature works; rarely authoritative for trim availability |
| `dealer_fleet_guide` | Fleet/dealer order guide | Standard vs optional matrix per trim |
| `third_party_press_long_form` | Long-form reviews (MotorTrend, Car and Driver, etc.) | Real-world behaviour, performance |
| `third_party_aggregator` | Edmunds, KBB, Cars.com | Trim availability matrices |
| `manufacturer_video` | OEM YouTube walkthroughs | How the feature works |
| `forum_or_ugc` | Owner forums, Reddit | Soft signal only — never sole authority for a Success verdict |

Use these exact `source_type` values when tagging sources in `STATE.md` and in all contract file evidence blocks. Never invent new values.

**Reliability tier order (highest → lowest):**
1. `manufacturer_official_spec` — definitive on trim availability
2. `dealer_fleet_guide` — defines standard vs optional per trim
3. `third_party_press_long_form` — behaviour, real-world performance
4. `third_party_aggregator` — trim availability matrices
5. `manufacturer_owner_manual` / `manufacturer_video` — feature behaviour, not availability
6. `forum_or_ugc` — soft signal only — never sufficient alone for a verdict

**Common edge cases to watch for:**
- **Trim-specific features:** A feature may be standard on the top trim but optional or absent on lower trims. Always anchor to the resolved full-option trim.
- **Market-specific availability:** Features standard in the US may be absent or different in EU/Middle East markets. Market `canonical` field governs.
- **Optional vs standard confusion:** A source saying "available with package X" means optional, not standard. Sources must explicitly confirm standard fitment for a Standard classification.
- **Feature renamed across markets:** Manufacturer may use different names in different markets (e.g. "Traffic Jam Assist" vs "Highway Driving Assist"). Use alias queries.
- **Package bundles:** A feature may only be listed as part of a bundle, not individually. Confirm the resolved trim includes the bundle.

**Ingestion check:** Before any parameter is declared no-information, confirm all approved sources were successfully downloaded and ingested into the RAGFlow dataset.

---

## Required Inputs

| Input | Required | Notes |
|-------|----------|-------|
| `brand` | Yes | Never default or guess |
| `model` | Yes | Never default or guess |
| `model_year` | Yes | Never default or guess |
| `market` | Yes | Never default or guess |
| `parameters` | No | If omitted, use all params in params.csv |
| `domains` / `urls` | No | If provided, scope web search to these domains via `site:` operators |
| `resolved_trim` | No | If omitted, resolve at stage 4 |

If any required input is missing, pause and ask for only the missing field(s). Never proceed with guesses.

---

## Workflow Stages

### Stage 1 — Parse Request

Extract `brand`, `model`, `model_year`, `market` from the user message. Identify optional inputs: parameter list, domains/URLs, resolved trim. Confirm required fields are present before proceeding.

---

### Stage 2 — Create RAGFlow Dataset

Create a RAGFlow dataset (knowledge base) for this run. Save the `dataset_id`.

---

### Stage 3 — Freeze Parameters

If the user specified parameters explicitly: use that list.
If not: copy `params.csv` to `classification-tasks/params-frozen.csv`. This is the authoritative reference for the run — do not modify it after this point.

**CSV column schema (exact column names):**

| Column | Used as |
|--------|---------|
| `Parameter` | Parameter name |
| `Domain / Category` | Category |
| `Category Description` | Short category description |
| `Category Overview` | Full category overview / context |
| `Parameter Description` | What the parameter measures |
| `Basic Criterion` | Level 1 classification criterion |
| `Medium Criterion` | Level 2 classification criterion |
| `High Criterion` | Level 3 classification criterion |
| `Optimised Parameter Name` | Cleaned name used in retrieval queries |

There is no ID column — generate a sequential ID `P{row_index:03d}` (1-indexed) per row when creating contract files. This generated ID is the canonical `parameter_id` for the run.

---

### Stage 4 — Resolve Trim

If `resolved_trim` was provided by the user: use it directly.

If not: use Serper to find the manufacturer's published full-option trim for `(brand, model, model_year, market)`. Search:
```
"<brand> <model> <model_year> <market> top trim full option"
site:<brand-official-site>.com <model> <model_year> configurator
```
Confirm the trim name from an official source. Record it as `resolved_trim`.

---

### Stage 5 — Create Classification Task Files

Create the `classification-tasks/` directory.

At this point `brand`, `model`, `model_year`, `market`, `resolved_trim`, and `dataset_id` are all known. **Write and execute a Python script inline** to generate all contract files in a single pass — do not spawn a subagent for this step.

The script must:
1. Read `params-frozen.csv` from `/mnt/user-data/workspace/classification-tasks/params-frozen.csv`
2. For each row (1-indexed), generate `parameter_id = f"P{i:03d}"`
3. Map CSV columns to contract fields using **exact column names**:
   - `Parameter` → `parameter_name`
   - `Domain / Category` → `category`
   - `Category Description` → `category_description`
   - `Category Overview` → `category_overview`
   - `Parameter Description` → `parameter_description`
   - `Optimised Parameter Name` → `optimised_parameter_name`
   - `Basic Criterion`, `Medium Criterion`, `High Criterion` → level rows (omit any column that is blank/NaN)
4. Render each contract file using the `TEMPLATE` string and `level_rows` helper defined in the `task-contracts` skill. Load that skill now if not already loaded — it contains the exact Python template to copy into this script.
5. Write each file to `/mnt/user-data/workspace/classification-tasks/<parameter_id>-<optimised_name_slug>.md`
6. Leave the `## Result` block as an empty JSON placeholder (all fields empty strings / null / false)
7. Print a count of files written on completion

Do not spawn any subagent for this step — the lead agent runs the script directly.

---

### Stage 6 — Write STATE.md

Write minimal state file to `/mnt/user-data/workspace/STATE.md`:

```markdown
# Run State
- **Car:** <brand> <model> <model_year> <market> — <resolved_trim>
- **Date:** <YYYY-MM-DD>
- **Dataset ID:** <ragflow_dataset_id>
- **Parameters:** <N> total
- **Classification tasks:** `classification-tasks/`
- **Status:** source-discovery

## Sources
| # | URL | source_type | doc_type | description | doc_id | status |
|---|-----|-------------|----------|-------------|--------|--------|
```

Update `Status` field at each stage transition. Update the `Sources` table whenever a source changes state (`pending` → `approved` → `downloading` → `ingesting` → `ingested`, or `failed: <reason>` at any step).

---

### Stage 7 — Source Discovery

**Before running any searches:** Invoke both skills via the Skill tool before running any queries — do not skip this step:
1. **`serper-search` skill** — invoke via Skill tool; provides search operator syntax, tool selection rules, and query construction patterns.
2. **`deep-research` skill** — invoke via Skill tool; provides deep research methodology and source evaluation guidelines.

**If domains or URLs were provided by user:** Use them to scope the web search — run the queries below but restrict each search to the provided domains using `site:` operators. Do not skip web search.

**If no domains/URLs:** Use Serper to find the most promising sources across the open web. Run at minimum:

```
"<brand> <model> <model_year>" "<resolved_trim>" specifications site:<brand>.com
"<brand> <model> <model_year>" owner manual filetype:pdf
"<brand> <model> <model_year>" "<market>" review (site:motortrend.com OR site:caranddriver.com OR site:thecarguide.com)
"<brand> <model> <model_year>" features standard optional trim (site:edmunds.com OR site:kbb.com)
"<brand> <model> <model_year>" "<market>" dealer order guide
```

Compile a candidate URL list. Remove duplicates, paywalled pages, and forum-only sources (keep at least one forum source if nothing else is available for a category).

Append each candidate as a row in the `## Sources` table in `STATE.md` with status `pending`.

---

### Stage 8 — PAUSE 1: Source Approval

Present the candidate URL list to the user as a detailed table:

```
Found N candidate sources for <brand> <model> <model_year> <market>:

| # | URL | `source_type` | `doc_type` | Reliability Tier | Description | Coverage | Notes |
|---|-----|--------------|-----------|-----------------|-------------|----------|-------|
| 1 | <url> | `manufacturer_official_spec` | `webpage` | Tier 1 — Manufacturer | Official trim specifications page | Full trim options, standard vs optional | Definitive for trim availability |
| 2 | <url> | `manufacturer_owner_manual` | `pdf` | Tier 1 — Manufacturer | Owner/press manual (PDF) | All features, detailed specs | Highest detail, may be region-specific |
| 3 | <url> | `dealer_fleet_guide` | `pdf` | Tier 2 — Dealer | Dealer fleet order guide | Standard vs optional per trim | Confirms fitment at trim level |
| 4 | <url> | `third_party_press_long_form` | `webpage` | Tier 3 — Editorial | Long-form review from <outlet> | Real-world feature behaviour | Corroborating signal, not definitive |
| 5 | <url> | `third_party_aggregator` | `webpage` | Tier 4 — Third-party | Trim comparison matrix (Edmunds/KBB) | Feature availability across trims | Good for cross-trim checks |
| 6 | <url> | `forum_or_ugc` | `webpage` | Tier 5 — Community | Owner discussion thread | Specific feature questions | Soft signal only — never sole source |

Column definitions:
- **`source_type`:** Canonical source type value (from the Evidence Source Types table above) — use these exact values in `STATE.md` and contract files
- **`doc_type`:** Document format — `pdf` for binary documents, `webpage` for scraped HTML pages, `docx`/`xlsx` for other binary formats. Infer from URL extension; the downloader confirms the actual type.
- **Reliability Tier:** Tier 1 (highest) → Tier 5 (lowest) per business domain rules
- **Description:** What the page/document contains
- **Coverage:** Which parameters or feature areas this source is likely to address
- **Notes:** Any caveats (region lock, paywall risk, partial data, etc.)

Please review the table and reply with one of:
- "approved" — accept all sources as listed
- "approved — remove #3, #5" — accept all except the specified rows
- "approved — add <url>" — accept all and add extra URLs
- Any combination of removals and additions
```

**STOP. Wait for user reply before proceeding.**

On approval: parse the reply for removals or additions. Update the candidate list accordingly. In `STATE.md`:
- Set status → `approved` for each kept source
- Remove rows for any sources the user rejected
- Add rows (status `approved`) for any URLs the user added

---

### Stage 9 — Download and Ingest

Spawn `document-downloader-agent` agents in **batches of maximum 5 in parallel**. Wait for each batch to complete before spawning the next batch.

The task prompt for each agent must include:
- `url`: the source URL to download
- `dataset_id`: the RAGFlow dataset ID for this run
- `source_type`: the canonical source type value for this URL (taken from the `source_type` column in the `## Sources` table in `STATE.md`) — classifier agents will read this directly from `STATE.md` using the `doc_id` returned after ingestion.
- `doc_type`: the document type (`pdf`, `webpage`, `docx`, etc.) from the `doc_type` column in `STATE.md`
- All other metadata about the source.

**Per-source status lifecycle:**

1. **Before spawning the agent:** set that source's status in `STATE.md` → `downloading`
2. **Agent returns `status: "success"`:** set status → `ingesting`, record `doc_id`
3. **Agent returns `status: "excluded"`:** set status → `excluded: <reason>` in `STATE.md`. Do NOT add this source's `doc_id` to `STATE.md` — there is no `doc_id`. Do not proceed to ingest step for this source.

Valid `excluded` reasons from the downloader agent: `access-denied`, `rate-limited`, `failed-to-download`, `upload-failed`, `missing-metadata`.

After all downloader agents in a batch complete: update `STATE.md` for all results, then spawn the next batch.

---

### Stage 9b — Deduplicate Ingested Documents

After **all** downloader batches complete, call `list_docs` with the `dataset_id` to retrieve the full document list. Group documents by name (filename). For any group with two or more documents sharing the same name:
1. Keep the document that was ingested or, if both not ingested, keep the one with the earliest `create_time`.
2. Delete all others using `delete_docs` (pass a list of their `doc_id` values).
3. Remove the deleted documents' rows from `STATE.md`.

If no duplicates are found, skip this step silently.

---

### Stage 10 — PAUSE 2: Classification Go-Ahead

**Step 1 — Query ingestion status:** Query RAGFlow for the current status of all documents in the dataset. For each document, update its row in `STATE.md`:
- If RAGFlow confirms parsing complete → set status → `ingested`
- If still processing → leave status as `ingesting`
- If RAGFlow reports an error → set status → `failed: <reason>`

**Step 2 — Present status table to the user:**

```
Ingestion status for <brand> <model> <model_year> <market>:

| # | URL | Source Type | Doc Type | Doc ID | Status | Notes |
|---|-----|-------------|----------|--------|--------|-------|
| 1 | <url> | Official Spec | webpage | <doc_id> | ingested | — |
| 2 | <url> | PDF Manual | pdf | <doc_id> | ingested | — |
| 3 | <url> | Press Review | webpage | — | failed: download error | Could not fetch page |
| 4 | <url> | Aggregator | webpage | <doc_id> | ingesting | Still processing |

Summary:
- <N> documents successfully ingested
- <P> documents still ingesting
- <M> documents failed: <list with reasons>

Dataset ID: <dataset_id>

Reply with one of:
- "start classification" — proceed with currently ingested documents
- "go back" — return to Stage 7 to find additional sources
```

**STOP. Wait for user reply before proceeding.**

---

### Stage 11 — Classify Parameters

> **Non-negotiable:** The lead agent must run this stage to full completion — classifying every parameter — before stopping or presenting any summary to the user. Do not pause, stop, or yield mid-stage.

**One agent per parameter — non-negotiable.** Each `parameter-classifier-agent` classifies exactly one parameter. Never pass multiple parameters to a single agent.

**Step 1 — Find all unclassified parameters.** Run this inline Python script once before dispatching any agents:

```python
import re, json
from pathlib import Path

contracts_dir = Path("/mnt/user-data/workspace/classification-tasks")
unclassified = []

for f in sorted(contracts_dir.glob("P*.md")):
    content = f.read_text(encoding="utf-8")
    match = re.search(r'## Result\s*```json\s*(\{.*?\})\s*```', content, re.DOTALL)
    if not match:
        unclassified.append(str(f.resolve()))
        continue
    try:
        result = json.loads(match.group(1))
        if not result.get("parameter_id") and not result.get("parameter_name"):
            unclassified.append(str(f.resolve()))
    except (json.JSONDecodeError, ValueError):
        unclassified.append(str(f.resolve()))

print(f"Unclassified: {len(unclassified)}")
for p in unclassified:
    print(p)
```

**Step 2 — Batch dispatch.** Using the list above, spawn agents in batches of up to the maximum concurrent subagent limit (default 7, max 15). Wait for all agents in a batch to complete before spawning the next batch.

After each batch completes, verify every contract file in the batch has a populated `## Result` block (non-empty `parameter_id`). If any block is still empty, re-spawn that agent once for that parameter only, then continue to the next batch.

For each agent, pass only the **absolute path to the parameter's contract file**. The agent must:
1. Read the contract file in full — the `## Task` section already contains all required context: car identity, `resolved_trim`, `dataset_id`, parameter definition, and level criteria
2. Apply decision rules and write the `## Result` block back to the contract file in the required JSON format, including:
   - `parameter_id`, `parameter_name`, `category`
   - `presence` — whether the feature is present on the resolved trim
   - `status` — Standard / Optional / Not Available
   - `classification` — Basic / Medium / High / N/A
   - `confidence` — Low / Medium / High
   - `decision_rule` — which rule governed the verdict
   - `traceability` — source URL(s) and quote(s) supporting the verdict

---

### Stage 11b — PAUSE 3: Post-Classification Summary

After all Stage 11 batches complete, **write and execute a Python script inline** to parse all contract files and compute summary statistics. The script must:
1. Parse the `## Result` JSON block from each `.md` file in `classification-tasks/`
2. Bucket each contract: `done` (Result present, any presence value), `no_info` (presence == "No Information Found"), `failed` (Result block missing or empty — agent failure, not a classification outcome)
3. Count presence values, classification levels, and collect names of no-info and failed parameters

Present the computed summary to the user:

```
Classification complete for <brand> <model> <model_year> <market> — <resolved_trim>

Total parameters: <N>

| Outcome | Count |
|---------|-------|
| Classified (presence confirmed) | <N> |
| Feature absent (No) | <N> |
| No information found | <N> |
| Conflict / Unable to Decide | <N> |
| Agent failure (empty Result — not classified) | <N> |

Classification breakdown (where classified):
| Level | Count |
|-------|-------|
| Basic | <N> |
| Medium | <N> |
| High | <N> |

No-information parameters (<N>): <list parameter names>
Agent failures (<N> — Result block empty): <list parameter names>
```

If there are agent failures, note: *"<N> parameters have an empty Result block. They will appear in the final output with empty/null values."*

Would you like to:
- "web fallback" — run Stage 12 web search for the <N> no-information parameters (agent failures will also be sent to web fallback)
- "skip to results" — proceed directly to Stage 13 and aggregate current results (all parameters included, failures with empty values)

**STOP. Wait for user reply before proceeding.**

---

### Stage 12 — Web Search Fallback (No-Information Parameters) *(optional — only if user chose "web fallback")*

**Step 1 — Identify no-information parameters via inline code:** Write and execute a Python script to scan all contract files in `classification-tasks/`. The script must:
1. For each `.md` file, extract the JSON block from the `## Result` section using a regex
2. Classify each contract into one of three buckets:
   - `no_info` — Result block is present and `presence == "No Information Found"`
   - `failed` — Result block is missing or empty (agent did not write a verdict) — these are **not** no-info; flag them separately as agent failures
   - `done` — Result block is present and `presence` is any other value
3. Return the absolute file paths for the `no_info` bucket, the count of `failed` contracts, and summary statistics (presence breakdown, level breakdown)

**Important:** Empty `## Result` blocks with empty json placeholder mean the classifier agent failed, these too should be included to be processed by the `parameter-searching-agent`

**Step 2 — Batch dispatch:** For each identified contract file, spawn one `parameter-searching-agent`. Use the same batch pattern as Stage 11 — dispatch up to the concurrent limit per batch, wait for all to complete, then spawn the next batch. One agent per parameter. The agent reads the contract file and overwrites the `## Result` block with web-search findings. The prompt format is identical to that used for `parameter-classifier-agent`.

---

### Stage 13 — Aggregate Results

After all agents complete:

**Step 1:** Write a Python script inline and execute it to aggregate `## Result` blocks into `results.json`. Every parameter file must produce exactly one record — files with an empty or missing Result block are included with null/empty values (never skipped). exmaple code:

```python
import json, re
from pathlib import Path

contracts_dir = Path("/mnt/user-data/workspace/classification-tasks")
results = []
empty_result_template = {
    "parameter_id": None,
    "parameter_name": None,
    "category": None,
    "presence": None,
    "status": None,
    "classification": None,
    "confidence": None,
    "decision_rule": None,
    "evidence_summary": "",
    "sources": [],
    "inverse_retrieval_attempted": False,
    "traceability": [],
    "_agent_failure": True
}

for md_file in sorted(contracts_dir.glob("*.md")):
    if md_file.name == "params-frozen.csv":
        continue
    content = md_file.read_text(encoding="utf-8")

    # Try to extract populated Result JSON
    match = re.search(r'## Result\s*```json\s*(\{.*?\})\s*```', content, re.DOTALL)
    if match:
        try:
            record = json.loads(match.group(1))
            # Detect empty placeholder (all fields blank/null)
            if record.get("parameter_id") or record.get("parameter_name"):
                results.append(record)
                continue
        except json.JSONDecodeError:
            pass

    # Empty or unparseable Result — extract metadata from file header and emit placeholder
    record = dict(empty_result_template)
    id_match = re.search(r'\*\*Internal ID:\*\*\s*(.+)', content)
    name_match = re.search(r'^# Parameter:\s*(.+)', content, re.MULTILINE)
    cat_match = re.search(r'\*\*Category:\*\*\s*(.+)', content)
    if id_match:
        record["parameter_id"] = id_match.group(1).strip()
    if name_match:
        record["parameter_name"] = name_match.group(1).strip()
    if cat_match:
        # Strip inline bold markers if present (e.g. "**ADAS**" → "ADAS")
        record["category"] = re.sub(r'\*\*', '', cat_match.group(1)).strip()
    results.append(record)

output = {
    "run_metadata": {
        "car": "<brand> <model> <model_year> <market>",
        "resolved_trim": "<resolved_trim>",
        "dataset_id": "<dataset_id>",
        "date": "<YYYY-MM-DD>",
        "total_parameters": len(results),
        "agent_failures": sum(1 for r in results if r.get("_agent_failure"))
    },
    "records": results
}

with open("/mnt/user-data/outputs/results.json", "w", encoding="utf-8") as f:
    json.dump(output, f, indent=2, ensure_ascii=False)

print(f"Aggregated {len(results)} records → results.json ({output['run_metadata']['agent_failures']} agent failures included with empty values)")
```

**Step 2:** Write and execute a second Python script to produce `results.csv` (no traceability):

```python
import json, csv
from pathlib import Path

with open("/mnt/user-data/outputs/results.json", encoding="utf-8") as f:
    data = json.load(f)

columns = ["parameter_id", "parameter_name", "category", "presence", "status", "classification", "confidence", "decision_rule"]

with open("/mnt/user-data/outputs/results.csv", "w", newline="", encoding="utf-8") as f:
    writer = csv.DictWriter(f, fieldnames=columns, extrasaction="ignore")
    writer.writeheader()
    for record in data["records"]:
        # Normalise None → "" so CSV cells are empty not "None"
        row = {col: ("" if record.get(col) is None else record.get(col, "")) for col in columns}
        writer.writerow(row)

print(f"Wrote {len(data['records'])} rows → results.csv")
```

**Step 3:** Verify both files exist and row counts match.

Update `STATE.md` status to `complete`. Report final counts to the user.
