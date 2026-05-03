# Workspace Layout

The runtime directory tree for a single run. Every path below is **normative**. If a stage needs a path not listed here, the path must be added to this file before it is written.

The agent operates inside a fixed host filesystem sandbox with **exactly three** accessible paths:

| Path | Mode | Purpose |
| :--- | :--- | :--- |
| `/mnt/user-data/workspace/` | read + write | Run workspace. Contains `.harness/` and nothing else. **All run state lives under `.harness/`.** |
| `/mnt/user-data/outputs/`   | read + write | Final client-facing deliverables only (`<brand>-<model>-<year>-<market>-<run-id>.json` and `.csv`). |
| `/mnt/user-data/uploads/`   | **read-only** | User-supplied input files (e.g. `params.csv`, attached PDFs, URL lists). The agent never writes here. |

Anything outside these three paths is out of scope. There is no `runs/<run-id>/` nesting; one workspace = one run.

Run ID format: `<YYYYMMDD>-<brand-slug>-<model-slug>-<year>-<market-slug>-<short-hash>`. Recorded in `.harness/STATE.md`; used to name the deliverables in `/mnt/user-data/outputs/`.

```
/mnt/user-data/
├── workspace/                                      # run workspace (read + write)
│   └── .harness/                                   # ALL agent-internal state \u2014 every working file lives here
│       ├── STATE.md                                ← main run state; written by Lead only
│       ├── params.csv                              ← frozen snapshot copied from /mnt/user-data/uploads/params.csv at preflight (LEAD-ONLY READER during a run)
│       ├── params.csv.hash                         ← sha256 of params.csv, recorded in STATE.md header
│       ├── source-candidates.md                    ← list posted to user at the approval gate
│       ├── source-approved.md                      ← parsed result of client reply; per-URL Approved/Rejected
│       ├── sources_excluded.md                     ← download/upload/ingestion failures; access-denied; rate-limited drops
│       ├── document-promise-board.md               ← Lead-aggregated R1 promise scores; drives R2 target selection
│       ├── partition-summaries.md                  ← Lead appends each R1 subagent's partition summary
│       ├── event-log.md                            ← append-only, Lead-only; mirrors STATE.md event log
│       ├── downloads/                              ← pre-upload staging; written by download/ingest subagents (W1 stage 4)
│       │   └── <slug>.<ext>                        ← fetched file; preserved post-upload for audit; R2 reads exactly one assigned file from here
│       ├── DownloadAgent/                          ← download/ingest subagent working files (W1 stage 4) \u2014 Lead seeds contract; subagent writes scratch + content-blind result envelope
│       │   └── <slug>-dl.md
│       ├── Category/
│       │   └── <category>.md                       ← per-category consolidated STATE; Lead-only writer
│       ├── SubAgent/                               ← R1 classification subagent working files (W1 stage 6a) \u2014 Lead seeds contract embedding the parameter slice
│       │   └── <agent-name>.md
│       ├── DeepDiveAgent/                          ← R2 deep-dive subagent working files (W1 stage 6d) \u2014 Lead seeds contract embedding the gap-parameter slice + target_doc
│       │   └── <agent-name>.md
│       ├── Archive/
│       │   ├── <agent-name>.md                     ← archived classification subagent files post-consolidation
│       │   └── <slug>-dl.md                        ← archived download/ingest subagent files post-consolidation
│       ├── advisories/
│       │   ├── undefined-tier.md                   ← evidence pointing at undefined tiers
│       │   └── out-of-list-findings.md             ← features found that fall outside the parameter list
│       └── scripts/                                ← ad-hoc helper scripts written by the Lead (e.g. json_to_csv.py)
│
├── outputs/                                        # final deliverables only (read + write)
│   ├── <brand>-<model>-<year>-<market>-<run-id>.json   ← emitted at W1 stage 7
│   └── <brand>-<model>-<year>-<market>-<run-id>.csv    ← emitted at W1 stage 7
│
└── uploads/                                        # user-supplied inputs (READ-ONLY)
    ├── params.csv                                  ← optional; overrides bundled reference list when present
    └── <other user-attached files>
```

## File naming rules

* `<run-id>` — see format above; max 80 chars. Slug = lowercase alnum + hyphens.
* `<category>` — taken verbatim from the `Domain / Category` column of `params.csv`, lowercased, spaces → `-`, non-alnum stripped. Example: `EV` → `ev`; `INFOT.` → `infot`.
* `<agent-name>` (R1) — `<category-slug>-<partition-index>` (e.g. `ev-1`, `cockpit-2`). **Max 15 parameters per subagent. Single-category only — partitions never cross categories.** If a category has ≤15 parameters, partition index is `1`.
* `<agent-name>` (R2 deep-dive) — `<category-slug>-deepdive-<index>` (e.g. `infot-deepdive-1`).
* `<slug>-dl` — download/ingest subagent name; `<slug>` derived from canonical URL host + path.

## Spawning rule

The **lead is the sole spawner** at every stage. No subagent (download/ingest, R1, R2, trim search) spawns another subagent. The lead spawns subagents in two stages:

1. W1 stage 4 — one download/ingest subagent per approved URL.
2. W1 stage 6 — R1 classification subagents per partition; then R2 deep-dive subagents per (target doc × gap parameters).

## Invariants

1. **Three-path sandbox is absolute.** The agent reads/writes only under `/mnt/user-data/workspace/`, `/mnt/user-data/outputs/`, and `/mnt/user-data/uploads/`. Any other path is a framework violation.
2. **`/mnt/user-data/uploads/` is read-only.** The agent copies needed inputs into `.harness/` at preflight; it never writes back.
3. **`/mnt/user-data/workspace/` contains exactly one entry: `.harness/`.** No deliverable, scratch, or log file at the workspace root.
4. **All run state lives in `.harness/`.** Deliverables \u2014 and only deliverables \u2014 live in `/mnt/user-data/outputs/`.
5. One run = one workspace. Concurrent runs require separate sandbox mounts, not subfolders.
6. `.harness/params.csv` inside the workspace is **immutable** for the life of the run. The workspace never reads back from `/mnt/user-data/uploads/params.csv` after preflight. **Only the lead reads `.harness/params.csv` during a run** — subagents receive parameter slices via their contract.
7. `.harness/STATE.md` must exist for as long as the workspace exists. Deletion is a framework violation.
8. `.harness/` retention after completion is **indefinite** until WF-5 closes.
9. **Download/ingest subagents are content-blind.** They never open or read the file they downloaded — only its byte size, captured via the fetch tool's return value or `os.stat`.
10. **R1 classification subagents read evidence only via the KB** (`retrieval`, `list_chunks`, `ragflow_download`). They do not read files in `.harness/downloads/` directly via `read_file`.
11. **R2 deep-dive subagents read exactly one local file** — the path in their contract's `target_doc.local_path` (always under `/mnt/user-data/workspace/.harness/downloads/`).

## Read/write matrix

See [`file-ownership-matrix.md`](file-ownership-matrix.md) for the exhaustive per-file permissions.
