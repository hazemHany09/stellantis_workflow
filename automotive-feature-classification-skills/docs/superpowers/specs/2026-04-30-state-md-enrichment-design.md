# Design Spec — STATE.md Enrichment

**Date:** 2026-04-30
**Topic:** Enriching `.harness/STATE.md` with richer operational data
**Status:** Approved

---

## Problem Statement

The current `STATE.main.md.tmpl` covers basic run metadata, source funnel counts, download tracking, and classification roster. The framework has since grown a two-round classification architecture (Round 1 search-mode + Round 2 deep-dive), but STATE.md has no sections for Round 2 subagents, gap parameter tracking, or inter-round hand-off data.

Three resume scenarios are currently under-served:

1. **Mid-classification resume** — agent wakes mid-R1 or mid-R2 and must reconstruct which subagents are live, queued, or gap-assigned
2. **Post-crash recovery** — agent restarts cold and must reconstruct full orchestration state without reading category files or subagent files
3. **Inter-round hand-off** — agent finishes R1 consolidation and must decide R2 targets without reading `.harness/document-promise-board.md`

---

## Design Decisions

| Decision | Choice | Rationale |
|:---------|:-------|:----------|
| Primary audience | Agent-first | STATE is the resume store; human readability is secondary |
| Resume scenarios covered | All three (mid-classification, post-crash, inter-round) | STATE must be the single source of truth |
| Format | Hybrid | Scalar fields + counts = markdown tables; complex structured data = embedded JSON blocks |
| Architecture | Tiered sections (T1/T2) | Makes resume-critical invariants explicit; future additions default to T2 |

---

## Tier Definitions

**Tier 1 — Operational (resume-critical).** Agent MUST write these sections before any pause or stage transition. A resume from any state must be possible using T1 sections alone.

**Tier 2 — Telemetry/Audit (informational).** Agent writes at stage completion. Not required for resume. A crash mid-T2 write does not corrupt resumability.

---

## Full Section Ordering

| # | Section | Tier | Write trigger |
|:--|:--------|:-----|:--------------|
| 1 | Header | T1 | Preflight — once |
| 2 | Summary counts + confidence distribution | T1 + T2 sub-table | After each consolidation pass |
| 3 | Source stats | T1 | After each source lifecycle change |
| 4 | Source download tracking | T1 | After each download subagent reports |
| 5 | Per-doc ingestion status | T1 | Phase 5 upload; updated at Stage 5 verify |
| 6 | KB dataset snapshot | T2 | Stage 5 success — once |
| 7 | Category roster | T1 | After each consolidation |
| 8 | Subagent roster (Round 1) | T1 | At spawn + status changes |
| 9 | Round 2 subagent roster | T1 | Written empty at R1 consolidation start; populated at R2 spawn |
| 10 | Gap parameters + R2 target assignments | T1 | Once at R1→R2 hand-off; never edited after |
| 11 | Stage timing telemetry | T2 | Stage entry + exit |
| 12 | Warnings and advisories summary | T2 | After each consolidation pass |
| 13 | Deduplication log | T2 | Phase 5b — once; append-only |
| 14 | Loaded skills | T2 | As skills load |
| 15 | Decisions made | T2 | Any notable decision |
| 16 | Event log | T1 | Append-only; every step |
| 17 | Pointers | T2 | Preflight — once |

---

## New Sections — Detailed Specifications

### Section 5 — [T1] Per-doc ingestion status

**Purpose:** Post-crash recovery. Agent can immediately see which doc_ids exist in the KB and their ingestion state without calling `doc_infos`.

**Write trigger:** Row written at upload (Phase 5). Status column updated at ingestion verify (Stage 5).

```markdown
## [T1] Per-doc ingestion status

| doc_id | canonical_url | slug | source_type | upload_timestamp | ingestion_status | ingestion_completed_at | failure_reason |
| :----- | :------------ | :--- | :---------- | :--------------- | :--------------- | :--------------------- | :------------- |
| `<<id>>` | `<<url>>` | `<<slug>>` | `<<type>>` | `<<ISO>>` | `pending | running | done | failed | dropped` | `<<ISO or —>>` | `<<reason or —>>` |
```

**Status values:** `pending` → `running` → `done` | `failed` | `dropped`

---

### Section 6 — [T2] KB dataset snapshot

**Purpose:** Audit. Captures KB state at classification start — doc count, source type distribution.

**Write trigger:** Once, immediately after Stage 5 success gate passes.

```markdown
## [T2] KB dataset snapshot

| Field | Value |
| :---- | :---- |
| Dataset ID | `<<id>>` |
| Documents ingested | `<<n>>` |
| Documents failed / dropped | `<<n>>` |
| Snapshot timestamp | `<<ISO>>` |

### Source type distribution

| Source type | Doc count |
| :---------- | :-------- |
| `manufacturer_official_spec` | `<<n>>` |
| `manufacturer_owner_manual` | `<<n>>` |
| `dealer_fleet_guide` | `<<n>>` |
| `third_party_press_long_form` | `<<n>>` |
| `third_party_aggregator` | `<<n>>` |
| `manufacturer_video` | `<<n>>` |
| `forum_or_ugc` | `<<n>>` |
```

---

### Section 9 — [T1] Round 2 subagent roster

**Purpose:** Mid-classification resume. Agent reconstructs which R2 deep-dive subagents are live, queued, or archived without reading SubAgent files.

**Write trigger:** Empty section (header + empty table) written at R1 consolidation start. Rows populated at R2 spawn. Status updated as subagents complete.

```markdown
## [T1] Round 2 subagent roster (deep-dive)

| Agent name | Target doc_id | Target doc slug | Gap parameters | Status | Spawned | Completed |
| :--------- | :------------ | :-------------- | :------------- | :----- | :------ | :-------- |
| `<<name>>` | `<<doc_id>>` | `<<slug>>` | `<<param_ids, comma-sep>>` | `<<SubagentStatus>>` | `<<ISO>>` | `<<ISO>>` |
```

---

### Section 10 — [T1] Gap parameters + R2 target assignments

**Purpose:** Inter-round hand-off. Agent reconstructs full R2 plan from this block alone — no need to read `.harness/document-promise-board.md`.

**Write trigger:** Once, at R1→R2 hand-off (lead consolidation pass A). Never edited after. If R2 cap is adjusted mid-run, append a note to event log only.

**Format:** Embedded JSON block.

```markdown
## [T1] Gap parameters and R2 target assignments

Written once at Round 1 → Round 2 hand-off. Read-only after this point.

```json
{
  "gap_parameters": [
    {
      "param_id": "<<id>>",
      "param_name": "<<name>>",
      "category": "<<cat>>",
      "round1_rule": "<<Rule 3 | Rule 4 | Rule 1 | Rule 5>>",
      "round1_confidence": "<<single-source | null>>",
      "gap_reason": "<<rule4-no-evidence | rule3-vague | single-source-corroboration-needed>>",
      "assigned_to_agent": "<<agent-name or null>>"
    }
  ],
  "r2_target_assignments": [
    {
      "agent_name": "<<name>>",
      "target_doc_id": "<<id>>",
      "target_doc_slug": "<<slug>>",
      "target_doc_local_path": "/mnt/user-data/workspace/.harness/downloads/<<slug>>.<ext>",
      "gap_param_ids": ["<<id1>>", "<<id2>>"],
      "promise_score": 0.0,
      "selection_rationale": "<<short string>>"
    }
  ],
  "gap_params_without_r2_target": ["<<param_id>>"],
  "r2_cap_reached": false,
  "generated_at": "<<ISO>>"
}
```
```

---

### Section 11 — [T2] Stage timing telemetry

**Purpose:** Audit. Diagnose slow runs without reading event log.

**Write trigger:** `started_at` written at stage entry; `completed_at` at stage exit.

```markdown
## [T2] Stage timing telemetry

| Stage | Started at | Completed at | Duration (s) |
| :---- | :--------- | :----------- | :----------- |
| `preflight` | `<<ISO>>` | `<<ISO>>` | `<<n>>` |
| `parse-request` | `<<ISO>>` | `<<ISO>>` | `<<n>>` |
| `resolve-trim` | `<<ISO>>` | `<<ISO>>` | `<<n>>` |
| `source-discovery` | `<<ISO>>` | `<<ISO>>` | `<<n>>` |
| `download-and-ingest` | `<<ISO>>` | `<<ISO>>` | `<<n>>` |
| `classification-round-1` | `<<ISO>>` | `<<ISO>>` | `<<n>>` |
| `classification-round-2` | `<<ISO>>` | `<<ISO>>` | `<<n>>` |
| `emit-deliverable` | `<<ISO>>` | `<<ISO>>` | `<<n>>` |
```

---

### Section 12 — [T2] Warnings and advisories summary

**Purpose:** Audit. Quick count of quality signals without reading category files.

**Write trigger:** After each consolidation pass (Round 1 and Round 2).

```markdown
## [T2] Warnings and advisories summary

| Bucket | Count |
| :----- | :---- |
| Run-level warnings | `<<n>>` |
| Category-level warnings | `<<n>>` |
| Rule 2a conflicts (Yes, Conflict) | `<<n>>` |
| Rule 2b conflicts (Disputed, Conflict) | `<<n>>` |
| Undefined-tier advisories | `<<n>>` |
| Out-of-list findings | `<<n>>` |
| R2 cap reached — unresolved gap params | `<<n>>` |
```

---

### Section 13 — [T2] Deduplication log

**Purpose:** Audit trail for Phase 5b. Empty table = explicit proof dedup ran and found nothing.

**Write trigger:** Phase 5b — once. Append-only.

```markdown
## [T2] Deduplication log

| Kept doc_id | Deleted doc_id | canonical_url | Reason |
| :---------- | :------------- | :------------ | :----- |
| `<<id>>` | `<<id>>` | `<<url>>` | `duplicate-upload` |
```

---

### Addition to Section 2 — Confidence distribution sub-table

Added immediately below existing summary counts table. Same write cadence.

```markdown
### Confidence distribution

| Bucket | Count |
| :----- | :---- |
| Confidence = consensus | `<<n>>` |
| Confidence = single-source | `<<n>>` |
| Round 2 upgrades (single-source → consensus) | `<<n>>` |
| Round 2 verdict changes (rule promoted) | `<<n>>` |
| Rule 4 records remaining after Round 2 | `<<n>>` |
```

---

## Write Invariants

1. **T1 sections must be written before any pause.** Setting `RunStatus = paused` requires all T1 sections to reflect current state first.
2. **T1 sections must be written before stage transition.** Changing `RunStage` = T1 sections flushed first.
3. **T2 sections are best-effort.** A crash mid-T2 write does not corrupt resumability.
4. **Gap parameters + R2 target assignments is write-once.** Never edited after R1→R2 hand-off.
5. **Round 2 subagent roster section must exist (even empty) before first R2 subagent spawns.** Prevents a resuming agent from inferring R2 hasn't started due to missing section.
6. **Deduplication log written even when zero duplicates found.** Empty table = explicit proof dedup ran.
7. **Per-doc ingestion status rows written at upload, not at ingestion trigger.** Agent resuming post-crash sees which doc_ids exist in the KB immediately.

---

## Resume Scenario Coverage

| Scenario | Sections required |
|:---------|:------------------|
| Mid-classification resume (R1 or R2) | Header, Subagent roster (R1), Round 2 subagent roster, Gap parameters + R2 assignments, Category roster, Event log |
| Post-crash recovery | All T1 sections |
| Inter-round hand-off | Gap parameters + R2 target assignments (JSON block alone is sufficient) |

---

## Files to Update

1. `stellantis-automotive-feature-classification/templates/STATE.main.md.tmpl` — add new sections
2. `stellantis-run-workspace/SKILL.md` — update State files section + layout tree
3. `stellantis-lead-agent-subagent-orchestration/SKILL.md` — update Final STATE.md update section; add R2 roster + gap block write rules
4. `stellantis-source-download-and-ingest/SKILL.md` — add per-doc ingestion status write rules (Phase 5 + Phase 5b)
5. `stellantis-automotive-feature-classification/SKILL.md` — update Quick Reference > STATE Files section
