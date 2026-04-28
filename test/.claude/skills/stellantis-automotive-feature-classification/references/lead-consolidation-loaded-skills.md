# Lead Consolidation Rule — Loaded Skills

Defines how the lead agent tracks and consolidates `loaded_skills` across the run lifecycle.

## Overview

Skill loading affects context budget and debugging. The lead maintains a consolidated audit log in `STATE.md` split into two sections:
- **Lead-loaded skills:** Loaded by lead at specific stages (preflight, source discovery, etc.)
- **Subagent-loaded skills:** Reported by each subagent in its result envelope

## Lead-loaded skills (preflight + operational)

### When updated

Lead updates `## Loaded skills` → `### Lead-loaded skills` **immediately after loading each skill**, not retroactively.

### Entry format

| Skill name | Stage loaded | Purpose |
| :--------- | :----------- | :------ |
| `skill-name` | `<stage>` | `<one-line role>` |

Example:
| Skill name | Stage loaded | Purpose |
| :--------- | :----------- | :------ |
| `stellantis-domain-context` | preflight | Shared vocabulary, mission, invariants |
| `stellantis-decision-rules` | preflight | Five decision rules + master matrix |
| `stellantis-car-trim-resolution` | W1-stage-1b | Resolve trim to manufacturer published top trim |
| `stellantis-source-discovery-with-client-domains` | W1-stage-3-modeA | Search within user-supplied domains |

### Completeness requirement

**Rule:** Every skill listed in the main `SKILL.md` frontmatter `requires:` field **must** appear in this table by workflow completion. If a skill is never loaded (e.g., Mode B branch never taken when user supplies URLs), document the reason in **Decisions made during this run**, not in the skills table.

### Stage naming

Use the exact stage name from `src/workflows/*/WORKFLOW.md`. Examples:
- `preflight`
- `W1-stage-1b` (from workflows/W1-normal-pipeline-mode-a/WORKFLOW.md)
- `W1-stage-3-modeA` (branching choice documented)
- `W1-stage-6a` (subagent spawn)

## Subagent-loaded skills (consolidated from result envelopes)

### When updated

Lead updates `## Loaded skills` → `### Subagent-loaded skills` **after consolidating each subagent's result envelope** (stage 6b, `consolidate`).

### Source of truth

Each subagent includes `"loaded_skills": [...]` in its result envelope (bottom of `.harness/SubAgent/<agent-name>.md`). Lead reads this array and promotes it to main STATE.md.

### Entry format

| Agent name | Skills loaded |
| :--------- | :------------ |
| `agent-name` | `skill-1`, `skill-2`, ... |

Example:
| Agent name | Skills loaded |
| :--------- | :------------ |
| `adas-1` | `stellantis-domain-context`, `stellantis-decision-rules`, `stellantis-subagent-classification-loop` |
| `ev-1` | `stellantis-domain-context`, `stellantis-decision-rules`, `stellantis-subagent-classification-loop` |
| `infot-1` | `stellantis-domain-context`, `stellantis-decision-rules`, `stellantis-subagent-classification-loop` |

### Contract obligation

**Rule:** Subagent must populate `loaded_skills` in result envelope. If not present or empty, lead must log an advisory (scope = `run`) and note the discrepancy in **Decisions made during this run**.

### Deduplication

If all subagents load the same skills (expected: foundational skills), the lead may collapse the table to:

| Agent name(s) | Skills loaded |
| :------------- | :------------ |
| All subagents | `stellantis-domain-context`, `stellantis-decision-rules`, `stellantis-subagent-classification-loop` |

But only if it's true for all agents. Divergence (one subagent loads an extra skill) must be shown per-agent.

## Audit value

This consolidation serves three purposes:

1. **Context budgeting:** Verify that no stage loads more skills than expected. If a subagent loads 18 skills, that's a red flag.
2. **Debugging:** If a subagent produced unexpected results, trace back which skills it had in scope.
3. **Governance:** Confirms that skill loading stays within the authored workflow (stage-driven, not ad-hoc).

## Related

- `src/references/subagent-to-lead-contract.md` — defines `loaded_skills` field in result envelope
- `templates/STATE.main.md.tmpl` — defines the "Loaded skills" section structure
- `framework-maintenance/impact-checklist.md` — when a new skill is added, update this rule

