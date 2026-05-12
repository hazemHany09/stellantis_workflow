# document-downloader-agent — Manifest

## Skills

None. All logic is embedded in the system prompt.

## Tools

| Tool | Purpose |
|------|---------|
| `fetch_url` | Download binary files (PDF, DOCX, XLSX) or scrape webpages |
| `fetch_webpage` | Scrape a public webpage to Markdown |
| `ingest_video_with_metadata` | Upload a YouTube video URL directly to a RAGFlow dataset (bypasses local download) |
| `upload_with_metadata` | Upload a locally downloaded file to RAGFlow dataset with a full metadata payload |
| `ragflow_run_document` | Trigger ingestion of uploaded document |

## Notes

- Input: URL + dataset_id + source_type (provided directly in prompt, no contract file)
- Output: JSON result in turn text — `{"status": "success", "doc_id": "...", "source_type": "..."}` or `{"status": "excluded", "reason": "..."}`
- Stores `source_type` in RAGFlow document metadata so that the classifier agent can resolve authority per document
- YouTube video URLs (youtube.com/watch, youtu.be, youtube.com/shorts) are handled via `ingest_video_with_metadata` — no local download required; `doc_type` is set to `video`
- Does not read parameter contract files
- Does not do classification
