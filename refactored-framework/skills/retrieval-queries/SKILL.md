---
name: retrieval-queries
description: Best practices for knowledge retrieval — multi-query strategy, negative queries, score interpretation, and escalation to no-information.
---

# RAGFlow Retrieval Query Strategy

## Core Rule: One Query Set Per Parameter

Never batch multiple parameters into one retrieval call. Vector search optimises for the centre of gravity of the query — loud terms swamp quieter ones and push specific evidence below the cut-off. Run each parameter independently.

You may run up to **10 queries per parameter**. Keep searching until you reach consensus (a clear, schema-mappable source) or exhaust all 10 slots. Adjust `top_k` between queries to surface different chunks and generate new query angles.

---

## Query Construction

For each parameter, run up to **10 queries total**. After each query, assess what you know and what gaps remain, then choose the most appropriate next query type. There is no fixed order beyond starting with the optimised name — every subsequent query is selected based on the current state of evidence.

---

### Query Types (use in any order after Query 1)

**Optimised name (primary — always run first)**
Use the parameter's exact optimised name as the core term. Anchor to the resolved trim.
```
"<optimised_parameter_name>" <brand> <model> <resolved_trim>
```

**Alias / paraphrase**
Use when the primary query returns sparse results or when you suspect the feature is named differently in sources. Use alternative industry phrasings.
```
"<alias>" <brand> <model> <year>
```

**Description terms**
Use when name-based queries return silent or off-topic chunks. Pull 2–3 key terms from the parameter's description, not the name.
```
<term1> <term2> <brand> <model> <market>
```

**Level criteria terms**
Use when evidence is vague — you have mentions but cannot map to a level. Use the specific language from the classification level definitions.
```
"<Basic level criterion phrase>" <brand> <model>
```

**Inverse / negative**
Use when: positive queries return nothing, evidence is ambiguous and absence is plausible, or you are approaching the end of the budget. Must be run at least once before emitting no-information (Rule 4).
```
"<feature_name>" "not available" <resolved_trim>
"<feature_name>" "deleted" OR "omits" OR "lacks"
"features not available on <resolved_trim>"
"<feature_name>" "excluded" OR "removed" OR "no longer"
"<alias>" "not included" <brand> <model> <resolved_trim>
```

---

### Decision Logic After Each Query

After each retrieval result, assess the current evidence state and choose the next query accordingly:
- **Clear, schema-mappable evidence found?** → Stop querying. Finalise the parameter.
- **Evidence is vague (mentioned but unmappable)?** → Run a level-criteria or alias query to sharpen the signal.
- **No evidence yet?** → Try a different query type (description terms, alias, or inverse).
- **Absence is plausible?** → Run an inverse query now, not only at the end.
- **Budget exhausted (10 queries)?** → Finalise the parameter. based on the classification rules → *Retrieval Prerequisites Before Applying Unable-to-Decide Rules* for the gates that must have been met before emitting Rule 3 or Rule 4.

Vary `top_k` (between 6 and 12) when switching query types to surface different chunks.

---

## Retrieval Parameters

- **top_k**: Start at 6. Vary between 6 and 12 across queries to surface different chunks — a change in top_k can expose evidence that earlier queries missed.
- **Query budget**: Maximum **10 queries per parameter**. Track your count and allocate remaining slots between positive rephrases and inverse variants.
- **Mode**: Use hybrid mode if available (keyword + semantic). Default to semantic-only otherwise.
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

If the parameter is still unresolved and budget remains within the 10-query cap:

1. Rebuild queries with different phrasing (use criterion text or terminology from retrieved chunks, not the feature name)
2. Toggle top_k (if you used 6, try 12, and vice versa)
3. Run an inverse query if not yet run
4. Continue until consensus is reached or the budget is exhausted

When the budget is exhausted, finalise the parameter. For the gates that must be met before emitting an Unable-to-Decide verdict, see classification rules. → *Retrieval Prerequisites Before Applying Unable-to-Decide Rules*.
