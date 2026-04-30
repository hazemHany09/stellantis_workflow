# 05 — Knowledge Base

## Role of the knowledge base

The knowledge base (KB) is the sole substrate for evidence retrieval during classification. After the source approval gate, every approved source document is downloaded, uploaded to the KB, and indexed. The classification stage then queries the KB by semantic retrieval to gather evidence per parameter.

Direct retrieval from the open web during classification is **not** allowed. If the skill needs extra evidence later, it must re-enter the source pipeline (propose → approve → download → ingest) rather than bypass the KB. This keeps every decision backed by an auditable source that the client has approved.

## KB scoping

The business rule is: **one dedicated KB per car**. A run covers exactly one car (see `03-inputs.md`), so each run creates, populates, and — at the end — archives or deletes its own isolated KB. A dedicated KB guarantees that:

* Evidence returned by retrieval cannot leak across cars.
* Citations remain unambiguous.
* Cleanup after analysis is a single deletion.

The KB must carry metadata identifying:

* The car being analysed (brand, model, year, market, trim).
* The analysis run (a unique identifier).
* Per-document metadata: original URL, capture date, source site, document type.

## Ingestion is asynchronous

Ingestion does not complete the moment a document is uploaded. The KB parses asynchronously, and parsing time varies per document. The business rule is:

> The classification stage must not start until **every** approved source document has finished ingestion.

The agent must actively wait for ingestion completion and must surface progress to the client during the wait. Starting classification against a partially ingested KB is a bug, not a shortcut.

## What the KB must offer

At the business level, the KB is required to support the following capabilities. Exact tool mapping is in `07-tools-and-constraints.md`.

| Capability                            | Why it is needed                                                                                                                                                                                                       |
| :------------------------------------ | :--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Create an isolated dataset            | To implement per-analysis scoping.                                                                                                                                                                                     |
| Upload documents with metadata        | To attach car identity and provenance to every document.                                                                                                                                                               |
| Report ingestion status               | To let the agent wait correctly before starting classification.                                                                                                                                                        |
| Semantic retrieval over the dataset   | The core query primitive the classification stage consumes.                                                                                                                                                            |
| Return citations for retrieved chunks | So every verdict can carry a citation back to the original document and, where possible, the chunk inside it.                                                                                                          |
| Delete documents                      | So that problematic sources (duplicate content, off-target car, content mismatch detected post-ingestion) can be removed from retrieval. Removal is a direct delete operation — there is no `Retired` lifecycle state. |
| Optional: knowledge graph queries     | Useful for cross-parameter consistency checks; not a hard requirement of the business logic.                                                                                                                           |

## Lifecycle of a KB

| Stage              | What happens                                                                                                           |
| :----------------- | :--------------------------------------------------------------------------------------------------------------------- |
| Create             | A fresh dataset is created at the start of an analysis run.                                                            |
| Populate           | Approved source documents are uploaded with car/run metadata.                                                          |
| Wait for ingestion | Agent polls ingestion status until every document is fully parsed.                                                     |
| Query              | Classification stage issues retrieval calls per parameter.                                                             |
| Archive or delete  | After the deliverable is produced, the KB is retained for audit or deleted. Which of the two is **OPEN** — see Q-KB-1. |

## Failure handling (interim, see `11-assumptions.md`)

The business rule "classification waits for full ingestion" is subject to the following interim operational rules, each pending a formal close:

* **Download failure** (AS-KB-A / Q-KB-4): 3 retries with backoff; then the URL is dropped from the active source set and recorded in the deliverable's `sources_excluded` list. The run is not aborted.
* **Ingestion timeout** (AS-KB-B / Q-KB-5): 30-minute per-document ceiling; on expiry the document is dropped with the same reporting treatment. Classification starts with whatever ingested successfully.
* **Gap-fill KB reuse** (AS-KB-C / Q-KB-6): the gap-fill workflow extends the original KB rather than creating a fresh one, so citation identity is continuous across the normal run and the gap-fill. New documents carry metadata identifying them as belonging to a gap-fill cycle.
* **Citation granularity** (AS-KB-D / Q-KB-7): the deliverable exposes URL-level `Source link` only, even when the KB supports chunk-level locators. Chunk precision is retained internally.

## What the KB is *not*

* It is not the decision-maker. It returns evidence. Verdicts are the harness's job.
* It is not a cache of the open web. Only approved documents enter it.
* It is not shared across analyses unless an explicit scoping decision allows it.

