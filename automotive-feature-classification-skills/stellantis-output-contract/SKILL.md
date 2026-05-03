---
name: stellantis-output-contract
description: Use whenever an agent emits a per-parameter record, a deliverable file, an excluded-sources footer, or a run header. Defines the field shapes, ordering, and required content of every output the framework produces.
loader: shared
stage: emission
requires:
  - stellantis-domain-context
  - stellantis-decision-rules
provides:
  - parameter-record-shape
  - deliverable-format-contract
  - run-header-shape
  - sources-excluded-shape
tools: []
---

## Dependencies

Hard prerequisites:
- `stellantis-domain-context` — vocabulary and the three-field model.
- `stellantis-decision-rules` — the only legal `(Presence, Status, Classification)` combinations.

Used by:
- `stellantis-subagent-classification-loop` — emits per-parameter records.
- `stellantis-lead-agent-subagent-orchestration` — merges records into the deliverable.
- `stellantis-failure-handling` — feeds the `sources_excluded` footer.
- `stellantis-run-workspace` — defines where each output is written.

---

# Sub-skill — Output contract

## Overview

The single source of truth for the **shape** of everything the framework writes. The decision-rule engine decides *what* the verdict is; this skill decides *how* it is recorded. Schemas are defined once in the core skill's `templates/` (notably `intermediate-parameter-record.json.tmpl` and `deliverable.schema.json`); this skill states the rules those schemas implement.

## When this runs

- Subagent: every time a parameter record is emitted into the result envelope.
- Lead: every time the consolidated deliverable is generated, the run header is finalised, or the `sources_excluded` footer is appended.

## Per-parameter record

Every parameter in the frozen reference list produces exactly **one** record. Required fields:

| Field | Notes |
| :--- | :--- |
| `category` | Internal category name from the reference list. |
| `parameter` | Internal parameter name from the reference list. |
| `optimised_parameter_name` | The externally reported name. **This is the one shown to the user.** The internal `parameter` field is for audit only. |
| `presence` | One of `{Yes, No, Disputed, No Information Found}`. |
| `status` | One of `{Success, Conflict, Unable to Decide}`. `No Information Found` is **never** a status value. |
| `classification` | One of `{Basic, Medium, High, Empty}`. Non-empty only under Rule 1. Must belong to the parameter's applicable level set. |
| `decision_rule` | One of `{Rule-1, Rule-2a, Rule-2b, Rule-3, Rule-4, Rule-5}`. The single rule that produced the record. |
| `presence_justification` | Prose. Explains why the presence value was assigned. Required for every record except Rule 4 (silent-all). |
| `level_justification` | Prose. Explains why the level was chosen and not the alternatives. Required only when `classification` is non-empty (Rule 1). |
| `confidence` | One of `{consensus, single-source, vague-only, silent-all}`. Set deterministically by the decision-rule engine — see `stellantis-decision-rules` "Consensus verification". `consensus` requires ≥ 2 clear sources from distinct `source_type` categories. |
| `inverse_retrieval_attempted` | Boolean. Required `true` on every Rule 4 record (the engine demands an inverse-retrieval pass before settling at silent-all). May be `true` on other rules too if an inverse pass ran. |
| `traceability_blocks[]` | Zero or more blocks. Cardinality determined by the rule (see below). |

### Traceability-block cardinality (per rule)

| Rule | Required blocks |
| :--- | :--- |
| Rule 1  | ≥ 1 clear-source block (vague blocks allowed) |
| Rule 5  | ≥ 1 clear-source block (vague blocks allowed) |
| Rule 2a | ≥ 2 clear-source blocks |
| Rule 2b | ≥ 2 clear-source blocks (mixed stances) |
| Rule 3  | ≥ 1 vague-source block |
| Rule 4  | 0 blocks |

### Traceability block fields

| Field | Notes |
| :--- | :--- |
| `kind` | `clear` or `vague`. |
| `stance` | For Rule 2b: `present` or `absent`. Optional otherwise. For inverse-retrieval evidence: `absent` (this is what promotes Rule 4 candidates to Rule 5). |
| `source_url` | The canonical URL stored as document metadata in the KB. **URL-level only**; no chunk IDs, fragments, or excerpts in the deliverable. |
| `source_type` | Mirrors the `source_type` from KB metadata. Required because the consensus check in `stellantis-decision-rules` reads it from the traceability block, not by re-querying the KB. |
| `note` | Optional short prose tying the source to the rule. |

## Deliverable format

The deliverable is dual-serialised, written to **`/mnt/user-data/outputs/`** — a separate sandbox path, not the workspace. Filenames omit any directory prefix beyond that root:

1. **Primary — JSON.** Path: `/mnt/user-data/outputs/<brand>-<model>-<year>-<market>-<run-id>.json`. Full schema, traceability blocks intact.
2. **Secondary — CSV.** Same basename, `.csv` extension, in `/mnt/user-data/outputs/`. Per-parameter records flattened to rows. Traceability is folded by URL concatenation or one-row-per-block expansion (whichever the CSV columns template prescribes).

Both files are emitted on every run. The internal `.harness/` folder (under `/mnt/user-data/workspace/`) is not part of the client-facing deliverable but is retained for audit.

### CSV emission — use a Python script

Direct LLM-authored CSV is error-prone: field quoting, comma escaping, and multi-value folding (e.g. multiple `source_url` values per record) regularly produce malformed output. Instead, the lead must produce the CSV by writing and executing a short Python script that reads the JSON deliverable and converts it to CSV.

**Procedure:**

1. Write `.harness/scripts/json_to_csv.py` — a self-contained Python 3 script that:
   - Reads `/mnt/user-data/outputs/<brand>-<model>-<year>-<market>-<run-id>.json`.
   - Iterates `records[]` in order.
   - Flattens each record to a row per the column specification in `templates/deliverable.csv.columns.md`.
   - Uses Python's `csv.writer` (not string concatenation) for correct quoting.
   - Writes to `/mnt/user-data/outputs/<brand>-<model>-<year>-<market>-<run-id>.csv`.
2. Execute the script via the code execution tool.
3. Verify the output file exists and its row count equals the JSON `records[]` length.

If the code execution tool is unavailable, fall back to direct CSV authoring — but note the fallback in the event log. The JSON deliverable is always the authoritative source; the CSV is derived from it.

### Record ordering

Records appear in the **exact order of the frozen reference list** — preserving category grouping and author intent. CSV row order = JSON array order. Stable across runs, which makes diffing trivial.

## Run header

The deliverable header (in JSON: top-level metadata object; in CSV: a banner block before the records) must contain at least:

| Field | Source |
| :--- | :--- |
| `run_id` | Generated at preflight |
| `submitted_at`, `completed_at` | Timestamps |
| `car_identity` | Resolved `(brand, model, model_year, market_canonical)` plus original market input |
| `resolved_full_option_trim` | Recorded by `stellantis-car-trim-resolution` |
| `reference_list_snapshot` | Hash of the frozen reference list |
| `source_counts` | `{approved, rejected, ingested, failed}` |
| `ingestion_window` | `{start, end}` timestamps |
| `classification_window` | `{start, end}` timestamps |
| `total_elapsed` | `HH:MM:SS` |
| `subagents_spawned` | Integer |
| `workflow` | The workflow identifier (e.g. `W1-mode-a`) |
| `cycles[]` | Optional. Present only when one or more gap-fill cycles ran. |
| `determinism_disclosure` | Fixed string stating that re-runs may differ. |

## Run-level summary

Below the header, before the per-parameter records, include:

- Counts: total parameters, `Yes`, `No`, `Disputed`, `No Information Found`, `Conflict`, `Unable to Decide`.
- Distribution by classification level (`Basic`, `Medium`, `High`).
- Lists of all parameters with `Conflict`, `Unable to Decide`, or `No Information Found` outcomes — for fast human review.

## Excluded-sources footer

A dedicated `sources_excluded` section at the end of the deliverable lists every URL that was approved but never made it into the queryable KB. Required columns:

| Field | Notes |
| :--- | :--- |
| `canonical_url` | Same canonicalisation as the approval store. |
| `failure_stage` | `download` or `ingestion`. |
| `failure_reason` | Short string (e.g. `HTTP 403`, `parse timeout`, `60-min ingestion ceiling exceeded`). |

No per-parameter notes appear here — failed sources were never queried, so claiming relevance to specific parameters is unsound.

## Schema-type encoding (compact)

The applicable level set per parameter is encoded compactly: `H`, `B/H`, `B/M/H`, `M/H`, etc. A legend defining the abbreviations is included in the run header (or in a schema-reference section).

## Tips & common pitfalls

- **Use `optimised_parameter_name` externally; never the internal `parameter` field.** Internal names exist only for audit.
- **`Classification = Empty` is mandatory wherever Rule ≠ 1.** Do not write `null`, `"-"`, or `"None"` in its place.
- **Justifications are prose, not quotes.** No verbatim source excerpts in the deliverable. The KB retains chunk-level content internally for audit.
- **URL-level citations only.** No chunk anchors, no `#fragment`, no excerpts in the traceability block.
- **Traceability cardinality is enforced.** A Rule 1 record with zero blocks is invalid; a Rule 4 record with any blocks is invalid.
- **Failed sources go to the footer, not into per-parameter records.**
- **Order matches the reference list.** Don't sort by category-then-status or any other key.

## What this skill is NOT

- Not the rule engine (see `stellantis-decision-rules`).
- Not a serialisation library — schemas live in the core skill's templates.
- Not a workspace layout (see `stellantis-run-workspace`).

## Maintenance

Any change to a record field, file format, or output ordering triggers:
1. Update the relevant template in the core skill (`intermediate-parameter-record.json.tmpl`, `deliverable.schema.json`, `deliverable.csv.columns.md`).
2. Update this skill.
3. Apply every follow-up under `framework-maintenance/impact-checklist.md` "Change a template" or "Add a new value to an enum".
4. Bump `contract_version` for breaking changes.
