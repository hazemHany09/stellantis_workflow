# Deliverable CSV — Column Mapping

Secondary deliverable per Q-DEL-1. Flattens the JSON deliverable into a tabular form for analyst review.

File: `runs/<run-id>/deliverable.csv`.

## Columns (in order)

| # | Column                              | Source (JSON path)                                  | Notes                                                                                               |
| - | :---------------------------------- | :-------------------------------------------------- | :-------------------------------------------------------------------------------------------------- |
| 1 | `category`                          | `records[].category`                                |                                                                                                     |
| 2 | `parameter_name`                    | `records[].parameter_name`                          | Optimised name (AS-DEL-G).                                                                          |
| 3 | `schema_type`                       | `records[].schema_type`                             | Compact notation (Q-DEL-4).                                                                         |
| 4 | `presence`                          | `records[].presence`                                | Enum value.                                                                                         |
| 5 | `status`                            | `records[].status`                                  | Enum value.                                                                                         |
| 6 | `classification`                    | `records[].classification`                          | Enum value; `Empty` is emitted as the literal string `Empty`.                                       |
| 7 | `decision_rule`                     | `records[].decision_rule`                           | `Rule-1 … Rule-5`.                                                                                  |
| 8 | `traceability_count`                | `records[].traceability_blocks` length              | Integer.                                                                                            |
| 9 | `traceability_blocks`               | `records[].traceability_blocks` concatenated        | See flattening below.                                                                               |

## Traceability-block flattening

A single cell in column 9 holds all blocks for the record, separated by `␞` (U+241E, Unicode Symbol For Record Separator). Within a block, fields are separated by `␟` (U+241F, Unit Separator).

Per-block field order: `source_name␟source_link␟kind␟source_justification␟classification_justification`.

For records with zero traceability blocks (Rule-4), the cell is empty.

## Header row

Quoted CSV fields, comma-separated, UTF-8, CRLF line endings. First row is the header exactly:

```
category,parameter_name,schema_type,presence,status,classification,decision_rule,traceability_count,traceability_blocks
```

## Row order

Identical to JSON `records[]` order, which is `params.csv` row order (AS-DEL-B).

## Summary + footer

CSV does not carry the JSON `header`, `summary`, `telemetry`, or `footer`. The analyst is expected to consult `deliverable.json` for that context. A pointer line is prepended as a comment:

```
# See deliverable.json for run header, summary counts, telemetry, and sources_excluded footer.
```

## Invariants

1. One row per parameter in `params.csv`.
2. No row is omitted for any reason — even Rule-4 records appear.
3. Column count is exactly 9; additional columns require a new release of this mapping (update in `framework-maintenance/change-log.md`).
