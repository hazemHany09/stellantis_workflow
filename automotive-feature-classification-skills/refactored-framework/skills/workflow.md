---
name: workflow
description: Lead agent entry point. Business domain summary + full 12-stage classification workflow. Load this skill first at the start of every run.
---

# Automotive Feature Classification — Workflow

## Business Domain Summary

**What this is:** You are automating a manual research task performed by an automotive analyst. For one car — identified by `(brand, model, model_year, market)` — you classify ~160 feature parameters against the manufacturer's full-option trim. For each parameter you answer: is the feature present, and if so, at which implementation level (Basic / Medium / High)?

**Parameters and categories:** Parameters are grouped into functional categories (e.g. ADAS, Infotainment, Comfort, EV, Safety). Each parameter has up to three classification levels; the criteria for each level are defined in the reference list (params.csv). Levels are cumulative — High implies Medium implies Basic.

**What makes a source reliable:**
1. Manufacturer official spec / configurator (highest — definitive on trim availability)
2. Dealer / fleet order guide (defines standard vs optional per trim)
3. Long-form press reviews from established outlets (behaviour, real-world performance)
4. Third-party aggregators like Edmunds/KBB (trim availability matrices)
5. Owner forums / UGC (soft signal only — never sufficient alone for a verdict)

**Common edge cases to watch for:**
- **Trim-specific features:** A feature may be standard on the top trim but optional or absent on lower trims. Always anchor to the resolved full-option trim.
- **Market-specific availability:** Features standard in the US may be absent or different in EU/Middle East markets. Market `canonical` field governs.
- **Optional vs standard confusion:** A source saying "available with package X" means optional, not standard. Sources must explicitly confirm standard fitment for a Standard classification.
- **Feature renamed across markets:** Manufacturer may use different names in different markets (e.g. "Traffic Jam Assist" vs "Highway Driving Assist"). Use alias queries.
- **Package bundles:** A feature may only be listed as part of a bundle, not individually. Confirm the resolved trim includes the bundle.

**When something seems wrong:** Use this domain summary to reason through anomalies:
- Source contradicts another? Check source types — manufacturer spec outranks press review.
- Feature missing from all sources? Check if it's market-specific or trim-specific.
- Level criteria ambiguous? Default to the most conservative (lower) level — do not upgrade without clear evidence.
- No sources at all? Confirm the KB was ingested correctly before declaring no-information.

---

## Required Inputs

| Input | Required | Notes |
|-------|----------|-------|
| `brand` | Yes | Never default or guess |
| `model` | Yes | Never default or guess |
| `model_year` | Yes | Never default or guess |
| `market` | Yes | Never default or guess |
| `parameters` | No | If omitted, use all params in params.csv |
| `domains` / `urls` | No | If omitted, run web search for sources |
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

### Stage 4 — Create Classification Task Files

Create the `classification-tasks/` directory.

Spawn one **general-purpose** subagent to write ALL contract files in a single pass. Pass it:
- The frozen parameter list (or params-frozen.csv path)
- The contract file template from `task-contracts` skill
- Instruction to write one `.md` file per parameter

Do not spawn one agent per parameter here — a single general-purpose agent writes all contracts.

---

### Stage 5 — Resolve Trim

If `resolved_trim` was provided by the user: use it directly.

If not: use Serper to find the manufacturer's published full-option trim for `(brand, model, model_year, market)`. Search:
```
"<brand> <model> <model_year> <market> top trim full option"
site:<brand-official-site>.com <model> <model_year> configurator
```
Confirm the trim name from an official source. Record it as `resolved_trim`.

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
```

Update `Status` field at each stage transition.

---

### Stage 7 — Source Discovery

**If domains or URLs were provided by user:** Use them directly as candidate sources. Skip web search.

**If no domains/URLs:** Use Serper to find the most promising sources. Run at minimum:

```
"<brand> <model> <model_year>" "<resolved_trim>" specifications site:<brand>.com
"<brand> <model> <model_year>" owner manual filetype:pdf
"<brand> <model> <model_year>" "<market>" review (site:motortrend.com OR site:caranddriver.com OR site:thecarguide.com)
"<brand> <model> <model_year>" features standard optional trim (site:edmunds.com OR site:kbb.com)
"<brand> <model> <model_year>" "<market>" dealer order guide
```

Compile a candidate URL list. Remove duplicates, paywalled pages, and forum-only sources (keep at least one forum source if nothing else is available for a category).

---

### Stage 8 — PAUSE 1: Source Approval

Present the candidate URL list to the user:

```
Found N candidate sources for <brand> <model> <model_year> <market>:

1. <url> — <source_type> — <brief description>
2. <url> — <source_type> — <brief description>
...

Please review and reply with approval (e.g. "approved", "approved — remove #3", or add URLs).
```

**STOP. Wait for user reply before proceeding.**

On approval: parse the reply for removals or additions. Update the candidate list accordingly.

---

### Stage 9 — Download and Ingest

Spawn one `document-downloader-agent` per approved URL.

Maximum **10 concurrent agents**. Manage in batches:
- Spawn batch of 10 → wait for completion → spawn next batch

Collect results: success (with `doc_id`) or excluded (with reason).

Update `STATUS.md` with ingestion summary.

---

### Stage 10 — PAUSE 2: Classification Go-Ahead

Inform the user ingestion is complete:

```
Ingestion complete.
- <N> documents successfully ingested
- <M> documents excluded: <list with reasons>

Dataset ID: <dataset_id>

Reply to start classification.
```

**STOP. Wait for user reply before proceeding.**

---

### Stage 11 — Classify Parameters

Spawn one `parameter-classifier-agent` per parameter.

Maximum **10 concurrent agents**. Inject `## Task` into each contract file before spawning (include `dataset_id`, `resolved_trim`, car identity). Spawn in batches of 10.

After each batch: scan contract files for completed `## Result` blocks.

---

### Stage 12 — Web Search Fallback (No-Information Parameters)

After all classifier batches complete, identify parameters where `presence == "No Information Found"`.

For each no-info parameter: spawn one `parameter-searching-agent` (max 10 concurrent). The agent reads the same contract file and overwrites the `## Result` block with web-search findings.

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
