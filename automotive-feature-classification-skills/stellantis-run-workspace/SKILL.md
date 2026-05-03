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

The agent runs inside a fixed host filesystem sandbox with **exactly three** accessible paths. All run state is concentrated in a single `.harness/` folder under the workspace path; the deliverable goes to a separate outputs path; user-supplied inputs arrive via a read-only uploads path. No other path on the host is read or written. This skill specifies what lives where, who can write what, and how the layout supports resumability.

## Filesystem sandbox (NORMATIVE)

The agent is allowed to access **only** these three paths. Any read or write outside them is a framework violation.

| Path | Mode | Purpose |
| :--- | :--- | :--- |
| `/mnt/user-data/workspace/` | read + write | The run workspace. Contains `/mnt/user-data/workspace/.harness/` and nothing else the agent writes. **All agent-internal run state lives under `.harness/`.** |
| `/mnt/user-data/outputs/` | read + write | Final client-facing deliverables only (`<brand>-<model>-<year>-<market>-<run-id>.json` and `.csv`). Nothing else is written here. |
| `/mnt/user-data/uploads/` | **read-only** | User-supplied input files (e.g. a custom `params.csv` to import at preflight, attached PDFs, URL lists). The agent never writes to this directory. Files needed during the run are copied into `.harness/` at preflight. |

Everything else — `/tmp`, `/home`, `~`, the repository root, current working directory, any other absolute path — is **out of scope**. Do not invent paths.

## When this runs

Inline, on any read or write to `/mnt/user-data/workspace/.harness/`, `/mnt/user-data/outputs/`, or `/mnt/user-data/uploads/`.

## Layout

All paths are absolute. There is no `runs/<run-id>/` nesting; one workspace = one run. Concurrent runs require separate sandboxes (separate workspace mounts), not subfolders.

```
/mnt/user-data/
├── workspace/                                              # the run workspace (read + write)
│   └── .harness/                                           # ALL agent-internal run state — every working file lives here
│       ├── STATE.md                                        # comprehensive run state (lead-write; T1/T2 tiered)
│       ├── params.csv                                      # frozen reference list snapshot (LEAD-ONLY READER during a run)
│       ├── params.csv.hash                                 # sha256 of params.csv
│       ├── event-log.md                                    # append-only mirror of STATE.md event log
│       ├── document-promise-board.md                       # aggregated R1 promise scores (lead-write from R1 envelopes)
│       ├── partition-summaries.md                          # R1 partition summaries (lead-write from R1 envelopes)
│       ├── downloads/                                      # PRE-UPLOAD STAGING AREA
│       │   ├── <slug>.md                                   # downloaded & converted markdown
│       │   ├── <slug>.pdf                                  # downloaded PDF
│       │   └── <slug>.<ext>                                # other document types
│       ├── DownloadAgent/                                  # download/ingest subagent working files (W1 stage 4) — lead spawns; subagents are content-blind
│       │   └── <slug>-dl.md                                # per-URL download/ingest subagent working file
│       ├── Category/
│       │   └── <category>.md                               # consolidated per-category records (lead-write directly from R1/R2 envelopes)
│       ├── SubAgent/                                       # R1 classification subagent working files (W1 stage 6a) — lead spawns directly
│       │   └── <agent-name>.md                             # Round 1 working file (lead seeds contract; R1 subagent writes)
│       ├── DeepDiveAgent/                                  # R2 deep-dive subagent working files (W1 stage 6d) — lead spawns directly
│       │   └── <agent-name>.md                             # Round 2 working file (lead seeds contract; R2 subagent writes)
│       ├── Archive/
│       │   ├── <agent-name>.md                             # R1/R2 subagent files moved by lead post-consolidation
│       │   └── <slug>-dl.md                                # archived download/ingest subagent working files
│       ├── advisories/                                     # internal-only artefacts
│       │   ├── undefined-tier-evidence.md
│       │   └── out-of-list-findings.md
│       ├── scripts/                                        # ad-hoc helper scripts written by the lead (e.g. json_to_csv.py)
│       ├── source-candidates.md                            # validated candidate list before approval
│       ├── source-approved.md                              # parsed approved URL set
│       └── sources_excluded.md                             # download/upload/ingestion drops; feeds deliverable footer
│
├── outputs/                                                # final deliverables only (read + write)
│   ├── <brand>-<model>-<year>-<market>-<run-id>.json       # client-facing deliverable (primary)
│   └── <brand>-<model>-<year>-<market>-<run-id>.csv        # client-facing deliverable (secondary)
│
└── uploads/                                                # user-supplied inputs (READ-ONLY)
    ├── params.csv                                          # optional — overrides the bundled reference list when present
    └── <other user-attached files>                         # e.g. URL lists, supplemental PDFs
```

Nothing else is created on the host. The workspace root (`/mnt/user-data/workspace/`) holds **only** `.harness/` — no deliverable, no scratch, no log file at the workspace root.

## Client-facing vs internal

| Audience | Path |
| :--- | :--- |
| **Client-facing (deliverable)** | The two files in `/mnt/user-data/outputs/`. |
| **Client-facing (audit)** | `/mnt/user-data/workspace/.harness/STATE.md`. Readable for audit; primary client output is in `outputs/`. |
| **Internal (agent working memory)** | Everything else under `/mnt/user-data/workspace/.harness/`. Staging area: `.harness/downloads/`. |
| **User-supplied inputs (read-only)** | `/mnt/user-data/uploads/`. |

The client may read `.harness/STATE.md` for audit trail; `.harness/downloads/` is for agent reference during the run and audit afterward. The two deliverable JSON/CSV files in `/mnt/user-data/outputs/` are the primary client output.

## Writer ownership

Strict ownership prevents corruption when subagents run concurrently.

| Path | Writer |
| :--- | :--- |
| `.harness/STATE.md` | **Lead only.** Never read or written by any subagent. |
| Deliverable files in `/mnt/user-data/outputs/` | **Lead only.** Written at W1 stage 7. |
| `/mnt/user-data/uploads/*` | **Lead only, read-only.** Read once at preflight; never written. |
| `.harness/params.csv` (frozen snapshot) | **Lead only**, written once at preflight; **lead-only reader during the run.** Never edited after. Subagents receive their parameter slice via the contract — they never read this file. |
| `.harness/downloads/*` | **Download/ingest subagents** — written during W1 stage 4; preserved for audit. Subagents never read the file content (size only). Never deleted after upload (except by the lead during dedup or post-block cleanup). R2 deep-dive subagents read exactly one assigned file via `read_file`. |
| `.harness/DownloadAgent/<slug>-dl.md` | **Lead** seeds the contract block at spawn; **named download/ingest subagent** writes scratch + content-blind result envelope. |
| `.harness/Category/*.md` | **Lead only.** Written directly during lead consolidation from R1/R2 envelopes. |
| `.harness/document-promise-board.md` | **Lead only.** Written during R1 consolidation as envelopes arrive. |
| `.harness/partition-summaries.md` | **Lead only.** Written during R1 consolidation as envelopes arrive. |
| `.harness/SubAgent/<agent-name>.md` | **Lead** seeds the contract block at spawn (with the parameter slice); **named R1 classification subagent** writes everything after. |
| `.harness/DeepDiveAgent/<agent-name>.md` | **Lead** seeds the contract block at spawn (with the gap-parameter slice); **named R2 deep-dive subagent** writes scratch + result envelope. |
| `.harness/Archive/*.md` | **Lead only.** Lead moves R1, R2, and download/ingest working files into Archive after consolidating their envelopes. |
| `.harness/advisories/*` | **Classification subagents** (R1 + R2) stage advisories in their result envelopes; **lead** writes the final advisory files during consolidation. |
| `.harness/source-*.md` | **Lead only.** |

**Spawning rule:** The **lead is the sole spawner** at every stage. No subagent of any kind (download/ingest, R1, R2, trim search) spawns another subagent. Any apparent need for nested spawning is satisfied by the lead consolidating the first envelope and spawning the second subagent itself.

**Subagent file isolation:** Every subagent writes only its own working file. Subagents never read `.harness/STATE.md`, `.harness/params.csv`, `.harness/Category/*.md`, or another subagent's working file. Inputs all arrive via the contract block at the top of the subagent's own working file.

## State files

### `.harness/STATE.md` (comprehensive run state)

Sections are tiered: **[T1] = resume-critical** (must be written before any pause or RunStage change); **[T2] = telemetry/audit** (written at stage completion; not required for resume).

Records, at minimum:

**[T1] sections:**
- `run_id`, `submitted_at`, `completed_at`, workflow, RunStage, RunStatus, PauseReason
- Resolved `car_identity` and `resolved_full_option_trim`; KB dataset ID; CSV hash; total parameters
- Source stats funnel (candidates → approved → ingested → dropped)
- Source download tracking: per-URL retry state, file size, local path, status
- Per-doc ingestion status: one row per uploaded doc with `doc_id`, `ingestion_status`, `ingestion_completed_at`
- Category roster (categories × subagent partitions × consolidated count × status)
- Round 1 subagent roster (agent-name, category, params, SubagentStatus, spawned/completed timestamps)
- Round 2 subagent roster (agent-name, target doc_id, gap params, SubagentStatus, spawned/completed timestamps) — written empty at R1 consolidation start
- Gap parameters + R2 target assignments (JSON block, write-once at R1→R2 hand-off)
- Event log (append-only)

**[T2] sections:**
- Confidence distribution sub-table (consensus vs single-source counts; R2 upgrade counts)
- KB dataset snapshot (doc count, source type distribution) — written once after Stage 5 success
- Stage timing telemetry (started_at / completed_at / duration per stage)
- Warnings and advisories summary (Rule 2a/2b conflict counts, advisory counts)
- Deduplication log (kept/deleted doc_id pairs from Phase 5b) — written even when empty
- Loaded skills (lead + subagent)
- Decisions made
- Pointers

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

### `.harness/DownloadAgent/<slug>-dl.md` (download/ingest subagent working file)

Has three blocks, in order:
1. **Contract header** (`<!-- DOWNLOAD CONTRACT -->`) — seeded once by the lead at spawn; never re-edited.
2. **Scratch** — the subagent's working notes (fetch tool calls, retry counts, upload outcome). **No content reads of the saved file.**
3. **Result envelope** (`<!-- DOWNLOAD RESULT -->`) — content-blind report (`status`, `file_size_bytes`, `is_html`, `error_message`, etc.). The lead derives `block_type` from these fields.

### `.harness/SubAgent/<agent-name>.md` (R1 classification subagent working file)

Has three blocks, in order:
1. **Contract header** (`<!-- AUTOGENERATED BY LEAD -->`) — seeded once by the **lead** at spawn (with the embedded parameter slice for the partition); never re-edited.
2. **Scratch** — the R1 subagent's working notes during retrieval / decision iterations.
3. **Result envelope** (`<!-- RESULT ENVELOPE -->`) — final output the **lead** consolidates directly.

### `.harness/DeepDiveAgent/<agent-name>.md` (R2 deep-dive subagent working file)

Has three blocks, in order:
1. **Contract header** (`<!-- AUTOGENERATED BY LEAD -->`) — seeded once by the **lead** at spawn (with the embedded gap-parameter slice and `target_doc.local_path`); never re-edited.
2. **Scratch** — the R2 subagent's working notes from reading the single assigned Markdown.
3. **Result envelope** (`<!-- RESULT ENVELOPE -->`) — final output the **lead** consolidates and merges with R1 records.

### `.harness/Category/<category>.md` (per category)

Lead-authored consolidation of all subagents that produced records for that category. Includes per-parameter records, assigned subagent names, and category-level warnings.

## Pause / resume contract

The workspace is the only state store. The agent is allowed to disappear and come back.

**Mandatory pause points in W1.** The lead must pause and wait for an explicit user signal at each of:
- `awaiting-source-approval` — after source validation, before download.
- `awaiting-client-resume` — after `ragflow_run_ingestion` is triggered, before verifying ingestion.
- `awaiting-classification-go-ahead` — **after ingestion is verified complete, before any R1 classification subagent is spawned.** The lead presents the ingested-document summary plus the planned R1 partition list and yields. Stage 6 spawning may not begin until the user explicitly says go.

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

- **The lead is the sole spawner.** The lead spawns download/ingest subagents (one per approved URL, max 5 concurrent) at stage 4, then R1 classification subagents directly at stage 6a, then R2 deep-dive subagents directly at stage 6d. No subagent ever spawns another subagent.
- **Max 3 classification subagents running concurrently** (R1 or R2; enforced by `stellantis-lead-agent-subagent-orchestration`). The lead queues the rest.
- **Each subagent writes only its own working file.** No file is shared as a write target between concurrent subagents.
- **Lead consolidation is serial.** Even if two subagents finish at the same time, the lead consolidates them one after another. A subagent's concurrency slot is freed only after the lead consolidates its envelope, not when it returns control.
- **Multiple concurrent car analyses each have their own `.harness/`.** No cross-run coordination.

## Resumability checklist

For a run to be safely resumable from a clean restart:
- `.harness/STATE.md` must always reflect the latest committed state (atomic-ish writes).
- All [T1] sections must be flushed before any `RunStatus = paused` or `RunStage` change.
- The frozen `.harness/params.csv` must remain unchanged for the entire run, including all gap-fill cycles.
- Every subagent working file (`DownloadAgent/`, `SubAgent/`, `DeepDiveAgent/`) must be self-describing — the contract block alone is enough for the lead to re-spawn the subagent.
- The `.harness/sources_excluded.md` log must be append-only.
- Downloaded files in `.harness/downloads/` are preserved and referenced in STATE.md.
- The lead records the R2 subagent roster and gap-parameters JSON block in `.harness/STATE.md` before spawning any R2 subagent.
- Per-doc ingestion status rows are written at upload, not at ingestion trigger — post-crash agent can see which doc_ids exist without calling `doc_infos`.

## Tips & common pitfalls

- **Create .harness FIRST.** Before any other processing in preflight, create the `.harness/` folder structure. Including `DownloadAgent/`, `SubAgent/`, `DeepDiveAgent/`, `Category/`, `Archive/`, `advisories/`, `downloads/`.
- **Save to downloads/ before uploading.** Every downloaded file goes to `.harness/downloads/` first, then upload references it there.
- **Don't write the deliverable until lead consolidation is complete.** Lead consolidation runs serially as each R1/R2 envelope arrives, then the final STATE.md update happens after the last R2 archive.
- **Don't let any subagent spawn another subagent.** The lead is the sole spawner. If a flow seems to need nested spawning, consolidate the first envelope back to the lead and let the lead spawn the second.
- **Don't let any subagent read `params.csv`.** The lead reads `.harness/params.csv` and embeds the parameter slice in each subagent's contract.
- **Don't let download/ingest subagents read the file content.** Size only, captured via `os.stat` or the fetch tool's return value. The lead does block-type classification.
- **Don't let R1 subagents read files in `.harness/downloads/` directly.** R1 reads evidence only via the KB. Local-file reads happen only at R2 deep-dive on the single `target_doc.local_path` in the contract.
- **Don't let any subagent write STATE.md or Category files.** Subagents write only their own working files.
- **Don't move R1/R2 subagent files out of `SubAgent/` / `DeepDiveAgent/` before the lead consolidates their envelope.** Archive comes after the lead records the envelope contents.
- **Don't put anything other than `.harness/` in the workspace root.** `/mnt/user-data/workspace/` contains exactly one entry: `.harness/`. Deliverables go to `/mnt/user-data/outputs/`, never to the workspace root.
- **Don't write to `/mnt/user-data/uploads/`.** It is read-only; copy any needed file into `.harness/` instead.
- **Don't read or write any path outside the three sandboxed paths.** No `/tmp`, no `~`, no relative paths that resolve elsewhere.
- **Don't write to another agent's working file.** The lead seeds every contract; subagents write only after the contract block in their own file.
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

