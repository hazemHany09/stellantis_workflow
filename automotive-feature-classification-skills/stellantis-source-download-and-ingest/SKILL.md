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
  - ragflow_run_ingestion
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

# Sub-skill — Download, upload, and ingest via subagents

## Overview

Downloads each approved URL using **download subagents** (one subagent per URL). Each subagent calls `fetch_url` to save the file to `.harness/downloads/`, performs file-size-based block detection (Cloudflare rate-limit vs access-denied), then calls `ragflow_upload` to submit the file to the KB dataset with structured metadata. File content never flows through the LLM context beyond the small peek required for block detection.

After all subagents report back, the lead:
1. Consolidates results into STATE.md source download tracking.
2. Reports access-denied sources (permanent flag, no retry).
3. Reports rate-limited sources to the user; awaits retry decision.
4. Runs auto-retries for network failures (no user prompt needed).
5. Auto-flags any URL that has been attempted **3 or more times** as invalid without prompting.
6. Calls `ragflow_run_ingestion` for **all successfully uploaded doc_ids at once**.
7. Pauses for user resume before verifying ingestion status.

## When this runs

W1 stages 4–5. After the client has approved at least one URL. Before the ingestion-verification resume pause.

## Inputs

| Input                | Source                                                       |
| :------------------- | :----------------------------------------------------------- |
| `source-approved.md` | Parsed client approval reply — canonical URLs with `origin`  |
| `kb_dataset_id`      | `.harness/STATE.md`                                          |
| `car_identity`       | `.harness/STATE.md` — for metadata enrichment                |

## Tools allowed

| Tool | Actor | Purpose |
| :--- | :---- | :------ |
| `fetch_url` | Download subagent | Fetch URL → save to `.harness/downloads/` |
| `ragflow_upload` | Download subagent | Upload saved file to KB dataset |
| `ragflow_run_ingestion` | Lead only | Trigger parse/embed pipeline for all uploaded docs at once |
| `doc_infos` | Lead only | Poll ingestion status per doc_id |
| `list_docs` | Lead only | List all docs in dataset |
| `get_metadata_summary` | Lead only | Sanity check after ingestion |
| `delete_docs` | Lead only | Remove docs where ingestion irrecoverably failed |

---

## Phase 1 — Pre-seed STATE.md download tracking

Before dispatching any subagent, the lead initialises the **Source download tracking** section in `.harness/STATE.md` for every approved URL:

Set `Attempts = 0`, `Status = pending` for every approved URL. Append event-log line: `"Download phase started — N URLs to process"`.

See `STATE.main.md.tmpl` for the table format.

---

## Phase 2 — Dispatch download subagents

The lead spawns **one download subagent per approved URL** (max 5 concurrent; queue the rest).

### Subagent working file

Pre-create `.harness/DownloadAgent/<slug>-dl.md` before spawning, using this structure:

```
<!-- DOWNLOAD CONTRACT -->
{
  "subagent_name": "<slug>-dl",
  "canonical_url": "<url>",
  "file_type": "<html|pdf|docx|xlsx|pptx|unknown>",
  "slug": "<slug>",
  "output_dir": ".harness/downloads/",
  "dataset_id": "<kb_dataset_id>",
  "attempt_number": <n>,
  "metadata": {
    "original_url": "...",
    "canonical_url": "...",
    "source_host": "...",
    "origin": "client|agent",
    "doc_type": "html|pdf|docx|xlsx|pptx",
    "capture_timestamp": "<ISO-8601 UTC>",
    "run_id": "...",
    "cycle": 1,
    "car_brand": "...",
    "car_model": "...",
    "car_model_year": <year>,
    "car_market_input": "...",
    "car_market_canonical": "...",
    "resolved_trim": "..."
  }
}
<!-- END DOWNLOAD CONTRACT -->
```

Pass the absolute path of the working file to the subagent at spawn as its first argument. The subagent's name is the filename stem (`<slug>-dl`).

### Download subagent actions (per subagent)

The subagent reads its contract from the working file and executes the following steps.

#### Step A — Cache check

Derive `<slug>` from the canonicalised URL's host + path (stripped of punctuation, max 100 chars). If `.harness/downloads/<slug>.<ext>` already exists **and** its size is > 10 KB (i.e. not a prior blocked download), skip `fetch_url` — log "Cache hit: reusing `<path>`" in scratch — and proceed directly to Step D (upload).

#### Step B — Fetch URL

Detect file type from the URL extension and call `fetch_url`:

| Detected type | `fetch_url` call | Saved as |
| :------------ | :--------------- | :------- |
| HTML / no extension / `.html` / `.htm` | `fetch_url(url=url, output_dir=".harness/downloads/")` | `<slug>.md` |
| PDF (`.pdf`) | `fetch_url(url=url, output_dir=".harness/downloads/", type="pdf")` | `<slug>.pdf` |
| Word (`.doc`, `.docx`) | `fetch_url(url=url, output_dir=".harness/downloads/", type="docx")` | `<slug>.docx` |
| Spreadsheet (`.xls`, `.xlsx`) | `fetch_url(url=url, output_dir=".harness/downloads/", type="xlsx")` | `<slug>.xlsx` |
| Presentation (`.ppt`, `.pptx`) | `fetch_url(url=url, output_dir=".harness/downloads/", type="pptx")` | `<slug>.pptx` |
| **Video / YouTube / streaming media** | **Not supported.** Write result with `status = unsupported-type`. Stop immediately. Do not attempt `fetch_url`. | — |
| Unknown extension | Try `fetch_url` without `type`; on failure try `type="pdf"`; on second failure write `status = failed`. | — |

`fetch_url` returns the saved file path. The subagent derives `size_bytes` from the returned path's file stat (or from the return value if it includes size metadata).

**Retry policy (within the subagent):** 3 retries with exponential backoff (1 s, 2 s, 4 s). On all retries exhausted, write result with `status = failed, failure_reason = fetch-exhausted`.

#### Step C — Block detection (HTML/markdown files only)

After a successful `fetch_url`, apply this check **only** when the saved file is `.md` (i.e. a Markdown conversion of a web page):

1. **Get `size_bytes`** from the `fetch_url` return value.
2. **If `size_bytes < 10240` (10 KB):** The page is suspiciously small for real content. Read the **first 500 characters** of the file.
3. **Scan for markers:**

   | Marker type | Trigger strings (case-insensitive) |
   | :---------- | :--------------------------------- |
   | Rate-limit (Cloudflare / throttle) | `"just a moment"`, `"cf-challenge"`, `"ray id:"`, `"checking your browser"`, `"enable javascript and cookies"`, `"cf_chl"`, `"attention required"`, `"ddos protection"` |
   | Access-denied | `"403 forbidden"`, `"access denied"`, `"not authorized"`, `"401 unauthorized"`, `"you don't have permission"`, `"permission denied"`, `"forbidden"` |

4. **Classify:**
   - Rate-limit markers found → `block_type = rate-limited`.
   - Access-denied markers found → `block_type = access-denied`.
   - Size < 10 KB but no known markers → `block_type = rate-limited` (default assumption; Cloudflare pages vary).
   - Size ≥ 10 KB → `block_type = null` (file appears valid).

5. **If `block_type` is non-null:** Stop. Write result envelope (Step E) with `status = <block_type>` and **do not call `ragflow_upload`**.

#### Step D — Upload (non-blocked files only)

For files where block detection passed (`block_type = null`) or the file is a non-HTML binary:

```
ragflow_upload(
    path=".harness/downloads/<slug>.<ext>",
    dataset_id=<kb_dataset_id>,
    filename="<slug>.<ext>",
    metadata={ ... }   ← from contract
)
```

- **On success:** Record returned `doc_id`.
- **On failure:** Retry up to 3 times (1 s, 2 s, 4 s). On exhaustion: `status = failed, failure_reason = upload-exhausted`.

#### Step E — Write result envelope

The subagent writes its result into the working file:

```
<!-- DOWNLOAD RESULT -->
{
  "subagent_name": "<slug>-dl",
  "canonical_url": "<url>",
  "status": "uploaded | rate-limited | access-denied | unsupported-type | failed",
  "doc_id": "<id or null>",
  "file_path": "<path or null>",
  "file_size_bytes": <n or null>,
  "block_type": "rate-limited | access-denied | null",
  "block_markers": ["<marker1>", ...],
  "failure_reason": "<reason or null>",
  "attempt_number": <n>
}
<!-- END DOWNLOAD RESULT -->
```

---

## Phase 3 — Lead consolidation of download results

After all download subagents for the current attempt have written their result envelope (lead detects presence of `<!-- DOWNLOAD RESULT -->` block in each working file):

1. **Read each result block** from `.harness/DownloadAgent/*.md`.
2. **Update STATE.md source download tracking** for each URL: set `Attempts`, `Last Failure Reason`, `File Size (bytes)`, `Local Path`, `Status`.
3. **Archive subagent working files** to `.harness/Archive/<slug>-dl.md`. Update status in archive.
4. **Categorise results:**
   - `status = uploaded` → has `doc_id`; ready for ingestion.
   - `status = rate-limited` → add to **rate-limited queue** (if `attempt_number < 3`).
   - `status = access-denied` → add to **access-denied list**.
   - `status = unsupported-type` → immediately drop.
   - `status = failed` → add to **retry queue** (if `attempt_number < 3`); otherwise auto-flag.

### Access-denied sources — immediate permanent flag

For every URL with `status = access-denied`:

1. Set `Status = access-denied` in STATE.md source download tracking.
2. Record in `.harness/sources_excluded.md`: `failure_stage = download, failure_reason = access-denied`.
3. **Report to user** once (consolidated block):
   > *"The following N source(s) returned access-denied and cannot be retrieved: `<url1>`, `<url2>`, …"*
4. **Do not retry.** Access-denied sources are permanently excluded.

### Rate-limited sources — user prompt and retry decision

For every URL with `status = rate-limited` AND `attempt_number < 3`:

1. Set `Status = rate-limited` in STATE.md source download tracking.
2. **Report to user** with a consolidated summary:
   > *"N source(s) appear to be rate-limited by Cloudflare or the server (attempt <n>/3). Would you like to retry them? `<url1>`, `<url2>`, …"*
3. **Await user decision:**
   - User approves retry (any affirmative) → add URLs to the **retry queue**.
   - User declines → drop. Record `failure_reason = rate-limited-user-declined` in `.harness/sources_excluded.md`.

For every URL with `status = rate-limited` AND `attempt_number ≥ 3`:

1. **Do not prompt the user.** Auto-flag immediately.
2. Set `Status = auto-flagged-invalid` in STATE.md source download tracking.
3. Record `failure_reason = rate-limited-max-retries` in `.harness/sources_excluded.md`.
4. Append event-log line: `"<url> auto-flagged invalid after 3 rate-limited attempts"`.

### Network-failed sources — auto-retry (no user prompt)

For every URL with `status = failed` AND `attempt_number < 3`:

1. Add to **retry queue** automatically (no user prompt needed).

For every URL with `status = failed` AND `attempt_number ≥ 3`:

1. Set `Status = auto-flagged-invalid`.
2. Record `failure_reason = download-failed-max-retries`.

### Retry loop

For each URL in the retry queue (rate-limited + failed, user-approved where required):

1. Increment `Attempts` counter in STATE.md source download tracking.
2. Re-seed `.harness/DownloadAgent/<slug>-dl.md` with updated `attempt_number`.
3. Re-dispatch download subagent.
4. On completion, **re-run Phase 3 consolidation** for retried URLs only.
5. Repeat until the URL succeeds, is access-denied, or has `attempt_number ≥ 3`.

---

## Phase 4 — Upload completion gate

After all retry loops settle (no more URLs in any retry or rate-limited queue):

1. **Count results:**
   - `uploads_successful` = URLs with a live `doc_id` in STATE.md.
   - `uploads_failed` = URLs in `.harness/sources_excluded.md`.
   - **Total must equal approved URL count.**

2. **Outcome:**
   - **`uploads_failed == 0`:** All sources uploaded. Proceed to Phase 5.
   - **`uploads_failed > 0`:** Some sources failed. **Pause and ask user:**
     - Display failed URL count and grouped reasons (access-denied / rate-limited-max-retries / download-failed / upload-failed).
     - Options: (a) Proceed with partial coverage, (b) Abort run.

3. **Log gate decision** and timestamp to `.harness/STATE.md`.

**This gate is non-negotiable.** Classification does not start until this decision is made.

---

## Phase 5 — Run ingestion for all documents

After the upload gate passes:

1. **Collect all `doc_id` values** from STATE.md source download tracking where `Status = uploaded`.
2. **Call `ragflow_run_ingestion` for ALL doc_ids at once:**
   ```
   ragflow_run_ingestion(
       dataset_id=<kb_dataset_id>,
       doc_ids=[<doc_id_1>, <doc_id_2>, ...]
   )
   ```
3. **Record `ingestion_started_at`** (current UTC timestamp) in STATE.md.
4. **Append event-log line:** `"Ingestion triggered for N documents at <timestamp>"`.

On failure of `ragflow_run_ingestion`: retry once after 30 seconds. If second attempt also fails, pause with `PauseReason = ingestion-trigger-failed` and surface the error to the user.

---

## Pause boundary (after ingestion triggered)

Immediately after Phase 5 completes:

* `RunStatus = paused`
* `PauseReason = awaiting-client-resume`
* `WorkflowStage = W1-stage-5-ingestion-polling`

Post message to user:
> *"Ingestion has been triggered for N documents. Send `resume` when you are ready and I will verify ingestion status."*

The lead does **not** poll `doc_infos` during this pause.

---

## On resume — Verify ingestion

Triggered when user replies `resume` (or any affirmative continuation).

1. Call `doc_infos(doc_ids=[…])` for every `doc_id` in the roster.
2. For each document, check parsing/run status:
   * `done` → set `SourceLifecycle = Ingested`.
   * `running` → still ingesting.
   * `failed` → set `SourceLifecycle = Dropped_IngestTimeout`, `FailureStage = ingestion`. Record in `.harness/sources_excluded.md`.
3. Apply the **60-minute per-document ceiling** (from `ingestion_started_at`):
   * If `now - ingestion_started_at < 60 min` and still `running` → pause again with `PauseReason = awaiting-ingestion-completion`. Ask user to wait; provide estimated time remaining.
   * If `now - ingestion_started_at ≥ 60 min` and still `running` → drop; `failure_reason = ingestion-timeout-60min`. Record in `.harness/sources_excluded.md`.
4. Gate conditions:
   * ≥ 1 doc `done` AND all others resolved → set `ingestion_completed_at`, `RunStage = classification`, `RunStatus = active`. Proceed.
   * **Every** doc `failed` or timed out → pause with `PauseReason = ingestion-error-client-confirmation-needed`. Ask user whether to wait further or abort.

---

## Outputs

* `.harness/DownloadAgent/*.md` — subagent working files (archived to `.harness/Archive/` post-consolidation).
* `.harness/downloads/` — all downloaded files; preserved for audit trail.
* `.harness/sources_excluded.md` — all drops with grouped reasons (access-denied / rate-limited-max-retries / download-failed / upload-failed / ingestion-timeout).
* `.harness/STATE.md` — source download tracking fully populated; `ingestion_started_at` and `ingestion_completed_at` set.
* Per-doc metadata in the KB (queryable via `get_metadata_summary`).

## Success criteria

* **All approved URLs resolved:** Every URL has either a live `doc_id` OR an entry in `.harness/sources_excluded.md`.
* **Ingestion triggered:** `ragflow_run_ingestion` called for all uploaded doc_ids simultaneously; `ingestion_started_at` recorded.
* **Ingestion complete:** At least one `Ingested` doc; all others resolved.
* **Audit trail:** `.harness/downloads/` contains all downloaded files; `.harness/Archive/` contains all download subagent working files.

## Failure modes

| Failure | Response |
| :------ | :------- |
| URL is video / YouTube / streaming | Drop immediately; `failure_reason = unsupported-type`. No retry. |
| `fetch_url` fails repeatedly (non-block) | Auto-retry up to 3 total attempts; then `auto-flagged-invalid`. |
| HTML file < 10 KB, Cloudflare markers | `status = rate-limited`; report to user; retry on approval (max 3 total). |
| HTML file < 10 KB, no known markers | `status = rate-limited` (default); same handling as above. |
| HTML file < 10 KB, access-denied markers | `status = access-denied`; flag immediately; report to user; no retry ever. |
| Rate-limit persists after 3 attempts | `auto-flagged-invalid`; `failure_reason = rate-limited-max-retries`; no user prompt. |
| `ragflow_upload` fails (3 retries) | `failure_reason = upload-exhausted`; URL excluded. |
| Upload gate triggered (partial failures) | Pause; ask user: proceed partial or abort. |
| `ragflow_run_ingestion` fails twice | Pause with `PauseReason = ingestion-trigger-failed`; surface error to user. |
| Document still `running` past 60-min ceiling | Drop; `failure_reason = ingestion-timeout-60min`. |
| All docs failed / timed out | Pause with `ingestion-error-client-confirmation-needed`. |
