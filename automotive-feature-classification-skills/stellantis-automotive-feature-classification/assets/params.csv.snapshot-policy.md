# params.csv Snapshot Policy

The reference parameter list is **frozen per run**. The bundled list (or a user-supplied override delivered via `/mnt/user-data/uploads/params.csv`) is copied into the run workspace at preflight, and every actor reads from that frozen copy for the rest of the run.

## Filesystem sandbox

The agent operates inside three host paths only:

- `/mnt/user-data/workspace/` — read + write. `params.csv` is frozen at `.harness/params.csv`.
- `/mnt/user-data/outputs/` — read + write. Deliverables only.
- `/mnt/user-data/uploads/` — **read-only**. Optional user-supplied `params.csv` lives here.

## Rule

1. At preflight (stage 1 of every workflow) the lead agent freezes the reference list:
   - If `/mnt/user-data/uploads/params.csv` exists, copy it into `/mnt/user-data/workspace/.harness/params.csv`.
   - Otherwise, copy the **bundled reference list** shipped with the skill. Its canonical location inside the skill repository is:
     `stellantis-automotive-feature-classification/assets/params.csv`
     Read it with `read_file` (absolute path) and write the content to `/mnt/user-data/workspace/.harness/params.csv`.
2. The sha256 hash of the copied file is recorded at `/mnt/user-data/workspace/.harness/params.csv.hash` and embedded in the main `STATE.md` header.
3. For the entire life of the run — including any gap-fill cycles (W5) — the lead reads parameters from `.harness/params.csv` **only**. Subagents never read this file; their parameter slice arrives in their contract block. `/mnt/user-data/uploads/params.csv` is never re-read.
4. New or changed parameters in the source list (uploads or bundled) take effect only in new runs started after the change.

## Rationale

* Eliminates race conditions if the user uploads an updated CSV mid-run.
* Makes the deliverable header's CSV hash a reproducibility anchor — a consumer can diff two runs on the same car by comparing hashes first.
* Keeps subagent contracts self-contained: parameter descriptors in the contract are extracted from the frozen snapshot, not from a live file.

## Enforcement

The lead must refuse to proceed if:

* No reference list is available (neither `/mnt/user-data/uploads/params.csv` nor the bundled fallback).
* The copied file is empty or fails CSV parsing.
* Any required column from the parameter specification is absent.

On any of those, pause with `RunStatus = failed` and surface the reason in `STATE.md`.
