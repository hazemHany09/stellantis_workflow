# Framework Maintenance

This folder is for an **operator or an agent explicitly invoked to modify the Automotive Feature Classification skill framework**. It is **not** part of the runtime. A production run must not touch any file here and must not read any file here.

## Audience

1. A developer evolving `src/` (adding a workflow, changing an enum, adjusting a template).
2. An agent explicitly asked *"update the framework"* and given a change request.

## Contents

* [`editing-rules.md`](editing-rules.md) — invariants anyone modifying `src/` must honour.
* [`impact-checklist.md`](impact-checklist.md) — when you change X, also update Y. Structured by change type.
* [`change-log.md`](change-log.md) — append-only history of framework edits. One entry per coherent change set.

## How to use this folder

1. Identify the intent (add a workflow, add an enum value, change a template shape, etc.).
2. Read [`editing-rules.md`](editing-rules.md).
3. Open [`impact-checklist.md`](impact-checklist.md), find the section matching the intent, apply every listed follow-up.
4. Make the edit.
5. Append a dated entry to [`change-log.md`](change-log.md) summarising what changed, why, and which impact-checklist items were applied.

## Why this is separate from `src/`

`src/` is the deployed runtime package. Agents loading `src/SKILL.md` should not be distracted by "how to edit" meta-content. Separating these prevents the lead or any subagent from accidentally reading editing instructions as operational rules.

## Why this is separate from `business-logic/`

`business-logic/` is the problem definition, signed off by the product / client side. `src/` is the implementation of that business logic. `framework-maintenance/` is instructions for modifying `src/`. Three distinct concerns, three directories.
