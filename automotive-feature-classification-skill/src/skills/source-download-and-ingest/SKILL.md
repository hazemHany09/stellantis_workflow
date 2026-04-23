---
name: source-download-and-ingest
description: Lead-agent sub-skill that takes the approved URL set, downloads each URL appropriately by type (HTML → Markdown via browser; PDF → direct PDF; presentation/xlsx → direct file), uploads every downloaded file into the run's RAGFlow dataset with metadata, triggers `run_document` on the full set, and polls `doc_infos` until every document has reached Ingested or the per-document 60-minute ceiling has expired.
loader: lead
---

# Sub-skill — Download, upload, ingest

## When this runs

W1 stages 4–5. After the client has approved at least one URL and sent the implicit continuation (approval reply itself). Before the ingestion-verification resume pause.

## Inputs

* `source-approved.md` (parsed client reply) — canonical URLs with `origin`.
* `kb_dataset_id` from main STATE.md.
* `car_identity` for metadata enrichment.

## Tools allowed

* **Download:** `browser_render_markdown`, `browser_render_pdf`, `browser_render_content` (fallback), plus plain HTTP-like fetch via `browser_render_content` when the file is already a static binary.
* **Upload:** `upload_with_metadata`.
* **Ingestion trigger:** `run_document`.
* **Polling:** `doc_infos`, `list_docs`, `get_metadata_summary`.

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

Per Q-KB-4: **3 retries with exponential backoff (1s, 2s, 4s)** for each URL's download. On final failure, add to `.harness/Archive/sources_excluded.md` with `stage = download, reason = <short>` and continue. The run is never aborted for a single failure.

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

## Ingestion trigger

After **every** approved URL has been uploaded (or accounted for as failed), call `run_document` **once** with the full list of returned `doc_ids`:

```json
{
  "doc_ids":   ["<doc_id_1>", "<doc_id_2>", "..."],
  "run":       "1",
  "delete":    false,
  "apply_kb":  false
}
```

Record the call time as `ingestion_started_at` in the main STATE.md.

## Pause boundary

Immediately after the `run_document` call, **pause**:

* `RunStatus = paused`
* `PauseReason = awaiting-client-resume`
* Post a message to the user: *"Ingestion has been triggered for N documents. Send `resume` when you are ready and I will verify ingestion status."*

The lead does **not** poll `doc_infos` during this pause. That happens on resume.

## On resume

1. Call `list_docs` or `doc_infos(doc_ids=[…])` for every `doc_id` in the roster.
2. For each document check `run_status` (RAGFlow reports one of: running, done, failed). Treat status semantics:
   * `done` → set SourceLifecycle = `Ingested`.
   * `running` → still ingesting.
   * `failed` → set SourceLifecycle = `Dropped_IngestTimeout`, `FailureStage = ingestion`, record in `sources_excluded.md`.
3. Apply the 60-minute per-document ceiling from Q-KB-5. For a doc still `running`:
   * If `now - ingestion_started_at < 60 minutes` — doc is within budget; pause again with `PauseReason = awaiting-ingestion-completion` and ask the user to wait.
   * If `now - ingestion_started_at ≥ 60 minutes` — drop the doc with `FailureStage = ingestion, reason = timeout-60min` and continue with the rest.
4. If at least one doc is `done` and **all others** are either `done`, `failed`, or `dropped on timeout`, proceed past the gate.
5. If **every** doc is `failed` or timed out, pause with `PauseReason = ingestion-error-client-confirmation-needed` and ask the user whether to wait further or abort (business-logic hard-constraint 3).

## Outputs

* `runs/<run-id>/.harness/Archive/sources_excluded.md` updated for any drops.
* Main STATE.md source stats block updated (Ingested / Dropped_IngestTimeout / etc.).
* Per-doc metadata persisted in RAGFlow (queryable via `get_metadata_summary`).

## Success criteria

* Every approved URL has either a live `doc_id` with `run_status = done` OR an entry in `sources_excluded.md`.
* `ingestion_started_at` and `ingestion_completed_at` are set in main STATE.md.

## Failure modes

| Failure                                            | Response                                                                   |
| :------------------------------------------------- | :------------------------------------------------------------------------- |
| `upload_with_metadata` fails for a file            | Retry 3× per Q-KB-4; on final failure record as `stage = download`.         |
| `run_document` returns `{"success": false}`        | Retry once with the same payload; on second failure, abort the run with `RunStatus = failed`. |
| No documents ingest within timeout                 | Pause with `ingestion-error-client-confirmation-needed` and surface the status breakdown. |

## Business-logic trace

ARCH-4, AS-KB-G, Q-KB-2, Q-KB-4, Q-KB-5, Q-KB-6 (cycle field), Q-KB-7, AS-DEL-C, hard-constraint 3 in `07-tools-and-constraints.md`.
