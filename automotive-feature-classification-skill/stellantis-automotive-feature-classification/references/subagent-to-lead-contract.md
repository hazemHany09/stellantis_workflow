# Subagent → Lead Result Envelope

The structured result the subagent writes at the bottom of its `.harness/SubAgent/<agent-name>.md` before returning control. The lead reads this block to consolidate.

## Shape

```json
{
  "envelope_version": "1.0",
  "agent_name": "<agent-name>",
  "run_id": "<run-id>",
  "category": "<category>",
  "partition_index": "<integer>",
  "status": "<SubagentStatus; expected: consolidated-ready or failed>",
  "completed_at": "<ISO-8601 UTC>",
  "loop_stats": {
    "main_loop_iterations_used": "<integer, 0..3>",
    "fallback_loop_iterations_used": "<integer, 0..3>",
    "retrieval_calls": "<integer>",
    "chunks_downloaded": "<integer>",
    "docs_touched": ["<doc_id>"]
  },
  "records": [
    { "...": "see src/templates/intermediate-parameter-record.json.tmpl" }
  ],
  "docs_metadata_updates": [
    {
      "doc_id": "<string>",
      "metadata_patch": {
        "parameters_found":       ["<parameter id>"],
        "parameters_classified":  ["<parameter id>"],
        "presence_Yes":           ["<parameter id>"],
        "presence_No":            ["<parameter id>"],
        "classification_High":    ["<parameter id>"],
        "classification_Medium":  ["<parameter id>"],
        "classification_Basic":   ["<parameter id>"]
      }
    }
  ],
  "warnings": [
    {
      "scope": "<parameter | category | run>",
      "parameter_id": "<id or null>",
      "message": "<short prose>"
    }
  ],
  "advisories": [
    {
      "kind": "<undefined_tier | out_of_list>",
      "parameter_id": "<id or null>",
      "evidence_doc_ids": ["<doc_id>"],
      "message": "<short prose>"
    }
  ],
  "loaded_skills": [
    "<skill-name-1>",
    "<skill-name-2>"
  ],
  "notes": "<free-form markdown>"
}
```

## Field notes

* **`status`** — must be `consolidated-ready` when all records are present and valid; `failed` when the subagent exhausted both budgets without producing a valid record for every parameter in its contract.
* **`records`** — exactly one entry per `parameter_id` in the inbound contract. Missing entries are a contract violation.
* **`docs_metadata_updates`** — patches the lead applies via `batch_update_doc_metadata`. The subagent does **not** call the metadata tool directly for final updates; it stages the patches here so the lead can batch across subagents. (The subagent may still call `set_doc_metadata` for intermediate working-memory metadata — e.g. tagging a doc as "read".)
* **`warnings`** — promoted into the category STATE by the lead during consolidation.
* **`advisories`** — promoted into `.harness/advisories/` by the lead; never into the deliverable.
* **`loaded_skills`** — array of skill names (as strings) that the subagent loaded during its run. Used by the lead for consolidation into main STATE.md. Example: `["stellantis-domain-context", "stellantis-decision-rules", "stellantis-subagent-classification-loop"]`.

## Writing rule

The envelope is emitted at the tail of the working file, in a single fenced block:

````markdown
...subagent scratch above...

<!-- RESULT ENVELOPE — LEAD WILL CONSUME THIS -->
```json
{ ...envelope json... }
```
<!-- END RESULT ENVELOPE -->
````

After writing this block, the subagent returns control. It must not touch the file again.
