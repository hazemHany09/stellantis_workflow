# Workspace Layout

The runtime directory tree for a single run. Every path below is **normative**. If a stage needs a path not listed here, the path must be added to this file before it is written.

Root: `runs/<run-id>/`.

Run ID format: `<YYYYMMDD>-<brand-slug>-<model-slug>-<year>-<market-slug>-<short-hash>`. Concurrent runs on the same car differ only in `<short-hash>`.

```
runs/<run-id>/
├── STATE.md                              ← main, client-facing; written by Lead only
├── params.csv                            ← frozen snapshot of artifacts/params.csv
├── params.csv.hash                       ← sha256 of params.csv, recorded in STATE.md header
├── deliverable.json                      ← emitted at stage 7 of W1
├── deliverable.csv                       ← emitted at stage 7 of W1
├── source-candidate-list.md              ← the list posted to user at the approval gate
├── source-approved.md                    ← parsed result of client reply; per-URL Approved/Rejected
└── .harness/                             ← agent-internal working files
    ├── downloads/                        ← pre-upload staging; written by download subagents (W1 stage 4)
    │   └── <slug>.<ext>                  ← fetched file; preserved post-upload for audit
    ├── DownloadAgent/                    ← download subagent working files (W1 stage 4)
    │   └── <slug>-dl.md                  ← contract + scratch + result envelope per URL
    ├── Category/
    │   └── <category>.md                 ← per-category consolidated STATE; Lead-only writer
    ├── SubAgent/
    │   └── <agent-name>.md               ← per-classification-subagent working file; Subagent-only writer
    ├── Archive/
    │   ├── <agent-name>.md               ← archived classification subagent files post-consolidation
    │   ├── <slug>-dl.md                  ← archived download subagent files post-consolidation
    │   └── sources_excluded.md           ← download/upload/ingestion failures; access-denied; rate-limited drops
    ├── advisories/
    │   ├── undefined-tier.md             ← evidence pointing at undefined tiers
    │   └── out-of-list-findings.md       ← features found that fall outside the parameter list
    └── event-log.md                      ← append-only, Lead-only; mirrors STATE.md event log
```

## File naming rules

* `<run-id>` — see format above; max 80 chars. Slug = lowercase alnum + hyphens.
* `<category>` — taken verbatim from the `Domain / Category` column of `params.csv`, lowercased, spaces → `-`, non-alnum stripped. Example: `EV` → `ev`; `INFOT.` → `infot`.
* `<agent-name>` — `<category-slug>-<partition-index>` (e.g. `ev-1`, `cockpit-2`). Max 15 parameters per subagent. If a category has ≤15 parameters, partition index is `1`.

## Invariants

1. One run = one workspace directory. Never two runs share a workspace.
2. `params.csv` inside the workspace is **immutable** for the life of the run. The workspace never reads back from `artifacts/params.csv` after preflight.
3. `STATE.md` must exist for as long as the workspace exists. Deletion of `STATE.md` is a framework violation.
4. `.harness/` is optional to retain after completion — see WF-5 (pending). Until WF-5 closes, retention is **indefinite**.
5. No file under `runs/<run-id>/` is named with a leading dot except `.harness/` itself.

## Read/write matrix

See [`file-ownership-matrix.md`](file-ownership-matrix.md) for the exhaustive per-file permissions.
