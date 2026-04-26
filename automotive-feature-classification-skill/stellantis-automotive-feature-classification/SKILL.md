---
name: stellantis-automotive-feature-classification
description: Use when user names a specific car (brand, model, year, market) and asks about features, parameters, options, capabilities, or levels.
stage: W1-entry
requires:
  - stellantis-domain-context
  - stellantis-decision-rules
  - stellantis-output-contract
  - stellantis-workflow-modes
  - stellantis-failure-handling
  - stellantis-knowledge-base-protocol
  - stellantis-run-workspace
  - stellantis-car-trim-resolution
  - stellantis-source-discovery-with-client-domains
  - stellantis-source-discovery-without-client-domains
  - stellantis-source-validation
  - stellantis-approval-gate
  - stellantis-source-download-and-ingest
  - stellantis-lead-agent-subagent-orchestration
  - stellantis-subagent-classification-loop
provides:
  - run-orchestration
  - deliverable.json
  - deliverable.csv
tools: []
---

## Dependencies

This skill orchestrates two layers of sub-skills, loaded lazily at the workflow stage that needs them.

### Foundational skills (load at preflight, used by every stage)

| Foundational skill | Role | Purpose |
|---|---|---|
| `stellantis-domain-context` | Lead + Subagent | Mission, vocabulary, master invariants, glossary, tips |
| `stellantis-decision-rules` | Lead + Subagent | Five decision rules + master matrix + 8 invariants |
| `stellantis-output-contract` | Lead + Subagent | Per-parameter record shape, deliverable format, run header, footer |
| `stellantis-workflow-modes` | Lead | Identify workflow at preflight; enforce composition rules |
| `stellantis-failure-handling` | Lead + Subagent | Retries, drops, timeouts, `sources_excluded` policy |
| `stellantis-knowledge-base-protocol` | Lead + Subagent | KB capability contract; ingestion wait; retrieval discipline |
| `stellantis-run-workspace` | Lead + Subagent | Workspace layout, writer ownership, pause/resume contract |

### Operational sub-skills (loaded per stage)

| Sub-skill | Stage(s) | Role | Purpose |
|-----------|----------|------|---------|
| `stellantis-car-trim-resolution` | W1-stage-1b | Lead | Resolve trim to manufacturer's published top trim |
| `stellantis-source-discovery-with-client-domains` | W1-stage-3-modeA | Lead | Search within user-supplied domains/URLs |
| `stellantis-source-discovery-without-client-domains` | W1-stage-3-modeB | Lead | Open-web search (Mode B) |
| `stellantis-source-validation` | W1-stage-3b | Lead | Validate & filter candidate source list |
| `stellantis-approval-gate` | W1-stage-3c | Lead | Flag URLs, yield, resume on explicit signal |
| `stellantis-source-download-and-ingest` | W1-stage-4, W1-stage-5 | Lead | Download URLs, upload to KB, wait for ingestion |
| `stellantis-lead-agent-subagent-orchestration` | W1-stage-6, W1-stage-6b | Lead | Partition (≤ 15 params/agent), spawn, consolidate |
| `stellantis-subagent-classification-loop` | W1-stage-6a | Subagent | Classify parameters using KB retrieval |

**Key reference:** See `REFERENCES.md` for template and schema IDs (`state-main-template`, `enums-reference`, etc.).

---

# Automotive Feature Classification

## Overview

Entry point for lead agent. Operationalizes the automotive feature classification framework — a structured workflow for mapping specific car parameters against a reference CSV, with defined roles (lead agent vs. subagent), state management, and deliverable formats.

**Vocabulary:** Use **category** (not domain) for features; **source site** (not domain) for hosts. Accept user's "domain" when context is features; acknowledge using "category".

## When to Use

Load this skill when the user's request matches ALL of:

1. Names a specific car (brand + model are named or clearly inferable)
2. Asks about features, parameters, options, capabilities, or levels
3. Does not instruct the agent to bypass the classification workflow

If any of the four required fields — brand, model, model year, market — cannot be extracted, pause immediately and ask **only for missing field(s)**.


## Core Pattern: Lead Agent Lifecycle

### Preflight

Executed exactly once per run, before loading any workflow or sub-skill:

1. **Load foundational skills.** `stellantis-domain-context`, `stellantis-decision-rules`, `stellantis-output-contract`, `stellantis-workflow-modes`, `stellantis-failure-handling`, `stellantis-knowledge-base-protocol`, `stellantis-run-workspace`. These stay in context for the rest of the run.
2. **Identify workflow.** Use `stellantis-workflow-modes` to classify the request. Default to W1; pick Mode A if the user supplied URLs or domains, otherwise Mode B. Record `workflow` and `mode` in the main state file.
3. **Generate run ID:** `<YYYYMMDD>-<brand-slug>-<model-slug>-<year>-<market-slug>-<short-hash>`.
4. **Create run workspace** per `stellantis-run-workspace`.
5. **Freeze the reference list:** copy `artifacts/params.csv` → `runs/<run-id>/params.csv`, record hash in the main state file.
6. **Initialize STATE.md** files from templates.

### Roles & Responsibilities

High-level split:

| Concern                                                    | Lead agent | Subagent |
| :--------------------------------------------------------- | :--------- | :------- |
| Parse request and resolve the 4 required fields            | ✅          | —        |
| Resolve ambiguous car identity + full-option trim          | ✅          | —        |
| Create + manage the KB dataset                             | ✅          | —        |
| Source discovery (web search) + validation                 | ✅          | —        |
| Post candidate URL list to user; interpret free-text reply | ✅          | —        |
| Download + upload + ingestion polling                      | ✅          | —        |
| Partition parameters + spawn subagents + monitor           | ✅          | —        |
| KB retrieval per parameter + read chunks + write verdicts  | —          | ✅        |
| Update per-document metadata with found parameters         | —          | ✅        |
| Consolidate subagent working files into category STATE     | ✅          | —        |
| Archive consolidated subagent files                        | ✅          | —        |
| Emit deliverable.json + deliverable.csv                    | ✅          | —        |

**Hard rule:** subagents **never** use open-web search or browser rendering. Their only tools are the knowledge-base retrieval/inspection set — `retrieval`, `retrieve_knowledge_graph`, `list_chunks`, `download_attachment`, `doc_infos`, `get_metadata_summary`.

**Concurrency:** maximum **3 subagents running concurrently** (enforced by `lead-agent-subagent-orchestration` skill).

## Quick Reference

### STATE Files

The framework uses a **hybrid** state layout. The client-facing summary lives at the run-workspace root; agent-internal detail lives under `.harness/`.

**Client-facing:**
- `runs/<run-id>/STATE.md` (lead-write only) — submission date, car identity, KB dataset ID, CSV hash, summary counts, per-category roster, event log

**Agent-internal (`.harness/`):**
- `.harness/Category/<category>.md` (lead-write only) — consolidated per-parameter records, assigned subagents, category warnings
- `.harness/SubAgent/<agent-name>.md` (subagent-write) — contract JSON header + scratch + final output
- `.harness/Archive/<agent-name>.md` — consolidated files moved after merge
- `.harness/advisories/` — undefined-tier advisories + out-of-list findings (internal only)

**Writer discipline:** Lead writes main STATE.md and `.harness/Category/*.md` only. Subagents write only their own `.harness/SubAgent/<agent-name>.md`. Subagent discovers its own name from file path stem (passed on spawn).

### Workflow & Sub-Skill Loading

1. After preflight, execute the W1 pipeline stage by stage
2. Load sub-skills lazily (not pre-loaded) at the stage that needs them. See **Dependencies** section above for the full list, stages, and purposes.
3. Sub-skill header specifies `loader: lead` or `loader: subagent` — enforce isolation

### Consolidation & Archiving

On subagent `status: consolidated-ready`:

1. Validate every per-parameter record against the intermediate parameter record schema and `enums-reference`
2. Merge validated records into `.harness/Category/<category>.md`
3. Update summary counts in main STATE.md
4. Archive: move `.harness/SubAgent/<agent-name>.md` → `.harness/Archive/<agent-name>.md`
5. Promote warnings → `.harness/Category/<category>.md` (surfaced in deliverable); advisories → `.harness/advisories/` (internal only)

**Serial constraint:** One consolidation must complete before next begins (even if subagents finish concurrently).

### Pause/Resume Protocol

**To pause** (deterministic state write, no busy-loop):
1. Set in STATE.md: `RunStatus = paused`, `PauseReason`, `PausedAt`, `WorkflowStage`
2. Append event-log line explaining user action required
3. Post required artifact:
   - `awaiting-source-approval`: URL list + instructions: approve all / approve indexes / reject indexes
   - `awaiting-client-resume`: short note, user sends `resume`
   - `ingestion-error-client-confirmation-needed`: list not-yet-ingested docs + "wait longer or proceed?"
   - `missing-required-input`: prompt for missing field(s) only
4. Return control

**To resume:**
1. Verify `RunStatus = paused`
2. Interpret user's free-text reply per rules in stage's sub-skill
3. Update STATE.md: `RunStatus = active`, clear `PauseReason`, set `ResumedAt`
4. Continue from `WorkflowStage`

## Implementation: File Reading Maps

**Lead agent reads:**
- Preflight: this skill, `enums-reference`, frozen `params.csv`
- Each stage: current stage definition (see `W1-workflow-definition`), named sub-skill(s), main `STATE.md` (status check)
- Consolidation: target `.harness/SubAgent/<agent-name>.md`, target `.harness/Category/<category>.md`

**Subagent reads (on spawn):**
- `stellantis-subagent-classification-loop` skill (REQUIRED)
- `enums-reference`
- `lead-to-subagent-contract` and `subagent-to-lead-contract`
- Own `.harness/SubAgent/<agent-name>.md` (contract embedded at top)
- Relevant category rows from frozen `params.csv`

**Subagent never touches:**
- Main `STATE.md`
- Any `.harness/Category/*.md` file
- Other subagent's working file
- Tool manifests beyond those defined in `subagent-classification-loop` skill

## Tool Allow-List

Per-actor permissions (tool bindings in sub-skill files):

| Capability          | Tools                                                                                                                       | Lead      | Subagent          |
|:---                 |:---                                                                                                                         |:---       |:---               |
| Open-web search     | `google_search`, `google_search_news`, `google_search_images`, `google_search_videos`, `google_search_autocomplete`         | ✅        | ❌                |
| Browser rendering   | `browser_render_content`, `browser_render_markdown`, `browser_render_pdf`, `browser_render_scrape`, `browser_render_links`  | ✅        | ❌                |
| Crawl workflow      | `browser_crawl_create`, `browser_crawl_status`, `browser_crawl_cancel`                                                      | ✅        | ❌                |
| Dataset mgmt        | `create_dataset`, `list_datasets`, `get_dataset_detail`, `update_dataset`, `delete_datasets`                                | ✅        | ❌                |
| Document mgmt       | `upload_with_metadata`, `list_docs`, `doc_infos`, `delete_docs`, `rename_doc`, `set_doc_metadata`, `batch_update_doc_metadata` | ✅ write  | ❌                |
| Metadata summary    | `get_metadata_summary`                                                                                                      | ✅        | ✅ read-only      |
| Chunk inspection    | `list_chunks`, `download_attachment`                                                                                        | ✅ rare   | ✅                |
| Retrieval           | `retrieval`, `retrieve_knowledge_graph`                                                                                     | ✅ rare   | ✅                |
| Tag operations      | `list_tags`, `set_tags`                                                                                                     | ✅        | ❌                |

Web search and browser rendering may be used only in source discovery and download phases. After ingestion completes, classification reads evidence exclusively from the knowledge base.

## Common Mistakes

- **Don't pre-load operational sub-skills** — load them lazily at the stage that needs them. Foundational skills (domain-context, decision-rules, output-contract, workflow-modes, failure-handling, kb-protocol, run-workspace) are the exception: they load at preflight and stay.
- **Don't let subagents use web search or browser tools** — they read evidence from the knowledge base only.
- **Don't let subagents write STATE.md or category files** — violations corrupt consolidation.
- **Don't batch consolidations** — serialize even if subagents finish concurrently.
- **Don't write records before validating** — always check against template schema and enums first.
- **Don't default required inputs.** If `brand`, `model`, `model_year`, or `market` is missing, pause and ask for that field only.
- **Don't re-search the web after the approval gate closes.** Coverage gaps surface as Rule 4 records; gap-fill (when authored) is the dedicated remedy.
- **Don't silently chain workflows.** A re-run is a new run. A gap-fill is a separate workflow. Each is initiated by an explicit user action.
- **Don't skip the foundational skills' tips sections.** They encode lessons learned across many runs.

## References & Maintenance

Review this skill and related workflows when operational rules change.

## Failure Modes

| Failure                              | Response                                                                              |
|:---                                  |:---                                                                                  |
| Missing required input               | Pause with `PauseReason = missing-required-input`; ask only for missing field(s)     |
| Download fails after 3 retries       | Drop URL; record in `sources_excluded.md`; `FailureStage = download`                |
| Ingestion exceeds 60-min per-doc     | Drop doc; record in `sources_excluded.md`; `FailureStage = ingestion`                |
| Partial ingestion, timeout not hit   | Pause with `PauseReason = ingestion-error-client-confirmation-needed`; list docs    |
| Subagent invalid record              | Mark failed; re-spawn once with same contract; if fails again, lead writes manually  |
| Concurrent-run collision             | Allowed; fully independent. No coordination required.                               |

## What This File Is NOT

- Not the workflow
- Not classification verdict rules
- Not a tool reference
- Not prose — it is normative. Violations are bugs.

