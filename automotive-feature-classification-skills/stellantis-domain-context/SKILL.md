---
name: stellantis-domain-context
description: Always-on shared context for any agent (lead or subagent) participating in an automotive feature classification run. Loads vocabulary, mission, and the cross-cutting invariants that every other skill assumes are already known.
loader: shared
stage: always-on
requires: []
provides:
  - shared-vocabulary
  - mission-statement
  - cross-cutting-invariants
  - glossary
tools: []
---

## Dependencies

None. This is a foundational context skill. Every other skill in this framework assumes the agent has read this one first.

Recommended `requires` from other skills:
- `stellantis-automotive-feature-classification` (lead entry) — load before anything else
- `stellantis-subagent-classification-loop` — load on every subagent spawn
- `stellantis-lead-agent-subagent-orchestration`
- All discovery / validation / download / approval / workspace / decision skills

---

# Sub-skill — Domain context (always-on)

## Overview

A compact, evergreen "why we are here" digest. Loaded by **every** agent role at start-up. It contains only stable facts: mission, vocabulary, the four hard constants, and the master invariants. Volatile rules live in their own dedicated skills (`stellantis-decision-rules`, `stellantis-output-contract`, `stellantis-failure-handling`, etc.) — never duplicate them here.

## When this runs

- **Lead agent:** loaded once at preflight, before parsing the request.
- **Subagent:** loaded once on spawn, before reading the contract.
- **Any other skill:** assumes this skill has already been read; do not redefine its terms.

## Mission (one paragraph)

The agent automates a manual research workflow performed by an automotive analyst. For one car at a time — identified by `(brand, model, model_year, market)` and scoped to the manufacturer's full-option trim — the agent must produce a structured classification deliverable answering, for each of ~160 reference parameters: **is the feature present, and if so, at which level (Basic / Medium / High)**. Every decision must be backed by a citation to an approved source. The agent is in charge of research, retrieval, and reasoning; the human stays in the loop only at defined approval and review points.

## The four hard constants

| Constant | Value |
| :--- | :--- |
| **Scope** | Exactly one car per run; full-option trim only. |
| **Required inputs** | `brand`, `model`, `model_year`, `market`. None defaulted, none guessed. |
| **Output triple** | `Presence` × `Status` × `Classification` per parameter. Every record matches exactly one row of the master decision matrix. |
| **Phase boundary** | Source discovery and classification are strictly sequential and non-overlapping. After the approval gate closes, no web search occurs in the run. |

## Vocabulary contract

Use these terms exactly. Other documents and skills depend on them.

| Term | Meaning |
| :--- | :--- |
| **Category** | A high-level functional area (e.g. `EV`, `INFOT.`, `ADAS`). The reference list groups parameters by category. The user may say *domain* — accept it as a synonym in user-facing prose only; never use *domain* internally for this concept. |
| **Source site** | A website host (e.g. `bmw.com`, `insideevs.com`). What the open web normally calls a *domain*. Internal term is always *source site*. |
| **Parameter** | One feature row in the reference list. Identified internally by `category + parameter`; reported externally by the `Optimised Parameter Name`. |
| **Level** | Implementation tier: `Basic`, `Medium`, `High`. A level is assignable only if the parameter's reference row defines a criterion for it. |
| **Applicable level set** | The non-empty subset of `{Basic, Medium, High}` defined for a given parameter. Evidence describing a level outside the applicable set must never snap to the nearest defined level. |
| **Presence / Status / Classification** | The three independent fields of a parameter verdict. They answer three different questions: does the feature exist, what is the outcome of the assignment process, and at which level. |
| **Clear / vague / silent source** | A source's relationship to a single parameter. *Clear* = mentions the parameter and gives usable, schema-mappable information. *Vague* = mentions but cannot be mapped. *Silent* = does not mention at all. The same document can be clear for one parameter and silent for another. |
| **Active source** | Any source that is not silent for the parameter. |
| **Knowledge base / KB** | The retrieval substrate. Populated only with approved, ingested source documents. The exclusive evidence channel during classification. |
| **Run workspace** | A per-run directory containing the deliverable plus an internal `.harness/` working folder. |
| **Full-option trim** | The manufacturer's published top trim for the (model, year, market). Resolved automatically; recorded in the deliverable header. |

## Master invariants

These hold in every workflow, every stage, every record. Other skills implement and check them; they must not contradict them.

1. **One car per run.** No cross-car sharing of sources, KB, or state. Multiple cars → multiple independent runs.
2. **No defaults on required inputs.** If `brand`, `model`, `model_year`, or `market` cannot be extracted, the agent pauses and asks **only** for the missing field(s).
3. **No web search inside classification.** Once the approval gate closes, evidence comes exclusively from the KB.
4. **Approved-only ingestion.** Only documents the user has approved (or, in automated mode, those that survived canonicalisation + paywall filters) are uploaded to the KB.
5. **Schema-bounded levels.** If a level has no criterion in the reference list, no source can promote evidence to that level. Out-of-schema evidence yields `Unable to Decide`, never a nearest-level snap.
6. **Three-field independence.** `Presence`, `Status`, `Classification` are not derivable from each other; each carries its own meaning. Every emitted record matches exactly one row of the master decision matrix.
7. **Citation discipline.** Every `Presence = Yes` record carries at least one traceability block referencing a clear or vague active source. Rule-4 (`No Information Found`) records carry no traceability blocks.
8. **Ownership boundaries.** The lead writes the main state file and the per-category state. Subagents write only their own working file. Subagents never use web search or browser rendering.
9. **Re-runs vs gap-fill.** A full-car re-run gets a new run ID and a new KB. A gap-fill cycle keeps the run ID and reuses the KB with cycle-tagged metadata. They are distinct workflows; the agent never silently chains them.
10. **The reference list is frozen per run.** A snapshot is taken at preflight. Any later change to the reference list affects new runs only.

## Glossary mini-reference

For full definitions of any term, the authoritative source is `enums-reference` in the `stellantis-automotive-feature-classification` skill. This skill repeats only the bare minimum needed for cross-skill alignment:

- `Presence` ∈ `{Yes, No, Disputed, No Information Found}`
- `Status` ∈ `{Success, Conflict, Unable to Decide}` *(`No Information Found` is **not** a status value)*
- `Classification` ∈ `{Basic, Medium, High, Empty}` (always within the parameter's applicable level set; `Empty` whenever `Status ≠ Success` or `Presence = No`)
- `Disputed` is only valid under presence-conflict (Rule 2b)
- `No Information Found` is only valid under silent-all (Rule 4)

## Tips & common pitfalls

These are the recurring mistakes observed in design review. Internalise them.

- **Do not collapse the three fields.** `Presence = Yes, Status = Conflict, Classification = Empty` is not the same as `Presence = Disputed, Status = Conflict, Classification = Empty`. Different rules, different meanings.
- **Vague sources never decide alone.** A vague-only evidence set yields `Unable to Decide` (Rule 3) with `Presence = Yes`, never a level.
- **Silence ≠ absence.** Rule 4 (`No Information Found`) means "no evidence found"; Rule 5 (`No`) means "evidence says absent". Both produce `Classification = Empty` but they are different presence values and different statuses.
- **One clear source is enough.** A single uncontradicted clear active source decides Rule 1. Do not require multiple.
- **A vague source coexisting with a clear source does not block agreement.** Only clear sources are weighed for `Success` / `Conflict` detection.
- **Out-of-schema evidence is filtered, not snapped.** A source describing a `Basic`-like behaviour for an `H`-only parameter is disregarded for classification (and recorded as an internal advisory).
- **`Disputed` is reserved.** It only appears under presence conflict (Rule 2b). Never anywhere else.
- **Domain ≠ source site.** When the user says *domain*, infer from context. If they mean a feature area, internally say *category*. If they mean a website, internally say *source site*. Do not echo *domain* in internal artefacts.
- **No defaulting, ever.** A run never proceeds with a guessed market, year, brand, or model. Pause and ask.
- **Determinism is not promised.** Two runs over the same approved set may emit subtly different deliverables. The deliverable header discloses this; re-runs are for coverage, not for reproducibility.

## What this skill is NOT

- Not a workflow definition (see workflow skills).
- Not the decision-rule engine (see `stellantis-decision-rules`).
- Not the deliverable schema (see `stellantis-output-contract`).
- Not a tool reference (each operational skill declares its own tools).
- Not exhaustive. Anything volatile or operational is in a dedicated skill.

## Maintenance

This skill is meant to be **stable**. Edits to it are rare and require a framework-maintenance change log entry. See `framework-maintenance/impact-checklist.md` for the trigger conditions and required follow-ups.
