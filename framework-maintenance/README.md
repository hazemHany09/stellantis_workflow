# Framework Maintenance

This folder is for an **operator or an agent explicitly invoked to modify the Automotive Feature Classification skill framework**. It is **not** part of the runtime. A production run must not touch any file here and must not read any file here.

## Audience

1. A developer evolving the skill packages (adding a workflow, changing an enum, adjusting a template).
2. An agent explicitly asked *"update the framework"* and given a change request.

## Skill Package Layout

The framework is now distributed as **8 independently importable skill packages** (one SKILL.md each):

| Package directory                          | Role                                                                   |
| :----------------------------------------- | :--------------------------------------------------------------------- |
| `automotive-feature-classification/`       | **Core skill** — entry point, shared references, templates, workflows  |
| `car-trim-resolution/`                     | Sub-skill — lead context                                               |
| `source-discovery-with-client-domains/`    | Sub-skill — lead context                                               |
| `source-discovery-without-client-domains/` | Sub-skill — lead context                                               |
| `source-validation/`                       | Sub-skill — lead context                                               |
| `source-download-and-ingest/`              | Sub-skill — lead context                                               |
| `lead-agent-subagent-orchestration/`       | Sub-skill — lead context                                               |
| `subagent-classification-loop/`            | Sub-skill — subagent context                                           |

**Core-only assets** (not duplicated in sub-skills):
- `automotive-feature-classification/references/` — enums, contracts, ownership matrix, workspace layout
- `automotive-feature-classification/templates/` — all JSON/Markdown templates
- `automotive-feature-classification/workflows/` — W1 and future workflow docs
- `automotive-feature-classification/assets/` — params CSV policy

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
