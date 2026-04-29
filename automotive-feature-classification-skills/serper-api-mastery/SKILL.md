---
name: serper-api-mastery
description: |
  Use this skill whenever you need to search the web using Serper API tools. ALWAYS invoke when:
  - Searching for YouTube videos or filtering by a specific YouTube channel
  - Filtering results to a specific domain (site: operator)
  - Filtering results by date or date range
  - Running multi-domain OR queries
  - Planning a chained/multi-step search strategy to decompose complex research
  - Any use of google_search, google_search_news, google_search_videos, google_search_images, or webpage_scrape tools
  Trigger even if the user just says "search for X" — this skill provides critical operator knowledge that turns naive queries into precise, high-signal searches.
---

# Serper API Mastery

You have access to a powerful Serper-backed search suite. This skill teaches you how to use it precisely and intelligently — including advanced operators, date filtering, YouTube channel targeting, and multi-step research chains.

---

## Available Tools

| Tool | Endpoint | Best For |
|------|----------|----------|
| `google_search` | `/search` | General web search — supports all operators + `tbs` date filter |
| `google_search_news` | `/news` | Time-sensitive content, breaking news, press releases |
| `google_search_videos` | `/videos` | Video content discovery (YouTube, Vimeo, etc.) |
| `google_search_images` | `/images` | Image results |
| `google_search_scholar` | `/scholar` | Academic papers, citations, research |
| `google_search_shopping` | `/shopping` | Product listings and prices |
| `google_search_places` | `/places` | Local businesses |
| `google_search_autocomplete` | `/autocomplete` | Discover related queries and terminology |
| `webpage_scrape` | `/scrape` | Read full text of a specific URL |

**Key parameters available on `google_search`:**
- `q` — query string, supports all Google operators
- `tbs` — time-based filter (see Date Filtering section)
- `num` — number of results (1–100, default 10)
- `page` — pagination (default 1)
- `gl` — country code (e.g. `us`, `uk`, `fr`, `eg`)
- `hl` — language code (e.g. `en`, `ar`, `fr`)
- `autocorrect` — set `false` for precise operator queries where autocorrect would corrupt operators

---

## Google Search Operators (embed in `q` field)

These operators work inside the `q` parameter and are passed verbatim to Google.

### Domain & URL Operators

| Operator | Syntax | Effect |
|----------|--------|--------|
| `site:` | `site:domain.com` | Restrict to one domain/subdomain |
| `inurl:` | `inurl:keyword` | URL must contain keyword |
| `intitle:` | `intitle:keyword` | Page title must contain keyword |
| `intext:` | `intext:keyword` | Page body must contain keyword |
| `filetype:` | `filetype:pdf` | Filter by file extension |

### Boolean & Phrase Operators

| Operator | Syntax | Effect |
|----------|--------|--------|
| Exact phrase | `"exact phrase"` | Match substring exactly |
| OR | `term1 OR term2` | Either term must appear |
| Exclude | `-term` | Exclude pages with this term |
| Wildcard | `*` | Matches any word in phrase |
| Grouping | `(term1 OR term2)` | Group boolean logic |

### Combine freely:
```
"climate change" site:nytimes.com OR site:theguardian.com -opinion
```

---

## Date Filtering via `tbs` Parameter

Pass `tbs` as a string to `google_search`. **Do not** put date info in the `q` field — it won't work reliably.

### Relative Date Ranges (quick)

| `tbs` value | Window |
|-------------|--------|
| `qdr:h` | Past hour |
| `qdr:h6` | Past 6 hours |
| `qdr:d` | Past day |
| `qdr:d3` | Past 3 days |
| `qdr:w` | Past week |
| `qdr:m` | Past month |
| `qdr:m3` | Past 3 months |
| `qdr:m6` | Past 6 months |
| `qdr:y` | Past year |
| `qdr:y2` | Past 2 years |

### Custom Date Range (precise)

Format: `cdr:1,cd_min:MM/DD/YYYY,cd_max:MM/DD/YYYY`

```
tbs="cdr:1,cd_min:01/01/2023,cd_max:12/31/2023"
```

### Sort by date (newest first)

Append `,sbd:1` to any `tbs` value:
```
tbs="cdr:1,cd_min:06/01/2024,cd_max:12/31/2024,sbd:1"
tbs="qdr:m,sbd:1"
```

> **Why sort by date matters:** Google defaults to relevance ranking. Adding `sbd:1` surfaces newest articles first, critical for tracking fast-moving topics.

---

## YouTube Channel-Specific Search

YouTube channel searches go through `google_search` (not `google_search_videos`), using `site:` and `inurl:` operators together to scope to a channel.

### YouTube URL patterns

| Channel type | URL pattern |
|--------------|-------------|
| Handle (new) | `youtube.com/@ChannelHandle` |
| Custom URL | `youtube.com/c/ChannelName` |
| User URL | `youtube.com/user/Username` |
| Channel ID | `youtube.com/channel/UCxxxxxxx` |

### Search within a specific channel

```
q="site:youtube.com/@LexFridman AI safety"
```

Or using the channel's videos path:
```
q="site:youtube.com inurl:@LexFridman AI"
```

### YouTube channel + date range

Combine `site:` in `q` with `tbs` for date-scoped channel searches:

```python
google_search(
    q='site:youtube.com/@3blue1brown "neural networks" OR "transformers"',
    tbs="cdr:1,cd_min:01/01/2022,cd_max:12/31/2023",
    num=20
)
```

---

## Multi-Domain OR Searches

Use `OR` to cast across multiple domains in one query. Parentheses group alternatives.

```
(site:youtube.com OR site:medium.com OR site:substack.com) "LLM fine-tuning" 2024
```

Mix domains and YouTube channels:
```
(site:youtube.com/@AnthropicAI OR site:youtube.com/@OpenAI OR site:blog.anthropic.com) Claude
```

> Keep OR-chains to 4–5 domains max. Beyond that, Google may silently drop some clauses.

---

## Chained Search Strategy (Multi-Step Research)

**The core idea:** Before running the precise search you ultimately need, run a *discovery search* first. Use its results to understand terminology, key players, and framing — then build a sharper second query.

### When to chain

Chain searches when:
- The topic is unfamiliar and you don't know the vocabulary experts use
- You need to find specific sources, authors, or channels before targeting them
- The first attempt returns noisy, broad, or off-target results
- The user's original question is vague but the underlying information need is specific

### The two-step pattern

**Step 1 — Discovery query** (broad, exploratory)
- Goal: learn the vocabulary, identify authoritative sources, find channel handles
- Use `autocorrect=true`, `num=10`, no date filter
- Read titles + snippets to extract: terminology, names, channel handles, relevant domains

**Step 2 — Targeted query** (precise, operator-rich)
- Goal: retrieve exactly what you need
- Apply operators learned from step 1: `site:`, `inurl:`, exact phrases, date filters
- Use `autocorrect=false` to prevent operator mangling

> You are the researcher here. Step 1 is for *you* to understand the landscape — not to show the user intermediate noise. Synthesize what you learned and use it silently to construct a superior Step 2 query.

---

## Complex Examples

### Example 1 — YouTube channel + date range

Find videos from a specific YouTube channel published between two dates:

```python
# Lex Fridman podcast episodes mentioning "AI regulation" from 2023 to 2024
google_search(
    q='site:youtube.com/@lexfridman "AI regulation" OR "AI governance" OR "AI policy"',
    tbs="cdr:1,cd_min:01/01/2023,cd_max:12/31/2024,sbd:1",
    num=20,
    autocorrect=False
)
```

---

### Example 2 — Multi-domain search across YouTube + news outlets with date window

Find coverage of a product launch across YouTube channels AND major tech blogs from a specific period:

```python
# Coverage of Claude 3 launch across YouTube + tech press, March–June 2024
google_search(
    q='(site:youtube.com OR site:techcrunch.com OR site:theverge.com OR site:wired.com) "Claude 3" Anthropic',
    tbs="cdr:1,cd_min:03/01/2024,cd_max:06/30/2024,sbd:1",
    num=30,
    autocorrect=False
)
```

---

### Example 3 — YouTube channel + specific domain, complex boolean

Find comparisons of two products from specific sources:

```python
# GPT-4 vs Claude comparison from specific credible channels and sites
google_search(
    q='(site:youtube.com/@yannicdepaepe OR site:youtube.com/@aiexplained OR site:youtube.com/@fireship) ("GPT-4" OR "GPT4") ("Claude 3" OR "Claude Opus") comparison',
    tbs="cdr:1,cd_min:01/01/2024,cd_max:12/31/2024",
    num=20,
    autocorrect=False
)
```

---

### Example 4 — Multi-step chained search (discovery → targeted)

**User goal:** "Find authoritative YouTube tutorials on fine-tuning Llama 3 using QLoRA"

**Step 1 — Discovery:** You don't yet know which YouTube educators cover this, or what terminology they use.

```python
# Step 1: Discover who covers LLM fine-tuning on YouTube
google_search(
    q="youtube channel LLM fine-tuning tutorial QLoRA LoRA Llama 2024",
    num=10
)
```

Step 1 returns results mentioning channels like `@hkproj`, `@brev-dev`, `@unsloth_ai`, `@huggingface`. You now know the authoritative handles and exact terminology ("Unsloth", "PEFT", "bitsandbytes").

**Step 2 — Targeted:** Use what you learned to search with channel-level precision:

```python
# Step 2: Targeted search on the most relevant channels from a specific window
google_search(
    q='(site:youtube.com/@hkproj OR site:youtube.com/@brev-dev OR site:youtube.com/@unsloth_ai) "QLoRA" "Llama 3" fine-tuning',
    tbs="cdr:1,cd_min:04/01/2024,cd_max:12/31/2024,sbd:1",
    num=20,
    autocorrect=False
)
```

> **Why this works:** The first query is cheap and broad — its purpose is to *educate you*, not the user. The second query is surgical. Without step 1, you would have guessed channel names incorrectly, used imprecise vocabulary, and returned poor results.

---

### Example 5 — Chained search with autocomplete for vocabulary discovery

When unsure of terminology, use `google_search_autocomplete` as step 1:

```python
# Step 1: Discover vocabulary around the topic
google_search_autocomplete(q="quantization LLM methods")
# Returns: "quantization LLM methods GGUF", "AWQ vs GPTQ", "4-bit quantization llama.cpp"

# Step 2: Now search with the precise vocabulary found
google_search(
    q='(site:youtube.com OR site:huggingface.co/blog) "AWQ" OR "GPTQ" OR "GGUF" quantization comparison "2024"',
    tbs="cdr:1,cd_min:01/01/2024,cd_max:12/31/2024",
    num=20,
    autocorrect=False
)
```

---

## Decision Guide: Which Tool to Use

```
User wants videos?
  → Yes: Use google_search_videos for discovery; google_search with site:youtube.com + operators for channel-specific
  → No: Continue...

Need date filtering?
  → Yes: google_search with tbs= parameter
  → No: Continue...

Need academic sources?
  → Yes: google_search_scholar
  → No: Continue...

Need full page content?
  → Yes: webpage_scrape after finding URL via google_search
  → No: google_search for snippets
```

---

## Common Mistakes to Avoid

| Mistake | Why it fails | Fix |
|---------|-------------|-----|
| Putting dates in `q` field | Google ignores natural language dates in queries | Use `tbs` parameter instead |
| `autocorrect=true` with operators | Google "fixes" `site:` or `inurl:` and breaks them | Set `autocorrect=False` when using operators |
| OR without spaces | `termAORtermB` treated as one term | Always `term1 OR term2` with spaces |
| Too many OR-chained sites | Google silently drops clauses | Max 4–5 `site:` in one OR chain |
| Using `google_search_videos` for channel-specific search | Returns YouTube search results, not channel-scoped | Use `google_search` with `site:youtube.com/@handle` |
| Skipping discovery on unfamiliar topics | Guessing vocabulary → irrelevant results | Run step 1 autocomplete or broad search first |
