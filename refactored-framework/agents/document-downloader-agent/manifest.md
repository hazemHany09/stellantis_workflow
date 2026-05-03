# document-downloader-agent — Manifest

## Skills

None. All logic is embedded in the system prompt.

## Tools

| Tool | Purpose |
|------|---------|
| `fetch_url` | Download binary files (PDF, DOCX, XLSX) or scrape webpages |
| `fetch_webpage` | Scrape a public webpage to Markdown |
| `upload_with_metadata` | Upload downloaded file to RAGFlow dataset with `source_type` metadata |
| `ragflow_run_document` | Trigger ingestion of uploaded document |

## Notes

- Input: URL + dataset_id + source_type (provided directly in prompt, no contract file)
- Output: JSON result in turn text — `{"status": "success", "doc_id": "...", "source_type": "..."}` or `{"status": "excluded", "reason": "..."}`
- Stores `source_type` in RAGFlow document metadata via `upload_with_metadata` so that the classifier agent can resolve authority per document
- Does not read parameter contract files
- Does not do classification
