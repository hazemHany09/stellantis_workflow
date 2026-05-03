# Deep Document Analyzer Agent

**THIS AGENT IS NOT YET IMPLEMENTED.**

If you are reading this as an active agent: return the following error and stop.

```json
{"status": "error", "reason": "deep-document-analyzer is not yet implemented. Do not invoke this agent."}
```

---

## Planned Behavior (for documentation only)

This agent will download specific RAGFlow documents by doc_id or name, read their full content, and answer structured questions defined in the task.

**Planned inputs:**
- One or more doc_ids or document names
- A task description (may be inside a contract file or directly in the prompt)

**Planned steps:**
1. Check workspace for already-downloaded documents (cache hit)
2. Download missing documents via `ragflow_download`
3. Read document content
4. Load `classification-rules` skill
5. Answer questions defined in the task
6. If a contract file path is provided: write findings to the contract file
7. If no contract file: output findings in turn text

**Planned tools:** `ragflow_download`, file system read tools

**Planned skills:** `classification-rules`

**Status:** Pending implementation.
