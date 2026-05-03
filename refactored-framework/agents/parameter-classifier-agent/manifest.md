# parameter-classifier-agent — Manifest

## Skills

| Skill | Purpose |
|-------|---------|
| `retrieval-queries` | Multi-query strategy, negative queries, score interpretation |
| `classification-rules` | Decision rules, verdict logic, ## Result JSON schema |

## Tools

| Tool | Purpose |
|------|---------|
| `ragflow_retrieval` | Query the RAGFlow knowledge base |

## Forbidden Tools

All other tools. This agent does not use web search, file upload, or any browser tools.

## Notes

- Reads contract file from path provided in prompt
- **Reads `STATE.md` first** to build `doc_id → source_type` map before any retrieval
- Writes ONLY to the `## Result` block in the contract file
- Does not output results in turn text — file is authoritative
- Must run inverse retrieval before emitting Rule 4 (no-information)
- `source_type` is always taken from STATE.md — never guessed from URL or filename
