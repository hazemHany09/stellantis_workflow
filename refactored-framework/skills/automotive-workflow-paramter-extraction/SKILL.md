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

Read classification levels (level labels + descriptions per parameter) from the frozen reference.

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

At this point `brand`, `model`, `model_year`, `market`, `resolved_trim`, and `dataset_id` are all known. Spawn one **general-purpose** subagent to write ALL contract files in a single pass with the `## Task` section fully pre-populated. The prompt must include:
- The absolute path to the output directory: `/mnt/user-data/workspace/classification-tasks/`
- The file naming convention: `<parameter_id>-<parameter_name_slug>.md`
- The full contract file template (copy it verbatim from `task-contracts` into the prompt — do **not** tell the agent to load `task-contracts`)
- The full parameter list from `params-frozen.csv` as a file path the agent can read
- Instruction to write one `.md` file per parameter, pre-populating the `## Task` section with:
  - `brand`, `model`, `model_year`, `market`
  - `resolved_trim`
  - `dataset_id`
  - The parameter's `id`, `name`, `category`, and all level definitions from params-frozen.csv
- Instruction to leave the `## Result` section empty (placeholder only)

Do not spawn one agent per parameter here — a single general-purpose agent writes all contracts.

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
| # | URL | source_type | description | doc_id | status |
|---|-----|-------------|-------------|--------|--------|
```

Update `Status` field at each stage transition. Update the `Sources` table whenever a source changes state (`pending` → `approved` → `downloading` → `ingesting` → `ingested`, or `failed: <reason>` at any step).

---

### Stage 7 — Source Discovery

**Before running any searches:** Read and apply the following skills:
1. **Serper skill** — load `serper` skill for operator knowledge, tool selection, and query construction rules.
2. **Deep-research skill** — invoke the `deep-research`.

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

| # | URL | `source_type` | Reliability Tier | Description | Coverage | Notes |
|---|-----|--------------|-----------------|-------------|----------|-------|
| 1 | <url> | `manufacturer_official_spec` | Tier 1 — Manufacturer | Official trim specifications page | Full trim options, standard vs optional | Definitive for trim availability |
| 2 | <url> | `manufacturer_owner_manual` | Tier 1 — Manufacturer | Owner/press manual (PDF) | All features, detailed specs | Highest detail, may be region-specific |
| 3 | <url> | `dealer_fleet_guide` | Tier 2 — Dealer | Dealer fleet order guide | Standard vs optional per trim | Confirms fitment at trim level |
| 4 | <url> | `third_party_press_long_form` | Tier 3 — Editorial | Long-form review from <outlet> | Real-world feature behaviour | Corroborating signal, not definitive |
| 5 | <url> | `third_party_aggregator` | Tier 4 — Third-party | Trim comparison matrix (Edmunds/KBB) | Feature availability across trims | Good for cross-trim checks |
| 6 | <url> | `forum_or_ugc` | Tier 5 — Community | Owner discussion thread | Specific feature questions | Soft signal only — never sole source |

Column definitions:
- **`source_type`:** Canonical source type value (from the Evidence Source Types table above) — use these exact values in `STATE.md` and contract files
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

Spawn one `document-downloader-agent` per approved URL. The task prompt for each agent must include:
- `url`: the source URL to download
- `dataset_id`: the RAGFlow dataset ID for this run
- `source_type`: the canonical source type value for this URL (taken from the `source_type` column in the `## Sources` table in `STATE.md`) — classifier agents will read this directly from `STATE.md` using the `doc_id` returned after ingestion.
- include all the metadata about the source in the prompt to the subagent.

**Per-source status lifecycle:**

1. **Before spawning the agent:** set that source's status in `STATE.md` → `downloading`
2. **The subagent downloads the file and triggers ingestion into RAGFlow:** once the subagent confirms the upload/ingest call was made, set status → `ingesting`

Collect results: success (with `doc_id` and `source_type`) or failed (with reason) so that you can update the `STATE.md` accordingly.

---

### Stage 10 — PAUSE 2: Classification Go-Ahead

**Step 1 — Query ingestion status:** Query RAGFlow for the current status of all documents in the dataset. For each document, update its row in `STATE.md`:
- If RAGFlow confirms parsing complete → set status → `ingested`
- If still processing → leave status as `ingesting`
- If RAGFlow reports an error → set status → `failed: <reason>`

**Step 2 — Present status table to the user:**

```
Ingestion status for <brand> <model> <model_year> <market>:

| # | URL | Source Type | Doc ID | Status | Notes |
|---|-----|-------------|--------|--------|-------|
| 1 | <url> | Official Spec | <doc_id> | ingested | — |
| 2 | <url> | PDF Manual | <doc_id> | ingested | — |
| 3 | <url> | Press Review | — | failed: download error | Could not fetch page |
| 4 | <url> | Aggregator | <doc_id> | ingesting | Still processing |

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

Spawn one `parameter-classifier-agent` per parameter.

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

### Stage 12 — Web Search Fallback (No-Information Parameters)

**Step 1 — Identify no-information parameters:** Spawn one general-purpose agent to scan all contract files in `classification-tasks/`. For each file, the agent reads the `## Result` block and checks whether `presence == "No Information Found"`. The agent returns to the lead agent a list of absolute file paths for every contract that meets this condition.

**Step 2 — Spawn search agents:** For each file path returned in Step 1, spawn one `parameter-searching-agent`. The agent reads the contract file and overwrites the `## Result` block with web-search findings. that's why the inputs to this agent as contract and prompt will be the same as the `parameter-classifier-agent` agent.

---

### Stage 13 — Aggregate Results

After all agents complete:

**Step 1:** Write a Python script inline and execute it to aggregate `## Result` blocks into `results.json`:

```python
import json, re
from pathlib import Path

contracts_dir = Path("/mnt/user-data/workspace/classification-tasks")
results = []

for md_file in sorted(contracts_dir.glob("*.md")):
    if md_file.name == "params-frozen.csv":
        continue
    content = md_file.read_text(encoding="utf-8")
    # Extract JSON block after ## Result
    match = re.search(r'## Result\s*```json\s*(\{.*?\})\s*```', content, re.DOTALL)
    if match:
        record = json.loads(match.group(1))
        results.append(record)

output = {
    "run_metadata": {
        "car": "<brand> <model> <model_year> <market>",
        "resolved_trim": "<resolved_trim>",
        "dataset_id": "<dataset_id>",
        "date": "<YYYY-MM-DD>",
        "total_parameters": len(results)
    },
    "records": results
}

with open("/mnt/user-data/outputs/results.json", "w", encoding="utf-8") as f:
    json.dump(output, f, indent=2, ensure_ascii=False)

print(f"Aggregated {len(results)} records → results.json")
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
        writer.writerow(record)

print(f"Wrote {len(data['records'])} rows → results.csv")
```

**Step 3:** Verify both files exist and row counts match.

Update `STATE.md` status to `complete`. Report final counts to the user.
