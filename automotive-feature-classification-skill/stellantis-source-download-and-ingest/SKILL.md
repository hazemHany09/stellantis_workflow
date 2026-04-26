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
  - browser_render_pdf
  - browser_render_content
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

# Sub-skill — Download, upload, ingest

## Overview

Downloads each approved URL to the appropriate file format, uploads to the knowledge-base dataset with structured metadata (which queues each document for ingestion), then pauses for user confirmation before verifying ingestion status against the 60-minute per-document ceiling.

## When this runs

W1 stages 4–5. After the client has approved at least one URL and sent the implicit continuation (approval reply itself). Before the ingestion-verification resume pause.

## Inputs

| Input                | Source                                                       |
| :------------------- | :----------------------------------------------------------- |
| `source-approved.md` | Parsed client approval reply — canonical URLs with `origin`  |
| `kb_dataset_id`      | Main STATE.md                                                |
| `car_identity`       | Main STATE.md — for metadata enrichment                      |

## Tools allowed

* **Download:** `browser_render_markdown`, `browser_render_pdf`, `browser_render_content` (fallback for already-static binaries).
* **Upload + ingestion submission:** `upload_with_metadata` — ingestion is queued by the knowledge base as part of upload.
* **Polling:** `doc_infos`, `list_docs`, `get_metadata_summary`.
* **Cleanup:** `delete_docs` (only to remove a document that ingestion irrecoverably failed on).

## Download decision tree

Given an approved URL, decide file type from the URL + a lightweight HEAD-style probe (or the Content-Type in a short `browser_render_content` call):

| Detected type                          | Action                                                                                                       |
| :------------------------------------- | :----------------------------------------------------------------------------------------------------------- |
| HTML / rendered web page               | `browser_render_markdown` → save as `<slug>.md`.                                                             |
| PDF                                    | `browser_render_pdf` (for rendered-page PDFs) OR direct fetch of the .pdf URL → save as `<slug>.pdf`.        |
| Presentation (`.ppt`, `.pptx`, `.key`) | Direct binary download → save as-is. **Do not** convert to markdown.                                         |
| Spreadsheet (`.xls`, `.xlsx`, `.csv`)  | Direct binary download → save as-is. **Do not** convert to markdown.                                         |
| Image / video                          | Not supported; record in `.harness/Archive/sources_excluded.md` with `stage = download`, reason `unsupported-type`, and skip. |
| Unknown                                | Try `browser_render_markdown`; on failure, direct fetch; on second failure, record as `sources_excluded`.    |

`<slug>` = the canonicalised URL's host + path stripped of punctuation; max 100 chars.

## Retry policy

**3 retries with exponential backoff (1s, 2s, 4s)** for each URL's download. On final failure, add to `.harness/Archive/sources_excluded.md` with `stage = download, reason = <short>` and continue. The run is never aborted for a single failure.

## Upload

For each successfully downloaded file, call `upload_with_metadata` with:

```json
{
  "dataset_id": "<kb_dataset_id>",
  "file_path":  "<local path>",
  "metadata": {
    "original_url":        "<pre-canonical url>",
    "canonical_url":       "<canonical url>",
    "source_host":         "<host>",
    "origin":              "<client | agent>",
    "doc_type":            "<html | pdf | pptx | xlsx | other>",
    "capture_timestamp":   "<ISO-8601 UTC>",
    "run_id":              "<run-id>",
    "cycle":               1,
    "car_brand":           "<brand>",
    "car_model":           "<model>",
    "car_model_year":      "<integer>",
    "car_market_input":    "<string>",
    "car_market_canonical":"<canonical>",
    "resolved_trim":       "<trim>"
  }
}
```

Record the returned `doc_id` in the main STATE.md's source roster.

## Ingestion submission

Each successful `upload_with_metadata` call submits the document for ingestion as part of the upload. Record the returned `doc_id` in the source roster and record the upload timestamp as `ingestion_started_at` in the main STATE.md (use the earliest upload timestamp when uploading multiple documents).

## Pause boundary

Immediately after the final upload call, **pause**:

* `RunStatus = paused`
* `PauseReason = awaiting-client-resume`
* Post a message to the user: *"Ingestion has been queued for N documents. Send `resume` when you are ready and I will verify ingestion status."*

The lead does **not** poll `doc_infos` during this pause. That happens on resume.

## On resume

1. Call `list_docs` or `doc_infos(doc_ids=[…])` for every `doc_id` in the roster.
2. For each document check the parsing/run status reported by the knowledge base. Treat status semantics:
   * `done` → set SourceLifecycle = `Ingested`.
   * `running` → still ingesting.
   * `failed` → set SourceLifecycle = `Dropped_IngestTimeout`, `FailureStage = ingestion`, record in `sources_excluded.md`.
3. Apply the 60-minute per-document ceiling. For a doc still `running`:
   * If `now - ingestion_started_at < 60 minutes` — doc is within budget; pause again with `PauseReason = awaiting-ingestion-completion` and ask the user to wait.
   * If `now - ingestion_started_at ≥ 60 minutes` — drop the doc with `FailureStage = ingestion, reason = timeout-60min` and continue with the rest.
4. If at least one doc is `done` and **all others** are either `done`, `failed`, or `dropped on timeout`, proceed past the gate.
5. If **every** doc is `failed` or timed out, pause with `PauseReason = ingestion-error-client-confirmation-needed` and ask the user whether to wait further or abort.

## Outputs

* `runs/<run-id>/.harness/Archive/sources_excluded.md` updated for any drops.
* Main STATE.md source stats block updated (Ingested / Dropped_IngestTimeout / etc.).
* Per-doc metadata persisted in the knowledge base (queryable via `get_metadata_summary`).

## Success criteria

* Every approved URL has either a live `doc_id` reporting status `done` OR an entry in `sources_excluded.md`.
* `ingestion_started_at` and `ingestion_completed_at` are set in main STATE.md.

## Failure modes

| Failure                                            | Response                                                                   |
| :------------------------------------------------- | :------------------------------------------------------------------------- |
| `upload_with_metadata` fails for a file            | Retry 3×; on final failure record as `stage = download`.                   |
| Document repeatedly reports `failed` after upload  | Mark `Dropped_IngestTimeout`; record in `sources_excluded.md`.             |
| No documents ingest within timeout                 | Pause with `ingestion-error-client-confirmation-needed` and surface the status breakdown. |
