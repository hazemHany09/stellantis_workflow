---
name: classification-rules
description: Decision rules for converting evidence into Presence/Status/Classification verdicts. Defines the output schema for the ## Result block. Used by lead agent, parameter-classifier-agent, and parameter-searching-agent.
---

# Classification Rules

## Vocabulary

| Term | Meaning |
|------|---------|
| **Parameter** | One feature row from the reference list |
| **Level** | Implementation tier: `Basic`, `Medium`, `High` |
| **Applicable level set** | The non-empty subset of `{Basic, Medium, High}` defined for this parameter — never assign a level outside this set |
| **Clear source** | Mentions the parameter AND gives schema-mappable information (level or explicit presence/absence) |
| **Vague source** | Mentions the parameter but cannot be mapped to a level or clear stance |
| **Silent source** | Does not mention the parameter — discard entirely |
| **Active source** | Any source that is clear or vague (not silent) |

**Level hierarchy is cumulative:** High implies Medium implies Basic. A source confirming High simultaneously confirms Medium and Basic — do not require separate evidence for lower levels.

---

## Evidence Source Types

| `source_type` | What it is | Authority for |
|----------------|------------|---------------|
| `manufacturer_official_spec` | OEM build-and-price, configurator, brochure, press kit | Trim availability, standard vs optional |
| `manufacturer_owner_manual` | Owner's manual, user guide | How the feature works; rarely authoritative for trim availability |
| `dealer_fleet_guide` | Fleet/dealer order guide | Standard vs optional matrix per trim |
| `third_party_press_long_form` | Long-form reviews (MotorTrend, Car and Driver, etc.) | Real-world behaviour, performance |
| `third_party_aggregator` | Edmunds, KBB, Cars.com | Trim availability matrices |
| `manufacturer_video` | OEM YouTube walkthroughs | How the feature works |
| `forum_or_ugc` | Owner forums, Reddit | Soft signal only — never sole authority for a Success verdict |

---

## The Five Decision Rules

| Rule | Evidence Situation | Presence | Status | Classification |
|------|--------------------|----------|--------|----------------|
| **Rule 1** | ≥1 clear source, all clear agree on same valid level | `Yes` | `Success` | the agreed level |
| **Rule 5** | ≥1 clear source, all clear agree feature is absent | `No` | `Success` | `Empty` |
| **Rule 2a** | ≥2 clear sources present, disagree on level | `Yes` | `Conflict` | `Empty` |
| **Rule 2b** | ≥2 clear sources disagree on presence vs absence | `Disputed` | `Conflict` | `Empty` |
| **Rule 3** | Active sources exist, but none are clear (all vague) | `Yes` | `Unable to Decide` | `Empty` |
| **Rule 4** | No active sources at all (every source silent) | `No Information Found` | `Unable to Decide` | `Empty` |

**Every record matches exactly one row. No exceptions.**

---

## Confidence Annotation

Every record carries a `confidence` field, set deterministically:

| Situation | `confidence` |
|-----------|-------------|
| Rule 1 / Rule 5: ≥2 clear sources from distinct `source_type` categories agree | `consensus` |
| Rule 1 / Rule 5: all clear sources share the same `source_type` category | `single-source` |
| Rule 1 / Rule 5: the only clear source is `forum_or_ugc` | Demote to Rule 3, `vague-only` |
| Rule 3: vague-only evidence | `vague-only` |
| Rule 4: silent-all | `silent-all` |
| Rule 2a / Rule 2b: ≥2 clear sources from distinct `source_type` categories | `consensus` |
| Rule 2a / Rule 2b: all clear sources share a single `source_type` category | `single-source` (also flag advisory `same-source-type-conflict`) |

### Promotion rule

If a parameter starts a cycle at `Rule 1` / `Rule 5` with `confidence = single-source`surfaces a clear corroborating source from a **different** `source_type` category, re-emit the parameter at the same rule with `confidence = consensus`. Promotion is unconditional — no human gate.

### Demotion rule

If the only clear source backing a Rule 1 / Rule 5 verdict is `forum_or_ugc`, demote the verdict to **Rule 3** (`Yes, Unable to Decide, Empty`) with `confidence = vague-only`. Forum claims alone never produce `Success` — there is no editorial accountability.

---

## Invariants

1. Silent sources never count toward any rule.
2. Vague sources alone never produce `Success` or `Conflict` — only `Unable to Decide`.
3. A single uncontradicted clear source is sufficient for Rule 1 (`single-source` confidence).
4. `Classification` is non-empty **only** under Rule 1.
5. `Disputed` is exclusive to Rule 2b. `No Information Found` is exclusive to Rule 4.
6. Every emitted record matches exactly one matrix row.
7. Out-of-schema evidence (a level not in the parameter's applicable level set) is filtered — never snapped to nearest.
8. `forum_or_ugc` alone never produces Success — demote to Rule 3.
9. Every Rule 4 record requires `inverse_retrieval_attempted: true`.
10. `No Information Found` is a `presence` value **only** — never a `status` value.
11. Out-of-schema evidence (a level the parameter's applicable level set does not define) is **disregarded** for classification — never snapped to the nearest defined level. If valid-level clear evidence still exists, apply Rule 1 (or Rule 2a if multiple valid-level sources still conflict). If no valid-level evidence remains, apply Rule 3.
---

## Retrieval Prerequisites Before Applying Unable-to-Decide Rules

Rule 3 and Rule 4 are last-resort verdicts. Before applying either, confirm the following gates were met during retrieval:

**Before applying Rule 3** (vague-only evidence):
- At least one query was run using level-criteria terms to attempt to sharpen vague evidence
- An alias or description-terms query was run to ensure the feature wasn't named differently in sources

**Before applying Rule 4** (no active sources at all):
- All viable positive query types were attempted (optimised name, alias, description terms)
- At least one inverse / negative query was run and also returned no active sources
- top_k was varied (e.g. tried both 6 and 12) across queries
- The query budget is exhausted or all viable query angles are genuinely exhausted

If any gate was skipped, do not emit the verdict — run the missing query first.

Every Rule 4 record **must** set `"inverse_retrieval_attempted": true` in the Result JSON.

### Inverse-retrieval query templates

Standard retrieval is biased toward presence: a search for "Night Vision" surfaces docs that mention the term, not docs that explicitly state it is unavailable. The inverse pass targets the language of negation, exclusion, and removal. Run **at least one** of these (1–2 queries per parameter) before settling at Rule 4:

- `"<feature_name>" "not available"`
- `"<feature_name>" "deleted"`
- `"<feature_name>" "no longer offered"`
- `"<feature_name>" "<top_trim>" "exclusion"`
- `"features not available on <top_trim>"`
- `"<top_trim>" "deletes" OR "omits" OR "lacks"`

Inspect top chunks. If any explicitly states the feature is absent on the resolved trim, **promote the verdict to Rule 5** (`No, Success, Empty`) and record the chunk as a clear-source traceability block with `stance: absent`. If the inverse pass produces no negative evidence either, the verdict stays at Rule 4 and the record carries `inverse_retrieval_attempted: true`.

---

## Edge Cases

- **Single clear + silent others.** Silent sources don't count. The lone clear source carries Rule 1 (or Rule 5 if its statement is "absent").
- **Mixed conflict — valid level vs out-of-schema tier.** A clear source describing a level in the parameter's applicable set, alongside a clear source describing a tier the schema does not define: **disregard the out-of-schema source** for classification. Apply Rule 1 (or Rule 2a if multiple valid-level sources still conflict). Record an internal advisory noting that out-of-schema evidence was observed.
- **All active sources support an out-of-schema tier only.** No valid-level evidence exists. Apply **Rule 3** (`Yes, Unable to Decide, Empty`).
- **Same-source-type conflict.** If a Rule 2a or Rule 2b verdict is backed only by clear sources sharing one `source_type` category, set `confidence = single-source` and add an advisory `same-source-type-conflict` to the evidence summary. Surface it for human review.
- **Vague + clear coexist under Rule 1 / Rule 5.** Vague blocks may appear alongside the clear block(s); they are not a conflict.

---

## Common Pitfalls

- **Source contradicts another?** Check source types — manufacturer spec outranks press review. Apply the hierarchy: official spec > dealer/fleet guide > long-form press > aggregator > UGC.
- **Feature missing from all sources?** Check if it is market-specific or trim-specific before concluding absence.
- **Level criteria ambiguous?** Default to the most conservative (lower) level — do not upgrade without clear evidence.
- **Silence ≠ absence.** Rule 4 ("no evidence found") ≠ Rule 5 ("evidence says absent"). Different verdicts.
- **One clear source is enough.** Do not require corroboration for Rule 1. but additional queries have to be made just to make sure that it isn't contradiciton, and make sure that it isn't not consensus.
- **Vague + clear = not a conflict.** Only clear sources determine Success/Conflict. Vague sources only matter when no clear source exists.
- **Don't snap to nearest level.** If a source describes a level not in the parameter's schema, discard it. Apply Rule 3.
- **`Conflict` never produces a level.** Emit Conflict and let the human adjudicate.

---

## Result Block Schema

Every parameter contract file has a `## Result` section. The agent writes this exact JSON structure:

```json
{
  "parameter_id": "",
  "parameter_name": "",
  "category": "",
  "presence": "",
  "status": "",
  "classification": null,
  "confidence": "",
  "decision_rule": "",
  "evidence_summary": "",
  "sources": [
    {
      "url": "",
      "source_type": "",
      "kind": "",
      "stance": "",
      "note": ""
    }
  ],
  "inverse_retrieval_attempted": false,
  "traceability": [
    {
      "chunk_id": "",
      "doc_id": "",
      "source_type": "",
      "quote": "",
      "kind": ""
    }
  ]
}
```

**Field rules:**
- `presence` ∈ `{Yes, No, Disputed, No Information Found}`
- `status` ∈ `{Success, Conflict, Unable to Decide}`
- `classification` ∈ `{Basic, Medium, High, null}` — `null` when not Rule 1
- `confidence` ∈ `{consensus, single-source, vague-only, silent-all}`
- `decision_rule` ∈ `{Rule-1, Rule-2a, Rule-2b, Rule-3, Rule-4, Rule-5}`
- `evidence_summary`: prose explaining the verdict (1–3 sentences)
- `sources`: one entry per **document** that is active (clear or vague); empty array `[]` for Rule 4
- `source.kind` ∈ `{clear, vague}`
- `source.stance` ∈ `{present, absent, null}` — required as `present` or `absent` on Rule 2b blocks (the field that disambiguates the conflict) and on inverse-retrieval evidence (`absent`); `null` otherwise
- `inverse_retrieval_attempted`: `true` on every Rule 4 record; `false` otherwise unless run
- `traceability`: one entry per **chunk** that contributed to the verdict (clear and vague chunks only — discard silent chunks)
- `traceability[].chunk_id`: the RAGFlow chunk identifier returned in the retrieval response
- `traceability[].doc_id`: the RAGFlow document identifier the chunk belongs to
- `traceability[].source_type`: resolved from document metadata via `doc_infos` — never guessed from URL or filename
- `traceability[].quote`: verbatim excerpt from the chunk text that supports (or informs) the verdict; keep to the most relevant sentence(s)
- `traceability[].kind` ∈ `{clear, vague}` — same classification applied to this chunk when building decision-rule evidence

**Traceability directives:**
1. Every chunk classified as `clear` or `vague` **must** have a corresponding traceability entry. Silent chunks are not recorded.
2. `source_type` in each traceability entry must come from the `doc_id → source_type` metadata map built before retrieval — it must match the `source_type` recorded in the `sources` array for that document.
3. The `quote` field must contain verbatim text — paraphrased or summarised text is not permitted.
4. For Rule 4 (no active sources), `traceability` is an empty array `[]`.
5. The `sources` array is a deduplicated, per-document view. The `traceability` array is the per-chunk raw evidence. Both must be consistent — every `doc_id` appearing in `traceability` must correspond to a `url` entry in `sources`.

### Traceability block cardinality per rule

| Rule | Required source blocks | Required traceability blocks |
|------|------------------------|------------------------------|
| Rule 1  | ≥1 `clear` (vague allowed) | ≥1 `clear` chunk entry |
| Rule 5  | ≥1 `clear` (vague allowed) | ≥1 `clear` chunk entry |
| Rule 2a | ≥2 `clear` | ≥2 `clear` chunk entries (from distinct docs) |
| Rule 2b | ≥2 `clear` with mixed `stance` | ≥2 `clear` chunk entries with opposing quotes |
| Rule 3  | ≥1 `vague` | ≥1 `vague` chunk entry |
| Rule 4  | `[]` (empty) | `[]` (empty) |

A Rule 1 record with zero clear blocks, or a Rule 4 record with any blocks, is invalid and must be re-emitted.
