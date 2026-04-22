# Automotive Feature Classification Skill — Business Logic

This directory is the **business-logic layer** of the skill. It describes the problem, the vocabulary, the data model, and the constraints — not the workflow, not the commands, not the prompts. The workflow, templates, commands, and rules will be authored in later passes that read from this layer.

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
