# params.csv Snapshot Policy

Per AS-SCOPE-2 and Q-SCOPE-2, `artifacts/params.csv` is **developer-owned, frozen per run**, and injected into the agent workspace.

## Rule

1. At preflight (stage 1 of every workflow) the lead agent copies `artifacts/params.csv` from the repository root into `runs/<run-id>/params.csv`.
2. The sha256 hash of the copied file is recorded at `runs/<run-id>/params.csv.hash` and embedded in the main `STATE.md` header.
3. For the entire life of the run — including any gap-fill cycles (W5) — the lead and all subagents read parameters from `runs/<run-id>/params.csv` **only**. The repository file is never re-read.
4. New or changed parameters in `artifacts/params.csv` take effect only in new runs started after the change.

## Rationale

* Eliminates race conditions when developers edit the CSV while runs are in flight.
* Makes the deliverable header's CSV hash a reproducibility anchor — a consumer can diff two runs on the same car by comparing hashes first.
* Keeps subagent contracts self-contained: parameter descriptors in the contract are extracted from the frozen snapshot, not from a live file.

## Enforcement

The lead must refuse to proceed if:

* `artifacts/params.csv` is missing at preflight.
* The copied file is empty or fails CSV parsing.
* Any required column from the parameter specification is absent.

On any of those, pause with `RunStatus = failed` and surface the reason in `STATE.md`.
