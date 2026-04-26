# File Ownership Matrix

Exhaustive per-path access rules. `R` = may read, `W` = may write, `C` = creates, `A` = archives (move), `—` = must not touch. Paths are relative to `runs/<run-id>/`.

| Path                                              | Lead    | Subagent | Notes                                                                 |
| :------------------------------------------------ | :------ | :------- | :-------------------------------------------------------------------- |
| `STATE.md`                                        | C W R   | —        | Main state file. Only lead updates it.                                |
| `params.csv`                                      | C R     | R        | Frozen snapshot; read-only to both.                                   |
| `params.csv.hash`                                 | C R     | —        | Written at preflight.                                                 |
| `deliverable.json`                                | C W     | —        | Emitted at W1 stage 7.                                                |
| `deliverable.csv`                                 | C W     | —        | Emitted at W1 stage 7.                                                |
| `source-candidate-list.md`                        | C W R   | —        | Posted to user at the approval gate.                                  |
| `source-approved.md`                              | C W R   | —        | Parsed result of the client reply.                                    |
| `.harness/Category/<category>.md`                 | C W R   | —        | Consolidated per-category STATE; lead-only writer.                    |
| `.harness/SubAgent/<agent-name>.md`               | C R (pre-seed contract) / W (never after spawn) | R W | Lead seeds it with the contract; subagent writes everything after. |
| `.harness/Archive/<agent-name>.md`                | C (via move) R | — | Destination of consolidated subagent working files.                   |
| `.harness/Archive/sources_excluded.md`            | C W R   | —        | Download/ingestion failures (AS-DEL-C).                              |
| `.harness/advisories/undefined-tier.md`           | W R     | —        | Lead promotes from subagent working files during consolidation.       |
| `.harness/advisories/out-of-list-findings.md`     | W R     | —        | Same as above.                                                        |
| `.harness/event-log.md`                           | C W     | —        | Append-only; mirrors STATE.md events.                                 |

Paths outside the run workspace:

| Path                                              | Lead | Subagent | Notes                                                                 |
| :------------------------------------------------ | :--- | :------- | :-------------------------------------------------------------------- |
| `artifacts/params.csv` (source repo)              | R    | —        | Read once at preflight; then never again this run.                    |
| `src/SKILL.md`                                    | R    | —        | Loaded by lead at session start.                                      |
| `src/references/enums.md`                         | R    | R        | Loaded by lead at preflight and by each subagent on spawn.            |
| `src/references/workspace-layout.md`              | R    | —        | Lead reads at preflight.                                              |
| `src/references/lead-to-subagent-contract.md`     | R    | R        | Contract shape.                                                       |
| `src/references/subagent-to-lead-contract.md`     | R    | R        | Result envelope shape.                                                |
| `src/templates/*`                                 | R    | R (those listed in §8.2 of `SKILL.md`) | Templates are instantiated, never mutated.   |
| `src/skills/<skill-name>/SKILL.md`                | R (lead skills) | R (subagent skills) | Loaded per actor per stage.                        |
| `src/workflows/W1-normal-pipeline-mode-a/**`      | R    | —        | Lead-only.                                                            |
| `business-logic/**`                               | R    | R (rare; only enums + decision rules reference) | Source of truth; never written from the skill. |
| `framework-maintenance/**`                        | —    | —        | Touched only by a framework-maintenance agent, not a run-time actor.  |

## Invariants

1. A subagent that writes to any path not listed as `W` for Subagent in this matrix is a bug. The lead must reject the subagent's output and treat it as `SubagentStatus = failed`.
2. A subagent that reads any `.harness/Category/*.md`, the main `STATE.md`, or another subagent's working file is a bug. The lead must verify this by inspection during consolidation.
3. `framework-maintenance/` is off-limits during a run. It is edited only by an operator explicitly invoking the self-modification workflow under [`../../framework-maintenance/README.md`](../../framework-maintenance/README.md).
