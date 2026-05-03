---
name: retrieval-queries
description: Best practices for RAGFlow KB retrieval — multi-query strategy, negative queries, score interpretation, and escalation to no-information. Used by parameter-classifier-agent.
---

# RAGFlow Retrieval Query Strategy

## Core Rule: One Query Set Per Parameter

Never batch multiple parameters into one retrieval call. Vector search optimises for the centre of gravity of the query — loud terms swamp quieter ones and push specific evidence below the cut-off. Run each parameter independently.

---

## Query Construction

For each parameter, generate **3–5 queries** before issuing any retrieval call:

**Query 1 — Optimised name (primary)**
Use the parameter's exact optimised name as the core term. Anchor to the resolved trim.
```
"<optimised_parameter_name>" <brand> <model> <resolved_trim>
```

**Query 2 — Alias / paraphrase**
Use alternative industry phrasings for the same feature. Example: "Wireless CarPlay" → "Wireless Apple CarPlay".
```
"<alias>" <brand> <model> <year>
```

**Query 3 — Description terms**
Pull 2–3 key terms from the parameter's description, not the name.
```
<term1> <term2> <brand> <model> <market>
```

**Query 4 — Level criteria terms** (when prior queries return vague evidence)
Use the specific language from the classification level definitions.
```
"<Basic level criterion phrase>" <brand> <model>
```

**Query 5 — Inverse / negative** (mandatory before emitting no-information)
Target language of absence, exclusion, or removal:
```
"<feature_name>" "not available" <resolved_trim>
"<feature_name>" "deleted" OR "omits" OR "lacks"
"features not available on <resolved_trim>"
```

Run inverse queries even when positive retrieval returned nothing. A negative mention is a clear source (Rule 5); silence after inverse pass confirms Rule 4.

---

## Retrieval Parameters

- **top_k**: Start at 6. Raise to 12 in fallback loop.
- **Mode**: Use hybrid mode if available (keyword + semantic). Default to semantic-only otherwise.
- **Knowledge graph**: Use `retrieve_knowledge_graph` when parameters cross-reference entities (e.g. a package that bundles multiple features). Rare — default to `retrieval`.

---

## Reading Retrieved Chunks

For each returned chunk, classify the source's relationship to the parameter:

| Classification | Criteria |
|----------------|----------|
| **Clear** | Mentions the parameter AND gives schema-mappable information (level, presence, or explicit absence) |
| **Vague** | Mentions the parameter but cannot be mapped to a level or clear presence/absence |
| **Silent** | Does not mention the parameter — discard |

A source is **active** if it is clear or vague. Silent sources count for nothing.

---

## Score Interpretation

High chunk score ≠ clear evidence. Always read the chunk text:
- A high-score chunk may mention the parameter in passing (vague) or in a different context (silent)
- A mid-score chunk may contain the exact level criterion language you need (clear)

Do not filter chunks by score alone. Read the top 6 chunks in full.

---

## Fallback Loop

If no parameter finalises (Rule 1, 2, or 5) after the main query set:

1. Rebuild queries with different phrasing (use criterion text, not feature name)
2. Raise top_k to 12
3. Run inverse queries if not yet run
4. If still unresolved: apply Rule 3 (vague evidence exists) or Rule 4 (no evidence at all)

---

## When to Escalate to No-Information (Rule 4)

Escalate only after **all** of these:
- All 3–5 primary queries returned no active sources
- Inverse retrieval pass ran and returned no negative evidence
- Fallback loop ran with raised top_k

If any of these steps was skipped, do not emit Rule 4. Run the missing step first.

Every Rule 4 record **must** include `"inverse_retrieval_attempted": true` in the result JSON.
