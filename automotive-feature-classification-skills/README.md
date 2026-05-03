# Automotive Feature Classification Skill

This skill automates per-parameter feature classification for a single car (brand + model + year + market). It is packaged as a constellation of cooperating skills, each in its own top-level folder named `stellantis-*`.

## Architectural rules (load-bearing)

These rules are enforced by every skill in the package. Violations are bugs.

1. **The lead is the sole spawner.** No subagent at any stage spawns another subagent. The lead spawns subagents in exactly two stages:
   - **W1 stage 4** — one **download/ingest subagent** per approved URL (governed by [`stellantis-source-download-and-ingest`](stellantis-source-download-and-ingest/SKILL.md)).
   - **W1 stage 6** — **R1 classification subagents** per single-category parameter partition, then **R2 deep-dive subagents** per (target document × gap parameters) (governed by [`stellantis-lead-agent-subagent-orchestration`](stellantis-lead-agent-subagent-orchestration/SKILL.md)).
2. **Lead initialises `.harness/` at `/mnt/user-data/workspace/.harness/` before doing anything else.** The agent operates inside a fixed three-path sandbox — `/mnt/user-data/workspace/` (read+write, holds `.harness/` and nothing else), `/mnt/user-data/outputs/` (read+write, deliverables only), `/mnt/user-data/uploads/` (read-only, user-supplied inputs). Source discovery, validation, the approval gate, and `.harness/` setup are all lead-only — the lead waits for explicit user approval before spawning any download subagent.
3. **Two mandatory user gates in W1.** The lead pauses (a) for **URL approval** before download, and (b) for the **classification go-ahead** after ingestion is verified and before any R1 classification subagent is spawned. After ingestion completes, the lead posts the ingestion + planned-partition summary and waits for the user's explicit `go` reply.
4. **Download/ingest subagents are content-blind.** They call `fetch_url` (or `fetch_webpage` in retry mode) and `ragflow_upload`, capture the saved file's byte size, and report findings. **They never open or read the file's contents.** The lead derives `block_type` (rate-limited / access-denied / null) from size + HTTP status.
5. **Reading documents is allowed only at the deep-dive (R2) stage.** R1 classification subagents read evidence only via the KB (`retrieval`, `list_chunks`, `ragflow_download`). R2 deep-dive subagents read exactly one local Markdown — the file at `target_doc.local_path` in their contract.
6. **No subagent reads `params.csv`.** The lead reads `.harness/params.csv` directly, partitions parameters **by category, max 15 per subagent, never crossing categories**, and embeds the exact parameter slice in each subagent's contract.
7. **Two stages with multiple subagents:** download/ingest (stage 4) and classification (stage 6). Both are spawned exclusively by the lead.

## Skill index (flat layout, all under repo root)

| Skill folder                                                                  | Role                                                              |
| :---------------------------------------------------------------------------- | :---------------------------------------------------------------- |
| [`stellantis-automotive-feature-classification/`](stellantis-automotive-feature-classification/SKILL.md) | **Agent entry point.** Core framework rules, preflight, role table, tool allow-list. Owns `references/`, `templates/`, `workflows/`. |
| [`stellantis-domain-context/`](stellantis-domain-context/SKILL.md)            | Mission, vocabulary, master invariants, tips.                     |
| [`stellantis-decision-rules/`](stellantis-decision-rules/SKILL.md)            | Decision rules + master matrix + invariants.                      |
| [`stellantis-output-contract/`](stellantis-output-contract/SKILL.md)          | Per-parameter record shape + deliverable format.                  |
| [`stellantis-workflow-modes/`](stellantis-workflow-modes/SKILL.md)            | Workflow identification at preflight.                             |
| [`stellantis-failure-handling/`](stellantis-failure-handling/SKILL.md)        | Retries, drops, timeouts, exclusion policy.                       |
| [`stellantis-knowledge-base-protocol/`](stellantis-knowledge-base-protocol/SKILL.md) | KB capability contract; ingestion wait; retrieval discipline. |
| [`stellantis-run-workspace/`](stellantis-run-workspace/SKILL.md)              | Workspace layout, writer ownership, pause/resume.                 |
| [`stellantis-car-trim-resolution/`](stellantis-car-trim-resolution/SKILL.md)  | Resolve trim to manufacturer's published top trim.                |
| [`stellantis-source-discovery-with-client-domains/`](stellantis-source-discovery-with-client-domains/SKILL.md) | Mode A — search inside user-supplied domains/URLs.   |
| [`stellantis-source-discovery-without-client-domains/`](stellantis-source-discovery-without-client-domains/SKILL.md) | Mode B — open-web search.                          |
| [`stellantis-source-validation/`](stellantis-source-validation/SKILL.md)      | Validate + filter candidate source list.                          |
| [`stellantis-approval-gate/`](stellantis-approval-gate/SKILL.md)              | Flag URLs, yield, resume on explicit signal.                      |
| [`stellantis-source-download-and-ingest/`](stellantis-source-download-and-ingest/SKILL.md) | **Lead spawns one content-blind download/ingest subagent per approved URL.** |
| [`stellantis-lead-agent-subagent-orchestration/`](stellantis-lead-agent-subagent-orchestration/SKILL.md) | **Lead-only spawner of R1 + R2 classification subagents.** No dispatch layer. |
| [`stellantis-subagent-classification-loop/`](stellantis-subagent-classification-loop/SKILL.md) | R1 classification subagent — KB retrieval only.            |
| [`stellantis-subagent-doc-deep-dive/`](stellantis-subagent-doc-deep-dive/SKILL.md) | R2 deep-dive subagent — single local Markdown read.          |
| [`stellantis-run-workspace/`](stellantis-run-workspace/SKILL.md)              | Workspace layout, file ownership.                                 |
| [`business-logic/`](business-logic/)                                          | Source-of-truth narrative for the problem (read in numeric order). |
| [`framework-maintenance/`](framework-maintenance/)                            | Editing rules, impact checklist, change log.                      |

## Where to start

* **Running the skill from an agent:** open [`stellantis-automotive-feature-classification/SKILL.md`](stellantis-automotive-feature-classification/SKILL.md). That is the entry point — every rule the lead agent needs to operate, plus pointers into references, templates, sub-skills, and workflows.
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

- The reference parameter list lives bundled with the skill; the user may override it by placing a custom file at `/mnt/user-data/uploads/params.csv` (read-only). At preflight the lead copies whichever applies into `/mnt/user-data/workspace/.harness/params.csv` — the lead-only frozen snapshot for the run.
- `tools/BUSINESS_LOGIC.md`, `tools/*_AGENT_TOOLS.md` — concrete tools available in this deployment. The business-logic documents deliberately refer to them only by capability category (search, browser rendering, knowledge base).

## Maintenance layer (`framework-maintenance/`)

| File                                                                                                    | Role                                                              |
| :------------------------------------------------------------------------------------------------------ | :---------------------------------------------------------------- |
| [`framework-maintenance/README.md`](framework-maintenance/README.md)                                    | Purpose + how to use the maintenance folder.                      |
| [`framework-maintenance/editing-rules.md`](framework-maintenance/editing-rules.md)                      | Invariants any skill edit must honour.                            |
| [`framework-maintenance/impact-checklist.md`](framework-maintenance/impact-checklist.md)                | When you change X, also update Y.                                 |
| [`framework-maintenance/change-log.md`](framework-maintenance/change-log.md)                            | Append-only history.                                              |
