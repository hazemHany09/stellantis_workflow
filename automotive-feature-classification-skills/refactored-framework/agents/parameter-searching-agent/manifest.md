# parameter-searching-agent — Manifest

## Skills

| Skill | Purpose |
|-------|---------|
| `serper` | Web search query strategy and tool usage |
| `classification-rules` | Decision rules, verdict logic, ## Result JSON schema |

## Tools

| Tool | Purpose |
|------|---------|
| `serper_google_search` | Search the web for feature evidence |

## Forbidden Tools

RAGFlow retrieval tools (this agent uses web, not KB). File upload tools. Browser rendering tools.

## Notes

- Reads contract file from path provided in prompt
- Writes ONLY to the `## Result` block in the contract file
- Appends `## Web Sources` section listing URLs used as evidence
- Does not output results in turn text — file is authoritative
- Invoked only for parameters where KB retrieval returned no-information
