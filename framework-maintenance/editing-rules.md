# Editing Rules

Invariants any modification to the skill packages **must** honour. Violations of these rules turn the framework inconsistent and are bugs even if the edit "works" locally.

## 1. Business logic is the source of truth

`business-logic/` is canonical. If a rule in any skill package contradicts `business-logic/`, the skill package is wrong. If you believe `business-logic/` is wrong, the fix starts in `business-logic/` first — skill packages follow.

Every normative rule in `automotive-feature-classification/SKILL.md`, `automotive-feature-classification/workflows/*/WORKFLOW.md`, and any sub-skill SKILL.md must cite the business-logic ID it implements (e.g. `AS-HARN-D`, `Q-SRC-6`, `ARCH-4`). If a rule has no citation, add one or delete the rule.

## 2. Enums are single-sourced

`automotive-feature-classification/references/enums.md` is the **only** place allowed to define a value set. Every other file must refer to it by name, not by repeating values.

Forbidden patterns:

* Copying `"Yes", "No", "Disputed", "No Information Found"` into a template's prose.
* Listing `RunStage` values inline in a workflow file.
* Defining a sibling enum file under any other path.

When adding a new value: edit `enums.md`, list the business-logic justification, then add a changelog entry.

## 3. Templates do not hard-code enum values

A template may show an example value in a placeholder comment (e.g. `<<one of enums.md::RunStage>>`). It must never hard-code a single value as *the* value. Examples in prose may quote values if they are examples; all normative references link to `enums.md`.

## 4. Cross-doc content is linked, never duplicated

If two skill package files need to state the same rule, one states it and the other references the first by name. Duplication drifts. Prefer references over copies.

Exception: short summary tables (e.g. the decision-rule matrix copy in `06-harness-interface.md`) are allowed to exist as validation copies provided they are labelled as such.

## 5. Workflow additions do not alter existing workflows

Adding W2 must not touch W1 files. Cross-workflow changes (a shared sub-skill gains a new argument) update the sub-skill + any workflow that invokes it. Never silently change a sub-skill's contract with a single workflow in mind.

## 6. Tool allow-lists are enforced

Two places define them: `automotive-feature-classification/SKILL.md` Tool Allow-List section, and each sub-skill's `Tools allowed` header. They must agree. A tool introduced in a sub-skill must also appear in `automotive-feature-classification/SKILL.md` with the correct actor (Lead / Subagent) flag.

## 7. ARCH-4 is absolute

Subagents never use `google_search*` or `browser_render*`. Any edit that implies otherwise must first modify `business-logic/11-assumptions.md` ARCH-4 and close it in a dated revision.

## 8. `.harness/` writer discipline

Main `STATE.md` and `.harness/Category/*.md` are **lead-only writable**. `.harness/SubAgent/<agent-name>.md` is subagent-writable after spawn (the contract header is lead-seeded once and never re-edited).

Any edit that adds a new file under `runs/<run-id>/` must specify the writer in `automotive-feature-classification/references/file-ownership-matrix.md` before it is created at runtime.

## 9. Deliverable schema is backward-compatible

Additions to `deliverable.schema.json` must be non-breaking (new optional fields; never remove or retype existing fields). A breaking change requires a new `deliverable.schema.v2.json` and a workflow-layer flag.

## 10. Never edit during a run

`framework-maintenance/` touches must not happen while a run is mid-flight against the same skill packages. This is an operational rule, enforced by the operator.

## 11. Record every change

Every coherent change set (one feature, one fix, one refactor) gets one entry in `change-log.md` with:

* Date.
* Short title.
* Files changed.
* Business-logic IDs touched.
* Migration notes (if any).

