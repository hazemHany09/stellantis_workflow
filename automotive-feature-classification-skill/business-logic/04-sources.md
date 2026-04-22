# 04 — Sources

## Why sources matter

Every decision emitted by the skill must be traceable to one or more source documents. A parameter decision with no citation is invalid. Therefore the source pipeline is a first-class business concern, not an implementation detail.

## Source granularity: the URL is the unit

Approval is **per URL**. Sites are not approved as a group; each candidate URL is a standalone entity that the client accepts or rejects individually. There is no notion of "an approved site whose URLs are pre-approved" and no cascade rule from site-level decisions. A rejected URL has no effect on other URLs from the same host.

## Source lifecycle

A source (a single URL) moves through the following business states, in order. The next state can only begin once the previous one is complete.

| State                  | Meaning                                                                                                                 |
| :--------------------- | :---------------------------------------------------------------------------------------------------------------------- |
| **Proposed**           | The URL has been identified — either by the client or by the agent's web search — and registered with the approval store (see below). |
| **Pending approval**   | The URL is awaiting the client's decision. No download happens in this state.                                           |
| **Approved**           | The client has explicitly signed off on using this URL. It is now retrievable from the approval store.                  |
| **Rejected**           | The client has explicitly rejected this URL. It is permanently excluded from the run.                                   |
| **Downloaded**         | The approved source document has been retrieved as a file (HTML → Markdown, PDF, or a scrape) and is ready for ingestion. |
| **Ingested**           | The knowledge base has parsed the document into retrievable chunks. It is now available to the classification stage.    |

## Source approval gate — asynchronous, store-mediated

The approval gate is **not** a synchronous conversation between the agent and the client. It is mediated by a dedicated approval store that decouples the two sides.

The store exposes two operations the agent uses directly:

- **Flag-for-approval.** The agent registers a proposed URL (with per-URL metadata: title, excerpt, why selected, car identity, run ID) into the store. The URL enters the `Pending approval` state. The agent does not wait on this call beyond acknowledgement.
- **Retrieve-approved.** The agent queries the store for URLs that have transitioned to `Approved` for the current run. It may call this repeatedly; approval can arrive at any later point.

The client interacts with the store through a separate interface (out of scope of the agent). From the agent's perspective:

1. Approval is **asynchronous**. Flagging and retrieving are two distinct calls, possibly separated by an arbitrary amount of time.
2. The agent does **not** drive the approval UI. The validation page and the harness continuation page are decoupled on the client side.
3. The agent must resume the run when, and only when, the approval store reports at least the required set of URLs as `Approved` (see `Minimum source coverage`, Q-SRC-2). Until then it waits or yields.
4. URLs never go from `Rejected` back to `Pending` within the same run. If the client changes their mind they must re-propose.

This decoupling is what enables the two client-side pages described by the client: one page to validate the sources (driven by the approval store), and a separate page to continue the classification work (driven by retrieval of the approved set).

### Interaction with existing tool categories

The approval store is a **fourth capability category**, complementing the three declared in `07-tools-and-constraints.md`. The concrete tools implementing *flag-for-approval* and *retrieve-approved* are planned for a later development stage and are not yet present in `tools/`. The business logic treats them as capabilities with the contract defined above; the workflow layer will bind them to concrete tools once built.

## Agent-discovered search: query driver

When the client does not supply a source site list, the agent builds its web-search queries from the four identifying fields of the car: **brand + model + year + market**. Those four fields, together, are what ties a returned source back to the exact car being analysed. A source that covers the right brand + model but the wrong year, or the wrong market, is not a valid source for this run.

## Selection criteria (guidance, not hard rules)

When the agent must discover sources itself, it should prefer:

- Official manufacturer press releases and spec sheets for the exact model/year/market.
- Official manufacturer owner's manuals and technical brochures.
- Reputable automotive press with detailed technical coverage of the exact model.
- Specialist sites for specific categories (e.g. EV-focused publications for the `EV` category).

It should avoid:

- User-generated forum posts as primary evidence.
- Listings on classified / resale sites.
- Aggregators that do not cite their own sources.

These preferences are **soft**. The client is the final judge at the approval gate.

## Per-car scoping

Sources are always scoped **per car**. An analysis run covers exactly one car (see `03-inputs.md`), so its source set is dedicated to that car and never merged with another run's sources. This keeps retrieval and citations unambiguous and makes cleanup a per-run operation.

## Minimum source coverage

Whether there is a minimum number of sources required before the classification stage may start, or a minimum per category, is **OPEN** — see Q-SRC-2. The interim is zero (no minimum). As a consequence, the approval gate cannot be satisfied purely by a count threshold — it needs an explicit "review complete" signal from the client side (see Q-SRC-6 and AS-SRC-C for the interim 15-minute quiescence rule).

## Operational rules layered on the lifecycle (interim, see `11-assumptions.md`)

The business lifecycle above is the contract. The following operational rules are interim decisions recorded in `11-assumptions.md` that affect how the lifecycle is driven in practice. Each has an open-question counterpart in `08-open-questions.md`:

- **URL canonicalisation** before *flag-for-approval* (AS-SRC-A / Q-SRC-7): strip fragment, strip tracking params, lowercase host, collapse trailing slash.
- **Client-supplied URL precedence** (AS-SRC-B / Q-SRC-7): Mode-A URLs that collide with agent discoveries are tagged `origin = client`.
- **Approval-complete signal** (AS-SRC-C / Q-SRC-6): proceed when approved set is stable for 15 minutes and Pending is empty.
- **Rejected URLs are sticky within a run** (AS-SRC-D / Q-SRC-8): not re-flagged even if re-discovered.
- **Candidate cap** (AS-SRC-E / Q-SRC-9): soft ceiling of 20 for agent-discovered URLs; client-supplied URLs are uncapped.
- **Per-URL metadata at flag time** (AS-SRC-G / Q-SRC-11): best-effort title, excerpt, why-selected, car identity, run ID — never blocking.
- **Download failure** (AS-KB-A / Q-KB-4): 3 retries, then drop; run continues. Surfaced in the deliverable header's `sources_excluded` list.
- **Ingestion timeout** (AS-KB-B / Q-KB-5): 30-minute cap; dropped on expiry; same reporting as download failure.

## What a source is *not*

- A source is not a parameter. One source typically covers many parameters.
- A source is not the knowledge base. It is the input to the knowledge base.
- A source is not a single chunk. Chunks are produced by ingestion.
