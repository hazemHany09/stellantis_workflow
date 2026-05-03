---
name: serper
description: Use whenever you need to search the web with serper. Provides operator knowledge for precise, high-signal searches using google_search and related tools.
---

# Serper Web Search

You have access to a Serper-backed search suite. Use it precisely — advanced operators, date filtering, and chained searches dramatically improve result quality.

---

## Available Tools

| Tool | Best For |
|------|----------|
| `google_search` | General web search — supports all operators + `tbs` date filter |
| `google_search_news` | Time-sensitive content, press releases |
| `google_search_autocomplete` | Discover related queries and terminology |
| `webpage_scrape` | Read full text of a specific URL |

**Key `google_search` parameters:**
- `q` — query string, supports all Google operators
- `tbs` — time-based filter
- `num` — number of results (1–100, default 10)
- `gl` — country code (e.g. `us`, `uk`, `fr`)
- `hl` — language code (e.g. `en`, `fr`)
- `autocorrect` — set `false` when using operators

---

## Google Search Operators

### Domain & URL
| Operator | Syntax | Effect |
|----------|--------|--------|
| `site:` | `site:domain.com` | Restrict to one domain |
| `inurl:` | `inurl:keyword` | URL must contain keyword |
| `intitle:` | `intitle:keyword` | Title must contain keyword |
| `filetype:` | `filetype:pdf` | Filter by file type |

### Boolean & Phrase
| Operator | Syntax | Effect |
|----------|--------|--------|
| Exact phrase | `"exact phrase"` | Match exactly |
| OR | `term1 OR term2` | Either term |
| Exclude | `-term` | Exclude pages with this term |
| Grouping | `(term1 OR term2)` | Group boolean logic |

---

## Date Filtering via `tbs`

Never put dates in `q` — use `tbs` instead.

| `tbs` value | Window |
|-------------|--------|
| `qdr:m` | Past month |
| `qdr:m6` | Past 6 months |
| `qdr:y` | Past year |
| `qdr:y2` | Past 2 years |
| `cdr:1,cd_min:MM/DD/YYYY,cd_max:MM/DD/YYYY` | Custom range |

Append `,sbd:1` to sort by date (newest first).

---

## Chained Search Strategy

**When the topic is unfamiliar, chain searches:**

**Step 1 — Discovery** (broad): learn vocabulary, identify sources.
```python
google_search(q="topic keywords", num=10)
```

**Step 2 — Targeted** (precise): use what you learned.
```python
google_search(
    q='site:specific-source.com "exact phrase" -unwanted',
    autocorrect=False,
    num=20
)
```

**Use `google_search_autocomplete` as step 1** when unsure of terminology:
```python
google_search_autocomplete(q="your topic")
# Returns related queries → use these terms in step 2
```

---

## Multi-Domain OR Searches

```
(site:site1.com OR site:site2.com OR site:site3.com) "keyword"
```

Keep OR-chains to 4–5 domains max — Google silently drops extras.

---

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Dates in `q` field | Use `tbs` parameter |
| `autocorrect=true` with operators | Set `autocorrect=False` |
| OR without spaces | Always `term1 OR term2` with spaces |
| Too many OR-chained sites | Max 4–5 per query |
| Skipping discovery on unfamiliar topics | Run step 1 first |

---

## Decision Guide

```
Need full page content? → webpage_scrape after finding URL
Need date filtering? → google_search with tbs=
Need academic sources? → google_search_scholar
Unfamiliar topic? → google_search_autocomplete first, then targeted search
Default → google_search
```
