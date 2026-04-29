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

Downloads each approved URL using a two-phase subagent flow where subagents provide structured findings to the lead, and the lead spawns specialized retry subagents based on those findings.

**Phase 2 — Initial download subagents.** One subagent per URL. Each calls `fetch_url`, performs file-size-based block detection, and reports detailed findings to the lead (status, block type, block markers, error messages, attempt count). On success, uploads to the KB dataset. On block or failure, **stops and reports** — does not retry internally.

**Phase 3 — Lead coordination and retry spawning.** The lead reads all initial subagent findings. For each URL that blocked or failed:
- If blocked (Cloudflare / 403 / access-denied on an HTML page): spawns a retry-download subagent with `mode = retry-webpage`.
- If fetch-exhausted (network errors): spawns a retry-download subagent with `mode = retry-network`.
- If access-denied on a binary file, or persistent failures: reports to user for retry decision.

Retry subagents provide findings to the lead, which decides whether to re-spawn, escalate, or flag the URL.

**Phase 4 — Upload consolidation gate.** After all initial + retry subagents settle, lead consolidates: which URLs have doc_ids, which failed permanently, which await user decision.

**Phase 5 — Ingestion.** Calls `ragflow_run_ingestion` for all successfully uploaded doc_ids at once. Pauses for user resume before verifying ingestion status.

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
| `fetch_url` | Download subagent | Primary fetch path — supports all file types (HTML, PDF, DOCX, XLSX, PPTX); save to `.harness/downloads/` |
| `fetch_webpage` | Download subagent | **Webpage-only fallback** — used when `fetch_url` returned a blocked / undersized HTML page (Cloudflare, 403, max-retry exhausted). Renders the page through Serper's webpage_scrape pipeline, which often bypasses simple bot blocks because the request headers, IP egress, and JS rendering differ from `fetch_url`. Markdown only; do not use for binary files. |
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
| HTML / no extension / `.html` / `.htm` | `fetch_url(url=url, output_dir=".harness/downloads/")` | `<slug>.md` |
| PDF (`.pdf`) | `fetch_url(url=url, output_dir=".harness/downloads/", type="pdf")` | `<slug>.pdf` |
| Word (`.doc`, `.docx`) | `fetch_url(url=url, output_dir=".harness/downloads/", type="docx")` | `<slug>.docx` |
| Spreadsheet (`.xls`, `.xlsx`) | `fetch_url(url=url, output_dir=".harness/downloads/", type="xlsx")` | `<slug>.xlsx` |
| Presentation (`.ppt`, `.pptx`) | `fetch_url(url=url, output_dir=".harness/downloads/", type="pptx")` | `<slug>.pptx` |
| **Video / YouTube / streaming media** | **Not supported.** Write result with `status = unsupported-type`. Stop immediately. Do not attempt `fetch_url`. | — |
| Unknown extension | Try `fetch_url` without `type`; on failure try `type="pdf"`; on second failure write `status = failed`. | — |

`fetch_url` returns the saved file path. The subagent derives `size_bytes` from the returned path's file stat (or from the return value if it includes size metadata).

**Retry policy (within the subagent):** 3 retries with exponential backoff (1 s, 2 s, 4 s). On all retries exhausted, write result with `status = failed, failure_reason = fetch-exhausted`. **The initial subagent does not attempt fallback tools or escalation** — it reports findings to the lead, which decides whether to spawn a retry subagent.

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
  "attempt_number": <n>,
  "is_html": true | false,
  "detailed_findings": {
    "block_detected": true | false,
    "block_type_if_detected": "rate-limited | access-denied | null",
    "error_message": "<if fetch failed: exact error from fetch_url>",
    "file_size_bytes": <n or null>,
    "http_status_if_available": <n or null>
  }
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

---

## Phase 3 — Lead coordination: read findings and spawn retry subagents

After all initial download subagents report, the lead reads their result envelopes from `.harness/DownloadAgent/*.md`. For each URL, the lead consults `detailed_findings` and decides the next action.

### Categorise initial-subagent outcomes

Walk all subagent results. Categorise each URL:

| Outcome | Condition | Lead action |
| :--- | :--- | :--- |
| **Uploaded** | `status = uploaded`, has `doc_id` | Record `doc_id` in STATE.md. Done. |
| **Blocked (HTML, bot-fingerprint)** | `is_html = true`, `block_type = rate-limited`, `attempt_number ≤ 3` | Spawn retry-subagent with `mode = retry-webpage` |
| **Blocked (HTML, 403 access-denied)** | `is_html = true`, `block_type = access-denied`, `http_status ≈ 403` | Spawn retry-subagent with `mode = retry-webpage`. Some 403s are bot-fingerprint; Serper pipeline may pass. |
| **Blocked (persistent, no HTTP clue)** | `is_html = true`, `block_type = rate-limited`, `attempt_number ≥ 3` | Ask user: "URL has been rate-limited 3 times. Retry with alternative fetch tool, or drop?" User decision routes to retry-webpage spawn or permanent flag. |
| **Access-denied (binary file, or clear entitlement gate)** | `block_type = access-denied`, (`is_html = false` OR `http_status = 401`) | Flag as `auto-flagged-invalid` with `failure_reason = access-denied`. No retry. Report to user. |
| **Fetch-exhausted (network errors)** | `failure_reason = fetch-exhausted`, `error_message` contains network / timeout / DNS signal | Spawn retry-subagent with `mode = retry-network` (same `fetch_url`, but fresh connection context, 2 attempts). |
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
  "output_dir": ".harness/downloads/",
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

The retry-download subagent is a variant of the initial subagent, activated by `mode`:

**Mode `retry-webpage` (HTML pages only):**
- Call `fetch_webpage(url=canonical_url, output_dir=".harness/downloads/")` instead of `fetch_url`.
- Block detection (Step C) runs on the Markdown returned by `fetch_webpage`.
- If block check passes, proceed to upload (Step D). If block persists, report `status = blocked, block_type = <new-type>, failure_reason = webpage-fallback-still-blocked`.
- **Retry budget:** 2 attempts (1 s, 4 s backoff). If both fail or block persists, stop.

**Mode `retry-network` (any file type):**
- Call `fetch_url` again (same parameters as initial attempt).
- This is for transient network errors: a fresh connection context often succeeds where a previous attempt timed out or hit DNS issues.
- Block detection (Step C) runs as normal.
- **Retry budget:** 2 attempts (1 s, 4 s backoff).

Both modes report findings back to the lead via the result envelope, using the same structure as initial subagents.

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
| HTML page blocked after 3 fetch_url attempts (rate-limit) | Initial subagent reports findings. Lead spawns retry-subagent with `mode = retry-webpage`. If retry-webpage succeeds, recover. If blocked again, permanently flag. |
| HTML page returns 403 Unauthorized | Initial subagent reports findings. Lead spawns retry-subagent with `mode = retry-webpage` (Serper pipeline may bypass bot-fingerprint). If it passes, recover. If blocked again, permanently flag. |
| Network timeout or DNS failure | Initial subagent reports findings (with error message). Lead spawns retry-subagent with `mode = retry-network` (fresh connection context). If it succeeds, recover. If still fails, permanently flag. |
| `ragflow_upload` fails (3 retries) | `failure_reason = upload-exhausted`; URL excluded. |
| Upload gate triggered (partial failures) | Pause; ask user: proceed partial or abort. |
| `ragflow_run_ingestion` fails twice | Pause with `PauseReason = ingestion-trigger-failed`; surface error to user. |
| Document still `running` past 60-min ceiling | Drop; `failure_reason = ingestion-timeout-60min`. |
| All docs failed / timed out | Pause with `ingestion-error-client-confirmation-needed`. |
