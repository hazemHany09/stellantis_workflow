---
name: stellantis-source-download-and-ingest
description: Use when source URLs have been approved by the user and are ready for ingestion into the knowledge base — W1 stages 4–5. Lead context only.
loader: lead
stage: W1-stage-4, W1-stage-5
requires:
  - stellantis-domain-context
  - stellantis-failure-handling
  - stellantis-knowledge-base-protocol
  - stellantis-run-workspace
  - stellantis-source-validation
  - stellantis-approval-gate
provides:
  - KB-dataset-with-documents
  - sources_excluded.md
tools:
  - browser_render_markdown
  - browser_render_content
  - run_python_file
  - upload_with_metadata
  - doc_infos
  - list_docs
  - get_metadata_summary
  - delete_docs
---

## Dependencies

Foundational:
- `stellantis-domain-context`, `stellantis-failure-handling`, `stellantis-knowledge-base-protocol`, `stellantis-run-workspace`.

Hard:
- `stellantis-source-validation` — produces validated candidate list.
- `stellantis-approval-gate` — produces the approved URL set.

Feeds into `stellantis-lead-agent-subagent-orchestration` after ingestion verification.

---

# Sub-skill — Download, save locally, upload, ingest

## Overview

Downloads each approved URL to the appropriate file format, **saves to `.harness/downloads/` for quality verification**, then uploads to the knowledge-base dataset with structured metadata (which queues each document for ingestion). Enforces a **completion gate**: cannot proceed until ALL approved sources are successfully uploaded to the KB (with per-source 5-minute timeout). Then pauses for user confirmation before verifying ingestion status against the 60-minute per-document ceiling.

## When this runs

W1 stages 4–5. After the client has approved at least one URL and sent the implicit continuation (approval reply itself). Before the ingestion-verification resume pause.

## Inputs

| Input                | Source                                                       |
| :------------------- | :----------------------------------------------------------- |
| `source-approved.md` | Parsed client approval reply — canonical URLs with `origin`  |
| `kb_dataset_id`      | `.harness/STATE.md`                                          |
| `car_identity`       | `.harness/STATE.md` — for metadata enrichment                |

## Tools allowed

* **Download:** `browser_render_markdown`, `browser_render_pdf`, `browser_render_content` (fallback for already-static binaries).
* **Upload + ingestion submission:** `upload_with_metadata` — ingestion is queued by the knowledge base as part of upload.
* **Polling:** `doc_infos`, `list_docs`, `get_metadata_summary`.
* **Cleanup:** `delete_docs` (only to remove a document that ingestion irrecoverably failed on).

## Download decision tree

Given an approved URL, detect file type from the URL extension and/or a lightweight `browser_render_content` Content-Type probe:

| Detected type                          | Action                                                                                                       |
| :------------------------------------- | :----------------------------------------------------------------------------------------------------------- |
| HTML / rendered web page               | `browser_render_markdown` → save as `.harness/downloads/<slug>.md`.                                          |
| PDF (`.pdf`)                           | **Python download** — write a script to `fetch_<slug>.py`, run via `run_python_file`, saves binary to `.harness/downloads/<slug>.pdf`. **Do not** use `browser_render_pdf`. |
| Presentation (`.ppt`, `.pptx`, `.key`) | **Python download** — write a script to `fetch_<slug>.py`, run via `run_python_file`, saves binary to `.harness/downloads/<slug>.<ext>`. **Do not** convert to markdown. |
| Spreadsheet (`.xls`, `.xlsx`, `.csv`)  | **Python download** — write a script to `fetch_<slug>.py`, run via `run_python_file`, saves binary to `.harness/downloads/<slug>.<ext>`. **Do not** convert to markdown. |
| Image / video                          | Not supported; record in `.harness/sources_excluded.md` with `stage = download`, reason `unsupported-type`, and skip. |
| Unknown                                | Try `browser_render_markdown`; on failure, Python download to `.harness/downloads/`; on second failure, record as `sources_excluded`. |

`<slug>` = the canonicalised URL's host + path stripped of punctuation; max 100 chars.

### Python download script pattern

For every binary file (PDF, PPTX, XLSX, etc.), write the following script to `.harness/downloads/fetch_<slug>.py` and run it via `run_python_file`. This keeps binary content off the context window entirely.

```python
# .harness/downloads/fetch_<slug>.py
import requests, os, sys

url = "<original_url>"
dest = ".harness/downloads/<slug>.<ext>"

os.makedirs(os.path.dirname(dest), exist_ok=True)
resp = requests.get(url, timeout=60, stream=True)
resp.raise_for_status()
with open(dest, "wb") as f:
    for chunk in resp.iter_content(chunk_size=65536):
        f.write(chunk)
print(f"Downloaded {dest} ({os.path.getsize(dest)} bytes)")
```

Delete the script file after successful execution.

## Retry policy — Download

**3 retries with exponential backoff (1s, 2s, 4s)** for each URL's download to filesystem. On final failure, add to `.harness/sources_excluded.md` with `stage = download, reason = <short>` and continue. The run is never aborted for a single failure.

**Update `.harness/STATE.md` after each download attempt:**
- Log download start (URL, timestamp)
- Log download complete (timestamp, file size, local path)
- Log download failure (timestamp, error reason, retry count)

## Upload process (with 5-minute per-source timeout)

**For each successfully downloaded file:**

1. **Start upload timer** for this URL (5-minute ceiling).
2. **Prepare metadata** with canonical_url, source_host, origin, doc_type, car_identity, etc.
3. **Convert to base64 and upload via Python** — write a script to `.harness/downloads/upload_<slug>.py`, run it via `run_python_file`. The script reads the local file, encodes it as base64, and POSTs it to the RAGFlow MCP API directly. File content never appears in the LLM context window. See the **Python upload script pattern** below.
4. **Log upload attempt** to `.harness/STATE.md`: timestamp, URL, file path.
5. **On success:** Record returned `doc_id` (printed by the script to stdout) in `.harness/STATE.md` source roster. Log: "Upload succeeded, doc_id=<id>".
6. **On failure:**
   - If time remaining < 5 minutes: **Retry up to 3 times** with exponential backoff (1s, 2s, 4s).
   - If 5-minute ceiling exceeded: **Drop the URL.** Record in `.harness/sources_excluded.md` with `stage = upload, reason = timeout-5min`.
   - Log each retry attempt and final decision to `.harness/STATE.md`.

### Python upload script pattern

For **every** file type (`.md`, `.pdf`, `.pptx`, `.xlsx`, etc.), upload via Python so that binary/base64 content never flows through the LLM context:

```python
# .harness/downloads/upload_<slug>.py
import base64, json, os, requests

RAGFLOW_BASE   = os.environ["RAGFLOW_BASE_URL"]   # e.g. http://localhost:9380
RAGFLOW_APIKEY = os.environ["RAGFLOW_API_KEY"]
DATASET_ID     = "<kb_dataset_id>"
FILE_PATH      = ".harness/downloads/<slug>.<ext>"

METADATA = {
    "original_url":         "<original_url>",
    "canonical_url":        "<canonical_url>",
    "source_host":          "<host>",
    "origin":               "<client|agent>",
    "doc_type":             "<html|pdf|pptx|xlsx|other>",
    "capture_timestamp":    "<ISO-8601 UTC>",
    "run_id":               "<run_id>",
    "cycle":                1,
    "car_brand":            "<brand>",
    "car_model":            "<model>",
    "car_model_year":       "<integer>",
    "car_market_input":     "<string>",
    "car_market_canonical": "<canonical>",
    "resolved_trim":        "<trim>"
}

with open(FILE_PATH, "rb") as f:
    content_b64 = base64.b64encode(f.read()).decode()

filename = os.path.basename(FILE_PATH)
url = f"{RAGFLOW_BASE}/api/v1/datasets/{DATASET_ID}/documents"
headers = {"Authorization": f"Bearer {RAGFLOW_APIKEY}", "Content-Type": "application/json"}
payload = {
    "name": filename,
    "blob": content_b64,
    "metadata": METADATA
}

resp = requests.post(url, headers=headers, json=payload, timeout=120)
resp.raise_for_status()
data = resp.json()
doc_id = data["data"]["id"]
print(f"doc_id={doc_id}")
```

The agent reads `doc_id=<id>` from stdout and records it in `.harness/STATE.md`. Delete the upload script after successful execution.

> **Note:** If `RAGFLOW_BASE_URL` / `RAGFLOW_API_KEY` are not in the environment, surface an error immediately — do not attempt upload.

## Upload completion gate (HARD BOUNDARY)

**After processing all approved URLs**, enforce this gate:

1. **Count results:**
   - `uploads_successful` = URLs with returned `doc_id` in STATE.md source roster
   - `uploads_failed` = URLs dropped to `.harness/sources_excluded.md`
   - **Total must equal approved URL count**

2. **Outcome:**
   - **If `uploads_failed == 0`:** All sources uploaded successfully. Continue to ingestion polling.
   - **If `uploads_failed > 0`:** Some sources failed or timed out. **PAUSE and ask user:**
     - Display the failed URL count and reasons
     - Options: (a) Retry failed sources (restart timer at 5 min per source), (b) Proceed to ingestion polling with partial coverage, (c) Abort run

3. **State update:** Log gate decision to `.harness/STATE.md`.

**This gate is non-negotiable:** Classification does not start until this decision is made.

## Ingestion submission & polling

Each successful Python upload script prints `doc_id=<id>` to stdout. The agent reads this value, records it in `.harness/STATE.md` source roster, and records the upload timestamp as `ingestion_started_at` in `.harness/STATE.md` (use the earliest upload timestamp when uploading multiple documents). Ingestion is queued by RAGFlow as part of the upload.

## Pause boundary (after upload gate passes)

**After all uploads complete (successfully or by user decision to proceed):**

* `RunStatus = paused` (update in `.harness/STATE.md`)
* `PauseReason = awaiting-client-resume`
* `WorkflowStage = W1-stage-5-ingestion-polling`
* Post a message to the user: *"Sources uploaded and ingestion queued for N documents. Send `resume` when you are ready and I will verify ingestion status."*

The lead does **not** poll `doc_infos` during this pause. That happens on resume.

## On resume

1. Call `list_docs` or `doc_infos(doc_ids=[…])` for every `doc_id` in the roster.
2. For each document check the parsing/run status reported by the knowledge base. Treat status semantics:
   * `done` → set SourceLifecycle = `Ingested`.
   * `running` → still ingesting.
   * `failed` → set SourceLifecycle = `Dropped_IngestTimeout`, `FailureStage = ingestion`, record in `.harness/sources_excluded.md`.
3. Apply the 60-minute per-document ceiling. For a doc still `running`:
   * If `now - ingestion_started_at < 60 minutes` — doc is within budget; pause again with `PauseReason = awaiting-ingestion-completion` and ask the user to wait.
   * If `now - ingestion_started_at ≥ 60 minutes` — drop the doc with `FailureStage = ingestion, reason = timeout-60min` and continue with the rest.
4. If at least one doc is `done` and **all others** are either `done`, `failed`, or `dropped on timeout`, proceed past the gate.
5. If **every** doc is `failed` or timed out, pause with `PauseReason = ingestion-error-client-confirmation-needed` and ask the user whether to wait further or abort.

## Outputs

* `.harness/sources_excluded.md` updated for any drops (download, upload timeout, ingestion failures).
* `.harness/STATE.md` source stats block updated (Uploaded / Ingested / Dropped_IngestTimeout / etc.).
* `.harness/downloads/` folder populated with all downloaded files (for audit trail).
* Per-doc metadata persisted in the knowledge base (queryable via `get_metadata_summary`).

## Success criteria

* **Upload gate passed:** Every approved URL has either a live `doc_id` reporting status in ingestion queue OR an entry in `.harness/sources_excluded.md` (with stage and reason).
* **Ingestion complete:** `ingestion_started_at` and `ingestion_completed_at` are set in `.harness/STATE.md`.
* **Audit trail:** `.harness/downloads/` contains all downloaded files.

## Failure modes

| Failure                                                      | Response                                                                                  |
| :----------------------------------------------------------- | :---------------------------------------------------------------------------------------- |
| `browser_render_markdown` fails for HTML URL (3 retries)     | Drop; record as `stage = download`.                                                       |
| Python download script fails or raises (3 retries)           | Drop; record as `stage = download`.                                                       |
| Python upload script fails or raises (3 retries or 5 min)   | Drop; record as `stage = upload, reason = timeout-5min` or error message.                 |
| `RAGFLOW_BASE_URL` / `RAGFLOW_API_KEY` missing in env        | Surface error immediately; do not attempt any uploads; ask user to fix environment.       |
| Upload gate triggered (some uploads failed)                  | Pause and ask user: retry, proceed partial, or abort.                                     |
| Document repeatedly reports `failed` after upload            | Mark `Dropped_IngestTimeout`; record in `.harness/sources_excluded.md`.                   |
| No documents ingest within timeout                           | Pause with `ingestion-error-client-confirmation-needed` and surface the status breakdown. |

