# File Ownership Matrix

Exhaustive per-path access rules. `R` = may read, `W` = may write, `C` = creates, `A` = archives (move), `—` = must not touch.

The agent operates inside a fixed three-path sandbox. Workspace-internal entries below use `.harness/` as shorthand for `/mnt/user-data/workspace/.harness/`.

| Path                                              | Lead    | Download/Ingest subagent | R1 Classification subagent | R2 Deep-Dive subagent | Notes                                                                 |
| :------------------------------------------------ | :------ | :----------------------- | :------------------------- | :-------------------- | :-------------------------------------------------------------------- |
| `.harness/STATE.md`                               | C W R   | —                        | —                          | —                     | Main state file. Only lead reads or writes it.                        |
| `.harness/params.csv`                             | C R     | —                        | —                          | —                     | Frozen snapshot. **Lead-only reader.** Subagents receive their parameter slice via the contract. |
| `.harness/params.csv.hash`                        | C R     | —                        | —                          | —                     | Written at preflight.                                                 |
| `/mnt/user-data/outputs/<brand>-<model>-<year>-<market>-<run-id>.json` | C W     | —                        | —                          | —                     | Emitted at W1 stage 7. |
| `/mnt/user-data/outputs/<brand>-<model>-<year>-<market>-<run-id>.csv`  | C W     | —                        | —                          | —                     | Emitted at W1 stage 7. |
| `.harness/source-candidates.md`                   | C W R   | —                        | —                          | —                     | Posted to user at the approval gate.                                  |
| `.harness/source-approved.md`                     | C W R   | —                        | —                          | —                     | Parsed result of the client reply.                                    |
| `.harness/downloads/<slug>.<ext>`                 | R       | C W (own slug only)      | —                          | R (own contract's `target_doc.local_path` only) | Written by download/ingest subagent via `fetch_url`. R1 retrieves through the KB, never via filesystem. R2 reads exactly one assigned file. |
| `.harness/DownloadAgent/<slug>-dl.md`             | C R (seeds contract) | W (scratch + result envelope) | —                  | —                     | Lead seeds contract; download/ingest subagent writes everything after. **Subagent does not read the saved file's content** — size only. |
| `.harness/Category/<category>.md`                 | C W R   | —                        | —                          | —                     | Consolidated per-category STATE; lead-only writer.                    |
| `.harness/SubAgent/<agent-name>.md`               | C R (seeds contract) | —               | W (scratch + result envelope, own file only) | —       | Lead seeds contract with parameter slice. R1 subagent writes; never reads `params.csv`. |
| `.harness/DeepDiveAgent/<agent-name>.md`          | C R (seeds contract) | —               | —                          | W (scratch + result envelope, own file only) | Lead seeds contract with gap-parameter slice. R2 subagent writes; never reads `params.csv`. |
| `.harness/document-promise-board.md`              | C W R   | —                        | —                          | —                     | Lead aggregates R1 promise scores here; drives R2 target selection.   |
| `.harness/partition-summaries.md`                 | C W R   | —                        | —                          | —                     | Lead appends each R1 subagent's `partition_summary`.                  |
| `.harness/Archive/<agent-name>.md`                | C (via move) R | —                | —                          | —                     | Archived classification subagent working files post-consolidation.    |
| `.harness/Archive/<slug>-dl.md`                   | C (via move) R | —                | —                          | —                     | Archived download subagent working files post-consolidation.          |
| `.harness/sources_excluded.md`                    | C W R   | —                        | —                          | —                     | All download/upload/ingestion failures (access-denied, rate-limited, timeouts). |
| `.harness/advisories/undefined-tier.md`           | W R     | —                        | W (via envelope only) R    | W (via envelope only) R | Subagents emit advisory entries in their result envelope; lead promotes them to this file. |
| `.harness/advisories/out-of-list-findings.md`     | W R     | —                        | W (via envelope only) R    | W (via envelope only) R | Same as above.                                                        |
| `.harness/event-log.md`                           | C W     | —                        | —                          | —                     | Append-only; mirrors STATE.md events.                                 |

Paths outside the run workspace:

| Path                                              | Lead | Subagent | Notes                                                                 |
| :------------------------------------------------ | :--- | :------- | :-------------------------------------------------------------------- |
| `/mnt/user-data/uploads/params.csv`               | R    | —        | **Read-only.** Read once at preflight; copied into `.harness/params.csv`; then never read again this run. |
| `/mnt/user-data/uploads/<other user-attached files>` | R | —       | **Read-only.** User-supplied PDFs, URL lists, etc. Copied into `.harness/` if needed. |
| Top-level entry SKILL.md                          | R    | —        | Loaded by lead at session start.                                      |
| `references/enums.md`                             | R    | R        | Loaded by lead at preflight and by each subagent on spawn.            |
| `references/workspace-layout.md`                  | R    | —        | Lead reads at preflight.                                              |
| `references/lead-to-subagent-contract.md`         | R    | R        | Contract shape.                                                       |
| `references/subagent-to-lead-contract.md`         | R    | R        | Result envelope shape.                                                |
| `templates/*`                                     | R    | R (those listed in main SKILL.md) | Templates are instantiated, never mutated.            |
| `<sub-skill>/SKILL.md`                            | R (lead skills) | R (subagent skills) | Loaded per actor per stage.                              |
| `workflows/W1-normal-pipeline-mode-a/**`          | R    | —        | Lead-only.                                                            |
| `framework-maintenance/**`                        | —    | —        | Touched only by a framework-maintenance agent, not a run-time actor.  |

## Invariants

0. **Three-path sandbox is absolute.** The only paths the agent may read or write are `/mnt/user-data/workspace/`, `/mnt/user-data/outputs/`, and `/mnt/user-data/uploads/`. `/mnt/user-data/uploads/` is read-only. Any other host path — `/tmp`, `~`, the repository root, the current working directory — is out of scope.
1. **Spawning is lead-only.** No subagent at any stage may spawn another subagent. The lead is the only actor allowed to invoke the host platform's subagent-spawn primitive.
2. **`params.csv` is lead-only during a run.** No subagent reads it. Each subagent's parameter slice arrives in its contract block. A subagent that reads `.harness/params.csv` is a bug — the lead must mark its result `SubagentStatus = failed`.
3. **Download/ingest subagents are content-blind.** A download/ingest subagent that opens, reads, or scans the file at `.harness/downloads/<slug>.<ext>` is a bug. Size is captured via the fetch tool's return value or `os.stat`. The lead must reject the subagent's output and treat it as `SubagentStatus = failed`.
4. **Classification subagents (R1) read evidence only via the KB.** A classification subagent that writes to any path not listed as `W` for it in this matrix, or reads `.harness/downloads/*` directly, is a bug.
5. **Deep-dive subagents (R2) read exactly one local file.** The path comes from `target_doc.local_path` in the contract. Reading any other file (including other docs in `.harness/downloads/`) is a bug.
6. **No subagent reads `STATE.md`, `Category/*.md`, or another subagent's working file.** Inputs all arrive via the contract block.
7. `framework-maintenance/` is off-limits during a run. It is edited only by an operator explicitly invoking the self-modification workflow under [`../../framework-maintenance/README.md`](../../framework-maintenance/README.md).
