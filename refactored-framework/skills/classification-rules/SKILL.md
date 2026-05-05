---
name: classification-rules
description: Decision rules for converting evidence into Presence/Classification verdicts. Defines the output schema for the ## Result block. Used by lead agent, parameter-classifier-agent, and parameter-searching-agent.
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

**Level assignment when evidence falls between two levels:** Always classify at the **highest level whose criteria are fully met** by the evidence. Because the hierarchy is cumulative, a car meeting Medium criteria also satisfies Basic — there is no need to "round up" to the unmet higher level. If evidence confirms the car meets Medium but not High, the classification is `Medium`.

*Example:* Levels defined are Medium (≥150 km/h) and High (≥180 km/h). The car's confirmed top speed is 160 km/h — Medium is met, High is not. Classification = `Medium`.

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
| `forum_or_ugc` | Owner forums, Reddit | Soft signal only — never sole authority for a confirmed verdict |

---

## Decision Rules

Two fields carry the verdict: **Presence** and **Classification**. There is no separate Status field.

| Situation | Presence | Classification |
|-----------|----------|----------------|
| ≥1 clear source, all clear agree on same valid level | `Yes` | the agreed level (`Basic` / `Medium` / `High`) |
| ≥1 clear source, all clear agree feature is absent | `No` | `Not Present` |
| ≥2 clear sources present, disagree on level | `Yes` | `Conflict` |
| ≥2 clear sources disagree on presence vs absence | `Conflict` | `Conflict` |
| Active sources exist, but none are clear (all vague) | `Yes` | `Unable to Decide` |
| No active sources at all (every source silent) | `No Information Found` | `No Information Found` |
| Clear source confirms feature exists, measured value is below the lowest defined level's threshold | `Yes` | `Didn't Meet Lowest Level` |

**Every record matches exactly one row. No exceptions.**

### `Didn't Meet Lowest Level` — conditions and boundaries

Apply this rule when **all three** of the following hold:
1. At least one clear source confirms the feature **exists** on the trim (not absent), AND
2. The source provides a **concrete measurable value** (e.g. a speed figure, a distance, a seat count), AND
3. That value **demonstrably falls below** the threshold stated for the **lowest defined level** in this parameter's applicable level set.

Do NOT apply if:
- The source is vague about the value (→ `Unable to Decide` applies instead).
- The source mentions the feature but provides no measured value at all (→ `Unable to Decide`).
- The car's relevant spec is not confirmed by a clear source (→ `Unable to Decide`).

**Example — apply:** Medium and High are the only defined levels; Medium threshold is 150 km/h. A clear manufacturer spec confirms the car's top speed is 140 km/h → `Didn't Meet Lowest Level`.

**Example — do not apply:** Car has massage seats but sources do not specify whether they are front, rear, or both (required to distinguish levels) → `Unable to Decide`.

---

## Confidence Annotation

Every record carries a `confidence` field, set deterministically:

| Situation | `confidence` |
|-----------|-------------|
| Confirmed / Absent: ≥2 clear sources from distinct `source_type` categories agree | `consensus` |
| Confirmed / Absent: all clear sources share the same `source_type` category | `single-source` |
| Confirmed / Absent: the only clear source is `forum_or_ugc` | Demote to `Unable to Decide`, `vague-only` |
| `Unable to Decide` (vague-only evidence) | `vague-only` |
| `No Information Found` (all sources silent) | `silent-all` |
| Conflict (level or presence): ≥2 clear sources from distinct `source_type` categories | `consensus` |
| Conflict (level or presence): all clear sources share a single `source_type` category | `single-source` (also flag advisory `same-source-type-conflict`) |
| `Didn't Meet Lowest Level`: ≥2 clear sources from distinct `source_type` categories agree | `consensus` |
| `Didn't Meet Lowest Level`: all clear sources share the same `source_type` category | `single-source` |

### Demotion rule

If the only clear source backing a Confirmed / Absent verdict is `forum_or_ugc`, demote the verdict to **`Unable to Decide`** (`presence = Yes`, `classification = Unable to Decide`) with `confidence = vague-only`. Forum claims alone never produce a confirmed level or `Not Present` — there is no editorial accountability.

---

## Invariants

1. Silent sources never count toward any rule.
2. Vague sources alone never produce a confirmed level, `Not Present`, `Conflict`, or `Didn't Meet Lowest Level` — only `Unable to Decide`.
3. A single uncontradicted clear source is sufficient for a Confirmed or Absent verdict (`single-source` confidence).
4. `Classification` is always populated — it is never `null` or empty.
5. `Conflict` in `presence` is exclusive to the presence-conflict situation (two clear sources disagree on whether the feature is present or absent). `No Information Found` appears in both `presence` and `classification` only for the all-silent situation.
6. Every emitted record matches exactly one row in the decision rules table.
7. Out-of-schema evidence (a level not in the parameter's applicable level set) is filtered — never snapped to nearest.
8. `forum_or_ugc` alone never produces a confirmed level or `Not Present` — demote to `Unable to Decide`.
9. Every `No Information Found` record requires `inverse_retrieval_attempted: true`.
10. `Didn't Meet Lowest Level` requires at least one clear source providing a concrete measured value that demonstrably falls below the lowest defined level's threshold. A vague or unmeasured source does not qualify.
11. Out-of-schema evidence (a level the parameter's applicable level set does not define) is **disregarded** for classification — never snapped to the nearest defined level. If valid-level clear evidence still exists, apply the confirmed row (or conflict-level if multiple valid-level sources conflict). If no valid-level evidence remains, apply `Unable to Decide`.

---

## Retrieval Prerequisites Before Applying Unable-to-Decide Rules

`Unable to Decide` and `No Information Found` are both last-resort verdicts. **All five gates below must be satisfied before applying either verdict.** The only difference between them is what remains after the gates are cleared — vague-only sources vs. no active sources.

1. At least one query was run using level-criteria terms to attempt to sharpen vague evidence
2. An alias or description-terms query was run to ensure the feature wasn't named differently in sources
3. All viable positive query types were attempted (optimised name, alias, description terms)
4. At least one inverse / negative query was run and returned no clear evidence
5. top_k was varied (e.g. tried both 6 and 12) across queries, and the query budget is exhausted or all viable query angles are genuinely exhausted

If any gate was skipped, do not emit the verdict — run the missing query first.

Every `No Information Found` record **must** set `"inverse_retrieval_attempted": true` in the Result JSON.

### Inverse-retrieval query templates

Standard retrieval is biased toward presence: a search for "Night Vision" surfaces docs that mention the term, not docs that explicitly state it is unavailable. The inverse pass targets the language of negation, exclusion, and removal. Run **at least one** of these (1–2 queries per parameter) before settling at `No Information Found`:

- `"<feature_name>" "not available"`
- `"<feature_name>" "deleted"`
- `"<feature_name>" "no longer offered"`
- `"<feature_name>" "<top_trim>" "exclusion"`
- `"features not available on <top_trim>"`
- `"<top_trim>" "deletes" OR "omits" OR "lacks"`

Inspect top chunks. If any explicitly states the feature is absent on the resolved trim, **emit an Absent verdict** (`presence = No`, `classification = Not Present`) and record the chunk as a clear-source traceability block with `stance: absent`. If the inverse pass produces no negative evidence either, the verdict stays at `No Information Found` and the record carries `inverse_retrieval_attempted: true`.

---

## Edge Cases

- **Single clear + silent others.** Silent sources don't count. The lone clear source carries the Confirmed or Absent verdict.
- **Mixed conflict — valid level vs out-of-schema tier.** A clear source describing a level in the parameter's applicable set, alongside a clear source describing a tier the schema does not define: **disregard the out-of-schema source** for classification. Apply the Confirmed verdict (or conflict-level if multiple valid-level sources still conflict). Record an internal advisory noting that out-of-schema evidence was observed.
- **All active sources support an out-of-schema tier only.** No valid-level evidence exists. Apply `Unable to Decide` (`Yes, Unable to Decide`).
- **Same-source-type conflict.** If a conflict verdict is backed only by clear sources sharing one `source_type` category, set `confidence = single-source` and add an advisory `same-source-type-conflict` to the evidence summary. Surface it for human review.
- **Vague + clear coexist under Confirmed / Absent.** Vague blocks may appear alongside the clear block(s); they are not a conflict.

---

## Common Pitfalls

- **Source contradicts another?** Check source types — manufacturer spec outranks press review. Apply the hierarchy: official spec > dealer/fleet guide > long-form press > aggregator > UGC.
- **Feature missing from all sources?** Check if it is market-specific or trim-specific before concluding absence.
- **Level criteria ambiguous?** Default to the most conservative (lower) level — do not upgrade without clear evidence.
- **Silence ≠ absence.** `No Information Found` ("no evidence found") ≠ `Not Present` ("evidence says absent"). Different verdicts.
- **One clear source is enough.** Do not require corroboration for a Confirmed verdict. But run additional queries to check for contradicting evidence and to determine whether `single-source` or `consensus` confidence applies.
- **Vague + clear = not a conflict.** Only clear sources determine a confirmed level or Conflict. Vague sources only matter when no clear source exists.
- **Don't snap to nearest level.** If a source describes a level not in the parameter's schema, discard it. Apply `Unable to Decide`.
- **Don't round up when between two levels.** If evidence clearly meets Medium but not High, classify as `Medium` — not `High`. The hierarchy is cumulative upward, not a reason to promote.
- **Conflict never produces a level.** Emit Conflict and let the human adjudicate.
- **`Didn't Meet Lowest Level` vs `Unable to Decide`.** The former requires a concrete measurable value from a clear source. If the source is vague about the value, or provides no measured number, use `Unable to Decide` instead.

---

## Result Block Schema

Every parameter contract file has a `## Result` section. The agent writes this exact JSON structure:

```json
{
  "parameter_id": "",
  "parameter_name": "",
  "category": "",
  "presence": "",
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
- `presence` ∈ `{Yes, No, Conflict, No Information Found}`
- `classification` ∈ `{Basic, Medium, High, Not Present, Conflict, Unable to Decide, No Information Found, Didn't Meet Lowest Level}`
- `confidence` ∈ `{consensus, single-source, vague-only, silent-all}`
- `decision_rule` ∈ `{confirmed, absent, conflict-level, conflict-presence, unable-to-decide, no-information, below-threshold}`
- `evidence_summary`: prose explaining the verdict (1–3 sentences)
- `sources`: one entry per **document** that is active (clear or vague); empty array `[]` for `No Information Found`
- `source.kind` ∈ `{clear, vague}`
- `source.stance` ∈ `{present, absent, null}` — required as `present` or `absent` on conflict-presence blocks (the field that disambiguates the conflict) and on inverse-retrieval evidence (`absent`); `null` otherwise
- `inverse_retrieval_attempted`: `true` on every `No Information Found` record; `false` otherwise unless run
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
4. For `No Information Found` (no active sources), `traceability` is an empty array `[]`.
5. The `sources` array is a deduplicated, per-document view. The `traceability` array is the per-chunk raw evidence. Both must be consistent — every `doc_id` appearing in `traceability` must correspond to a `url` entry in `sources`.

### Traceability block cardinality per verdict

| Verdict | Required source blocks | Required traceability blocks |
|---------|------------------------|------------------------------|
| Confirmed (Basic/Medium/High) | ≥1 `clear` (vague allowed) | ≥1 `clear` chunk entry |
| Absent (Not Present) | ≥1 `clear` (vague allowed) | ≥1 `clear` chunk entry |
| Conflict — level | ≥2 `clear` | ≥2 `clear` chunk entries (from distinct docs) |
| Conflict — presence | ≥2 `clear` with mixed `stance` | ≥2 `clear` chunk entries with opposing quotes |
| Unable to Decide | ≥1 `vague` | ≥1 `vague` chunk entry |
| No Information Found | `[]` (empty) | `[]` (empty) |
| Didn't Meet Lowest Level | ≥1 `clear` with measured value | ≥1 `clear` chunk entry containing the measured value |

A Confirmed record with zero clear blocks, or a `No Information Found` record with any blocks, is invalid and must be re-emitted.
