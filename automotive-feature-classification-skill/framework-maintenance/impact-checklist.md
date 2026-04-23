# Impact Checklist

When you make a change of the kind listed in the leftmost column, apply **every** item in the right column before landing it. Omissions are bugs.

## Add a new value to an enum

| Change                                              | Follow-ups                                                                                                                                                                                                                                                                                                                                                                                                                     |
| :-------------------------------------------------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Add `Presence` value                                | Update `src/references/enums.md`. Update `src/templates/intermediate-parameter-record.json.tmpl` `enum`. Update `src/templates/deliverable.schema.json` `enum` inside `records[].presence`. Update `business-logic/10-decision-rules.md` (add a new rule). Update `business-logic/06-harness-interface.md` invariants + matrix. Update `src/skills/subagent-classification-loop/SKILL.md` decision table. Add changelog entry. |
| Add `Status` value                                  | Same pattern as Presence.                                                                                                                                                                                                                                                                                                                                                                                                      |
| Add `Classification` value                          | Same pattern as Presence + update `SchemaType` section of enums if it implies a new tier.                                                                                                                                                                                                                                                                                                                                      |
| Add `RunStage`, `RunStatus`, or `PauseReason` value | Update `enums.md`. Update `src/templates/STATE.main.md.tmpl`. Update `src/SKILL.md` §7 pause-reason table. Update any workflow file that emits the new value. Add changelog entry.                                                                                                                                                                                                                                             |
| Add `SourceLifecycle` value                         | Update `enums.md`. Update `src/skills/source-download-and-ingest/SKILL.md` decision tree. Update `business-logic/12-workflow-diagram.md` section B. Add changelog entry.                                                                                                                                                                                                                                                       |
| Add a `DecisionRule` value (e.g. Rule-6)            | Update `enums.md`. Update `business-logic/10-decision-rules.md`. Update `business-logic/06-harness-interface.md` matrix. Update `src/templates/intermediate-parameter-record.json.tmpl` `allOf` matrix. Update `src/templates/deliverable.schema.json` `decision_rule` enum. Update `src/skills/subagent-classification-loop/SKILL.md`. Add changelog entry.                                                                   |

## Add a new workflow

| Change                                   | Follow-ups                                                                                                                                                                                                                 |
| :--------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| New workflow (e.g. W2 Mode B standalone) | Create `src/workflows/<id>/WORKFLOW.md`. Link it from `src/SKILL.md` §5 workflow load rules. Add a section to `business-logic/13-potential-workflows.md`. Add an entry to the decision tree in §12.G. Add changelog entry. |

## Add a new sub-skill

| Change        | Follow-ups                                                                                                                                                                                                                            |
| :------------ | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| New sub-skill | Create `src/skills/<name>/SKILL.md` with `name`, `description`, `loader` front-matter. Declare its tools in both its header and `src/SKILL.md` §9. Reference it from at least one workflow or another sub-skill. Add changelog entry. |

## Change a contract

| Change                                | Follow-ups                                                                                                                                                                                                                                                                                                                                                          |
| :------------------------------------ | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Modify `lead-to-subagent-contract.md` | Update `src/templates/subagent-contract.json.tmpl` schema. Update `src/skills/lead-agent-subagent-orchestration/SKILL.md` contract construction section. Update `src/skills/subagent-classification-loop/SKILL.md` identity-discovery section. Add changelog entry. **Breaking changes require a version bump in the contract's** **`contract_version`** **const.** |
| Modify `subagent-to-lead-contract.md` | Update any consumer (lead orchestration skill). Update `src/templates/intermediate-parameter-record.json.tmpl` if record shape changes. Add changelog entry.                                                                                                                                                                                                        |

## Change a template

| Change                                        | Follow-ups                                                                                                                                                                                                                                                    |
| :-------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Template shape changes (fields added/removed) | Update every producer in `src/skills/` and `src/workflows/`. Update any consumer. Update `src/references/file-ownership-matrix.md` if a new file is introduced. Validate the change by hand-building one example matching the new shape. Add changelog entry. |
| Template wording changes only                 | Still log. No consumer updates needed.                                                                                                                                                                                                                        |

## Change a tool allow-list

| Change                              | Follow-ups                                                                                                                                                                                                     |
| :---------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Add a tool to Lead's allow-list     | Update `src/SKILL.md` §9. Update any sub-skill that will use it. If the tool is in `tools/*.md`, confirm the capability-category mapping in `business-logic/07-tools-and-constraints.md`. Add changelog entry. |
| Add a tool to Subagent's allow-list | Same as above + **audit ARCH-4**: if the tool is a web-search or browser-rendering tool, the addition requires a business-logic amendment first.                                                               |

## Add a new file under `runs/<run-id>/`

| Change           | Follow-ups                                                                                                                                                                                             |
| :--------------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| New runtime file | Update `src/references/workspace-layout.md` tree. Update `src/references/file-ownership-matrix.md` with the writer/reader roles. Update the producer sub-skill or workflow stage. Add changelog entry. |

## Change a decision rule in business-logic

| Change                                | Follow-ups                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| :------------------------------------ | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Add / change / remove a decision rule | Update `business-logic/10-decision-rules.md`. Update `business-logic/06-harness-interface.md` matrix. Update `src/references/enums.md::DecisionRule` if new ids. Update `src/templates/intermediate-parameter-record.json.tmpl` `allOf` validator. Update `src/templates/deliverable.schema.json` `decision_rule` enum. Update `src/skills/subagent-classification-loop/SKILL.md` rule table. Update `business-logic/09-deliverable.md` cardinality rules if traceability block cardinality changes. Add changelog entry. |

## Before landing any change

1. Every referenced file path resolves (no dead links).
2. Every enum value referenced in prose is spelled exactly as in `src/references/enums.md`.
3. `change-log.md` has a new entry.
4. At least one example instance (hand-written) validates against any schema you touched.

