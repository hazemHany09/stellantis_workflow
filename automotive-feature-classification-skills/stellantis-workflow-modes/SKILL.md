---
name: stellantis-workflow-modes
description: Use at request entry to identify which workflow shape applies to the run, and to enforce composition rules between workflows. Lead context only.
loader: lead
stage: dispatch
requires:
  - stellantis-domain-context
provides:
  - workflow-identifier
  - workflow-composition-rules
tools: []
---

## Dependencies

- `stellantis-domain-context` — required.

Consumed by:
- `stellantis-automotive-feature-classification` (lead entry) — uses this skill at preflight to record the chosen workflow in the main state file.

---

# Sub-skill — Workflow modes

## Overview

A workflow is a specific composition of the framework's stages. This skill enumerates the **currently authored** workflows, declares which ones are mutually exclusive, and defines composition rules. Additional workflows may be added later — the skill is designed to grow without disturbing existing entries.

## When this runs

- Lead, once at preflight: parse the request, classify the run as a workflow instance, record the chosen workflow id in the main state file.
- Lead, on every later "client wants more" event: pick the workflow that fits the new request (full re-run? gap-fill? per-parameter flagging?). Each picked workflow runs independently.

## Currently authored workflows

Only **W1** has a full workflow document at the time of writing. The slots for additional workflows are reserved; see *Adding a new workflow* below.

### W1 — Normal pipeline (Mode A or Mode B)

The single end-to-end workflow currently authored. Covers the full path: parse → resolve trim → freeze reference list → discover sources (Mode A if user supplied URLs/domains; Mode B if not) → validate candidates → **approval gate (pause for URL approval)** → download + ingest → **classification go-ahead gate (pause after ingestion verified, before any classification subagent is spawned — user must reply `go`)** → spawn subagents by category → consolidate → emit deliverable.

| Slot | Value |
| :--- | :--- |
| Trigger | User submits a request naming a specific car. |
| Approval gate | **Yes**. URL-level approval required before ingestion. |
| Classification go-ahead gate | **Yes**. After ingestion is verified, the lead pauses with `PauseReason = awaiting-classification-go-ahead`, posts the ingestion + planned-partition summary, and waits for the user's explicit `go` before spawning any R1 classification subagent. |
| Scope | Full car, every parameter in the frozen reference list. |
| KB | New dataset per run. |
| Run ID | New per run. |
| Workflow document | `automotive-feature-classification/workflows/W1-normal-pipeline-mode-a/WORKFLOW.md` |

W1 has two **internal modes** that differ only in the discovery sub-skill loaded at stage 3:
- **Mode A** — user supplied at least one URL or domain → load `stellantis-source-discovery-with-client-domains`.
- **Mode B** — user supplied none → load `stellantis-source-discovery-without-client-domains`.

The rest of the pipeline is identical. Modes are not separate workflows; they are a fork inside W1.

## Reserved slots (not yet authored)

The framework's design accounts for these workflows but their workflow documents do not exist yet. They must not be invoked at runtime until a workflow document is authored and registered. If a request matches one of these patterns, the lead either falls back to W1 with a warning or pauses for clarification.

- **W2 — Automated mode (no approval gate).** Same as W1 minus the approval gate. Sub-skill `stellantis-source-discovery-without-client-domains` is ready; the workflow document is not.
- **W3 — Full-car re-run.** Mechanically identical to W1; the trigger differs.
- **W4 — Gap-fill cycle.** Targeted second round on an existing run; reuses the KB; same run ID; cycle-tagged metadata.
- **W5 — Per-parameter flagging re-run.** Re-classify selected parameters on the existing KB.
- **W6 — Concurrent runs on the same car.** Pure orchestration pattern; each run is independent.
- **W7 — Multi-car batch.** Fan-out to N independent W1 runs.
- **W8 — Input clarification sub-workflow.** Pre-pipeline interrupt when a required field is missing.

## Composition rules

These constraints govern how workflows interact. Adding a new workflow must not break them.

1. **W1 modes are mutually exclusive within a run.** Mode A and Mode B cannot both run in one W1 invocation.
2. **A re-run is a new invocation.** "Re-run" is a user-facing label, not a chained workflow. The new run gets a new run ID and a new KB.
3. **Gap-fill and per-parameter flagging (when authored) attach only to a completed run.** They cannot be invoked standalone.
4. **Gap-fill (when authored) keeps the run ID; full re-run does not.** This is the architectural distinction between the two patterns.
5. **The agent never silently chains workflows.** Each workflow is initiated by an explicit user action.
6. **Input clarification is invoked inline at most once per run.** Once the four required fields are present, it does not re-enter.

## Workflow id storage

The chosen workflow id and (for W1) the mode is recorded in the main state file at preflight under `workflow` and `mode`. The deliverable header copies these values into the output for audit.

## Tips & common pitfalls

- **Default to W1 when ambiguous.** If the request fits W1 cleanly, use it. Do not invent a workflow.
- **A request to "redo with different sources" is a new W1**, not a "modification" of a previous run.
- **Concurrent runs on the same car are allowed.** Do not coordinate, de-duplicate, or merge them. Each is independent.
- **"Automated mode" means W2.** Do not skip the approval gate inside a W1 run "as a shortcut" — that is a workflow, not a flag.

## What this skill is NOT

- Not the workflow content. Each workflow document defines its own stages.
- Not a state machine. Stage-level state is managed by the workflow document and the main state file.
- Not a tool reference.

## Maintenance — adding a new workflow

Adding W2..Wn is a deliberate, multi-step change. Procedure:

1. Author the workflow document under `automotive-feature-classification/workflows/<id>-<slug>/WORKFLOW.md`.
2. In this skill: promote the entry from *Reserved slots* to *Currently authored workflows* with a full slot table.
3. Update the lead skill (`stellantis-automotive-feature-classification`) `Workflow & Sub-Skill Loading` section to dispatch to it.
4. Update `business-logic/13-potential-workflows.md` row.
5. Apply every follow-up under `framework-maintenance/impact-checklist.md` "Add a new workflow".
6. Append a change-log entry.

Do not author a new workflow by editing an existing workflow document. New workflows live in their own directory.
