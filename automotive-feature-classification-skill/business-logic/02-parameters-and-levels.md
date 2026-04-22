# 02 — Parameters and Levels

## The reference parameter list

The authoritative list of parameters lives in `artifacts/params.csv`. That file is the single source of truth. This skill must never hard-code parameter definitions — it must read them from the CSV at runtime so that when the CSV changes, the skill follows.

The file currently contains ~160 parameters grouped under categories such as `EV` and `INFOT.`. More categories exist further down the file. The count and the categories may evolve; the skill must not assume a fixed number.

## CSV structure

| Column                      | Meaning                                                                                                                                                                            |
| :-------------------------- | :--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Domain / Category`         | Category (see glossary). Rows whose `Parameter` column is empty and whose `Domain / Category` starts with `▶` are **category header rows** that introduce a new category.          |
| `Parameter`                 | Human-readable parameter name.                                                                                                                                                     |
| `Category Description`      | Present only on some rows (typically a `Category Overview` row placed beneath a category header). Provides a human-readable summary of the whole category.                         |
| `Parameter Description`     | What the parameter covers. This is the primary semantic description used to disambiguate the parameter during research and retrieval.                                              |
| `Basic — Criterion`         | Text defining the Basic level. **An empty cell means Basic is not applicable to this parameter.**                                                                                  |
| `Medium — Criterion`        | Text defining the Medium level. **An empty cell means Medium is not applicable to this parameter.**                                                                                |
| `High — Criterion`          | Text defining the High level. **An empty cell means High is not applicable to this parameter.**                                                                                    |
| `Optimised Parameter Name`  | Standardised name used when reporting results externally. The deliverable must use this name.                                                                                      |

## Applicable level set

For each parameter, the applicable level set is the set of levels whose criterion cell is non-empty.

- If only `High — Criterion` is filled, the only possible verdicts are `Not supported` or `High`. No intermediate level exists.
- If `Basic — Criterion` and `High — Criterion` are filled but `Medium — Criterion` is empty, the possible verdicts are `Not supported`, `Basic`, or `High`. The skill must never emit `Medium` for such a parameter.
- If all three criteria are filled, any of the three levels is valid.
- Combinations with only `Basic` or only `Medium` are possible in principle. The skill must handle all non-empty subsets of `{Basic, Medium, High}`.

## Presence vs level

Two distinct questions must be answered for each parameter on each car:

1. **Presence.** Does the car support this parameter at all? This is a binary Yes/No.
2. **Level.** If Yes, which level in the applicable set does the car's implementation meet?

The justifications for the two questions are also distinct. The presence justification explains why the feature is deemed present (or absent). The level justification explains why the implementation matches a particular criterion and not the others.

## Category headers and overview rows

Some rows in the CSV are not parameters:

- Rows that are a category banner (e.g. `▶  EV` with all other columns empty).
- Rows labelled `Category Overview` under such a banner, whose purpose is to carry the `Category Description` text for the whole category.

These rows must be recognised and skipped during parameter enumeration but their text must be preserved for display and for providing context during retrieval.

## Parameter identity

The combination of `Domain / Category` + `Parameter` is treated as the parameter identity for internal processing. The `Optimised Parameter Name` is used for the final deliverable. The skill must maintain a stable mapping between the two so that results can be traced back to the CSV row.
