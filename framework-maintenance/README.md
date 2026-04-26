# Framework Maintenance

This folder is for an **operator or an agent explicitly invoked to modify the Automotive Feature Classification skill framework**. It is **not** part of the runtime. A production run must not touch any file here and must not read any file here.

## Audience

1. A developer evolving the skill packages (adding a workflow, changing an enum, adjusting a template).
2. An agent explicitly asked *"update the framework"* and given a change request.

## Skill Package Layout

The framework is distributed as **independently importable skill packages** (one SKILL.md each), grouped into two tiers.

### Foundational skills (always loaded at preflight)

| Package directory                          | Role                                                                                                                                       |
| :----------------------------------------- | :----------------------------------------------------------------------------------------------------------------------------------------- |
| `stellantis-domain-context/`               | Mission, vocabulary, master invariants, glossary, evergreen tips. Loaded by **every** agent role.                                          |
| `stellantis-decision-rules/`               | Five decision rules + master matrix + 8 invariants. Loaded by lead and every classification subagent.                                      |
| `stellantis-output-contract/`              | Per-parameter record shape, deliverable format, run-header, `sources_excluded` footer.                                                     |
| `stellantis-workflow-modes/`               | Currently authored workflows + reserved slots + composition rules. Loaded by the lead at preflight.                                        |
| `stellantis-failure-handling/`             | Retry / drop / timeout policy. Centralises canonicalisation rules and the `sources_excluded` emission rule.                                |
| `stellantis-knowledge-base-protocol/`      | Vendor-agnostic KB capability contract. Ingestion wait, retrieval discipline, delete protocol.                                             |
| `stellantis-run-workspace/`                | Workspace layout, writer ownership, pause/resume contract, resumability checklist.                                                         |
| `stellantis-approval-gate/`                | URL-level approval gate, flag/retrieve operations, hard phase boundary enforcement.                                                        |

### Operational skills (loaded lazily per stage)

| Package directory                          | Role                                                                   |
| :----------------------------------------- | :--------------------------------------------------------------------- |
| `stellantis-automotive-feature-classification/`       | **Core skill** — entry point, shared references, templates, workflows  |
| `stellantis-car-trim-resolution/`                     | Lead context — trim resolution at preflight                            |
| `stellantis-source-discovery-with-client-domains/`    | Lead context — Mode A discovery                                        |
| `stellantis-source-discovery-without-client-domains/` | Lead context — Mode B discovery                                        |
| `stellantis-source-validation/`                       | Lead context — candidate validation                                    |
| `stellantis-source-download-and-ingest/`              | Lead context — download + ingest                                       |
| `stellantis-lead-agent-subagent-orchestration/`       | Lead context — partition + spawn + consolidate                         |
| `stellantis-subagent-classification-loop/`            | Subagent context — KB retrieval + classification                       |

**Core-only assets** (not duplicated in sub-skills):
- `stellantis-automotive-feature-classification/references/` — enums, contracts, ownership matrix, workspace layout
- `stellantis-automotive-feature-classification/templates/` — all JSON/Markdown templates
- `stellantis-automotive-feature-classification/workflows/` — W1 and future workflow docs
- `stellantis-automotive-feature-classification/assets/` — params CSV policy

Sub-skills reference core assets by prose (e.g. *"per the `enums.md` reference in **automotive-feature-classification skill**"*) rather than relative file paths.

## Contents

* [`editing-rules.md`](editing-rules.md) — invariants anyone modifying a skill package must honour.
* [`impact-checklist.md`](impact-checklist.md) — when you change X, also update Y. Structured by change type.
* [`change-log.md`](change-log.md) — append-only history of framework edits. One entry per coherent change set.

## How to use this folder

1. Identify the intent (add a workflow, add an enum value, change a template shape, etc.).
2. Read [`editing-rules.md`](editing-rules.md).
3. Open [`impact-checklist.md`](impact-checklist.md), find the section matching the intent, apply every listed follow-up.
4. Make the edit.
5. Append a dated entry to [`change-log.md`](change-log.md) summarising what changed, why, and which impact-checklist items were applied.

## Why this is separate from the skill packages

Agents loading `automotive-feature-classification/SKILL.md` should not be distracted by "how to edit" meta-content. Separating these prevents the lead or any subagent from accidentally reading editing instructions as operational rules.

## Why this is separate from `business-logic/`

`business-logic/` is the problem definition, signed off by the product / client side. The skill packages are the implementation of that business logic. `framework-maintenance/` is instructions for modifying the skill packages. Three distinct concerns, three directories.
