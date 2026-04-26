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

