# Cloudflare Browser Rendering Tools Available To The Agent

This markdown describes the MCP tools exposed by `Cloudflare_mcp`.

Server name: `cloudflare-browser-rendering`

## What The Agent Can Do

The agent can use this server when it needs a browser-capable retrieval layer instead of a plain HTTP fetch.

Typical uses:

* render JavaScript-heavy pages
* capture screenshots or PDFs
* convert rendered pages to Markdown
* scrape structured content from selectors
* extract JSON from a page
* crawl a site asynchronously

## Accessible Tools

| Tool                      | Purpose                                               |
| ------------------------- | ----------------------------------------------------- |
| `browser_render_content`  | Return fully rendered HTML after JavaScript execution |
| `browser_render_pdf`      | Generate a rendered PDF                               |
| `browser_render_markdown` | Convert a rendered page to Markdown                   |
| `browser_render_scrape`   | Extract selector-based data from a page               |
| `browser_render_links`    | Return links discovered on a page                     |
| `browser_crawl_create`    | Start an asynchronous site crawl job                  |
| `browser_crawl_status`    | Poll crawl progress and results                       |
| `browser_crawl_cancel`    | Cancel a running crawl                                |

## Tool Families

### Single-page rendering

* `browser_render_content`
* `browser_render_pdf`
* `browser_render_markdown`

### Extraction and scraping

* `browser_render_scrape`
* `browser_render_links`

### Multi-page crawl workflow

* `browser_crawl_create`
* `browser_crawl_status`
* `browser_crawl_cancel`

##
