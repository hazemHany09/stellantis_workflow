---
name: task-contracts
description: How the lead agent creates parameter contract files and spawns subagents via task_tool. Defines the contract file schema, task injection format, and spawning protocol.
---

# Task Contracts — Lead Agent Protocol

## Contract File

Each parameter gets one contract file in `classification-tasks/`. The lead writes the full file at stage 4 (before spawning). When spawning, the lead **injects the `## Task` section** into the file first, then calls `task_tool`.

### File naming
```
classification-tasks/<parameter_id>-<parameter_name_slug>.md
```
Example: `classification-tasks/P042-wireless-charging.md`

### Full contract file template

```markdown
# Parameter: <optimised_parameter_name>
**Category:** <category>
**Internal ID:** <parameter_id>

## Classification Levels
| Level | Label | Description |
|-------|-------|-------------|
| 1 | Basic | <basic_criterion> |
| 2 | Medium | <medium_criterion> |
| 3 | High | <high_criterion> |

*(Only include levels defined in the reference list. Omit rows with no criterion.)*

<working_directory>
You have access to the same sandbox environment as the parent agent:
- User uploads: `/mnt/user-data/uploads`
- User workspace: `/mnt/user-data/workspace`
- Output files: `/mnt/user-data/outputs`
- Deployment-configured custom mounts may also be available at other absolute container paths
- Treat `/mnt/user-data/workspace` as the default working directory
- Prefer relative paths from the workspace when writing scripts or shell commands
</working_directory>

## Task
**Dataset ID:** <ragflow_dataset_id>
**Car:** <brand> <model> <year> <market> — Full-option trim: <resolved_trim>

Your task: classify this parameter for the car above using the RAGFlow knowledge base.

1. Load your skills (retrieval-queries, classification-rules)
2. Query the KB using the dataset ID above — follow retrieval-queries skill for multi-query strategy
3. Apply classification-rules to determine the verdict
4. Write your verdict into the `## Result` block below — follow the exact JSON schema in classification-rules

Do not output anything outside the `## Result` block. The lead reads the file, not your turn output.

## Result
```json
{
  "parameter_id": "",
  "parameter_name": "",
  "category": "",
  "presence": "",
  "status": "",
  "classification": null,
  "confidence": "",
  "decision_rule": "",
  "evidence_summary": "",
  "sources": [],
  "inverse_retrieval_attempted": false
}
```
```

---

## Spawning via `task_tool`

```python
task_tool(
    description="Classify <parameter_name>",        # 3-5 words
    prompt="Contract file: /mnt/user-data/workspace/classification-tasks/<filename>.md\n\nRead the contract file. Follow the ## Task instructions inside it.",
    subagent_type="parameter-classifier-agent",
    max_turns=None
)
```

**Spawning rules:**
- Always inject `## Task` into the contract file **before** calling `task_tool`
- The `prompt` must include the absolute path to the contract file
- The agent reads the file — do not duplicate contract content in the prompt
- For `parameter-searching-agent`: same protocol, `subagent_type="parameter-searching-agent"`, no dataset_id needed in ## Task
- For `document-downloader-agent`: prompt includes URL + dataset_id directly (no contract file)

---

## Batch Spawning Protocol

Maximum **10 agents per batch**. Do not spawn the next batch until the current batch completes.

```python
# Spawn batch (max 10)
batch = parameters[offset:offset+10]
for param in batch:
    # inject ## Task into contract file
    # call task_tool
# Wait for batch to complete before next batch
```

After each batch: scan completed contract files for results before spawning next batch.

---

## Lead Reads Files, Not Agent Output

The agent's turn output is not the source of truth. After a batch completes:

1. Scan `classification-tasks/*.md` files
2. Read the `## Result` JSON block from each
3. A missing or empty `## Result` block = agent failure → re-spawn that agent

Never parse the agent's turn output string. The file is authoritative.

---

## Identifying No-Information Parameters

After classifier batch completes, scan all contract files:

```
no_info = [f for f in contracts if result["presence"] == "No Information Found"]
```

These are handed to `parameter-searching-agent` in the next batch. The same contract file is reused — the searching agent overwrites the `## Result` block with web-search-based findings.

---

## Subagent Types Reference

| Agent | `subagent_type` | Input |
|-------|-----------------|-------|
| Classifier | `parameter-classifier-agent` | Contract file path |
| Downloader | `document-downloader-agent` | URL + dataset_id in prompt |
| Searcher | `parameter-searching-agent` | Contract file path |
| General purpose | `general-purpose` | Direct instructions |
