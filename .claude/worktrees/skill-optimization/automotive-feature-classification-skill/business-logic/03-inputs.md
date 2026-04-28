# 03 — Inputs

## What the client submits

The client initiates an analysis by providing, at minimum:

1. **Target car identification** — enough information to uniquely identify the car to be analysed.
2. **Source site list** — *optional*. A list of websites to restrict research to. If omitted, the agent must discover a suitable list itself.

## Target car identification

The client **must** provide all four of the following fields. None is optional.

| Field      | Required | Notes                                                                                                                              |
| :--------- | :------- | :--------------------------------------------------------------------------------------------------------------------------------- |
| Brand      | Yes      | E.g. `BMW`, `Peugeot`, `Tesla`.                                                                                                    |
| Model      | Yes      | E.g. `iX`, `3008`, `Model Y`.                                                                                                       |
| Model year | Yes      | Feature availability changes across model years. The agent must not guess the year.                                                |
| Market     | Yes      | Feature availability changes across markets (e.g. EU vs US vs MEA). The agent must not guess the market.                           |
| Trim       | Fixed    | Always the full-option trim (see `00-overview.md`). The client does not name the trim; the skill treats "full option" as scope.    |

If any of brand / model / year / market is missing, the run is not started — the skill must request the missing field before proceeding.

## Source site list — two modes

### Mode A: client-supplied

The client hands over one or more source sites that must be used. In this mode:

- The agent **must** restrict research to those sites.
- The agent may still need to identify specific pages within those sites; site-level restriction does not remove the need for per-page discovery.
- The client-supplied list is still subject to the source approval gate (see `04-sources.md`) because additional documents (specific URLs) will need approval even if the sites are pre-approved.

### Mode B: agent-discovered

The client provides no source site list. In this mode:

- The agent runs an open web search using the four identifying fields (brand + model + year + market) as the query context to find candidate sources that cover the exact car.
- The candidate list is then presented to the client at the source approval gate.
- Nothing is downloaded or ingested before approval.

## One car per request

**Resolved.** An analysis run covers exactly **one car**. One agent is responsible for one car. Multiple cars are handled by spawning multiple independent runs (potentially in parallel) — each with its own inputs, its own source set, its own knowledge base, and its own deliverable. Cross-car sharing of sources, KBs, or state is out of scope.

This rule governs all downstream scoping: sources (see `04-sources.md`) and knowledge base (see `05-knowledge-base.md`) are always per-car.

## Open questions

- **Q-INPUT-5**: What is the acceptable structured shape of the client's submission — a JSON object, a form, a free-text brief? This affects how the skill reads the request.
- **Q-INPUT-6**: Car disambiguation when brand + model + year + market maps to multiple variants (body styles, regional rebadges, facelifts, sub-models). Interim rule in `11-assumptions.md` (AS-INPUT-A) is to prompt the client; no silent picking.
- **Q-INPUT-7**: Market vocabulary (ISO country codes vs country names vs sales regions). Interim rule AS-INPUT-B accepts all and normalises.
- **Q-INPUT-8**: Model-year validity range and pre-launch / discontinued handling. Interim rule AS-INPUT-C restricts to `(now − 10)` to `(now + 2)`.

*(Q-INPUT-1, Q-INPUT-2, Q-INPUT-3, Q-INPUT-4 all closed — see `08-open-questions.md`.)*
