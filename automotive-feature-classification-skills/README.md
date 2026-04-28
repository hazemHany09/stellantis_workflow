# Automotive Feature Classification Skill

This skill automates per-parameter feature classification for a single car (brand + model + year + market). It has three layers, each in its own directory:

| Layer                                  | Directory                    | Role                                                                                     |
| :------------------------------------- | :--------------------------- | :--------------------------------------------------------------------------------------- |
| **Business logic** (source of truth)   | [`business-logic/`](business-logic/) | What the problem is. Vocabulary, data model, decision rules, constraints, open questions. |
| **Runtime skill package**              | [`src/`](src/)               | How the problem is solved. Agent entry point, references, templates, sub-skills, workflows. |
| **Self-modification rules**            | [`framework-maintenance/`](framework-maintenance/) | How the runtime is edited. Editing rules, impact checklist, change log.               |

## Where to start

* **Running the skill from an agent:** open [`src/SKILL.md`](src/SKILL.md). That is the entry point — every rule the lead agent needs to operate, plus pointers into references, templates, sub-skills, and workflows.
* **Understanding the problem:** read `business-logic/` in numeric order, starting at [`business-logic/00-overview.md`](business-logic/00-overview.md).
* **Modifying the framework:** read [`framework-maintenance/README.md`](framework-maintenance/README.md) and apply [`framework-maintenance/impact-checklist.md`](framework-maintenance/impact-checklist.md) before making changes.

## Business-logic layer

The skill is designed to be portable. Any system that can read files and invoke a web-search tool, a browser-rendering tool, and a knowledge-base tool can run it. No vendor names or cloud-specific terms appear in the business logic.

## Read order

Read the documents in numeric order. Each one builds on the previous.

| # | File                                                              | What it covers                                                                                                                          |
| - | :---------------------------------------------------------------- | :-------------------------------------------------------------------------------------------------------------------------------------- |
| 0 | [`business-logic/00-overview.md`](business-logic/00-overview.md)                               | Purpose, actors, scope, what is in/out of scope for this document set.                                                                 |
| 1 | [`business-logic/01-glossary.md`](business-logic/01-glossary.md)                               | Every term used elsewhere. Resolves the *category vs source site* name collision.                                                      |
| 2 | [`business-logic/02-parameters-and-levels.md`](business-logic/02-parameters-and-levels.md)     | Structure of `artifacts/params.csv`, applicable level set, presence vs level.                                                           |
| 3 | [`business-logic/03-inputs.md`](business-logic/03-inputs.md)                                   | What the client submits. Two modes of source-site provisioning (supplied / discovered).                                                |
| 4 | [`business-logic/04-sources.md`](business-logic/04-sources.md)                                 | Source lifecycle, approval gate, soft selection guidance.                                                                              |
| 5 | [`business-logic/05-knowledge-base.md`](business-logic/05-knowledge-base.md)                   | Role, scoping, async ingestion, required capabilities, lifecycle.                                                                     |
| 6 | [`business-logic/06-harness-interface.md`](business-logic/06-harness-interface.md)             | Input/output contract of the classification stage. Algorithm deliberately out of scope.                                               |
| 7 | [`business-logic/07-tools-and-constraints.md`](business-logic/07-tools-and-constraints.md)     | Vendor-neutral tool capabilities (four categories, incl. planned source approval store), hard and soft constraints.                   |
| 8 | [`business-logic/08-open-questions.md`](business-logic/08-open-questions.md)                   | Everything still unresolved. Must close before the workflow layer is written.                                                          |
| 9 | [`business-logic/09-deliverable.md`](business-logic/09-deliverable.md)                         | Final per-parameter record schema + traceability block schema. Serialisation format still open.                                       |
| 10 | [`business-logic/10-decision-rules.md`](business-logic/10-decision-rules.md)                  | Source taxonomy (clear / vague / silent) and the four decision rules governing Presence / Status / Classification.                    |

## External references

- `artifacts/params.csv` — the authoritative reference parameter list.
- `tools/BUSINESS_LOGIC.md`, `tools/*_AGENT_TOOLS.md` — concrete tools available in this deployment. The business-logic documents deliberately refer to them only by capability category (search, browser rendering, knowledge base).

## Runtime layer (`src/`)

| File / folder                                                                             | Role                                                                                               |
| :---------------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------- |
| [`src/SKILL.md`](src/SKILL.md)                                                            | **Agent entry point.** Core framework rules, preflight, pause/resume, tool allow-list, roles.     |
| [`src/assets/`](src/assets/)                                                              | Per-run snapshot policy for `artifacts/params.csv`.                                                |
| [`src/references/`](src/references/)                                                      | Enums (single source of truth), workspace layout, file-ownership matrix, lead↔subagent contracts. |
| [`src/templates/`](src/templates/)                                                        | STATE.md templates, working-file template, source-list template, JSON schemas for deliverable / contract / traceability / intermediate records, CSV column mapping. |
| [`src/skills/`](src/skills/)                                                              | Runtime-loaded sub-skills: source discovery (Mode A + Mode B), source validation, download + ingest, trim resolution, orchestration, subagent classification loop. |
| [`src/workflows/W1-normal-pipeline-mode-a/`](src/workflows/W1-normal-pipeline-mode-a/)    | The only authored workflow today: normal pipeline with client-supplied URLs.                       |

## Maintenance layer (`framework-maintenance/`)

| File                                                                                                    | Role                                                              |
| :------------------------------------------------------------------------------------------------------ | :---------------------------------------------------------------- |
| [`framework-maintenance/README.md`](framework-maintenance/README.md)                                    | Purpose + how to use the maintenance folder.                      |
| [`framework-maintenance/editing-rules.md`](framework-maintenance/editing-rules.md)                      | Invariants any `src/` edit must honour.                           |
| [`framework-maintenance/impact-checklist.md`](framework-maintenance/impact-checklist.md)                | When you change X, also update Y.                                 |
| [`framework-maintenance/change-log.md`](framework-maintenance/change-log.md)                            | Append-only history.                                              |
