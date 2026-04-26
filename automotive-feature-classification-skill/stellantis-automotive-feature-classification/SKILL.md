---
name: stellantis-automotive-feature-classification
description: Use when user names a specific car (brand, model, year, market) and asks about features, parameters, options, capabilities, or levels
stage: W1-entry
requires: []
provides:
  - run-orchestration
  - deliverable.json
  - deliverable.csv
tools: []
---

## Dependencies

This skill orchestrates all 7 sub-skills, loading each lazily at the workflow stage that needs it:

| Sub-skill | Stage(s) | Role | Purpose |
|-----------|----------|------|---------|
| `stellantis-car-trim-resolution` | W1-stage-1b | Lead | Resolve ambiguous trim to full-option variant |
| `stellantis-source-discovery-with-client-domains` | W1-stage-3-modeA | Lead | Search within client-supplied domains/URLs |
| `stellantis-source-discovery-without-client-domains` | W1-stage-3-modeB | Lead | Open-web search (Mode B fallback) |
| `stellantis-source-validation` | W1-stage-3b | Lead | Validate & filter candidate source list |
| `stellantis-source-download-and-ingest` | W1-stage-4, W1-stage-5 | Lead | Download URLs, upload to KB, trigger ingestion |
| `stellantis-lead-agent-subagent-orchestration` | W1-stage-6, W1-stage-6b | Lead | Partition params, spawn subagents, consolidate results |
| `stellantis-subagent-classification-loop` | W1-stage-6a | Subagent | Classify parameters using KB retrieval (spawned per partition) |

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

1. Generate run ID: `<YYYYMMDD>-<brand-slug>-<model-slug>-<year>-<market-slug>-<short-hash>`
2. Create run workspace with standard directory structure
3. Freeze CSV: copy `artifacts/params.csv` → `runs/<run-id>/params.csv`, record hash in STATE.md
4. Initialize STATE.md files from templates
5. Detect workflow: default W1 (Mode A) if user supplied URLs, else W2; record in STATE.md

### Roles & Responsibilities

High-level split:

| Concern                                                    | Lead agent | Subagent |
| :--------------------------------------------------------- | :--------- | :------- |
| Parse request and resolve the 4 required fields            | ✅          | —        |
| Resolve ambiguous car identity + full-option trim          | ✅          | —        |
| Create + manage the KB dataset                             | ✅          | —        |
| Source discovery (web search) + validation                 | ✅          | —        |
| Post candidate URL list to user; interpret free-text reply | ✅          | —        |
| Download + upload + `run_document` + ingestion polling     | ✅          | —        |
| Partition parameters + spawn subagents + monitor           | ✅          | —        |
| KB retrieval per parameter + read chunks + write verdicts  | —          | ✅        |
| Update per-document metadata with found parameters         | —          | ✅        |
| Consolidate subagent working files into category STATE     | ✅          | —        |
| Archive consolidated subagent files                        | ✅          | —        |
| Emit deliverable.json + deliverable.csv                    | ✅          | —        |

**Hard rule:** subagents **never** use open-web search or browser rendering. Their only tools are the knowledge-base retrieval/inspection set — `retrieval`, `list_chunks`, `download_attachment`, `doc_infos`, `get_metadata_summary`, `set_doc_metadata`, `batch_update_doc_metadata`.

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

| Capability          | Tools                                                                                    | Lead      | Subagent          |
|:---                 |:---                                                                                      |:---       |:---               |
| Open-web search     | google_search*, google_search_news, google_search_images, etc.                          | ✅        | ❌                |
| Browser rendering   | browser_render_*, browser_crawl_*                                                       | ✅        | ❌                |
| Dataset mgmt        | create_dataset, list_datasets, get_dataset_detail, update_dataset, delete_datasets      | ✅        | ❌                |
| Document mgmt       | upload_with_metadata, list_docs, doc_infos, delete_docs, rename_doc, set_doc_metadata  | ✅ write  | ✅ metadata only  |
| Document ingestion  | run_document                                                                             | ✅        | ❌                |
| Metadata summary    | get_metadata_summary                                                                     | ✅        | ✅ read-only      |
| Chunk inspection    | list_chunks, download_attachment                                                        | ✅ rare   | ✅                |
| Retrieval           | retrieval, retrieve_knowledge_graph                                                      | ✅ rare   | ✅                |

*Enforces hard phase separation*

## Common Mistakes

- **Don't pre-load sub-skills** — load lazily at the stage that needs them
- **Don't let subagents use web search or browser tools** — ARCH-4 violation; they use KB retrieval only
- **Don't let subagents write STATE.md or category files** — violations corrupt consolidation
- **Don't batch consolidations** — serialize even if subagents finish concurrently
- **Don't write records before validating** — always check against template schema and enums first

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

