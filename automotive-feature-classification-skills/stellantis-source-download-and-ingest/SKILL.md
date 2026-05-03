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
  - fetch_webpage
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

Downloads each approved URL using a subagent-per-URL flow. The **lead is the sole spawner** of these subagents (and of any retry subagents). Subagents do the network I/O and the upload, but **never read the content of the downloaded file** — they observe only its byte size and report that size to the lead. The lead is the only actor that interprets size or content signals to decide whether a download is healthy, blocked, or must be retried.

**Phase 2 — Initial download subagents.** One subagent per URL. Each calls `fetch_url`, captures the resulting file's byte size from the tool's return value (or `os.stat` on the saved path), and reports detailed findings to the lead (status, byte size, file extension, fetch error message if any, attempt count). On a healthy size, uploads to the KB dataset via `ragflow_upload` (still without reading the file). On a network failure or after upload, **stops and reports** — does not retry internally and does not interpret block markers.

**Phase 3 — Lead coordination and retry spawning.** The lead reads each subagent's result envelope. The lead — not the subagent — decides whether the reported size indicates a healthy fetch, a likely Cloudflare/bot block, or an access-denied page, and routes accordingly:
- HTML/markdown output with `file_size_bytes < 10 240` → likely block → lead spawns a retry-download subagent with `mode = retry-webpage` (uses `fetch_webpage` instead of `fetch_url`).
- Network error / fetch-exhausted → lead spawns a retry-download subagent with `mode = retry-network`.
- Persistent small-size after retry, or unsupported file type → lead reports to the user for a retry decision.

Retry subagents follow the same content-blind rule: they fetch, upload, and report size; the lead interprets.

**Phase 4 — Upload consolidation gate.** After all initial + retry subagents settle, the lead consolidates: which URLs have `doc_id`s, which failed permanently, which await user decision.

**Phase 5 — Ingestion.** Lead calls `ragflow_run_ingestion` for all successfully uploaded doc_ids at once. Pauses for user resume before verifying ingestion status.

**Hard rules at this stage:**
- The lead is the only actor that may spawn download or retry subagents.
- Download subagents never read `params.csv`, `.harness/STATE.md`, the file they downloaded, or any other subagent's working file. Their inputs are entirely contained in their contract.
- Download subagents never invoke any KB retrieval, browser, or search tool.

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
| `fetch_url` | Download subagent | **Primary fetch path** — supports all file types (HTML, PDF, DOCX, XLSX, PPTX); save to `.harness/downloads/`. Always try this first. |
| `fetch_webpage` | Retry subagent only | **Backup fetch — never used on a first attempt.** The lead may spawn a retry subagent with `mode = retry-webpage` that calls `fetch_webpage` **only** when a previous `fetch_url` attempt on that URL definitively failed due to: (a) rate-limiting / Cloudflare block, (b) network timeout, or (c) authentication failure (HTTP 401/403 on an HTML page). Renders the page through Serper's webpage_scrape pipeline. Markdown only; never use for binary files. |
| `ragflow_upload` | Download subagent | Upload saved file to KB dataset |
| `ragflow_run_ingestion` | Lead only | Trigger parse/embed pipeline for all uploaded docs at once |
| `doc_infos` | Lead only | Poll ingestion status per doc_id |
| `list_docs` | Lead only | List all docs in dataset |
| `get_metadata_summary` | Lead only | Sanity check after ingestion |
| `delete_docs` | Lead only | Remove duplicate or failed docs post-ingestion |

---

## Phase 1 — Pre-seed STATE.md download tracking

Before dispatching any subagent, the lead initialises two sections in `.harness/STATE.md` for every approved URL:

1. **Source download tracking** — set `Attempts = 0`, `Status = pending` for every approved URL.
2. **Per-doc ingestion status** — add a row with `ingestion_status = pending` for every approved URL (using the URL slug as placeholder; `doc_id` filled in after upload succeeds).

Append event-log line: `"Download phase started — N URLs to process"`.

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
  "output_dir": "/mnt/user-data/workspace/.harness/downloads/",
  "dataset_id": "<kb_dataset_id>",
  "attempt_number": <n>,
  "metadata": {
    "original_url": "...",
    "canonical_url": "...",
    "source_host": "...",
    "origin": "client|agent",
    "source_type": "manufacturer_official_spec|manufacturer_owner_manual|dealer_fleet_guide|third_party_press_long_form|third_party_aggregator|manufacturer_video|forum_or_ugc",
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
| HTML / no extension / `.html` / `.htm` | `fetch_url(url=url, output_dir="/mnt/user-data/workspace/.harness/downloads/")` | `<slug>.md` |
| PDF (`.pdf`) | `fetch_url(url=url, output_dir="/mnt/user-data/workspace/.harness/downloads/", type="pdf")` | `<slug>.pdf` |
| Word (`.doc`, `.docx`) | `fetch_url(url=url, output_dir="/mnt/user-data/workspace/.harness/downloads/", type="docx")` | `<slug>.docx` |
| Spreadsheet (`.xls`, `.xlsx`) | `fetch_url(url=url, output_dir="/mnt/user-data/workspace/.harness/downloads/", type="xlsx")` | `<slug>.xlsx` |
| Presentation (`.ppt`, `.pptx`) | `fetch_url(url=url, output_dir="/mnt/user-data/workspace/.harness/downloads/", type="pptx")` | `<slug>.pptx` |
| **Video / YouTube / streaming media** | **Not supported.** Write result with `status = unsupported-type`. Stop immediately. Do not attempt `fetch_url`. | — |
| Unknown extension | Try `fetch_url` without `type`; on failure try `type="pdf"`; on second failure write `status = failed`. | — |

`fetch_url` returns the saved file path. The subagent derives `size_bytes` from the returned path's file stat (or from the return value if it includes size metadata).

**Retry policy (within the subagent):** 3 retries with exponential backoff (1 s, 2 s, 4 s). On all retries exhausted, write result with `status = failed, failure_reason = fetch-exhausted`. **The initial subagent does not attempt fallback tools or escalation** — it reports findings to the lead, which decides whether to spawn a retry subagent.

#### Step C — Size capture (no content read)

After a successful `fetch_url`, the subagent records the byte size of the saved file. **The subagent does not open, read, or scan the file's contents at any point.** The size is taken from the `fetch_url` return value if it exposes one, or from `os.stat(path).st_size` (or platform equivalent) on the saved path.

The subagent records:
- `file_size_bytes` (integer) — byte size of the saved file
- `file_path` — the path returned by `fetch_url`
- `is_html` — true iff the saved file extension is `.md` or `.html` (i.e. a webpage conversion)
- `http_status_if_available` — the HTTP status code from the fetch tool's return value, if exposed; otherwise `null`
- `error_message` — populated only on tool errors; `null` on success

The subagent does **not** decide whether the file is blocked, rate-limited, or access-denied. That interpretation is the lead's job in Phase 3, based on `file_size_bytes` and `is_html`. The subagent always proceeds to Step D when Step B succeeded — even when the file is suspiciously small — and lets the lead route the URL to a retry subagent if needed after upload.

(If the lead later determines the upload was a block page, the lead deletes the doc via `delete_docs` during Phase 3 consolidation. Subagents never call `delete_docs`.)

#### Step D — Upload

For every successfully fetched file (regardless of size — interpretation is the lead's job):

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

The subagent writes its result into the working file. The envelope is **content-blind** — it carries fetch outcome and size only, no block classification:

```
<!-- DOWNLOAD RESULT -->
{
  "subagent_name": "<slug>-dl",
  "canonical_url": "<url>",
  "status": "uploaded | unsupported-type | fetch-failed | upload-failed",
  "doc_id": "<id or null>",
  "file_path": "<path or null>",
  "file_size_bytes": <n or null>,
  "file_extension": "<ext or null>",
  "is_html": true | false,
  "attempt_number": <n>,
  "failure_reason": "<reason or null>",
  "error_message": "<exact error string from fetch_url or ragflow_upload, if any>",
  "http_status_if_available": <n or null>
}
<!-- END DOWNLOAD RESULT -->
```

The subagent never sets a `block_type` or scans for marker strings. Block classification is exclusively the lead's job in Phase 3.

---

## Phase 3 — Lead consolidation of download results

After all download subagents for the current attempt have written their result envelope (lead detects presence of `<!-- DOWNLOAD RESULT -->` block in each working file):

1. **Read each result block** from `.harness/DownloadAgent/*.md`.
2. **Lead-side block classification (size-driven).** For each result, the lead derives `block_type` from the reported `file_size_bytes`, `is_html`, and `http_status_if_available`:
   - `is_html = true` AND `file_size_bytes < 10 240` AND `http_status ∈ {null, 200, 503, 429}` → `block_type = rate-limited` (probable Cloudflare/bot interstitial). The lead calls `delete_docs` on the just-uploaded `doc_id` (if any) to keep the KB clean.
   - `http_status ∈ {401, 403}` → `block_type = access-denied`. Lead calls `delete_docs` on the uploaded `doc_id` if any.
   - Otherwise → `block_type = null` (file accepted as is).
3. **Update STATE.md source download tracking** for each URL: set `Attempts`, `Last Failure Reason`, `File Size (bytes)`, `Local Path`, `Status`, lead-derived `block_type`.
4. **Update STATE.md per-doc ingestion status** for each successfully uploaded URL with `block_type = null`: set `doc_id`, `source_type`, `upload_timestamp`. Leave `ingestion_status = pending`.
5. **Archive subagent working files** to `.harness/Archive/<slug>-dl.md`. Update status in archive.
6. **Categorise results (lead, post-classification):**
   - `status = uploaded` AND `block_type = null` → ready for ingestion.
   - `status = uploaded` AND `block_type = rate-limited` → add to **rate-limited queue** (if `attempt_number < 3`).
   - `status = uploaded` AND `block_type = access-denied` → add to **access-denied list**.
   - `status = unsupported-type` → immediately drop.
   - `status = fetch-failed` or `upload-failed` → add to **retry queue** (if `attempt_number < 3`); otherwise auto-flag.

### Access-denied sources — immediate permanent flag

For every URL where the lead derived `block_type = access-denied`:

1. Set `Status = access-denied` in STATE.md source download tracking.
2. Record in `.harness/sources_excluded.md`: `failure_stage = download, failure_reason = access-denied`.
3. **Report to user** once (consolidated block):
   > *"The following N source(s) returned access-denied and cannot be retrieved: `<url1>`, `<url2>`, …"*
4. **Do not retry.** Access-denied sources are permanently excluded.

### Rate-limited sources — user prompt and retry decision

For every URL where the lead derived `block_type = rate-limited` AND `attempt_number < 3`:

1. Set `Status = rate-limited` in STATE.md source download tracking.
2. **Report to user** with a consolidated summary:
   > *"N source(s) appear to be rate-limited by Cloudflare or the server (attempt <n>/3). Would you like to retry them? `<url1>`, `<url2>`, …"*
3. **Await user decision:**
   - User approves retry (any affirmative) → add URLs to the **retry queue**.
   - User declines → drop. Record `failure_reason = rate-limited-user-declined` in `.harness/sources_excluded.md`.

For every URL with derived `block_type = rate-limited` AND `attempt_number ≥ 3`:

1. **Do not prompt the user.** Auto-flag immediately.
2. Set `Status = auto-flagged-invalid` in STATE.md source download tracking.
3. Record `failure_reason = rate-limited-max-retries` in `.harness/sources_excluded.md`.
4. Append event-log line: `"<url> auto-flagged invalid after 3 rate-limited attempts"`.

### Network-failed sources — auto-retry (no user prompt)

For every URL with `status = fetch-failed` AND `attempt_number < 3`:

1. Add to **retry queue** automatically (no user prompt needed).

For every URL with `status = fetch-failed` AND `attempt_number ≥ 3`:

1. Set `Status = auto-flagged-invalid`.
2. Record `failure_reason = download-failed-max-retries`.

---

## Phase 3 (continued) — Lead retry-subagent dispatch

After the lead has classified each result via the size-driven rules above, the lead spawns retry subagents for URLs in the rate-limited / network-failed categories. Reminder: **the lead is the only actor that may spawn subagents**, including retry subagents.

### Categorise initial-subagent outcomes (lead-derived `block_type`)

Walk all subagent results. Categorise each URL using the lead's derived `block_type` (Phase 3 step 2):

| Outcome | Condition | Lead action |
| :--- | :--- | :--- |
| **Uploaded** | `status = uploaded`, `block_type = null` | Record `doc_id` in STATE.md. Done. |
| **Blocked (HTML, bot-fingerprint)** | `is_html = true`, lead-derived `block_type = rate-limited`, `attempt_number ≤ 3` | Spawn retry-subagent with `mode = retry-webpage` |
| **Blocked (HTML, 403 access-denied)** | `is_html = true`, lead-derived `block_type = access-denied`, `http_status ≈ 403` | Spawn retry-subagent with `mode = retry-webpage`. Some 403s are bot-fingerprint; Serper pipeline may pass. |
| **Blocked (persistent, no HTTP clue)** | `is_html = true`, lead-derived `block_type = rate-limited`, `attempt_number ≥ 3` | Ask user: "URL has been rate-limited 3 times. Retry with alternative fetch tool, or drop?" User decision routes to retry-webpage spawn or permanent flag. |
| **Access-denied (binary file, or clear entitlement gate)** | lead-derived `block_type = access-denied`, (`is_html = false` OR `http_status = 401`) | Flag as `auto-flagged-invalid` with `failure_reason = access-denied`. No retry. Report to user. |
| **Fetch-exhausted (network errors)** | `status = fetch-failed`, `error_message` contains network / timeout / DNS signal | Spawn retry-subagent with `mode = retry-network` (same `fetch_url`, but fresh connection context, 2 attempts). |
| **Unsupported type** | `status = unsupported-type` (video, streaming) | Drop immediately. Record in `.harness/sources_excluded.md`. |

### Spawn retry-download subagents

For each URL that triggers a retry action:

1. Create a new contract file at `.harness/DownloadAgent/<slug>-retry-<mode>.md` with `mode ∈ {retry-webpage, retry-network}`.
2. Embed the retry contract block (see below).
3. Spawn a retry-download subagent, passing the working file path.

### Retry-download subagent contract

```
<!-- AUTOGENERATED BY LEAD -->
{
  "subagent_name": "<slug>-retry-<mode>",
  "mode": "retry-webpage | retry-network",
  "canonical_url": "<url>",
  "file_type": "html | pdf | ...",
  "slug": "<slug>",
  "output_dir": "/mnt/user-data/workspace/.harness/downloads/",
  "dataset_id": "<kb_dataset_id>",
  "prior_findings": {
    "initial_block_type": "rate-limited | access-denied | ...",
    "initial_attempt_count": <n>,
    "initial_error_message": "<error>"
  },
  "retry_budget": 2,
  "metadata": { ... same as initial ... }
}
<!-- END AUTOGENERATED BY LEAD -->
```

### Retry-download subagent behaviour

The retry-download subagent is a variant of the initial subagent, activated by `mode`. Like the initial subagent it is **content-blind** — it never opens or reads the saved file:

**Mode `retry-webpage` (HTML pages only):**
- Call `fetch_webpage(url=canonical_url, output_dir="/mnt/user-data/workspace/.harness/downloads/")` instead of `fetch_url`.
- After fetch, capture `file_size_bytes` (Step C — size only, no content read) and proceed to upload (Step D).
- Always report findings back to the lead. The lead will re-classify `block_type` on the new size.
- **Retry budget:** 2 attempts (1 s, 4 s backoff). If both fail, stop and report.

**Mode `retry-network` (any file type):**
- Call `fetch_url` again (same parameters as initial attempt).
- This is for transient network errors: a fresh connection context often succeeds where a previous attempt timed out or hit DNS issues.
- Capture size; upload; report.
- **Retry budget:** 2 attempts (1 s, 4 s backoff).

Both modes report findings back to the lead via the same content-blind result envelope.

### Lead second-pass consolidation

After all retry-subagents report:

1. For each retry result, update STATE.md:
   - Success (retry-webpage or retry-network) → set `Status = uploaded`, record the new `doc_id`, remove from exclusion list if present.
   - Still blocked (retry-webpage) → set `Status = auto-flagged-invalid`, record `failure_reason = webpage-fallback-still-blocked`, log to exclusion list.
   - Still failed (retry-network) → set `Status = auto-flagged-invalid`, record `failure_reason = network-retry-exhausted`, log to exclusion list.
2. Count final outcomes: `uploads_successful`, `uploads_failed`, `uploads_pending-user-decision`.
3. If `uploads_pending-user-decision > 0`, ask user (once, consolidated): retry again, drop, or abort run.

### Event logging

Append to `.harness/STATE.md` event log:

```
- Initial download phase completed: N URLs uploaded, K blocked, M network-failed
- Retry-subagents spawned: M retry-webpage, N retry-network
- Retry phase completed: X recovered, Y still blocked
```

---

---

## Phase 4 — Upload completion gate

After all initial download subagents + all retry-download subagents settle (no more retry subagents in flight):

1. **Count results:**
   - `uploads_successful` = URLs with a live `doc_id` in STATE.md.
   - `uploads_failed` = URLs in `.harness/sources_excluded.md`.
   - `uploads_pending_user_decision` = URLs that require user retry decision (rate-limit persisting after retry-webpage, or user judgment call).
   - **Total must equal approved URL count.**

2. **Outcome:**
   - **`uploads_pending_user_decision == 0` and `uploads_failed == 0`:** All sources uploaded. Proceed to Phase 5.
   - **`uploads_pending_user_decision > 0`:** Some URLs await user decision. **Pause and ask user (consolidated block):**
     - Display the URLs and their block/error findings.
     - Options: (a) Retry those URLs again (lead re-spawns retry subagents), (b) Drop them and proceed with partial coverage, (c) Abort run.
   - **`uploads_failed > 0` and `uploads_pending_user_decision == 0`:** All remaining URLs have been permanently excluded. **Pause and ask user:**
     - Display failed URL count and grouped reasons.
     - Options: (a) Proceed with partial coverage (which docs were successfully ingested), (b) Abort run.

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

## Phase 5b — Deduplicate documents (lead only)

Immediately after `ragflow_run_ingestion` is called and before the pause boundary, the lead checks for duplicate documents in the dataset. All deduplication outcomes (including zero duplicates found) are written to the **[T2] Deduplication log** section in STATE.md. Duplicates arise when the same canonical URL was uploaded more than once (e.g., from retry flows that produced a second `doc_id` for the same source).

1. Call `list_docs(dataset_id=<kb_dataset_id>)` to retrieve all documents currently in the dataset.
2. Group documents by `canonical_url` (from their metadata). Any group with more than one `doc_id` contains duplicates.
3. For each duplicate group:
   - Keep the **first-uploaded** `doc_id` (lowest `created_at` or smallest index in the list).
   - Collect all other `doc_id` values in the group as duplicates.
   - Call `delete_docs(dataset_id=<kb_dataset_id>, doc_ids=[<duplicate_id_1>, ...])` to remove them.
   - Append an event-log line: `"Deleted duplicate doc <id> for <canonical_url> (kept <kept_id>)"`.
4. Update STATE.md source download tracking: any deleted `doc_id` is removed from the roster; the kept `doc_id` remains.
5. Append a summary event-log line: `"Deduplication complete — N duplicates removed, M unique documents remain"`.

**When to run:** Always. Even if no duplicates are expected, the list-and-check is cheap and prevents silent double-ingestion corrupting retrieval recall.

**This is a lead-only step.** Subagents are never involved in document management.

---

## Pause boundary (after ingestion triggered)

Immediately after Phase 5b completes:

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
   * `done` → set `SourceLifecycle = Ingested`. Update STATE.md per-doc ingestion status: `ingestion_status = done`, `ingestion_completed_at = <ISO>`.
   * `running` → still ingesting.
   * `failed` → set `SourceLifecycle = Dropped_IngestTimeout`, `FailureStage = ingestion`. Record in `.harness/sources_excluded.md`. Update STATE.md per-doc ingestion status: `ingestion_status = failed`, `failure_reason = <reason>`.
3. Apply the **60-minute per-document ceiling** (from `ingestion_started_at`):
   * If `now - ingestion_started_at < 60 min` and still `running` → pause again with `PauseReason = awaiting-ingestion-completion`. Ask user to wait; provide estimated time remaining.
   * If `now - ingestion_started_at ≥ 60 min` and still `running` → drop; `failure_reason = ingestion-timeout-60min`. Record in `.harness/sources_excluded.md`.
4. Gate conditions:
   * ≥ 1 doc `done` AND all others resolved → set `ingestion_completed_at`, `RunStage = classification`. **Do not auto-spawn classification subagents.** Instead, immediately pause for the classification go-ahead (next section).
   * **Every** doc `failed` or timed out → pause with `PauseReason = ingestion-error-client-confirmation-needed`. Ask user whether to wait further or abort.

---

## Pause boundary (after ingestion verified, before classification)

This is a **mandatory** pause in W1. The lead has finished downloading and ingesting all approved sources; classification subagents have not yet been spawned. The lead must yield control to the user and wait for an explicit go-ahead before stage 6 begins.

1. Update `.harness/STATE.md`: `RunStatus = paused`, `PauseReason = awaiting-classification-go-ahead`, `PausedAt = <timestamp>`, `WorkflowStage = W1-stage-5b-pre-classification`.
2. Read `.harness/params.csv` and pre-compute the R1 category partitions (single-category, ≤ 15 parameters each, in CSV row order). Record the planned partition list in STATE.md under the **R1 subagent roster** section with `SubagentStatus = planned` for each row. Do **not** spawn any subagent yet.
3. Append event-log line: `"Ingestion verified — N documents ingested. Awaiting user go-ahead to begin classification."`
4. Post a summary message to the user:
   > *"Ingestion is complete. **N documents** are live in the knowledge base. The frozen reference list contains **M parameters** across **K categories**, planned as **P R1 partitions** (each a single category, ≤ 15 parameters). Q sources were dropped during download/ingest (see `.harness/sources_excluded.md`). Reply `go` when you are ready and I will spawn the classification subagents. Reply with scope changes (e.g. category filters) first if you want to amend the run."*
5. Return control. Do not poll, do not auto-resume.

**On resume:** When the user replies `go` (or any affirmative), the lead verifies `PauseReason = awaiting-classification-go-ahead`, sets `RunStatus = active`, clears `PauseReason`, sets `ResumedAt`, appends an event-log line `"Classification go-ahead received — spawning R1 subagents"`, and hands control to `stellantis-lead-agent-subagent-orchestration` for stage 6.

If the user's reply is a scope amendment (e.g. "only powertrain category"), apply the amendment to `.harness/params.csv` (re-filter and re-hash; record the new hash in STATE.md), recompute the partition plan, and re-post the summary. Do not begin classification until the user explicitly says go on the amended plan.

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
| HTML page blocked after 3 fetch_url attempts (rate-limit) | Initial subagent reports findings. Lead spawns retry-subagent with `mode = retry-webpage`. If retry-webpage succeeds, recover. If blocked again, permanently flag. |
| HTML page returns 403 Unauthorized | Initial subagent reports findings. Lead spawns retry-subagent with `mode = retry-webpage` (Serper pipeline may bypass bot-fingerprint). If it passes, recover. If blocked again, permanently flag. |
| Network timeout or DNS failure | Initial subagent reports findings (with error message). Lead spawns retry-subagent with `mode = retry-network` (fresh connection context). If it succeeds, recover. If still fails, permanently flag. |
| `ragflow_upload` fails (3 retries) | `failure_reason = upload-exhausted`; URL excluded. |
| Upload gate triggered (partial failures) | Pause; ask user: proceed partial or abort. |
| `ragflow_run_ingestion` fails twice | Pause with `PauseReason = ingestion-trigger-failed`; surface error to user. |
| Document still `running` past 60-min ceiling | Drop; `failure_reason = ingestion-timeout-60min`. |
| All docs failed / timed out | Pause with `ingestion-error-client-confirmation-needed`. |
