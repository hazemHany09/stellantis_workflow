# Framework Change Log

Append-only. One entry per coherent change set. Newest on top.

Entry format:

```
## YYYY-MM-DD — <short title>

**Intent.** <one sentence>
**Files changed.**
- <path>
**Business-logic IDs touched.** <list or "—">
**Impact-checklist items applied.** <list>
**Migration notes.** <text or "none">
```

***

## 2026-04-30 — Source-type consensus, inverse retrieval, two-round R1/R2 architecture, and STATE.md enrichment

**Intent.** Resync `business-logic/` and `framework-maintenance/` with the substantial skill-package changes landed since the foundational tier. Three domain additions (consensus / confidence, inverse retrieval, source-type taxonomy) were promoted into the decision rule contract; the orchestration layer was reshaped around a two-round flow (Round 1 broad search-mode + Round 2 doc-deep-dive); the download stage was refactored to a subagent-driven flow with block detection and retry modes; STATE.md gained a tiered (T1/T2) section catalogue. Documentation now reflects what the skills actually do.

**Files changed.**

* `business-logic/10-decision-rules.md` — added *Source-type categories and consensus* (seven categories, independence rule, confidence annotation table, promotion / UGC-demotion rules) and *Inverse retrieval — prerequisite for Rule 4* (query templates, Rule 4→Rule 5 promotion path, `inverse_retrieval_attempted` flag). Added invariants 9–11.
* `business-logic/09-deliverable.md` — added `Confidence`, `Inverse retrieval attempted`, `Decision rule` to the per-parameter record table; added `Source type` and `Stance` to the traceability-block table.
* `business-logic/06-harness-interface.md` — added `confidence`, `inverse_retrieval_attempted`, `decision_rule` to the harness output contract; added invariants 11–13 for confidence / inverse-retrieval / no-UGC-only-Success enforcement.
* `framework-maintenance/README.md` — added `stellantis-subagent-doc-deep-dive` to the operational-skills table; clarified that orchestration is now a two-round R1/R2 flow.
* `framework-maintenance/editing-rules.md` — rule 8 expanded: writer discipline now covers `DeepDiveAgent/`, `DownloadAgent/`, and adds a STATE.md-section-addition checklist (template + write trigger + T1/T2 marker + resumability check).
* `framework-maintenance/impact-checklist.md` — new rows: `Add a Confidence value`, `Add a SourceType value`, `Change source-type consensus rules`, `Change inverse-retrieval contract`. New per-skill sections: `stellantis-lead-agent-subagent-orchestration` (R2 cap / partition / consolidator), `stellantis-subagent-doc-deep-dive` (contract + gap-input shape), `stellantis-source-download-and-ingest` (download-subagent contract / block detection / retry modes). `stellantis-run-workspace` row expanded for STATE.md sections and R1/R2 subagent file conventions. Path references retargeted from `runs/<run-id>/` to `.harness/`.

**Business-logic IDs touched.** Decision-rule scope expanded for source-type / inverse retrieval / confidence — these are domain rules, not implementation. No business-logic ID was reopened; the additions slot into the existing Rule 1–5 structure as new invariants and a new annotation field.

**Impact-checklist items applied.** Confidence enum row added. SourceType row added. Inverse-retrieval contract row added. Two-round R1/R2 sections registered for `stellantis-lead-agent-subagent-orchestration` and `stellantis-subagent-doc-deep-dive`. Download-subagent contract section registered for `stellantis-source-download-and-ingest`. STATE.md section-addition row registered for `stellantis-run-workspace`.

**Migration notes.**

* `business-logic/` deliberately does **not** document two-round R1/R2 orchestration, partition / concurrency ceilings, document-promise scoring, deep-dive subagents, STATE.md sections, RagFlow tool names, or Python CSV scripts. These are implementation concerns owned by the skill packages and `framework-maintenance/`. The business-logic surface stays domain-only — what the deliverable carries, how decisions are made, what evidence rules apply.
* `business-logic/05-knowledge-base.md` and `12-workflow-diagram.md` still reference RagFlow primitives (`doc_infos`, `delete_docs`, `create_dataset`) and `.harness/` from earlier passes. These are not corrected here. A future cleanup pass should rephrase them in vendor-neutral / role-neutral language so the document set reads cleanly for a non-technical client reviewer.
* The `automotive-feature-classification-skills/docs/` folder (untracked in git) holds design notes (`2026-04-30-state-md-enrichment-design.md` and the subagents/orchestration brainstorm). These are working artefacts, not part of either canonical layer; they should not be cited from `business-logic/` or referenced as authoritative by skill packages.

***

## 2026-04-26 — Add foundational-skill tier (domain-context, decision-rules, output-contract, workflow-modes, failure-handling, kb-protocol, approval-gate, run-workspace)

**Intent.** Promote cross-cutting concerns out of the operational sub-skills and into a tier of always-on foundational skills, so every agent role (lead and subagent) shares the same vocabulary, rule engine, output contract, retry policy, KB protocol, and workspace ownership rules. Eliminates drift between agents and centralises the high-value tips/pitfalls accumulated during design.

**Files changed.**

* `stellantis-domain-context/SKILL.md` — new. Mission, vocabulary contract, four hard constants, master invariants, glossary mini-reference, evergreen tips.
* `stellantis-decision-rules/SKILL.md` — new. Five rules, master matrix, edge cases, eight invariants, tips.
* `stellantis-output-contract/SKILL.md` — new. Per-parameter record shape, deliverable format, run header, summary, `sources_excluded` footer, ordering rules, tips.
* `stellantis-workflow-modes/SKILL.md` — new. W1 (Mode A / Mode B) authored; W2..W8 reserved as named slots; composition rules; maintenance procedure for adding workflows later.
* `stellantis-failure-handling/SKILL.md` — new. Six failure classes, retry budgets, drop policy, URL canonicalisation, `sources_excluded` emission, run-abort criteria.
* `stellantis-knowledge-base-protocol/SKILL.md` — new. Capability categories, scoping rule, document metadata, lifecycle, ingestion wait, retrieval contract, hard rules.
* `stellantis-approval-gate/SKILL.md` — new. `flag-for-approval` / `retrieve-approved` operations, URL-level semantics, yield/resume protocol, hard phase boundary (ARCH-4) enforcement.
* `stellantis-run-workspace/SKILL.md` — new. Workspace layout, writer ownership, pause/resume contract, resumability checklist.
* `stellantis-automotive-feature-classification/SKILL.md` — updated `requires` to include all eight foundational skills; preflight section now loads them first; common-mistakes section expanded.
* `stellantis-car-trim-resolution/SKILL.md` — added `requires: stellantis-domain-context`; added explicit "full option = manufacturer's published top trim" policy section; added Tips section.
* `stellantis-subagent-classification-loop/SKILL.md` — `requires` extended to include foundational skills (domain-context, decision-rules, output-contract, failure-handling, kb-protocol, run-workspace).
* `stellantis-lead-agent-subagent-orchestration/SKILL.md` — `requires` extended with foundational skills.
* `stellantis-source-download-and-ingest/SKILL.md` — `requires` extended with foundational skills + `stellantis-approval-gate`.
* `stellantis-source-validation/SKILL.md` — `requires` extended with foundational skills; downstream now feeds `stellantis-approval-gate`.
* `stellantis-source-discovery-with-client-domains/SKILL.md` — `requires` extended with foundational skills.
* `stellantis-source-discovery-without-client-domains/SKILL.md` — `requires` extended with foundational skills.
* `framework-maintenance/README.md` — replaced flat package table with foundational + operational tiers.
* `framework-maintenance/impact-checklist.md` — added `Edit a foundational skill` section (per-skill trigger tables), `Add tips / common pitfalls to a skill` section (where tips go and rules for adding them), `Add a new foundational skill` section.

**Business-logic IDs touched.** None. This is a structural redistribution; no business rule changed. The new skills are operational restatements of already-settled assumptions and decision rules.

**Impact-checklist items applied.** New foundational-skill tier registered. Lead skill `requires` updated. Cross-skill `requires` updated. Subagent inheritance of foundational skills documented. New impact-checklist sections added in lockstep with the new skills.

**Migration notes.**

* The partition cap of ≤ 15 parameters per subagent is already implemented in `stellantis-lead-agent-subagent-orchestration` and is unchanged. No separate partitioning skill is added.
* `stellantis-approval-gate` is now the canonical source for the URL-level approval contract. `stellantis-source-download-and-ingest` and `stellantis-source-validation` delegate to it instead of restating the gate semantics.
* The W1 workflow document (`workflows/W1-normal-pipeline-mode-a/WORKFLOW.md`) is unchanged and remains canonical for stage-by-stage flow. `stellantis-workflow-modes` only adds the dispatch and composition layer above it.
* When a new workflow is authored later, follow the procedure documented in `stellantis-workflow-modes` *Maintenance — adding a new workflow* and the new *Edit a foundational skill — `stellantis-workflow-modes`* row in the impact checklist.
* Tips are now first-class. The locations and rules for adding them are documented in the new *Add tips / common pitfalls to a skill* impact-checklist section.

***

## 2026-04-24 — Decompose monolithic `src/` into sibling skill packages

**Intent.** Replace the nested `src/skills/*/SKILL.md` layout with 8 independently importable skill packages so each can be zipped and distributed separately (Claude Desktop, DeerFlow, OpenClaw, etc. each require exactly one SKILL.md per import).

**Files changed.**

* `automotive-feature-classification/SKILL.md` — renamed from `src/SKILL.md`; removed broken `../../tools/` reference; updated `framework-maintenance` cross-reference to prose.
* `car-trim-resolution/SKILL.md` — promoted from `src/skills/car-trim-resolution/SKILL.md` to top-level sibling package.
* `source-discovery-with-client-domains/SKILL.md` — promoted; updated `../../templates/source-candidate-list.md.tmpl` reference to prose citing core skill.
* `source-discovery-without-client-domains/SKILL.md` — promoted.
* `source-validation/SKILL.md` — promoted.
* `source-download-and-ingest/SKILL.md` — promoted.
* `lead-agent-subagent-orchestration/SKILL.md` — promoted; updated 3 `../../references/` and `../../templates/` paths to prose citing core skill.
* `subagent-classification-loop/SKILL.md` — promoted; updated 4 `../../references/` and `../../templates/` paths to prose citing core skill.
* `framework-maintenance/README.md` — updated package layout table, replaced `src/` references.
* `framework-maintenance/editing-rules.md` — replaced all `src/` path references with new package paths.
* `framework-maintenance/impact-checklist.md` — replaced all `src/skills/*` and `src/references/*` and `src/templates/*` path references with new package-relative paths.

**Business-logic IDs touched.** None — structural redistribution only; no rule semantics changed.

**Impact-checklist items applied.** New sub-skill registration pattern updated (sibling package model). Tool allow-list sync rule updated. Enum single-source path updated.

**Migration notes.**

* All shared assets (references, templates, workflows) remain in `automotive-feature-classification/` only — sub-skills must never duplicate them.
* Sub-skill cross-references now use prose form: *"per `<file>` in **automotive-feature-classification skill**"* — never relative file paths.
* The old `src/` directory name is retired. Any external tooling or CI that referenced `src/SKILL.md` must be updated to `automotive-feature-classification/SKILL.md`.

***

## 2026-04-23 — Initial `src/` + `framework-maintenance/` scaffold

**Intent.** Author the runtime skill package under `src/` (entry point, references, templates, sub-skills, W1 workflow) and the editing-rules under `framework-maintenance/`. Operationalises the full business-logic layer in `business-logic/00-overview.md` .. `business-logic/14-client-questions.md`.

**Files changed.**

* `src/SKILL.md` — new. Agent entry point.
* `src/assets/params.csv.snapshot-policy.md` — new.
* `src/references/enums.md` — new. Single source of truth for Presence, Status, Classification, SchemaType, DecisionRule, SourceLifecycle, SourceOrigin, FailureStage, RunStage, RunStatus, PauseReason, SubagentStatus, TraceabilityBlockKind.
* `src/references/workspace-layout.md` — new.
* `src/references/file-ownership-matrix.md` — new.
* `src/references/lead-to-subagent-contract.md` — new.
* `src/references/subagent-to-lead-contract.md` — new.
* `src/templates/STATE.main.md.tmpl` — new.
* `src/templates/STATE.category.md.tmpl` — new.
* `src/templates/subagent-working-file.md.tmpl` — new.
* `src/templates/source-candidate-list.md.tmpl` — new.
* `src/templates/traceability-block.json.tmpl` — new.
* `src/templates/traceability-block.md.tmpl` — new.
* `src/templates/intermediate-parameter-record.json.tmpl` — new. Includes decision-rule matrix validator.
* `src/templates/subagent-contract.json.tmpl` — new.
* `src/templates/deliverable.schema.json` — new.
* `src/templates/deliverable.csv.columns.md` — new.
* `src/skills/source-discovery-with-client-domains/SKILL.md` — new.
* `src/skills/source-discovery-without-client-domains/SKILL.md` — new.
* `src/skills/source-validation/SKILL.md` — new.
* `src/skills/source-download-and-ingest/SKILL.md` — new. Integrates the new `run_document` and `get_metadata_summary` tools.
* `src/skills/car-trim-resolution/SKILL.md` — new.
* `src/skills/lead-agent-subagent-orchestration/SKILL.md` — new.
* `src/skills/subagent-classification-loop/SKILL.md` — new.
* `src/workflows/W1-normal-pipeline-mode-a/WORKFLOW.md` — new.
* `framework-maintenance/README.md` — new.
* `framework-maintenance/editing-rules.md` — new.
* `framework-maintenance/impact-checklist.md` — new.

**Business-logic IDs touched.** None (this change consumes the business-logic layer; it does not modify it).

**Impact-checklist items applied.** New workflow scaffolding; new sub-skills authored; tool allow-lists reconciled between `src/SKILL.md` §9 and each sub-skill's header; workspace layout + ownership matrix authored in lockstep.

**Migration notes.**

* W2 (Mode B standalone) is not yet authored as a workflow file; the Mode B **sub-skill** exists and is ready to plug in when the W2 workflow doc is written.
* Approval-store tool integration is deferred. The approval gate runs on free-text posts; when the approval-store capability lands, only stage 3b and 3c of `W1-normal-pipeline-mode-a/WORKFLOW.md` need to change.
* New RAGFlow tools `run_document` and `get_metadata_summary` are referenced in `src/SKILL.md` §9, `src/skills/source-download-and-ingest/SKILL.md`, and `src/skills/subagent-classification-loop/SKILL.md` / `src/skills/lead-agent-subagent-orchestration/SKILL.md` respectively. The tool manifest in `tools/RAGFLOW_AGENT_TOOLS.md` does not yet list them — **TODO for a future change set**: update that manifest to match.

