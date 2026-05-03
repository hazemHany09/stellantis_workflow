# document-downloader-agent — Manifest

## Skills

None. All logic is embedded in the system prompt.

## Tools

| Tool | Purpose |
|------|---------|
| `fetch_url` | Download binary files (PDF, DOCX, XLSX) or scrape webpages |
| `fetch_webpage` | Scrape a public webpage to Markdown |
| `ragflow_upload` | Upload downloaded file to RAGFlow dataset |
| `ragflow_run_document` | Trigger ingestion of uploaded document |

## Notes

- Input: URL + dataset_id (provided directly in prompt, no contract file)
- Output: JSON result in turn text — `{"status": "success", "doc_id": "..."}` or `{"status": "excluded", "reason": "..."}`
- Does not read parameter contract files
- Does not do classification
