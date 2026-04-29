---
name: stellantis-knowledge-base-protocol
description: Use whenever an agent creates, populates, polls, queries, or deletes from the knowledge base. Vendor-agnostic contract for the four KB capability categories.
loader: shared
stage: any
requires:
  - stellantis-domain-context
  - stellantis-failure-handling
provides:
  - kb-create-protocol
  - kb-upload-protocol
  - kb-ingestion-wait-protocol
  - kb-retrieval-protocol
  - kb-delete-protocol
tools: []
---

## Dependencies

- `stellantis-domain-context` — vocabulary.
- `stellantis-failure-handling` — retry, timeout, drop policy.

Used by:
- `stellantis-source-download-and-ingest` — create, upload, wait.
- `stellantis-subagent-classification-loop` — retrieval, chunk inspection.
- `stellantis-lead-agent-subagent-orchestration` — metadata summary checks.

---

# Sub-skill — Knowledge-base protocol

## Overview

The KB is the exclusive evidence substrate during classification. This skill specifies what the KB must offer, in vendor-agnostic terms, and the contract every agent honours when interacting with it. Concrete tool bindings (e.g. RAGFlow function names) live in the operational sub-skills that consume this contract.

## When this runs

- Lead, once per run: create the dataset, populate it, wait for ingestion.
- Subagent, on every parameter: retrieve evidence.
- Lead, optionally: delete duplicate / off-target documents.
- Gap-fill workflows (when authored): extend the existing dataset with cycle-tagged documents.

## Upload approach

**All files are uploaded via `ragflow_upload`.** File content (binary or base64 encoding) is handled internally by the tool — the lead never reads file content or passes it through the LLM context window. The lead calls `ragflow_upload(path="...", dataset_id="...", filename="...", metadata={...})`, reads the `id` field from the returned JSON as `doc_id`, and records it in STATE.md. No Python scripts are needed. See `stellantis-source-download-and-ingest` for the full upload sequence and required metadata fields.

## Capability categories

The KB must offer four distinct capability categories. The framework relies only on these — no vendor-specific feature is required.

| Category | Operations | Used by |
| :--- | :--- | :--- |
| **Dataset management** | Create dataset, list datasets, get/update/delete dataset | Lead (preflight, archival) |
| **Document management** | Upload with metadata, list, info-poll, delete, set/batch-update metadata | Lead |
| **Retrieval** | Semantic search, optional knowledge-graph query, list chunks, download document to local directory (`ragflow_download`) | Subagent (read-only); Lead (rare) |
| **Metadata summary** | Aggregate metadata over the dataset | Both, read-only |

## Shared download directory

All downloaded files — whether fetched by the lead during the download phase or by a subagent during retrieval — are saved to **`.harness/downloads/`**. This directory serves as both a cache and an audit trail.

**Cache-check directive (applies to both lead and subagent):** Before issuing any download tool call (`fetch_url` for the lead, `ragflow_download` for the subagent), check if the target file already exists in `.harness/downloads/`. If it does, reuse it. Log "Cache hit: reusing `.harness/downloads/<filename>`" and skip the network call.

This prevents redundant downloads and keeps the run reproducible across pauses and resumes.

## Scoping rule

**One dedicated dataset per run.** Created at preflight, populated only with that run's approved sources, queried only by that run's agents.

A run never queries another run's dataset. Cross-run reuse is forbidden in the normal pipeline. Gap-fill cycles extend the *same* run's dataset, not a different run's.

## Required document metadata

Every uploaded document carries at minimum:

| Field | Required | Notes |
| :--- | :--- | :--- |
| `canonical_url` | yes | Result of canonicalisation rules in `stellantis-failure-handling`. |
| `original_url` | yes | Pre-canonicalisation, as flagged. |
| `source_host` | yes | The website host. |
| `document_type` | yes | `html`, `pdf`, `scrape`. |
| `capture_timestamp` | yes | When the document was fetched. |
| `run_id` | yes | The owning run. |
| `car_identity` | yes | `(brand, model, model_year, market_canonical)`. |
| `cycle` | optional | `1` for the original ingestion. `2`, `3`, … for gap-fill cycles. Absent or `1` means original. |
| `origin` | optional | `client` (user-supplied URL) or `agent` (agent-discovered). |
| `source_type` | yes | One of the source-type categories defined in `stellantis-source-validation`. Required for consensus verification in `stellantis-decision-rules`. |

Metadata enables citation traceability and cycle-aware retrieval. The deliverable's `source_url` field reads `canonical_url` from this metadata.

## Lifecycle

| Phase | What happens | Who |
| :--- | :--- | :--- |
| Create | New dataset created at preflight, tagged with `run_id` and `car_identity`. | Lead |
| Populate | Each approved URL is downloaded, then uploaded with the metadata above. | Lead |
| Wait for ingestion | Lead polls per-document status until every uploaded document reports `Ingested`, or the 60-min ceiling is reached. | Lead |
| Query | Subagents issue retrieval calls per parameter. No web search. | Subagent |
| Mid-run delete | Lead may delete a document discovered post-ingestion as off-target / duplicate. There is no `Retired` state. | Lead |
| Archive | Dataset is retained indefinitely after deliverable emission, tagged with completion timestamp. No automatic deletion. | Lead |

## Ingestion wait protocol

1. Upload every approved document.
2. Poll the document-info endpoint silently — no per-document progress messages.
3. For each document, track `(canonical_url, status)`. Stop polling that document when `status = Ingested`.
4. When all documents are `Ingested`, emit a single "Ingestion complete" notification to the user.
5. If any document exceeds the 60-min per-document ceiling, drop it (per `stellantis-failure-handling`) and continue with the rest.
6. Proceed to classification only when all surviving documents are `Ingested`.

## Retrieval protocol (subagent)

The subagent is the primary retrieval consumer. The contract:

1. **Tools allowed.** Only `retrieval`, `retrieve_knowledge_graph`, `list_chunks`, `ragflow_download`, `doc_infos`, `get_metadata_summary`. No web search, no browser rendering, no `fetch_url`. Ever.
2. **Iteration budget.** Up to 3 main retrieval iterations per parameter, plus a 3-iteration fallback if the main iterations yield no usable evidence. Beyond that, emit Rule 4.
3. **Cycle filter (gap-fill).** When the active workflow is a gap-fill cycle `N`, retrieval may filter on `cycle ∈ {1, …, N}`. The original cycle's evidence remains visible.
4. **No web fallback.** A failed retrieval becomes a Rule 4 record, not a web search.

## Delete protocol

When the lead detects, post-ingestion, that a document is off-target (e.g. wrong car, duplicate content), it:
1. Deletes the document via the document-management capability.
2. Records the deletion in the run's event log with a reason.
3. Does **not** add the document to the `sources_excluded` footer — that footer is for download/ingestion failures, not post-ingestion deletions.

There is no separate `Retired` lifecycle state.

## Hard rules

- **No web search inside classification.** This is the architectural phase boundary. Subagents cannot reach the web by any means.
- **No cross-run queries.** Even if two runs cover the same car, they query separate datasets.
- **No silent metadata drift.** Once uploaded, a document's `canonical_url`, `run_id`, and `car_identity` metadata are never edited.
- **No partial-ingestion classification.** Classification waits for full ingestion (or timeout drops) before starting.
- **One uploaded document per canonical URL.** Duplicates collapse upstream of upload (canonicalisation).

## Tips & common pitfalls

- **Don't query before all docs are `Ingested`.** Premature retrieval yields a degraded evidence set and a non-reproducible run.
- **Don't reuse a dataset across runs.** Per-run scoping is the only way citations stay unambiguous.
- **Don't strip metadata you set.** `canonical_url` and `run_id` are load-bearing; everything in the deliverable traces back through them.
- **Don't escalate a retrieval miss into a web search.** Emit Rule 4 instead.
- **Don't poll noisily.** Silent polling, single completion message.

## What this skill is NOT

- Not a vendor binding. Tool names live in the operational skills that implement this contract.
- Not the source-discovery contract.
- Not the failure policy (see `stellantis-failure-handling`); this skill calls into it.

## Maintenance

Capability changes (e.g. adding a new metadata field, changing the ingestion ceiling) require:
1. Update this skill.
2. Update the consuming operational skill (`stellantis-source-download-and-ingest`, `stellantis-subagent-classification-loop`).
3. Update the relevant template in the core skill (e.g. document-metadata template).
4. Apply follow-ups per `framework-maintenance/impact-checklist.md`.
