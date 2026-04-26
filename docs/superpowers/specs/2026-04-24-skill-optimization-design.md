# Skill Optimization Design: Template-First Success Metrics

**Date:** 2026-04-24\
**Scope:** Optimize 8 automotive classification skills + 1 workflow (W1)\
**Goal:** Improve clarity, fix gaps, add context, define success metrics for first production run

## Problem Statement

Current skills lack:

1. **Shared success criteria** — no definition of "workflow complete"
2. **Context chains** — subagents don't know what state they inherit or what they must pass downstream
3. **Failure guidance** — no "if X fails, do Y" instructions
4. **State tracking clarity** — unclear how to monitor progress across parallel subagent runs
5. **Output validation** — no criteria for accepting consolidation/deliverables

This causes risk in first run: subagents may produce outputs that don't meet unstated expectations, lead agent can't detect incomplete work, deliverables fail validation.

## Solution: Template-First Approach

Define success metrics in templates first (single source of truth), then update all 8 skills to reference those criteria. Subagents, lead agent, and consolidation all use same metrics.

### Phase 1: Audit & Define Metrics

**Audit scope:**

* `src/templates/STATE.main.md.tmpl` — lead-agent STATE
* `src/templates/STATE.category.md.tmpl` — per-category consolidated records
* `src/templates/intermediate-parameter-record.json.tmpl` — subagent parameter output schema
* `src/templates/subagent-working-file.md.tmpl` — subagent scratch workspace
* `src/templates/traceability-block.md.tmpl` and `.json.tmpl` — evidence linking

**Deliverable:**
For each template, add `## Success Criteria` section with:

* Required fields (which must be present for "done")
* Quality thresholds (e.g., tier assignment, evidence count)
* Validation rules (schema compliance, enum values)
* Exit conditions (when to stop searching / consolidating)

Example for parameter record:

```
## Success Criteria
- [Tier assigned] One of: T1-confirmed, T2-partial, T3-inferred, T4-not-found
- [Evidence block] ≥1 chunk ID, source site, quote
- [Subagent name] Must match deployed subagent
- [Confidence] 0-100, linked to tier
- No missing required_key values
```

### Phase 2: Update Skill Instructions

Update all 8 skills with:

1. **"Success Metric" inline** — top of each skill, what this skill's output looks like when done
2. **"You receive" section** — explicit state/context passed to this skill
3. **"You produce" section** — explicit state/output this skill leaves for next skill
4. **"If X fails, do Y" section** — error recovery (retry, escalate, skip)

**Skills to update (execution order):**

1. **stellantis-automotive-feature-classification** (lead only)
   * Success: `W1-Status = complete`, `deliverable.csv generated`, all categories have ≥1 parameter
   * Receive: user request, 4 required fields
   * Produce: run workspace, STATE.md, W1 mode detection, KB dataset
   * Fails: can't resolve car identity → re-ask user

2. **stellantis-car-trim-resolution** (lead only)
   * Success: ambiguous car resolved to single trim + full option set, recorded in STATE
   * Receive: brand, model, year, market; existing trim guesses
   * Produce: resolved trim ID, option list, confidence level
   * Fails: multiple valid trims exist → post to user with visual comparison

3. **stellantis-source-discovery-with-client-domains** (lead only)
   * Success: ≥5 unique candidate URLs, no duplicates, validated reachable
   * Receive: car identity, client-provided domains list
   * Produce: candidate-list.md (source, snippet, relevance score)
   * Fails: < 5 URLs found → escalate to discovery-without-domains

4. **stellantis-source-discovery-without-client-domains** (lead only)
   * Success: ≥5 unique candidate URLs from open web (if needed)
   * Receive: car identity, optionally: failed domain list
   * Produce: appended candidate-list.md
   * Fails: still < 5 URLs → log advisory, proceed with available sources

5. **stellantis-source-validation** (lead only)
   * Success: all candidate URLs checked, marked reachable/stale/error
   * Receive: candidate-list.md
   * Produce: validated candidate-list.md (reachability status)
   * Fails: 50%+ URLs unreachable → log warning, proceed with available

6. **stellantis-source-download-and-ingest** (lead only)
   * Success: all validated URLs downloaded, indexed in KB, ingestion polling confirms searchable
   * Receive: validated candidate-list.md
   * Produce: KB dataset ID, doc\_info records with metadata
   * Fails: ingestion timeout → retry once, then log + proceed

7. **stellantis-lead-agent-subagent-orchestration** (lead only)
   * Success: all subagents spawned, all report final status, consolidation queue ordered
   * Receive: KB dataset, parameter partition plan (from automotive-feature-classification)
   * Produce: subagent roster with .harness/SubAgent/<name>.md files
   * Fails: subagent doesn't report → mark as timed-out, log advisories for missing parameters

8. **stellantis-subagent-classification-loop** (subagent only)
   * Success: for each assigned parameter, all tiers attempted (T1→T4), final verdict + evidence block
   * Receive: parameter name, KB dataset ID, subagent name, search budget
   * Produce: .harness/SubAgent/<name>.md with consolidated parameter records (all tiers, all searches logged)
   * Fails: KB timeout → use cached results, mark tier as "inconclusive", proceed to next parameter

### Phase 3: Add Error Handling & Context Bridges

**Error handling pattern** (add to each skill "If X fails" section):

* **Critical** (workflow can't proceed): lead escalates to user (e.g., car ID unresolvable)
* **High** (reduces data quality): log warning + advisory, proceed (e.g., <5 URLs, source unreachable)
* **Medium** (parameter-level): subagent retries tier N+1 or marks inconclusive (e.g., KB timeout)
* **Low** (internal): silently skip (e.g., malformed response, retry succeeds)

**Context bridges** (add to each skill "You receive/produce" section):

* **STATE.md fields** — what this skill reads, what it updates
* **.harness/ directories** — what files this skill reads from subagents, what it writes for next skill
* **KB API contracts** — what retrieve/list\_chunks/doc\_infos expect and return

### Phase 4: Create Validation Checklist

Add to lead-agent workflow (post-consolidation):

```
- [ ] All categories have ≥1 parameter record
- [ ] All parameter records have assigned tier (T1-T4)
- [ ] All records have evidence block + subagent name
- [ ] No "confidence" field is null
- [ ] deliverable.csv matches STATE.md counts
- [ ] All subagent status = consolidated or timed-out
- [ ] No orphaned .harness/SubAgent files
```

## Expected Outcomes

1. **Subagents:** Know exactly what success looks like before starting (tier definitions, evidence expectations, exit conditions)
2. **Lead agent:** Can track progress objectively (state changes, consolidation validation)
3. **Deliverables:** Automatically validate against schema (success criteria enforced)
4. **Errors:** Classified + handled per severity (no ambiguity about retry vs. escalate)
5. **First run:** Predictable, traceable, debuggable

## Scope & Constraints

* **In scope:** Optimize existing 8 skills + W1 workflow; no new workflows
* **Out of scope:** Add new skills, new workflow modes, change harness API contracts
* **Prerequisite:** No blocking changes; all templates/skills already exist
* **Testing:** Design assumes first real run will validate metrics; adjust based on observed failures

## Timeline

1. Audit templates → define metrics (\~2 hours review)
2. Update skill instructions (\~4-6 hours, 8 skills)
3. Create validation checklist (\~1 hour)
4. Commit design + updated skills
5. First run validation

***

**Next:** Writing plans skill will break this into implementation tasks.
