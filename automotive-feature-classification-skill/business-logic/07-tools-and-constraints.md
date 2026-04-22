# 07 — Tools and Constraints

## Tool capability model (vendor-neutral)

The skill depends on four **capability categories**. Any concrete tool that fulfils the capability can be substituted.

| Capability category              | What it must do                                                                                                                                    | Status in this project                                  |
| :------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------- | :------------------------------------------------------ |
| **Open-web search**              | Accept a free-text query and return ranked URLs with titles and snippets. Must support site-restricted queries. Image/news/video sub-modes are a plus. | Available today — see `tools/` folder.                |
| **Browser rendering / scraping** | Fetch a URL, execute JavaScript, and return the fully rendered content as HTML, Markdown, or a structured scrape by selector. Must also export the rendered page as PDF. Must support multi-page crawls. | Available today — see `tools/` folder.                |
| **Knowledge base**               | Create isolated datasets, upload documents with metadata, expose ingestion status, perform semantic retrieval with chunk-level citations, and delete datasets or documents. | Available today — see `tools/` folder.                |
| **Source approval store**        | Expose two operations: (1) *flag-for-approval* — register a proposed URL with metadata into a pending queue scoped to a run; (2) *retrieve-approved* — return the URLs the client has approved for that run. Approval is URL-level. Pending / Approved / Rejected are the three states the store must expose. | **Planned.** Two built-in tools will be added in a later development stage. Business logic treats them as capabilities with the contract above. |

The **source approval store** is what decouples the client-side validation UI from the agent's classification work. The agent writes proposed URLs into it and, separately, reads back the approved set. It never drives the approval UI directly.

No other capabilities are assumed. In particular, the skill must not assume access to:

- A local filesystem beyond reading `artifacts/params.csv` and writing deliverables.
- Code execution / sandbox tools.
- Database tools beyond the knowledge base and the approval store.
- Credential-bearing scrapers of paywalled content.
- Any vendor-specific API beyond what the four capability categories describe.

## Vendor-neutral language

Throughout the business-logic documents, tools are referred to by **capability category**, not by product name:

- "web search tool" — not the product brand behind it.
- "browser rendering tool" — not the platform.
- "knowledge base tool" — not the product.
- "source approval store" — not the database or queue product that backs it.

Concrete tool names live only in `tools/*.md` and in the workflow layer (designed later). If a capability category is ever fulfilled by a different tool, only the workflow layer changes; the business logic stays unchanged.

## Hard constraints

1. **No evidence outside the KB.** Classification reads evidence from the KB only. The search and browser-rendering tools are used during the sourcing phase and are not used during classification.
2. **No action past the source approval gate without explicit client approval.** Downloads, uploads, and ingestion are blocked until approval.
3. **No classification before full ingestion.** The skill polls and waits.
4. **Every verdict must carry citations when presence is `yes`.** A verdict without citation is rejected.
5. **Every parameter in the CSV produces exactly one verdict.** No parameter may be silently dropped.
6. **Only applicable levels may be emitted.** Parameters with a restricted applicable level set must never receive a level outside that set.

## Soft constraints (recommendations)

- Prefer Markdown extraction over raw HTML when ingesting pages — it parses more reliably in most knowledge bases.
- Prefer the manufacturer's own documents over third-party summaries when both cover the same feature.
- Prefer newer documents for the exact model year over older documents for a previous year of the same model.
