---
name: stellantis-failure-handling
description: Use whenever a download, ingestion, retrieval, or subagent operation fails. Centralises the retry, drop, timeout, and reporting policy so the run never aborts on a single failure.
loader: shared
stage: any
requires:
  - stellantis-domain-context
  - stellantis-output-contract
provides:
  - retry-policy
  - drop-and-report-policy
  - timeout-policy
  - sources_excluded-emission-rule
tools: []
---

## Dependencies

- `stellantis-domain-context` — vocabulary.
- `stellantis-output-contract` — defines the `sources_excluded` footer this skill populates.

Consumed by:
- `stellantis-source-validation` — paywall + dead-link drops.
- `stellantis-source-download-and-ingest` — download retries, ingestion timeouts.
- `stellantis-knowledge-base-protocol` — retrieval failure handling.
- `stellantis-lead-agent-subagent-orchestration` — subagent failure / re-spawn.

---

# Sub-skill — Failure handling

## Overview

A run never aborts on a single operational failure. Each failure class has a defined retry budget, a drop point, and a reporting destination. The run continues with whatever survives; the deliverable footer surfaces what was lost.

## When this runs

Inline, whenever an operation fails or times out:
- A URL fails to download.
- An uploaded document does not reach `Ingested` within the per-document ceiling.
- A retrieval call returns an error.
- A subagent surrenders, hangs, or returns invalid output.

## Failure classes

### 1. Download failure

A canonicalised, approved URL fails to fetch.

| Stage | Action |
| :--- | :--- |
| Attempt 1 fails | Retry after **1 second**. |
| Attempt 2 fails | Retry after **2 seconds**. |
| Attempt 3 fails | Retry after **4 seconds**. |
| Attempt 4 fails | **Drop the URL.** Record `failure_stage = download`. |
| Run | Continues. Never aborts on a single download failure. |

Record the dropped URL in the workspace's exclusion log. It will surface in the `sources_excluded` footer.

### 2. Ingestion timeout / parse failure

A document was uploaded successfully but does not reach `Ingested` within the **60-minute per-document ceiling**, or fails parsing.

| Stage | Action |
| :--- | :--- |
| Within 60 min | Continue polling. |
| 60 min reached, still not `Ingested` | **Drop the document.** Record `failure_stage = ingestion`, `failure_reason = timeout` or the parser-reported error. |
| Run | Continues with the documents that did reach `Ingested`. |

### 3. Paywall / login wall detected at fetch time

Detected by the discovery or download stage (login wall, severely truncated content). Silently drop **before** flagging for approval if detected during discovery; if detected during download (post-approval), drop and record as a download failure.

### 4. Retrieval error during classification

The KB returns an error or empty result for a parameter query.

| Condition | Action |
| :--- | :--- |
| Transient error (timeout, 5xx) | Retry up to 3 times with backoff (same pattern as downloads). |
| Persistent error after retries | Treat the parameter as Rule 4 (`No Information Found`) for that retrieval iteration. The subagent's fallback iteration may still recover evidence. |
| All iterations exhausted with no evidence | Emit Rule 4 record per the decision rules. |

### 5. Subagent failure

A subagent does not return a valid result envelope.

| Symptom | Action |
| :--- | :--- |
| Subagent posts `"status": "failed"` | Re-spawn **once** with the same contract. |
| Subagent exceeds per-partition ceiling (default 30 min) with no progress for ≥ 10 min | Mark `SubagentStatus = failed`; re-spawn once. |
| Re-spawn also fails | Lead manually writes Rule-4 records for every parameter in the partition. Add a run-level warning to the deliverable footer. |
| Returned envelope fails schema validation | Re-spawn once; if re-spawn also returns invalid, lead writes Rule-4 fallback. |

### 6. Concurrency-cap protection

The lead enforces a maximum of 3 subagents running concurrently. If a slot is needed and none is free, queue the partition. A queued partition is not a failure.

### 7. Rate limit (Cloudflare / server throttle)

Detected by a download subagent when a fetched HTML/Markdown file is suspiciously small (`size_bytes < 10 240`) AND the first 500 characters contain one or more of: `"just a moment"`, `"cf-challenge"`, `"ray id:"`, `"checking your browser"`, `"enable javascript and cookies"`, `"cf_chl"`, `"attention required"`, `"ddos protection"`. If size is < 10 KB but none of these markers are found, treat as rate-limited by default (Cloudflare pages vary).

| Stage | Action |
| :--- | :--- |
| Detected by subagent | Subagent writes `status = rate-limited` in result envelope. Does **not** call `ragflow_upload`. |
| Lead consolidates | Reports rate-limited URL(s) to user: count, URLs, current attempt number out of 3. |
| User approves retry | Lead re-dispatches download subagent. Increments `Attempts` in STATE.md source download tracking. |
| User declines retry | Drop URL. Record `failure_stage = download, failure_reason = rate-limited-user-declined`. |
| `Attempts ≥ 3` (any attempt outcome) | **Auto-flag without prompting.** Set `Status = auto-flagged-invalid`. Record `failure_reason = rate-limited-max-retries`. |

### 8. Access denied (HTTP 403 / hard block)

Detected by a download subagent when a fetched file contains one or more of: `"403 forbidden"`, `"access denied"`, `"not authorized"`, `"401 unauthorized"`, `"you don't have permission"`, `"permission denied"`, `"forbidden"` (case-insensitive, in the first 500 characters).

| Stage | Action |
| :--- | :--- |
| Detected by subagent | Subagent writes `status = access-denied` in result envelope. Does **not** call `ragflow_upload`. |
| Lead consolidates | Sets `Status = access-denied` in STATE.md. Records `failure_stage = download, failure_reason = access-denied` in `sources_excluded.md`. |
| Lead reporting | Surfaces all access-denied URLs to user in a single consolidated block. |
| Retry | **Never attempted.** Access-denied sources are permanently excluded regardless of attempt count. |

## URL canonicalisation (drop reduction)

Before any URL enters the approval / download pipeline, canonicalise to eliminate technical duplicates:

1. Lowercase hostname.
2. Strip trailing slashes.
3. Strip URL fragments (`#...`).
4. Strip tracking query parameters (`utm_*`, `fbclid`, `gclid`, etc.).
5. Sort remaining query parameters alphabetically.

Two URLs that canonicalise to the same form are treated as the same URL — they are flagged once, downloaded once, ingested once. This applies to both agent-discovered and user-supplied URLs.

## Reporting destination — the `sources_excluded` footer

Every dropped item is appended to the workspace's exclusion log. At deliverable emission time, the footer aggregates them into a `sources_excluded` section per the output contract:

| `canonical_url` | `failure_stage` | `failure_reason` |
| :--- | :--- | :--- |

No per-parameter notes appear here. Failed sources were never queried, so attributing relevance to a parameter is unsound.

## What never aborts the run

| Cause | Outcome |
| :--- | :--- |
| One URL fails to download | Drop URL; continue. |
| One document fails to ingest | Drop document; continue. |
| All sources fail | Run continues; deliverable is full of Rule 4 records. The run completes, just with very thin coverage. |
| One subagent fails twice | Lead authors fallback Rule-4 records for the partition; continue. |
| Ingestion takes longer than expected (within ceiling) | Continue polling. |

## What does abort the run

The run stops only when:
- A required input cannot be obtained from the user (input clarification rejected or abandoned).
- The user issues an explicit cancel signal.
- A platform-level error renders the workspace unwriteable.

Operational failures of individual sources, documents, or subagents never qualify.

## Tips & common pitfalls

- **Don't escalate one failure into a run abort.** A degraded deliverable is preferred to no deliverable.
- **Don't retry forever.** The retry budget per failure class is fixed. Exhausted budget = drop, not infinite loop.
- **Don't re-spawn a subagent more than once for the same partition.** A second failure means the partition is structurally problematic; fall through to manual Rule-4 fallback.
- **Don't skip the `sources_excluded` footer.** Even when only one URL is dropped, it must be visible in the deliverable.
- **Don't conflate paywall with download failure.** Paywall = silent drop in discovery; download failure = recorded with retry history.

## What this skill is NOT

- Not a tool reference.
- Not the retry engine itself — each operational skill executes its own retries; this skill defines the policy they implement.
- Not the rule engine for classification (see `stellantis-decision-rules`).

## Maintenance

Changing a retry budget, ceiling, or drop policy requires:
1. Update this skill.
2. Update the operational skill that implements it.
3. Update `business-logic/11-assumptions.md` if the change crosses a settled assumption.
4. Append a change-log entry per `framework-maintenance/impact-checklist.md`.
