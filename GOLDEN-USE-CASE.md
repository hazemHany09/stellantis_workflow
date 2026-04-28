# Golden Use Case — Automotive Feature Classification (W1 Normal Pipeline)

> **Purpose:** Reference scenario for assessing and auditing the lead agent and subagents. Describes what _should_ happen, step by step, for a standard W1 run. Use this as the benchmark when evaluating actual agent behavior.

---

## Scenario

**Request:** "Classify all features for the 2024 Jeep Grand Cherokee L in the US market. I'll supply a couple of source URLs."

**User-supplied URLs:** `https://www.jeep.com/grand-cherokee-l` and `https://media.stellantis.com/2024-jeep-grand-cherokee-l-specs`

**Workflow:** W1, Mode A (user supplied URLs → source-discovery-with-client-domains).

---

## Part 1 — Lead Agent Lifecycle

### Preflight (before any stage)

**What the lead does:**

1. **Creates `.harness/` folder structure immediately** — before loading any skill or reading any file:
   ```
   .harness/
   ├── downloads/
   ├── Category/
   ├── SubAgent/
   ├── Archive/
   └── advisories/
   ```

2. **Loads foundational skills** into context (they stay for the entire run):
   - `stellantis-domain-context`, `stellantis-decision-rules`, `stellantis-output-contract`
   - `stellantis-workflow-modes`, `stellantis-failure-handling`
   - `stellantis-knowledge-base-protocol`, `stellantis-run-workspace`

3. **Parses request** → extracts:
   - `brand = Jeep`, `model = Grand Cherokee L`, `model_year = 2024`, `market_canonical = US`
   - `client_urls = ["https://www.jeep.com/grand-cherokee-l", "https://media.stellantis.com/2024-jeep-grand-cherokee-l-specs"]`
   - Since URLs are present → **Mode A**

4. **Identifies workflow** via `stellantis-workflow-modes`: W1, Mode A.

5. **Generates run ID:** e.g. `20240415-jeep-grand-cherokee-l-2024-us-a3f9b2`

6. **Initialises `.harness/STATE.md`** with all run metadata.

7. **Resolves trim** via `stellantis-car-trim-resolution` → e.g. `Overland 4x4`. Records `resolved_trim` in STATE.md.

8. **Creates KB dataset** via `create_dataset` → gets back `kb_dataset_id = ds_abc123`. Records in STATE.md.

9. **Freezes reference list** → copies `artifacts/params.csv` → `.harness/params.csv`, records SHA-256 hash in STATE.md.

**STATE.md after preflight:**
```
RunStatus: active
RunStage: source-discovery
run_id: 20240415-jeep-grand-cherokee-l-2024-us-a3f9b2
workflow: W1
mode: A
resolved_trim: Overland 4x4
kb_dataset_id: ds_abc123
params_csv_hash: <sha256>
event_log:
  - [timestamp] Preflight complete, .harness created
  - [timestamp] Trim resolved: Overland 4x4
  - [timestamp] KB dataset created: ds_abc123
  - [timestamp] params.csv frozen, hash: <sha256>
```

---

### Stage 1 — Parse request ✓ (done during preflight)

All four required fields extracted. No pause needed.

---

### Stage 1b — Resolve trim ✓ (done during preflight)

`resolved_trim = "Overland 4x4"` recorded.

---

### Stage 2 — Create KB dataset ✓ (done during preflight)

`kb_dataset_id` recorded.

---

### Stage 3 — Source discovery (Mode A)

**What the lead does:**

1. Loads `stellantis-source-discovery-with-client-domains`.
2. Since both inputs are full URLs (not bare domains), they are tagged `origin = client` immediately.
3. Runs `stellantis-source-validation` on both URLs:
   - Canonicalises each URL (strip tracking params, lowercase host, etc.).
   - Checks for duplicates, paywall signals, and format support.
   - Both URLs pass validation.
4. Writes `.harness/source-candidates.md` with 2 validated entries.

**STATE.md update:**
```
RunStage: awaiting-approval
source_candidates_count: 2
event_log:
  - [timestamp] Source discovery (Mode A) complete: 2 candidates validated
```

---

### Stage 3b — Post candidate list + pause

**What the lead does:**

1. Posts the candidate list markdown to the user (title, excerpt, origin for each URL).
2. Sets STATE.md:
   ```
   RunStatus: paused
   PauseReason: awaiting-source-approval
   PausedAt: [timestamp]
   WorkflowStage: awaiting-approval
   ```
3. Returns control — no polling.

**User sees:** A numbered list of the 2 candidate URLs with metadata, plus instructions: "Reply with `approve all`, `approve 1,2`, or `reject <n>`."

---

### Stage 3c — Parse approval reply

**User replies:** "approve all"

**What the lead does:**

1. Parses reply → both URLs = `Approved`.
2. Writes `.harness/source-approved.md`:
   ```
   1. https://www.jeep.com/grand-cherokee-l — Approved (origin: client)
   2. https://media.stellantis.com/2024-jeep-grand-cherokee-l-specs — Approved (origin: client)
   ```
3. Updates STATE.md:
   ```
   RunStatus: active
   PauseReason: (cleared)
   ResumedAt: [timestamp]
   RunStage: download-and-ingest
   source_approved_count: 2
   ```

---

### Stage 4 — Download and upload

**What the lead does** (per `stellantis-source-download-and-ingest`):

**For URL 1 — `https://www.jeep.com/grand-cherokee-l` (HTML page):**

1. **Cache check:** Look for `.harness/downloads/wwwjeepcomesuv-grand-cherokee-l.md` → not found.
2. **Download:**
   ```
   fetch_url(
       url="https://www.jeep.com/grand-cherokee-l",
       output_dir=".harness/downloads/"
   )
   → returns ".harness/downloads/wwwjeepcomesuv-grand-cherokee-l.md"
   ```
   Lead records the path in STATE.md. **Does NOT read the file content.**
3. **Upload:**
   ```
   ragflow_upload(
       path=".harness/downloads/wwwjeepcomesuv-grand-cherokee-l.md",
       dataset_id="ds_abc123",
       filename="wwwjeepcomesuv-grand-cherokee-l.md",
       metadata={
           "canonical_url": "https://www.jeep.com/grand-cherokee-l",
           "original_url": "https://www.jeep.com/grand-cherokee-l",
           "source_host": "www.jeep.com",
           "origin": "client",
           "doc_type": "html",
           "capture_timestamp": "2024-04-15T10:23:00Z",
           "run_id": "20240415-jeep-grand-cherokee-l-2024-us-a3f9b2",
           "cycle": 1,
           "car_brand": "Jeep",
           "car_model": "Grand Cherokee L",
           "car_model_year": 2024,
           "car_market_input": "US",
           "car_market_canonical": "US",
           "resolved_trim": "Overland 4x4"
       }
   )
   → returns {"id": "doc_001", "name": "wwwjeepcomesuv-grand-cherokee-l.md", "progress": 0.0}
   ```
4. Records `doc_id = doc_001` in STATE.md.

**For URL 2 — `https://media.stellantis.com/2024-jeep-grand-cherokee-l-specs` (inferred as HTML):**

Same pattern → saved as `.harness/downloads/mediastellantiscom2024-jeep-grand-cherokee-l-specs.md`, uploaded → `doc_id = doc_002`.

**STATE.md after downloads + uploads:**
```
RunStage: awaiting-ingestion
source_stats:
  approved: 2
  downloaded: 2
  uploaded: 2
  failed: 0
source_roster:
  - url: https://www.jeep.com/grand-cherokee-l
    local_path: .harness/downloads/wwwjeepcomesuv-grand-cherokee-l.md
    doc_id: doc_001
    upload_status: success
    ingestion_started_at: 2024-04-15T10:24:00Z
  - url: https://media.stellantis.com/2024-jeep-grand-cherokee-l-specs
    local_path: .harness/downloads/mediastellantiscom2024-jeep-grand-cherokee-l-specs.md
    doc_id: doc_002
    upload_status: success
    ingestion_started_at: 2024-04-15T10:24:30Z
event_log:
  - [timestamp] Upload gate passed: 2/2 uploaded successfully
```

---

### Stage 4b — Await ingestion resume

**What the lead does:**

1. Sets STATE.md:
   ```
   RunStatus: paused
   PauseReason: awaiting-client-resume
   WorkflowStage: W1-stage-5-ingestion-polling
   ```
2. Posts to user: *"Sources uploaded and ingestion queued for 2 documents. Send `resume` when you are ready and I will verify ingestion status."*
3. Returns control — does NOT poll RAGFlow during this pause.

---

### Stage 5 — Verify ingestion

**User replies:** `resume`

**What the lead does:**

1. Reads STATE.md → `PauseReason = awaiting-client-resume`, `WorkflowStage = W1-stage-5-ingestion-polling`.
2. Calls `doc_infos(doc_ids=["doc_001", "doc_002"])`:
   - `doc_001` status = `done` → `Ingested`
   - `doc_002` status = `done` → `Ingested`
3. Sets:
   ```
   source_stats.ingested: 2
   ingestion_completed_at: 2024-04-15T10:31:00Z
   RunStage: classification
   RunStatus: active
   ```
4. Emits single message: "Ingestion complete — 2 documents ready. Starting classification."

**STATE.md after ingestion:**
```
RunStatus: active
RunStage: classification
ingestion_completed_at: 2024-04-15T10:31:00Z
source_stats:
  approved: 2 / downloaded: 2 / uploaded: 2 / ingested: 2 / dropped: 0
```

---

### Stage 6 — Partition and spawn subagents

**What the lead does** (per `stellantis-lead-agent-subagent-orchestration`):

1. **Partitions** `.harness/params.csv` by category. Example output (4 categories, ~60 params total):
   | Partition | Category | Param count |
   |-----------|----------|-------------|
   | `powertrain-1` | Powertrain | 14 |
   | `safety-1` | Safety | 15 |
   | `connectivity-1` | Connectivity | 12 |
   | `interior-1` | Interior | 11 |
   | `exterior-1` | Exterior | 8 |

2. **Records subagent roster** in STATE.md.

3. **Pre-seeds working files** for the first 3 subagents (concurrency limit). For `powertrain-1`:
   - Writes `.harness/SubAgent/powertrain-1.md` with contract block:
     ```
     <!-- AUTOGENERATED BY LEAD -->
     {
       "agent_name": "powertrain-1",
       "kb_dataset_id": "ds_abc123",
       "car_identity": {...},
       "resolved_trim": "Overland 4x4",
       "parameters": [...14 powertrain params...],
       "tool_allowlist": ["retrieval", "retrieve_knowledge_graph", "list_chunks",
                          "ragflow_download", "doc_infos", "get_metadata_summary"]
     }
     ```

4. **Spawns 3 subagents concurrently:** `powertrain-1`, `safety-1`, `connectivity-1`.
   - `interior-1` and `exterior-1` are queued.

**STATE.md after spawn:**
```
RunStage: classification
subagent_roster:
  - name: powertrain-1    status: running    params: 14
  - name: safety-1        status: running    params: 15
  - name: connectivity-1  status: running    params: 12
  - name: interior-1      status: queued     params: 11
  - name: exterior-1      status: queued     params: 8
```

---

## Part 2 — Subagent Lifecycle (example: `powertrain-1`)

The subagent receives its working file path. Everything below happens **independently** in a separate agent context.

### Startup

1. Reads identity from working file path stem → `agent_name = "powertrain-1"`.
2. Opens `.harness/SubAgent/powertrain-1.md`, reads the contract block.
3. Cross-checks `agent_name` in JSON == `"powertrain-1"`. ✓
4. Loads required skills: `stellantis-domain-context`, `stellantis-decision-rules`, `stellantis-subagent-classification-loop`.
5. Reads `enums-reference`, `lead-to-subagent-contract`, `subagent-to-lead-contract`.

### Main Loop — Iteration 1 (up to 3 iterations)

**Step 1 — Document relevance scan:**

For each of the 14 powertrain parameters, queries `retrieval` against `ds_abc123`:
```
retrieval(
    query="engine displacement horsepower powertrain Jeep Grand Cherokee L 2024",
    dataset_id="ds_abc123",
    top_k=6
)
→ returns top 6 doc_ids with relevance scores: [doc_001 (0.92), doc_002 (0.87), ...]
```

**Step 2 — Read:**

For `doc_001`:
1. Calls `list_chunks("doc_001")` → gets chunk IDs.
2. **Cache check:** `.harness/downloads/doc_001` not found yet.
3. Calls `ragflow_download(doc_ids=["doc_001"], output_dir=".harness/downloads/")` → saves file, returns path.
4. Reads relevant chunks. Does NOT read the entire file — targets chunks matching parameter descriptions.
5. Classifies each source chunk per decision-rule taxonomy:
   - `clear` — "3.6L Pentastar V6 / 293 hp @ 6400 rpm" → maps to engine capacity level
   - `clear` — "5.7L HEMI V8 / 357 hp" → maps to engine capacity level

**Step 3 — Update working file:**

Appends to `.harness/SubAgent/powertrain-1.md` under Iteration 1 heading:
- Queries issued
- Doc IDs and chunk IDs accessed
- Per-parameter evidence snippets and classification (`clear`/`vague`/`silent`)

**Step 4 — Tentative verdicts (Iteration 1):**

| Parameter | Evidence situation | Rule | Triple |
|-----------|-------------------|------|--------|
| Engine displacement | 2× clear sources agree on V6/V8 options | Rule-2a | `(Yes, Conflict, Empty)` — two valid trims offer different levels |
| Max horsepower | 1× clear source, 357 hp | Rule-1 | `(Yes, Success, Level-4)` |
| EV capability | 1× clear — "no battery/motor" | Rule-5 | `(No, Success, Empty)` |
| … | … | … | … |

Parameters with finalised verdicts (Rule-1, -2a, -2b, -5) are locked. Rule-3/4 candidates go to fallback.

### Fallback Loop (for unresolved params only)

If 2 parameters remain at Rule-4 after main loop:

1. Builds targeted queries using parameter name + description + applicable criteria text.
2. Calls `retrieval` with `top_k=12`.
3. If evidence found → promotes to Rule-1 or Rule-3.
4. Still nothing at end of iteration 3 → Rule-4 emitted.

### Final Output

After all parameters are resolved, the subagent writes the result envelope at the tail of `.harness/SubAgent/powertrain-1.md`:

```
<!-- RESULT ENVELOPE -->
{
  "status": "consolidated-ready",
  "agent_name": "powertrain-1",
  "records": [
    {
      "parameter_id": "PWR-001",
      "presence": "Yes",
      "status": "Success",
      "classification": "Level-4",
      "traceability": [{"doc_id": "doc_001", "chunk_id": "c_0042", "excerpt": "357 hp @ 5600 rpm"}]
    },
    ...
  ],
  "docs_metadata_updates": [
    {"doc_id": "doc_001", "metadata_patch": {"evidenced_parameters": ["PWR-001", "PWR-003"]}}
  ],
  "warnings": [],
  "advisories": [],
  "loaded_skills": ["stellantis-domain-context", "stellantis-decision-rules", "stellantis-subagent-classification-loop"]
}
```

Returns control. **Does NOT touch STATE.md, Category files, or other SubAgent files.**

---

## Part 3 — Lead Agent: Consolidation

### What the lead does when a subagent signals `consolidated-ready`

For `powertrain-1` (first to finish):

1. **Reads** `.harness/SubAgent/powertrain-1.md`.
2. **Validates** every record against the intermediate parameter record schema and enums reference.
3. **Applies metadata patches:** calls `batch_update_doc_metadata` with `docs_metadata_updates[]`.
4. **Merges** validated records into `.harness/Category/Powertrain.md`.
5. **Promotes** any `warnings[]` into the category STATE file's warnings section.
6. **Promotes** `advisories[]` into `.harness/advisories/undefined-tier.md` or `out-of-list-findings.md`.
7. **Archives:** moves `.harness/SubAgent/powertrain-1.md` → `.harness/Archive/powertrain-1.md`.
8. **Updates STATE.md:**
   ```
   subagent_roster:
     - name: powertrain-1    status: archived   records_consolidated: 14
   classification_counts.powertrain:
     yes_success: 8, yes_conflict: 2, no: 2, rule4: 2
   ```
9. **Spawns next queued subagent:** `interior-1` to refill the concurrency slot.

**Serial constraint:** Even if `safety-1` and `connectivity-1` finish simultaneously, the lead consolidates them one at a time.

**STATE.md during consolidation (rolling update):**
```
subagent_roster:
  - powertrain-1:   archived
  - safety-1:       archived
  - connectivity-1: archived
  - interior-1:     running
  - exterior-1:     running
records_consolidated_total: 41 / 60
```

---

## Part 4 — Lead Agent: Deliverable Emission

### Stage 7 — Emit deliverable

Once all 5 subagents are `archived` (or `failed` with lead-written Rule-4 fallbacks):

1. **Assembles `deliverable.json`** from all `.harness/Category/*.md` files plus STATE.md header:
   - Header: run ID, timestamps, car identity, resolved trim, CSV hash, KB dataset ID, source counts, subagents spawned.
   - Legend for enum values.
   - Summary: counts per presence/status/classification distribution.
   - `records[]` in CSV row order — exactly one record per parameter.
   - Footer: `sources_excluded` list (empty in the happy path).

2. **Assembles `deliverable.csv`** — flattened view of the same records.

3. **Writes both files** at project root:
   ```
   jeep-grand-cherokee-l-2024-us-20240415-a3f9b2.json
   jeep-grand-cherokee-l-2024-us-20240415-a3f9b2.csv
   ```

**STATE.md update:**
```
RunStage: deliverable
event_log:
  - [timestamp] deliverable.json written: 60 records
  - [timestamp] deliverable.csv written
```

---

### Stage 8 — Mark complete

```
RunStatus: complete
RunStage: complete
completed_at: 2024-04-15T11:45:00Z
```

Lead posts summary to user with counts and file paths.

---

## Part 5 — Communication Patterns

### How lead and subagents communicate

| Channel | Direction | Mechanism | Notes |
|---------|-----------|-----------|-------|
| Contract block | Lead → Subagent | `.harness/SubAgent/<name>.md` header | Written once by lead at spawn; never edited after. |
| Working notes | Subagent internal | `.harness/SubAgent/<name>.md` scratch section | Subagent-only write target during run. |
| Result envelope | Subagent → Lead | `.harness/SubAgent/<name>.md` tail | `<!-- RESULT ENVELOPE -->` block; lead reads on consolidation. |
| Category records | Lead only | `.harness/Category/<category>.md` | Lead writes after consolidating each envelope. |
| Run state | Lead only | `.harness/STATE.md` | Source of truth for run lifecycle; subagents never read or write this. |
| User chat surface | Lead ↔ User | Chat messages | Pauses, approvals, resume signals, deliverable summary. |

**Subagents never read STATE.md.** They receive all context they need via the contract block at spawn.

**The lead never reads subagent scratch.** It reads only the `<!-- RESULT ENVELOPE -->` block.

---

## Part 6 — STATE.md Lifecycle Summary

| Transition | `RunStatus` | `RunStage` | Key STATE fields set |
|-----------|------------|-----------|---------------------|
| Preflight complete | `active` | `source-discovery` | `run_id`, `workflow`, `mode`, `kb_dataset_id`, `params_csv_hash`, `resolved_trim` |
| Candidates ready | `active` | `awaiting-approval` | `source_candidates_count` |
| **Pause #1 — approval** | `paused` | `awaiting-approval` | `PauseReason = awaiting-source-approval` |
| Approval received | `active` | `download-and-ingest` | `source_approved_count`, `PauseReason` cleared |
| Downloads + uploads complete | `active` | `awaiting-ingestion` | `source_stats.uploaded`, per-URL `doc_id`, `ingestion_started_at` |
| **Pause #2 — resume** | `paused` | `W1-stage-5-ingestion-polling` | `PauseReason = awaiting-client-resume` |
| Resume signal received | `active` | `classification` | `ingestion_completed_at`, `source_stats.ingested` |
| Subagents spawned | `active` | `classification` | `subagent_roster` (all entries pending/running/queued) |
| Each subagent archived | `active` | `classification` | `subagent_roster[i].status = archived`, rolling counts |
| Deliverable written | `active` | `deliverable` | `deliverable_json_path`, `deliverable_csv_path` |
| **Run complete** | `complete` | `complete` | `completed_at` |

---

## Part 7 — Tool Call Summary

| Stage | Actor | Tool | Purpose |
|-------|-------|------|---------|
| Preflight | Lead | `create_dataset` | Create the KB dataset for this run |
| Stage 3 | Lead | `google_search` (Mode B only) | Discover candidate URLs when none supplied |
| Stage 4 | Lead | `fetch_url` | Download each approved URL → `.harness/downloads/` |
| Stage 4 | Lead | `ragflow_upload` | Upload local file to KB with metadata |
| Stage 5 | Lead | `doc_infos` / `list_docs` | Poll ingestion status |
| Stage 6 | Lead | `batch_update_doc_metadata` | Apply subagent-staged metadata patches |
| Stage 6 | Lead | `get_metadata_summary` | Sanity-check after consolidation |
| Stage 6a | Subagent | `retrieval` | Semantic search in KB |
| Stage 6a | Subagent | `list_chunks` | Enumerate document chunks |
| Stage 6a | Subagent | `ragflow_download` | Save KB document to `.harness/downloads/` for reading |
| Stage 6a | Subagent | `doc_infos` | Check doc metadata |
| Stage 6a | Subagent | `get_metadata_summary` | Dataset-level metadata (rare) |

**Cache check (lead and subagent):** Before any download call (`fetch_url` or `ragflow_download`), check `.harness/downloads/` for existing file. Reuse if present.

**Forbidden at all times:**
- Subagents using `fetch_url`, `google_search`, any `browser_render_*`, `ragflow_upload`, or `create_dataset`.
- Lead or subagent using video/YouTube URLs (drop with `unsupported-type` at download stage).
- Any agent reading another's working file as a write target.

---

## Part 8 — Failure Paths (Brief)

| Failure | Stage | What should happen |
|---------|-------|--------------------|
| `fetch_url` fails | Stage 4 | 3 retries, then `sources_excluded.md` entry with `stage=download`. Run continues. |
| `ragflow_upload` fails / 5-min timeout | Stage 4 | 3 retries, then excluded. Upload gate pauses run for user decision. |
| Ingestion `failed` status or 60-min timeout | Stage 5 | Drop doc; continue if ≥1 doc ingested. If all fail, pause for user. |
| Subagent envelope fails validation | Stage 6b | Re-spawn once. On second failure, lead writes Rule-4 for partition. |
| URL is video/YouTube | Stage 4 | Excluded immediately, no retry, `reason=unsupported-type`. |
| KB retrieval returns nothing | Stage 6a | Rule-4 for parameter. No web fallback — ever. |

---

> **Audit note:** In a conforming run, `.harness/STATE.md` must contain a continuous event log covering every state transition listed in Part 6. `.harness/downloads/` must contain one file per successfully downloaded URL. The deliverable's `records[]` count must equal the number of rows in `.harness/params.csv` (excluding header and category-banner rows).
