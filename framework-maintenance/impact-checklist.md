# Impact Checklist

When you make a change of the kind listed in the leftmost column, apply **every** item in the right column before landing it. Omissions are bugs.

## Add a new value to an enum

| Change                                              | Follow-ups                                                                                                                                                                                                                                                                                                                                                                                                                     |
| :-------------------------------------------------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Add `Presence` value                                | Update `automotive-feature-classification/references/enums.md`. Update `automotive-feature-classification/templates/intermediate-parameter-record.json.tmpl` `enum`. Update `automotive-feature-classification/templates/deliverable.schema.json` `enum` inside `records[].presence`. Update `business-logic/10-decision-rules.md` (add a new rule). Update `business-logic/06-harness-interface.md` invariants + matrix. Update `subagent-classification-loop/SKILL.md` decision table. Add changelog entry. |
| Add `Status` value                                  | Same pattern as Presence.                                                                                                                                                                                                                                                                                                                                                                                                                           |
| Add `Classification` value                          | Same pattern as Presence + update `SchemaType` section of enums if it implies a new tier.                                                                                                                                                                                                                                                                                                                                                           |
| Add `RunStage`, `RunStatus`, or `PauseReason` value | Update `enums.md`. Update `automotive-feature-classification/templates/STATE.main.md.tmpl`. Update `automotive-feature-classification/SKILL.md` Pause/Resume Protocol section. Update any workflow file that emits the new value. Add changelog entry.                                                                                                                                                                                               |
| Add `SourceLifecycle` value                         | Update `enums.md`. Update `source-download-and-ingest/SKILL.md` decision tree. Update `business-logic/12-workflow-diagram.md` section B. Add changelog entry.                                                                                                                                                                                                                                                                                       |
| Add a `DecisionRule` value (e.g. Rule-6)            | Update `enums.md`. Update `business-logic/10-decision-rules.md`. Update `business-logic/06-harness-interface.md` matrix. Update `automotive-feature-classification/templates/intermediate-parameter-record.json.tmpl` `allOf` matrix. Update `automotive-feature-classification/templates/deliverable.schema.json` `decision_rule` enum. Update `subagent-classification-loop/SKILL.md`. Add changelog entry.                                        |
| Add a `Confidence` value (e.g. new label beyond `consensus / single-source / vague-only / silent-all`) | Update `enums.md::Confidence`. Update `intermediate-parameter-record.json.tmpl` `confidence` enum + the rule-to-confidence validator. Update `deliverable.schema.json` `confidence` enum. Update `stellantis-decision-rules/SKILL.md` consensus table + invariant 9. Update `stellantis-output-contract/SKILL.md` per-parameter record table. Update `business-logic/10-decision-rules.md` *Confidence annotation* table. Update `business-logic/06-harness-interface.md` invariant 11. Add changelog entry. |
| Add a `SourceType` value (new category beyond the seven defined) | Update `enums.md::SourceType`. Update `stellantis-decision-rules/SKILL.md` source-type table + which categories count as independent. Update `stellantis-source-validation/SKILL.md` assignment logic. Update `stellantis-output-contract/SKILL.md` traceability-block fields. Update `business-logic/10-decision-rules.md` source-type table. If the new category implies a new authority profile, document which decisions it can/can't authoritatively back. Add changelog entry. |

## Add a new workflow

| Change                                   | Follow-ups                                                                                                                                                                                                                 |
| :--------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| New workflow (e.g. W2 Mode B standalone) | Create `automotive-feature-classification/workflows/<id>/WORKFLOW.md`. Link it from `automotive-feature-classification/SKILL.md` Workflow & Sub-Skill Loading section. Add a section to `business-logic/13-potential-workflows.md`. Add an entry to the decision tree in §12.G. Add changelog entry. |

## Add a new sub-skill

| Change        | Follow-ups                                                                                                                                                                                                                            |
| :------------ | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| New sub-skill | Create `<name>/SKILL.md` as a sibling package (same level as `automotive-feature-classification/`) with `name`, `description`, `loader` front-matter. Declare its tools in both its header and `automotive-feature-classification/SKILL.md` Tool Allow-List. Reference it from at least one workflow or another sub-skill by skill name (not file path). Add changelog entry. |

## Edit a foundational skill

Foundational skills are loaded at preflight by **every** agent role and stay in context for the whole run. They are therefore the highest-impact skills in the framework. The trigger conditions and follow-ups below are stricter than for operational skills.

### `stellantis-domain-context`

| Trigger to edit | Follow-ups |
| :--- | :--- |
| A vocabulary term changes meaning, is added, or is removed | Update this skill's vocabulary table. Update `business-logic/01-glossary.md`. Update `automotive-feature-classification/references/enums.md` if the term has an enum. Audit every other skill for stale usages of the old term. Add changelog entry. |
| A master invariant is added, removed, or reworded | Update this skill's invariant list. Update `business-logic/00-overview.md` and `business-logic/11-assumptions.md` to match. Audit `stellantis-decision-rules` for cross-references. Add changelog entry. |
| A new evergreen tip / common pitfall is identified across runs | Add to the `Tips & common pitfalls` section. **Do not** add tips that belong to a specific operational skill \u2014 those go in that skill's own Tips section. Tips here must be vocabulary-, invariant-, or rule-shaped. Add changelog entry. |
| Mission scope changes | Edit `business-logic/00-overview.md` first. Then update this skill's "Mission" paragraph. Then audit every operational skill. Add changelog entry. |

This skill must remain **compact** (\u2264 300 lines). Volatile content belongs in dedicated skills, not here.

### `stellantis-decision-rules`

| Trigger to edit | Follow-ups |
| :--- | :--- |
| A decision rule is added, removed, or modified | Apply the *Change a decision rule in business-logic* row above in full. Update this skill's rule list, master matrix, and invariants. Update `stellantis-subagent-classification-loop` rule references. Update `stellantis-output-contract` traceability cardinality table. |
| An enum value used in a rule changes | Apply the *Add a new value to an enum* row above. Update this skill's matrix entries. |
| A new edge case is observed in real runs | Add to the `Edge cases` section. If the edge case implies a new rule, escalate to the rule-change procedure. Otherwise update tips. |

### `stellantis-lead-agent-subagent-orchestration`

| Trigger to edit | Follow-ups |
| :--- | :--- |
| Round-2 deep-dive cap, gap-parameter detection rules, or document-promise-board scoring change | Update this skill's R1→R2 transition section. Update `stellantis-subagent-doc-deep-dive` if the contract or input shape changes. Update `STATE.main.md.tmpl` *R2 subagent roster* and *gap parameters + R2 target assignments* sections. Update `document-promise-board.md` schema if scoring inputs change. Update consolidation merge rules (single-source → consensus promotion). Add changelog entry. |
| Partition strategy change (≤15 cap, category-split heuristic, max-3 concurrent) | Update this skill's partition section. Update `business-logic/11-assumptions.md::AS-HARN-D` if the threshold itself moves (this is a settled assumption — reopening requires explicit justification). Update partition-summaries.md schema if outputs change. Add changelog entry. |
| Consolidator validation rules change (e.g. Rule-4 inverse-retrieval check, UGC-only Rule-1 demotion, confidence assignment) | Update this skill's *consolidation pass* section. Update `stellantis-decision-rules/SKILL.md` invariants 9–11 if validators are added or removed. Update `intermediate-parameter-record.json.tmpl` validators. Add changelog entry. |

### `stellantis-subagent-doc-deep-dive`

| Trigger to edit | Follow-ups |
| :--- | :--- |
| New deep-dive subagent contract field or result envelope shape | Update this skill. Update `stellantis-lead-agent-subagent-orchestration` Round-2 spawn / harvest sections. Update `STATE.main.md.tmpl` *R2 roster* fields if surfaced. Update file-ownership matrix for `.harness/DeepDiveAgent/<agent-name>.md`. Add changelog entry. |
| Gap-parameter input shape changes | Update this skill's input section. Update `stellantis-lead-agent-subagent-orchestration` gap-detection output. Update STATE.md *gap parameters + R2 target assignments* JSON block. Add changelog entry. |

### `stellantis-source-download-and-ingest`

| Trigger to edit | Follow-ups |
| :--- | :--- |
| Phase-2 / Phase-3 download subagent contract or result envelope shape change | Update this skill's Phase 2 + Phase 3 sections. Update `.harness/DownloadAgent/<slug>-dl.md` template if it exists, otherwise document the inline JSON-block contract. Update `stellantis-lead-agent-subagent-orchestration` if the lead reads the same envelope shape. Update file-ownership matrix. Add changelog entry. |
| Block-detection heuristics (file-size thresholds, marker scanning rules) change | Update this skill's block-detection section. Update `stellantis-failure-handling` failure-class definitions if a new class surfaces. Add changelog entry. |
| Retry-mode set changes (`retry-webpage`, `retry-network`, future modes) | Update this skill's retry section. Update tool allow-list (`fetch_webpage` is retry-only — mode gating must be enforced). Add changelog entry. |

### `stellantis-output-contract`

| Trigger to edit | Follow-ups |
| :--- | :--- |
| A field is added/removed/retyped in a per-parameter record | Apply the *Change a template* row above for `intermediate-parameter-record.json.tmpl` and `deliverable.schema.json`. Update this skill. Bump `contract_version` on breaking changes. |
| The deliverable filename pattern changes | Update this skill, `automotive-feature-classification/SKILL.md` Preflight section, and the workspace layout reference. |
| The run header / footer / summary section changes | Update this skill plus the `STATE.main.md.tmpl` and `deliverable.schema.json` templates. |

### `stellantis-workflow-modes`

| Trigger to edit | Follow-ups |
| :--- | :--- |
| A new workflow is authored | Apply the *Add a new workflow* row above. **Promote** the workflow's entry in this skill from *Reserved slots* to *Currently authored workflows*. Update the lead skill's dispatch logic. |
| A composition rule changes | Update the `Composition rules` section here. Audit every workflow doc for compatibility. Add changelog entry. |
| A reserved slot becomes obsolete | Remove its entry from *Reserved slots* and document the rationale in the change log. |

### `stellantis-failure-handling`

| Trigger to edit | Follow-ups |
| :--- | :--- |
| A retry budget, ceiling, or backoff schedule changes | Update this skill's Failure-class section. Update `stellantis-source-download-and-ingest` and `stellantis-knowledge-base-protocol` to reflect. Update `business-logic/11-assumptions.md` if a settled assumption is touched. |
| A new failure class is identified | Add a new section to this skill. Wire it into the relevant operational skill. Update the `sources_excluded` footer schema if the failure surfaces in the deliverable. |
| URL canonicalisation rules change | Update this skill **and** `stellantis-approval-gate` (which calls into the canonicaliser). |

### `stellantis-knowledge-base-protocol`

| Trigger to edit | Follow-ups |
| :--- | :--- |
| A new KB capability becomes mandatory or optional | Update this skill's capability-categories table. Update `business-logic/05-knowledge-base.md`. |
| A required document-metadata field is added/removed | Update this skill. Update `stellantis-source-download-and-ingest` upload logic. Update `stellantis-output-contract` if the field surfaces in the deliverable. |
| The ingestion wait protocol changes | Update this skill, `stellantis-source-download-and-ingest`, and the relevant pause-reason values in `enums.md`. |
| A new retrieval iteration budget is set | Update this skill and `stellantis-subagent-classification-loop`. |

### `stellantis-approval-gate`

| Trigger to edit | Follow-ups |
| :--- | :--- |
| The yield/resume signal contract changes | Update this skill, `stellantis-automotive-feature-classification` Pause/Resume Protocol section, and the relevant `PauseReason` enum value. |
| A new per-URL approval-entry field is required | Update this skill, `stellantis-source-validation` (it now produces the field), and any UI/store binding that consumes it. |
| The hard phase boundary (ARCH-4) changes | This is a business-logic-level edit. Touch `business-logic/11-assumptions.md` ARCH-4 first; do not edit this skill before the business-logic change is closed. |

### `stellantis-run-workspace`

| Trigger to edit | Follow-ups |
| :--- | :--- |
| A new file under `.harness/` is introduced | Apply the *Add a new file under `.harness/`* row below. Update this skill's layout tree and writer-ownership table. |
| Writer ownership for an existing file changes | Update this skill's writer-ownership table **and** `automotive-feature-classification/references/file-ownership-matrix.md`. The two must agree. |
| The pause/resume contract changes | Update this skill and the lead skill's Pause/Resume Protocol section. |
| A new section is added to `STATE.md` (T1 or T2) | Update `STATE.main.md.tmpl` with the section header + placeholder content. Mark the section [T1] (lifecycle/critical) or [T2] (analytics/audit). Document in the producer skill the exact write trigger (which phase, on which event). Update this skill's STATE-section index and resumability checklist. Update `stellantis-output-contract` if the section surfaces in the deliverable header. Add changelog entry. |
| Round 1 / Round 2 subagent file conventions change (e.g. naming under `SubAgent/`, `DeepDiveAgent/`) | Update this skill's layout tree. Update `automotive-feature-classification/references/file-ownership-matrix.md`. Update the relevant orchestration / loop / deep-dive skills. Add changelog entry. |

## Add tips / common pitfalls to a skill

Tips are derived from observed run failures, design reviews, or recurring user questions. They are not free editorial commentary.

| Trigger to add a tip | Where it goes | Follow-ups |
| :--- | :--- | :--- |
| A vocabulary or rule mistake is observed across multiple skills | `stellantis-domain-context` Tips section | Audit other skills for related mis-statements. Add changelog entry. |
| A mistake is specific to interpreting the decision rules | `stellantis-decision-rules` Tips section | Cross-check the master matrix. |
| A mistake is specific to record shape or deliverable format | `stellantis-output-contract` Tips section | Update the relevant template if the rule needs schema-level enforcement. |
| A mistake is specific to retries / drops / timeouts | `stellantis-failure-handling` Tips section | None additional. |
| A mistake is specific to KB usage | `stellantis-knowledge-base-protocol` Tips section | None additional. |
| A mistake is specific to one operational stage | That stage's sub-skill Tips section | Do not duplicate into foundational skills. |
| A mistake is generic across the framework | `stellantis-domain-context` Tips section | Keep wording short \u2014 one bullet, one idea. |

**Rules for tips:**
- Each tip is one sentence (or one short bullet) describing the wrong behaviour and the correct behaviour.
- Tips never invent new rules; they reinforce existing ones. If a tip implies a rule change, open a business-logic change instead.
- Tips never reference internal business-logic file names. They speak in plain operational terms.
- Tips that become obsolete (the underlying mistake is no longer possible) are deleted, not retained as historical notes.

## Add a new foundational skill

Adding a foundational skill is **rare** and high-impact. Use this only when a new domain of cross-cutting concern emerges.

| Change | Follow-ups |
| :--- | :--- |
| New foundational skill | Create the package with `loader: shared` and `stage: always-on` (or `stage: any`). Add it to the `requires` of `stellantis-automotive-feature-classification`, `stellantis-subagent-classification-loop` (if subagent-relevant), and any operational sub-skill that consumes it. Update `framework-maintenance/README.md` Foundational-skills table. Update the lead skill's Preflight section to load it. Add a dedicated section to this checklist with its trigger-to-edit rules. Add changelog entry. |

A new foundational skill must not duplicate content from an existing one. Before creating, justify in the changelog entry **why** the concern is genuinely cross-cutting and not local to one operational stage.

## Change a contract

| Change                                | Follow-ups                                                                                                                                                                                                                                                                                                                                                          |
| :------------------------------------ | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Modify `lead-to-subagent-contract.md` | Update `automotive-feature-classification/templates/subagent-contract.json.tmpl` schema. Update `lead-agent-subagent-orchestration/SKILL.md` contract construction section. Update `subagent-classification-loop/SKILL.md` identity-discovery section. Add changelog entry. **Breaking changes require a version bump in the contract's** **`contract_version`** **const.** |
| Modify `subagent-to-lead-contract.md` | Update any consumer (lead orchestration skill). Update `automotive-feature-classification/templates/intermediate-parameter-record.json.tmpl` if record shape changes. Add changelog entry.                                                                                                                                                                              |

## Change a template

| Change                                        | Follow-ups                                                                                                                                                                                                                                                    |
| :-------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Template shape changes (fields added/removed) | Update every producer sub-skill and `automotive-feature-classification/workflows/`. Update any consumer sub-skill. Update `automotive-feature-classification/references/file-ownership-matrix.md` if a new file is introduced. Validate the change by hand-building one example matching the new shape. Add changelog entry. |
| Template wording changes only                 | Still log. No consumer updates needed.                                                                                                                                                                                                                        |

## Change a tool allow-list

| Change                              | Follow-ups                                                                                                                                                                                                     |
| :---------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Add a tool to Lead's allow-list     | Update `automotive-feature-classification/SKILL.md` Tool Allow-List. Update any sub-skill that will use it. Confirm the capability-category mapping in `business-logic/07-tools-and-constraints.md`. Add changelog entry. |
| Add a tool to Subagent's allow-list | Same as above + **audit ARCH-4**: if the tool is a web-search or browser-rendering tool, the addition requires a business-logic amendment first.                                                                          |

## Add a new file under `.harness/`

| Change           | Follow-ups                                                                                                                                                                                             |
| :--------------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| New runtime file | Update `stellantis-run-workspace/SKILL.md` layout tree. Update `automotive-feature-classification/references/workspace-layout.md` tree. Update `automotive-feature-classification/references/file-ownership-matrix.md` with the writer/reader roles. Update the producer sub-skill or workflow stage. If the file is a subagent contract or result envelope, document the JSON block schema inline in the producer skill. Add changelog entry. |

## Change a decision rule in business-logic

| Change                                | Follow-ups                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| :------------------------------------ | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Add / change / remove a decision rule | Update `business-logic/10-decision-rules.md`. Update `business-logic/06-harness-interface.md` matrix. Update `automotive-feature-classification/references/enums.md::DecisionRule` if new ids. Update `automotive-feature-classification/templates/intermediate-parameter-record.json.tmpl` `allOf` validator (including the rule-to-confidence and rule-to-traceability constraints). Update `automotive-feature-classification/templates/deliverable.schema.json` `decision_rule` enum. Update `subagent-classification-loop/SKILL.md` rule table. Update `business-logic/09-deliverable.md` cardinality rules if traceability block cardinality changes. Update consolidation merge rules in `stellantis-lead-agent-subagent-orchestration` if the new rule changes promotion / demotion logic. Add changelog entry. |
| Change source-type consensus rules (which categories count as independent, demotion targets, promotion triggers) | Update `business-logic/10-decision-rules.md` *Source-type categories and consensus* section. Update `stellantis-decision-rules/SKILL.md` consensus + demotion sections + invariants 9–10. Update `stellantis-lead-agent-subagent-orchestration` consolidator promotion logic. Update `intermediate-parameter-record.json.tmpl` validators. Add changelog entry. |
| Change inverse-retrieval contract (query templates, when it runs, what flag is recorded) | Update `business-logic/10-decision-rules.md` *Inverse retrieval* section. Update `stellantis-decision-rules/SKILL.md` inverse-retrieval section + invariant 11. Update `stellantis-subagent-classification-loop` and `stellantis-subagent-doc-deep-dive` to honour the new contract. Update `intermediate-parameter-record.json.tmpl` `inverse_retrieval_attempted` field. Add changelog entry. |

## Before landing any change

1. Every referenced file path resolves (no dead links). Sub-skill cross-references use skill names in prose, not relative paths.
2. Every enum value referenced in prose is spelled exactly as in `automotive-feature-classification/references/enums.md`.
3. `change-log.md` has a new entry.
4. At least one example instance (hand-written) validates against any schema you touched.

