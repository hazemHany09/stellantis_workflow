# 14 — Client Questions (product-facing)

This document converts the working assumptions recorded in `11-assumptions.md` into concrete questions to put to the client. **It deliberately excludes internal-implementation decisions** (e.g. RAGFlow polling cadence, retention in days, KB chunking, partition heuristics, tool vendors). Only questions that change the **product's behaviour as the client experiences it** — inputs, outputs, workflows, trade-offs visible in the deliverable — are included.

Each question states:

* **What we assumed** — the interim decision currently in `11-assumptions.md`.
* **Why we are asking** — what the client needs to confirm or override.
* **What happens if we are wrong** — risk if the assumption does not match the client's expectation.
* **Options** — the alternatives the client can pick from.

If the client confirms every assumption below, no change is needed. Each override triggers targeted rework in the business-logic documents and the workflow layer.

***

## Section 1 — Inputs (how the client submits a request)

### Q-C-1 — Is any submission format acceptable?

**Assumption.** The skill accepts free text, JSON, a form, or a conversational message, and extracts the four required fields (brand, model, year, market) from whatever you provide. If any field is missing, we ask **only** for that field.

**Why we ask.** If your internal tooling will always send a structured format, we can tighten validation and skip the extraction step.

**Options.**

* (A) Confirm: fully format-agnostic. *(current assumption)*
* (B) Fix the format to JSON with a specific schema.
* (C) Fix the format to a web form with fixed fields.

**Risk if wrong.** If you expect a strict schema but we accept free text, malformed submissions may silently pass through the extraction step and run on wrong inputs.

***

### Q-C-2 — How should we handle ambiguous car identity?

**Status: Resolved 2026-04-23. Option A confirmed.**

**Decision.** When `(brand, model, year, market)` maps to multiple variants (body styles, facelifts, regional rebadges, sub-models), the agent **auto-selects the most prominent / widespread variant** and records the resolved identity in the deliverable header. The client may request a re-run with a different variant if needed. No disambiguation gate is added.

**Risk accepted.** A deliverable may be built on the agent's choice of variant without the client noticing the header line. Mitigated by making the resolved identity prominent in the deliverable header.

***

### Q-C-3 — What market vocabulary do you want to use?

**Assumption.** We accept ISO 3166-1 alpha-2 codes (`DE`, `US`, `AE`), country names (`Germany`), and regulatory-zone shortcuts (`EU`, `MEA`, `NAFTA`, `GCC`). Zones are expanded to their constituent countries during source discovery.

**Why we ask.** If you only ever think in terms of single countries, zone expansion adds noise; if you only ever think in terms of zones, single-country runs may be too narrow.

**Options.**

* (A) Confirm hybrid. *(current assumption)*
* (B) Single-country only (reject zone inputs).
* (C) Zone-only (require a zone; reject single-country inputs).

**Risk if wrong.** Mismatch between your mental model and ours produces over- or under-scoped source discovery.

***

### Q-C-4 — Should we pre-validate that `(model, year)` is a real pair?

**Assumption.** No pre-validation. We accept any year (including future or discontinued) and let sources speak for themselves. A `(model, year)` pair with no public coverage simply produces a deliverable full of Rule 4 (silent-all) records.

**Why we ask.** Pre-validation prevents wasted runs on impossible pairs but requires a maintained list of valid years per model.

**Options.**

* (A) Confirm no pre-validation. *(current assumption)*
* (B) Pre-validate against a manufacturer catalogue; reject impossible pairs up-front.
* (C) Warn on "suspicious" years (e.g. > current year + 2) but still run.

**Risk if wrong.** With (A), you may get empty deliverables for typos (e.g. `2029` instead of `2024`). With (B), legitimate early-access or leaked-spec runs are blocked.

***

### Q-C-5 — Do you want to submit multiple cars in one request?

**Assumption.** Yes, but internally we decompose the batch into independent single-car runs; each run has its own approval gate, its own KB, and its own deliverable. No cross-car sharing and no single batch deliverable.

**Why we ask.** If you expected a single consolidated batch deliverable, the multi-deliverable output may surprise you.

**Options.**

* (A) Confirm: N deliverables for N cars. *(current assumption)*
* (B) Add a consolidated batch summary deliverable on top of the per-car deliverables.
* (C) Only accept one car per request; reject batches at the entry point.

**Risk if wrong.** You expect one consolidated report but receive N separate ones.

***

## Section 2 — Source selection and approval

### Q-C-6 — What should "full option" mean precisely?

**Assumption.** "Full option" = the manufacturer's **published top trim** for the specified model / year / market. If multiple top trims exist (e.g. Sport vs Luxury premium variants), we select the most feature-rich one. The resolved trim name is recorded in the deliverable header.

**Why we ask.** This definition is the scope contract. If your analysts mean something different by "full option" (e.g. "all options ticked", independent of trim), every deliverable is scoped wrong.

**Options.**

* (A) Confirm: manufacturer's published top trim. *(current assumption)*
* (B) "All options ticked" — the superset of every option available on any trim.
* (C) Named trim supplied by you (override the auto-resolution).

**Risk if wrong.** Every deliverable is either over-scoped or under-scoped.

***

### Q-C-7 — Can classification start with a single approved URL?

**Assumption.** Yes. There is no minimum source coverage. If you approve only one URL (or even zero), classification proceeds and coverage gaps surface as Rule 4 records.

**Why we ask.** Some clients prefer the skill to refuse to run on too-thin evidence.

**Options.**

* (A) Confirm: no minimum. *(current assumption)*
* (B) Require a minimum number of approved URLs before classification starts.
* (C) Require a minimum per category (e.g. at least one source covering each category present in the CSV).

**Risk if wrong.** With (A), very thin approval sets produce deliverables that look like the car has almost no features (mostly Rule 4 silent-all).

***

### Q-C-8 — How should paywalled sources be handled?

**Assumption.** Paywalled URLs are **silently dropped** during discovery and never proposed to you.

**Why we ask.** If you have subscriptions or credentials to specific paywalled sites, silent dropping throws away evidence you could legitimately use.

**Options.**

* (A) Confirm silent drop. *(current assumption)*
* (B) Surface paywalled URLs to you at the approval gate with a paywall flag; you decide whether to provide credentials or drop them.
* (C) Silently drop, but include the dropped URLs in the deliverable footer for transparency.

**Risk if wrong.** You lose access to authoritative paywalled sources (trade publications, OEM subscription portals) without ever knowing they were found.

***

### Q-C-9 — Do you want multilingual sources?

**Assumption.** All languages are treated equally. Non-English sources can be proposed, approved, and ingested.

**Why we ask.** If you only review English sources at the approval gate, non-English proposals may waste review cycles.

**Options.**

* (A) Confirm all languages. *(current assumption)*
* (B) English-only.
* (C) Configurable per run (you specify the allowed languages at submit time).

**Risk if wrong.** With (A), you may receive approval requests in languages you cannot read; with (B), you lose valuable market-specific sources.

***

### Q-C-10 — Default cap of 20 candidate URLs per run (Mode B) — acceptable?

**Assumption.** In agent-discovered mode we flag at most 20 candidate URLs per run unless you explicitly set a different cap at submission. URLs you supply directly (Mode A) are uncapped.

**Why we ask.** 20 is a soft default that balances review burden and coverage breadth. Your analysts may have a different tolerance.

**Options.**

* (A) Confirm 20 default. *(current assumption)*
* (B) Change default to a different number.
* (C) Remove the cap entirely (review everything the web search returns).

**Risk if wrong.** Too low → coverage gaps, many Rule 4 verdicts. Too high → review burden at the approval gate becomes unmanageable.

***

### Q-C-11 — Who triggers the resume-classification signal?

**Assumption.** After flagging candidate URLs, the agent yields. You review via the approval store and send an explicit **"resume classification"** message back to the agent.

**Why we ask.** The alternative is an automated listener that detects when your approval activity has stopped.

**Options.**

* (A) Confirm explicit resume message. *(current assumption)*
* (B) Automated listener — agent resumes when approval activity quiesces for a configured interval.
* (C) Either works — let us choose internally.

**Risk if wrong.** With (A), a forgotten resume message stalls the run indefinitely. With (B), you may still be mid-review when the agent resumes.

***

### Q-C-12 — Can the agent run without the approval gate (automated mode)?

**Assumption.** Yes, as an opt-in workflow (W3 in `13-potential-workflows.md`). If you flag a run as "automated", we skip the approval gate and ingest all canonicalised, non-paywalled URLs directly.

**Why we ask.** Automated runs remove the bottleneck of human approval but give up the ability to curate sources before ingestion.

**Options.**

* (A) Confirm: automated mode is opt-in. *(current assumption)*
* (B) Make automated mode the default; approval gate is opt-in.
* (C) Automated mode is disallowed; approval gate is always on.

**Risk if wrong.** Mismatch produces either too much review friction (C) or deliverables you never had a chance to curate (B).

***

## Section 3 — Decision behaviour and conflict handling

### Q-C-13 — Every disagreement between clear sources emits `Conflict` — acceptable?

**Assumption.** Any disagreement between two or more clear active sources always emits `Status = Conflict` immediately. We do not attempt any tiebreak (no "official > press > blog" ranking, no recency, no market-match preference). You adjudicate every conflict.

**Why we ask.** This is the design decision that most directly affects your review burden. The trade-off is: more `Conflict` records you must resolve manually, versus the risk of the agent silently picking the "wrong" source to trust.

**Options.**

* (A) Confirm: no tiebreak, emit `Conflict`. *(current assumption)*
* (B) Rank sources by a fixed hierarchy (e.g. manufacturer official > automotive press > blog) and auto-resolve.
* (C) Prefer the most recent source within a configurable window (e.g. last 12 months).
* (D) Prefer sources whose market matches the run's market exactly.

**Risk if wrong.** (A) produces many Conflict records and high review load. (B/C/D) produces silent overrides you may disagree with.

***

### Q-C-14 — Silent-all outcome — Resolved 2026-04-23

**Status: Resolved. Option B was chosen.**

**Decision.** When every reviewed source is silent on a parameter (Rule 4 — no active sources at all), the skill emits:

* `Presence = No Information Found`
* `Status = Unable to Decide`
* `Classification = Empty`
* No traceability blocks.

This is directly distinguishable from Rule 5 (explicit absence), which emits `Presence = No`, `Status = Success`, with at least one clear-source traceability block. Analysts can identify silent-all parameters by the `No Information Found` presence value without inspecting JSON structure. `No Information Found` as a `status` value remains retired.

***

### Q-C-15 — Evidence for undefined tiers — disregard and advise?

**Assumption.** If a source describes the feature at a tier not defined in `params.csv` for that parameter (e.g. source says "Basic" but the CSV only defines `High`), we **disregard** that source for classification and record an internal advisory note (in `.harness/`) so the CSV can be evolved later. The advisory is **not** part of the client-facing deliverable.

**Why we ask.** The alternative is to surface these findings directly to you so you can adjust the CSV in near real-time.

**Options.**

* (A) Confirm: advisory only, hidden from deliverable. *(current assumption)*
* (B) Surface advisories in the deliverable (e.g. a `schema_gaps` section).
* (C) Treat undefined-tier evidence as Rule 3 (`Unable to Decide`), not as "disregarded".

**Risk if wrong.** With (A), the `params.csv` evolves only when developers inspect `.harness/`. With (B), every run ships extra noise.

***

### Q-C-16 — Out-of-list findings — hidden from deliverable?

**Assumption.** If a source describes a feature that does not exist as a parameter in `params.csv`, we record it in `.harness/` for developer review only. It does not appear in the client-facing deliverable.

**Why we ask.** Same trade-off as Q-C-15. You may want to see out-of-list findings.

**Options.**

* (A) Confirm: hidden from deliverable. *(current assumption)*
* (B) Surface in a `discovered_parameters` section of the deliverable.
* (C) Hidden by default; surface on request via a workflow flag.

**Risk if wrong.** With (A), the CSV evolves slowly and new features go unreported to you.

***

### Q-C-17 — Two runs on the same car are not guaranteed to be identical — acceptable?

**Assumption.** Two runs on the same car, same CSV, and same approved source set may produce different deliverables. The deliverable header discloses this explicitly. Re-runs are for coverage, not reproducibility.

**Why we ask.** If your downstream processes require reproducibility (e.g. compliance audit), non-determinism is a hard constraint.

**Options.**

* (A) Confirm: accept non-determinism with disclosure. *(current assumption)*
* (B) Add a determinism mode with fixed random seeds and deterministic retrieval ordering (higher implementation cost, slower runs).
* (C) Version every deliverable with a content-hash; require regeneration only when the hash changes.

**Risk if wrong.** Audit / compliance teams may refuse to accept deliverables that cannot be reproduced bit-for-bit.

***

## Section 4 — Deliverable

### Q-C-18 — JSON primary + CSV secondary — correct formats?

**Assumption.** Primary deliverable: a structured JSON file named `<brand>-<model>-<year>-<market>-<run-id>.json`. Secondary: a flattened CSV for spreadsheet review. Both are delivered side-by-side.

**Why we ask.** If you only want one format, the other is wasted work; if you want additional formats (Markdown, PDF, XLSX), we need to design them.

**Options.**

* (A) Confirm JSON + CSV. *(current assumption)*
* (B) JSON only.
* (C) CSV only.
* (D) JSON + CSV + human-readable report (Markdown or PDF).

**Risk if wrong.** Formats you cannot consume ship unused; formats you need ship missing.

***

### Q-C-19 — `Schema type` encoding: compact (`B/H`, `B/M/H`) — acceptable?

**Assumption.** The `Schema type` field uses compact notation: `B/H` means Basic+High are defined; `B/M/H` means all three; `H` means High only. A legend is in the deliverable header.

**Why we ask.** Compact notation is dense; your analysts may prefer explicit.

**Options.**

* (A) Confirm compact (`B/H`). *(current assumption)*
* (B) Verbose (`Basic+High`, `Basic+Medium+High`).
* (C) Structured list (`["Basic", "High"]` in JSON; pipe-separated in CSV).

**Risk if wrong.** Notation mismatch breaks downstream parsers or dashboard visualisations.

***

### Q-C-20 — Record order = `params.csv` row order — acceptable?

**Assumption.** Records in the deliverable appear in the same order as rows in `params.csv`, preserving category grouping and author intent.

**Options.**

* (A) Confirm CSV row order. *(current assumption)*
* (B) Sort by category + parameter name alphabetically.
* (C) Sort by Status (Conflicts first, then Unable to Decide, then Success) to surface review-worthy records at the top.

**Risk if wrong.** Your analysts may expect a specific ordering and mis-read the file otherwise.

***

### Q-C-21 — Run-level summary + analytics section — include?

**Assumption.** The deliverable includes a run-level summary with:

* Counts: Total Parameters, Yes, No, Conflict, Unable to Decide.
* Distribution by schema type (Basic / Medium / High).
* Full list of Conflict and Unable-to-Decide parameters.
* Extended telemetry: approved / ingested / failed source counts, ingestion and classification durations, sub-agent execution log excerpt.

**Options.**

* (A) Confirm summary + telemetry. *(current assumption)*
* (B) Summary only (no telemetry).
* (C) Minimal — per-parameter records only, no run-level summary.

**Risk if wrong.** Too much → deliverable bloat. Too little → you lose visibility into runs.

***

### Q-C-22 — Extended header with timestamps — acceptable?

**Assumption.** Each deliverable carries an extended header: run ID, submission + completion timestamps, resolved car identity, resolved trim name, CSV version/hash, source counts, ingestion start/end, classification start/end, total elapsed time, sub-agent count.

**Options.**

* (A) Confirm extended header. *(current assumption)*
* (B) Minimal header (run ID + car identity only).

**Risk if wrong.** Extended header is metadata your downstream may want to ignore; minimal may hide diagnostic info you need.

***

### Q-C-23 — Failed sources reported in a footer section only?

**Assumption.** A dedicated `sources_excluded` footer lists every URL that failed to download or ingest, with stage of failure and a short reason. No per-parameter mention of failed sources (because failed sources were never ingested, so we have no evidence of their relevance to specific parameters).

**Options.**

* (A) Confirm footer-only. *(current assumption)*
* (B) Annotate every parameter record with "potentially missing sources" based on URL / category heuristics.
* (C) Drop failed sources entirely — no footer.

**Risk if wrong.** Too little visibility into failures undermines your confidence in the deliverable.

***

### Q-C-24 — No intermediate review gate — acceptable?

**Assumption.** The merged deliverable is emitted directly as the final output. No intermediate harness-review step before you see it. If you disagree with any classification you trigger a gap-fill (W5) or a re-run (W4).

**Options.**

* (A) Confirm: no intermediate gate. *(current assumption)*
* (B) Insert an intermediate review pass where you approve the deliverable before it is finalised.
* (C) Auto-flag records for mandatory review when Status = Conflict or Unable to Decide; emit only after you resolve flags.

**Risk if wrong.** (A) makes post-delivery corrections the client's responsibility; (B/C) lengthens every run.

***

## Section 5 — Re-runs and gap-fill

### Q-C-25 — Three re-run patterns — correct set?

**Assumption.** Three re-run workflows are supported:

1. **Full-car re-run** (W4) — new run ID, new approval round, full re-classification.
2. **Gap-fill cycle** (W5) — targets silent / weak-coverage parameters. Reuses the same KB and run ID; adds new sources tagged with a cycle number.
3. **Per-parameter flagging** (W6) — re-classifies specific parameters, optionally with new sources.

Gap-fill and per-parameter flagging are **optional** and triggered by you after reviewing the deliverable.

**Why we ask.** If you only ever use one of these patterns, the others are dead weight in the workflow layer.

**Options.**

* (A) Confirm all three. *(current assumption)*
* (B) Drop per-parameter flagging; only full re-run and gap-fill.
* (C) Drop gap-fill; only full re-run.
* (D) Add a fourth pattern not listed above (specify).

**Risk if wrong.** Patterns you do not use become confusing surface area; patterns you need become unsupported friction.

***

### Q-C-26 — Gap-fill gate: your choice per cycle — acceptable?

**Assumption.** Each gap-fill cycle can run with or without the approval gate — you choose at the time of triggering gap-fill.

**Options.**

* (A) Confirm: your choice per cycle. *(current assumption)*
* (B) Gap-fill always has the approval gate.
* (C) Gap-fill never has the approval gate (always automated).

**Risk if wrong.** Wrong default creates either bottleneck (B) or uncurated expansions (C).

***

### Q-C-27 — Concurrent runs on the same car — independent?

**Assumption.** Two or more runs on the same `(brand, model, year, market)` tuple are allowed and treated as fully independent. Each has its own run ID, KB, approval round, and deliverable. No coordination, no merging.

**Options.**

* (A) Confirm: fully independent concurrent runs. *(current assumption)*
* (B) Disallow concurrent runs on the same car; reject the second until the first completes.
* (C) Allow concurrent runs but merge their results at the end.

**Risk if wrong.** (C) is complex and not designed for; (B) may frustrate parallel teams.

***

### Q-C-28 — CSV is frozen per run — acceptable?

**Assumption.** `artifacts/params.csv` is developer-owned (not client-owned). It is injected into the run workspace at the start of each run and frozen for the entire run including every gap-fill cycle. A new CSV version only takes effect in a completely new run.

**Options.**

* (A) Confirm: frozen per run, developer-owned. *(current assumption)*
* (B) Client-owned CSV — you upload it with each request.
* (C) Hot-reload mid-run (new CSV version affects remaining parameters in the current run).

**Risk if wrong.** (A) means CSV updates propagate only on the next new run; (B/C) place CSV management on you.

***

## Section 6 — Things we are **not** asking the client about

For transparency, here is the list of decisions recorded in `11-assumptions.md` that we consider **internal** and are not raising with the client. If any of these is actually client-facing in your view, flag it and we will escalate it:

* KB retention window in days (audit internals).
* KB reuse across runs (internal; client-facing effect is latency only).
* RAGFlow polling cadence and progress reporting (internal).
* Download retry count and backoff schedule (internal).
* Ingestion timeout value (internal; client-facing effect is dropped docs in footer).
* Chunk-level citation granularity inside the KB (internal; `Source link` is URL-level).
* Sub-agent partition heuristic and category-splitting thresholds (internal).
* Tool vendors and concrete platform bindings (ARCH-1 — platform-agnostic by design).
* `.harness/` workspace layout (internal developer tooling).
* URL canonicalisation rules (internal; visible only as dedup behaviour).

***

## How to respond

For each question `Q-C-N`, reply with one of:

* `Q-C-N = A` (confirm current assumption).
* `Q-C-N = B` (or C, D — one of the alternatives listed).
* `Q-C-N = custom:` followed by a freeform answer.

Multiple confirmations in a single message are welcome. We will record each response back into `11-assumptions.md` (promoting working assumptions to settled, or reopening them with your alternative).
