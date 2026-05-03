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

---

## Common Pitfalls

- **Silence ≠ absence.** Rule 4 ("no evidence found") ≠ Rule 5 ("evidence says absent"). Different verdicts.
- **One clear source is enough.** Do not require corroboration for Rule 1.
- **Vague + clear = not a conflict.** Only clear sources determine Success/Conflict. Vague sources only matter when no clear source exists.
- **Don't snap to nearest level.** If a source describes a level not in the parameter's schema, discard it. Apply Rule 3.
- **`Conflict` never produces a level.** Emit Conflict and let the human adjudicate.

---

## ## Result Block Schema

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
      "note": ""
    }
  ],
  "inverse_retrieval_attempted": false
}
```

**Field rules:**
- `presence` ∈ `{Yes, No, Disputed, No Information Found}`
- `status` ∈ `{Success, Conflict, Unable to Decide}`
- `classification` ∈ `{Basic, Medium, High, null}` — `null` when not Rule 1
- `confidence` ∈ `{consensus, single-source, vague-only, silent-all}`
- `decision_rule` ∈ `{Rule-1, Rule-2a, Rule-2b, Rule-3, Rule-4, Rule-5}`
- `evidence_summary`: prose explaining the verdict (1–3 sentences)
- `sources`: one entry per traceability block; empty array `[]` for Rule 4
- `source.kind` ∈ `{clear, vague}`
- `inverse_retrieval_attempted`: `true` on every Rule 4 record; `false` otherwise unless run
