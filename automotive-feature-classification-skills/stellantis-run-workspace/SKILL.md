---
name: stellantis-run-workspace
description: Use whenever an agent reads or writes a run-scoped file. Defines the workspace layout, writer ownership, client-facing vs internal split, and the resumability contract.
loader: shared
stage: any
requires:
  - stellantis-domain-context
  - stellantis-output-contract
provides:
  - workspace-layout
  - writer-ownership-rules
  - resumability-contract
tools: []
---

## Dependencies

- `stellantis-domain-context` — vocabulary.
- `stellantis-output-contract` — defines deliverable file shapes.

Used by every operational skill — the lead and every subagent — whenever they touch the workspace.

---

# Sub-skill — Run workspace (flat .harness structure)

## Overview

Every car analysis uses a single `.harness/` folder at the project root (no `runs/<run-id>/` nesting). Inside it: the internal agent working memory organized by concern. Client-facing deliverables live at the project root alongside `.harness/`. This skill specifies what lives where, who can write what, and how the layout supports resumability.

## When this runs

Inline, on any read or write to `.harness/` or the project-root deliverables.

## Layout

```
<project-root>/
├── .harness/                                         # ALL agent-internal state (one per car)
│   ├── STATE.md                                      # comprehensive run state (lead-write)
│   ├── params.csv                                    # frozen reference list snapshot
│   ├── downloads/                                    # PRE-UPLOAD STAGING AREA
│   │   ├── <slug>.md                                 # downloaded & converted markdown
│   │   ├── <slug>.pdf                                # downloaded PDF
│   │   └── <slug>.<ext>                              # other document types
│   ├── DownloadAgent/                                # download subagent working files (W1 stage 4)
│   │   └── <slug>-dl.md                              # per-URL download subagent working file
│   ├── Category/
│   │   └── <category>.md                             # consolidated per-category records (lead-write)
│   ├── SubAgent/
│   │   └── <agent-name>.md                           # active classification subagent working file (subagent-write)
│   ├── Archive/
│   │   ├── <agent-name>.md                           # consolidated subagent files moved post-merge
│   │   └── <slug>-dl.md                              # archived download subagent working files
│   ├── advisories/                                   # internal-only artefacts
│   │   ├── undefined-tier-evidence.md
│   │   └── out-of-list-findings.md
│   ├── source-candidates.md                          # validated candidate list before approval
│   ├── source-approved.md                            # parsed approved URL set
│   └── sources_excluded.md                           # download/upload/ingestion drops; feeds deliverable footer
│
├── <brand>-<model>-<year>-<market>-<run-id>.json     # client-facing deliverable (primary)
├── <brand>-<model>-<year>-<market>-<run-id>.csv      # client-facing deliverable (secondary)
└── <other project files>
```

## Client-facing vs internal

| Audience | Path |
| :--- | :--- |
| **Client-facing (deliverable + audit)** | The two deliverable files at project root. `.harness/STATE.md` is readable but primarily for audit. |
| **Internal (agent working memory)** | Everything under `.harness/`. Staging area: `.harness/downloads/`. |

The client may read `.harness/STATE.md` for audit trail; `.harness/downloads/` is for agent reference during the run and audit afterward. The two deliverable JSON/CSV files are the primary client output.

## Writer ownership

Strict ownership prevents corruption when subagents run concurrently.

| Path | Writer |
| :--- | :--- |
| `.harness/STATE.md` | **Lead only.** |
| Deliverable files (project root) | **Lead only.** |
| `.harness/params.csv` (frozen snapshot) | **Lead only**, written once at preflight. Never edited after. |
| `.harness/downloads/*` | **Download subagents** — written by each subagent during W1 stage 4; preserved for audit. Never deleted after upload. |
| `.harness/DownloadAgent/<slug>-dl.md` | **Lead** seeds the contract block at spawn; **named download subagent** writes scratch + result envelope. |
| `.harness/Category/*.md` | **Lead only.** Updated during consolidation. |
| `.harness/SubAgent/<agent-name>.md` | **Lead** seeds the contract block at spawn; **named classification subagent** writes everything after. |
| `.harness/Archive/*.md` | **Lead only.** Files are moved here after consolidation (both download and classification subagents). |
| `.harness/advisories/*` | **Classification subagents** for parameter-level advisories; **lead** for run-level advisories. |
| `.harness/source-*.md` | **Lead only.** |

Subagents never touch any file outside their own `.harness/SubAgent/<agent-name>.md` and the advisory files they explicitly own.

## State files

### `.harness/STATE.md` (comprehensive run state)

Records, at minimum:
- `run_id`, `submitted_at`, `completed_at`
- Resolved `car_identity` and `resolved_full_option_trim`
- Frozen reference-list snapshot hash
- Workflow id and (if W1) mode
- KB dataset ID
- Source counts: `{approved, rejected, uploaded, failed, ingested}`
- Download & upload tracking:
  - Per-URL: download timestamp, file size, local path in `.harness/downloads/`
  - Per-URL: upload attempt timestamps, retry counts, 5-min timeout status
- Upload completion gate decision (timestamp, outcome)
- Ingestion tracking:
  - Per-doc: ingestion_started_at, ingestion_completed_at, final status
- Per-category roster (categories × subagent partitions × status)
- Comprehensive event log (every step logged here)

### `.harness/downloads/` (pre-upload staging)

New directory structure introduced to address the architectural requirement: **"agent should scrape everything with the markdown and put it in .harness under download and then upload them"**

Files here:
- `<slug>.md` — Markdown-converted HTML pages
- `<slug>.pdf` — Downloaded PDFs
- `<slug>.<ext>` — Other document types (binary, unchanged)

Lifecycle:
1. **Created** during download phase (W1 stage 4).
2. **Referenced** during upload phase — each file uploaded from here with full metadata.
3. **Preserved** after upload for audit trail (not deleted).
4. **Accessible** to user if they wish to verify markdown quality before ingestion.

### `.harness/SubAgent/<agent-name>.md` (per subagent)

Has three blocks, in order:
1. **Contract header** (`<!-- AUTOGENERATED BY LEAD -->`) — seeded once by the lead at spawn; never re-edited.
2. **Scratch** — the subagent's working notes during retrieval / decision iterations.
3. **Result envelope** (`<!-- RESULT ENVELOPE -->`) — final output the lead consolidates.

### `.harness/Category/<category>.md` (per category)

Lead-authored consolidation of all subagents that produced records for that category. Includes per-parameter records, assigned subagent names, and category-level warnings.

## Pause / resume contract

The workspace is the only state store. The agent is allowed to disappear and come back.

To **pause**:
1. Update `.harness/STATE.md`: `RunStatus = paused`, `PauseReason = <reason>`, `PausedAt = <timestamp>`, `WorkflowStage = <stage id>`.
2. Append an event-log line stating the user action required.
3. Post the required artefact in the chat surface (URL list, missing-field prompt, etc.).
4. Return control. Do not poll.

To **resume**:
1. Read `.harness/STATE.md` for `RunStatus`, `PauseReason`, `WorkflowStage`.
2. Verify `RunStatus = paused`.
3. Interpret the user's reply per the rules of the stage's sub-skill.
4. Set `RunStatus = active`, clear `PauseReason`, set `ResumedAt = <timestamp>`.
5. Continue from `WorkflowStage`.

If the user's reply is insufficient (e.g. still ambiguous), re-prompt — never default silently.

## Concurrency rules

- **Multiple subagents may run concurrently.** Up to 3 (enforced by `stellantis-lead-agent-subagent-orchestration`).
- **Each subagent writes only its own working file.** No file is shared as a write target.
- **Consolidation is serial.** Even if two subagents finish at the same time, the lead consolidates them one after another.
- **Multiple concurrent car analyses each have their own `.harness/`.** No cross-run coordination. (To support this, car identity must be part of the analysis context, or separate projects must be used.)

## Resumability checklist

For a run to be safely resumable from a clean restart:
- `.harness/STATE.md` must always reflect the latest committed state (atomic-ish writes).
- The frozen `.harness/params.csv` must remain unchanged for the entire run, including all gap-fill cycles.
- Subagent working files must be self-describing — the contract block alone is enough to re-spawn.
- The `.harness/sources_excluded.md` log must be append-only.
- Downloaded files in `.harness/downloads/` are preserved and referenced in STATE.md.

## Tips & common pitfalls

- **Create .harness FIRST.** Before any other processing in preflight, create the `.harness/` folder structure.
- **Save to downloads/ before uploading.** Every downloaded file goes to `.harness/downloads/` first, then upload references it there.
- **Don't write the deliverable until consolidation is complete for every partition.** The deliverable is the final commit.
- **Don't move subagent files out of `SubAgent/` before consolidation finishes.** Archive comes after merge, not before.
- **Don't put internal artefacts into the root.** The root has only deliverables; `.harness/` contains working files.
- **Don't write to another agent's working file.** Even the lead does not edit a subagent's scratch — it reads and consolidates.
- **Don't edit the frozen reference list mid-run.** A new reference list applies to the next run only.
- **Don't rely on memory across pauses.** Read state from `.harness/STATE.md` on every resume.
- **Don't delete `.harness/downloads/` after upload.** Preserve for audit trail.

## What this skill is NOT

- Not a tool reference.
- Not a deliverable schema (see `stellantis-output-contract`).
- Not the failure policy.

## Maintenance

Adding a new file under `.harness/` requires:
1. Update this skill's layout tree.
2. Update any reference docs (e.g., `automotive-feature-classification/references/file-ownership-matrix.md` and `workspace-layout.md`).
3. Update the producer skill or workflow stage.
4. Apply follow-ups per `framework-maintenance/impact-checklist.md` "Add a new file under `.harness/`".

