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
  - fetch_url
  - ragflow_upload
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

Downloads each approved URL to `.harness/downloads/` using `fetch_url`, then uploads to the knowledge-base dataset via `ragflow_upload` with structured metadata (which queues each document for ingestion). File content never flows through the LLM context — `fetch_url` saves to disk and returns the path; `ragflow_upload` reads from disk and handles encoding internally. Enforces a **completion gate**: cannot proceed until ALL approved sources are successfully uploaded to the KB (with per-source 5-minute timeout). Then pauses for user confirmation before verifying ingestion status against the 60-minute per-document ceiling.

## When this runs

W1 stages 4–5. After the client has approved at least one URL and sent the implicit continuation (approval reply itself). Before the ingestion-verification resume pause.

## Inputs

| Input                | Source                                                       |
| :------------------- | :----------------------------------------------------------- |
| `source-approved.md` | Parsed client approval reply — canonical URLs with `origin`  |
| `kb_dataset_id`      | `.harness/STATE.md`                                          |
| `car_identity`       | `.harness/STATE.md` — for metadata enrichment                |

## Tools allowed

* **Download:** `fetch_url` — fetches any URL and saves to `.harness/downloads/`. Handles both web page rendering (→ Markdown) and binary file download (PDF, DOCX, XLSX, PPTX). Never reads the saved content into context.
* **Upload + ingestion submission:** `ragflow_upload` — reads file from disk, handles encoding internally, uploads to the KB dataset with metadata. Returns JSON with `id` (= `doc_id`).
* **Polling:** `doc_infos`, `list_docs`, `get_metadata_summary`.
* **Cleanup:** `delete_docs` (only to remove a document that ingestion irrecoverably failed on).

## Download decision tree

Before calling `fetch_url` for any URL, **check the cache first**: derive `<slug>` from the canonicalised URL's host + path (stripped of punctuation, max 100 chars). If `.harness/downloads/<slug>.<ext>` already exists, skip the download and use the cached file — log "Cache hit: reusing existing file".

Given an approved URL, detect file type from the URL extension:

| Detected type                          | `fetch_url` call                                                                                  | Saved as                                 |
| :------------------------------------- | :------------------------------------------------------------------------------------------------ | :--------------------------------------- |
| HTML / rendered web page (no extension or `.html`, `.htm`) | `fetch_url(url=url, output_dir=".harness/downloads/")` — auto-detects as webpage | `.harness/downloads/<slug>.md`           |
| PDF (`.pdf`)                           | `fetch_url(url=url, output_dir=".harness/downloads/", type="pdf")`                                | `.harness/downloads/<slug>.pdf`          |
| Word document (`.doc`, `.docx`)        | `fetch_url(url=url, output_dir=".harness/downloads/", type="docx")`                               | `.harness/downloads/<slug>.docx`         |
| Spreadsheet (`.xls`, `.xlsx`)          | `fetch_url(url=url, output_dir=".harness/downloads/", type="xlsx")`                               | `.harness/downloads/<slug>.xlsx`         |
| Presentation (`.ppt`, `.pptx`)         | `fetch_url(url=url, output_dir=".harness/downloads/", type="pptx")`                               | `.harness/downloads/<slug>.pptx`         |
| **Video / YouTube / streaming media**  | **Not supported.** Record in `.harness/sources_excluded.md` with `stage = download`, reason `unsupported-type`, and skip immediately. Do not attempt `fetch_url`. | — |
| Unknown extension                      | `fetch_url(url=url, output_dir=".harness/downloads/")` (defaults to webpage); on failure, try with explicit `type="pdf"`; on second failure, record as `sources_excluded`. | `.harness/downloads/<slug>.md` or `.pdf` |

`<slug>` = the canonicalised URL's host + path stripped of punctuation; max 100 chars.

`fetch_url` returns the saved file path (a `/mnt/user-data/workspace/...` virtual path). The lead agent records this path in `.harness/STATE.md` — it never reads the file content into the context window.

## Retry policy — Download

**3 retries with exponential backoff (1s, 2s, 4s)** for each URL's `fetch_url` call. On final failure, add to `.harness/sources_excluded.md` with `stage = download, reason = <short>` and continue. The run is never aborted for a single failure.

**Update `.harness/STATE.md` after each download attempt:**
- Log download start (URL, timestamp)
- Log download complete (timestamp, file size, local path)
- Log download failure (timestamp, error reason, retry count)

## Upload process (with 5-minute per-source timeout)

**For each successfully downloaded file:**

1. **Start upload timer** for this URL (5-minute ceiling).
2. **Prepare metadata** dict (see fields below).
3. **Call `ragflow_upload`** directly — no Python script needed; `ragflow_upload` handles base64 encoding internally:
   ```
   ragflow_upload(
       path=".harness/downloads/<slug>.<ext>",
       dataset_id=<kb_dataset_id>,
       filename="<slug>.<ext>",
       metadata={...}
   )
   ```
4. **Log upload attempt** to `.harness/STATE.md`: timestamp, URL, file path.
5. **On success:** Read `id` from the returned JSON; record as `doc_id` in `.harness/STATE.md` source roster. Log: "Upload succeeded, doc_id=<id>".
6. **On failure:**
   - If time remaining < 5 minutes: **Retry up to 3 times** with exponential backoff (1s, 2s, 4s).
   - If 5-minute ceiling exceeded: **Drop the URL.** Record in `.harness/sources_excluded.md` with `stage = upload, reason = timeout-5min`.
   - Log each retry attempt and final decision to `.harness/STATE.md`.

### Required metadata fields

Pass these as the `metadata` dict to `ragflow_upload`:

| Field | Value |
| :---- | :---- |
| `original_url` | Pre-canonicalisation URL as flagged |
| `canonical_url` | After canonicalisation rules |
| `source_host` | Website host |
| `origin` | `client` or `agent` |
| `doc_type` | `html`, `pdf`, `docx`, `xlsx`, `pptx` |
| `capture_timestamp` | ISO-8601 UTC when `fetch_url` was called |
| `run_id` | Owning run ID |
| `cycle` | `1` for original ingestion |
| `car_brand` | From STATE.md |
| `car_model` | From STATE.md |
| `car_model_year` | Integer, from STATE.md |
| `car_market_input` | From STATE.md |
| `car_market_canonical` | Canonical market code |
| `resolved_trim` | From STATE.md |

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

Each successful `ragflow_upload` call returns a JSON string with `id` (the `doc_id`). The agent reads this value, records it in `.harness/STATE.md` source roster, and records the upload timestamp as `ingestion_started_at` in `.harness/STATE.md` (use the earliest upload timestamp when uploading multiple documents). Ingestion is queued by RAGFlow as part of the upload.

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
* `.harness/downloads/` folder populated with all downloaded files (for audit trail and quality verification).
* Per-doc metadata persisted in the knowledge base (queryable via `get_metadata_summary`).

## Success criteria

* **Upload gate passed:** Every approved URL has either a live `doc_id` reporting status in ingestion queue OR an entry in `.harness/sources_excluded.md` (with stage and reason).
* **Ingestion complete:** `ingestion_started_at` and `ingestion_completed_at` are set in `.harness/STATE.md`.
* **Audit trail:** `.harness/downloads/` contains all downloaded files.

## Failure modes

| Failure                                                      | Response                                                                                  |
| :----------------------------------------------------------- | :---------------------------------------------------------------------------------------- |
| `fetch_url` fails for HTML URL (3 retries)                   | Drop; record as `stage = download`.                                                       |
| `fetch_url` fails for binary URL (3 retries)                 | Drop; record as `stage = download`.                                                       |
| URL is a video / YouTube / streaming media                   | Drop immediately; record as `stage = download, reason = unsupported-type`. No retries.   |
| `ragflow_upload` fails or raises (3 retries or 5 min)        | Drop; record as `stage = upload, reason = timeout-5min` or error message.                 |
| Upload gate triggered (some uploads failed)                  | Pause and ask user: retry, proceed partial, or abort.                                     |
| Document repeatedly reports `failed` after upload            | Mark `Dropped_IngestTimeout`; record in `.harness/sources_excluded.md`.                   |
| No documents ingest within timeout                           | Pause with `ingestion-error-client-confirmation-needed` and surface the status breakdown. |
