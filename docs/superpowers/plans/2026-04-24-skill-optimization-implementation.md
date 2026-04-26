# Skill Optimization Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add success criteria sections to all templates, update 8 skill instructions with context bridges and error handling, create validation checklist. Enable first production run with clear metrics.

**Architecture:**

* Phase 1: Audit templates, add `## Success Criteria` sections (6 templates)
* Phase 2: Update each skill instruction with "Success Metric", "You receive", "You produce", "If X fails" sections (8 skills)
* Phase 3: Create consolidated validation checklist for lead agent
* Phase 4: Syntax validation + commit

**Tech Stack:** Markdown templates, JSON schema, shell scripting for validation

***

## Phase 1: Update Templates with Success Criteria

### Task 1: Add Success Criteria to STATE.main.md.tmpl

**Files:**

* Modify: `stellantis-automotive-feature-classification/templates/STATE.main.md.tmpl`

**Context:** Main state file is read by lead agent and consolidation. Add explicit success criteria for workflow completion.

* [ ] **Step 1: Read current template**

```Shell
cat stellantis-automotive-feature-classification/templates/STATE.main.md.tmpl
```

* [ ] **Step 2: Add Success Criteria section after Pointers section**

Append to end of file (before the closing comment if any):

```Markdown
## Success Criteria for Workflow Completion

State file is "done" (workflow complete) when ALL of:

1. **Run status** = `complete` (not `active` or `paused`)
2. **All categories** have `Consolidated = total` (not pending subagents)
3. **Subagent roster:** all agents have status `consolidated` or `timed-out` (no `in-progress`)
4. **Summary counts:**
   - `Status = Success` + `Status = Conflict` + `Status = Unable to Decide` = `Total parameters`
   - No `Not yet classified` remaining
   - No `Failed records` unless logged with reason in Event log
5. **Deliverable files exist:**
   - `runs/<<run-id>>/deliverable.json` (valid JSON schema)
   - `runs/<<run-id>>/deliverable.csv` (matches JSON counts)
6. **Event log** has final event: `<<final ISO>> · consolidation-complete · ...`

**If any criterion not met:** Workflow is not done. Log advisory in `.harness/advisories/`, update RunStatus to `paused`, set PauseReason, append event log, post to user.
```

* [ ] **Step 3: Verify syntax**

```Shell
cat stellantis-automotive-feature-classification/templates/STATE.main.md.tmpl | tail -20
```

Expected: Last section should be "Success Criteria for Workflow Completion" with numbered criteria.

* [ ] **Step 4: Commit**

```Shell
git add stellantis-automotive-feature-classification/templates/STATE.main.md.tmpl
git commit -m "docs(template): add success criteria to STATE.main.md.tmpl"
```

***

### Task 2: Add Success Criteria to STATE.category.md.tmpl

**Files:**

* Modify: `stellantis-automotive-feature-classification/templates/STATE.category.md.tmpl`

**Context:** Per-category consolidated state. Add criteria for "category consolidation complete".

* [ ] **Step 1: Read current template**

```Shell
cat stellantis-automotive-feature-classification/templates/STATE.category.md.tmpl
```

* [ ] **Step 2: Append Success Criteria section**

Add this after the final section:

```Markdown

## Success Criteria for Category Consolidation

Category STATE is "consolidated" when ALL of:

1. **All parameters** assigned (no empty parameter slots)
2. **All parameter records** have:
   - `presence` = one of [Yes, No, Disputed, No Information Found]
   - `status` = one of [Success, Conflict, Unable to Decide]
   - `classification` = one of [High, Medium, Basic, Empty] (or Empty if status ≠ Success)
   - `decision_rule` = one of [Rule-1 through Rule-5] matching (presence, status) from matrix
   - `traceability_blocks` = array with correct cardinality per rule
3. **Parameter records validate** against intermediate-parameter-record.json schema
4. **All subagent files** that touched this category have been:
   - Validated for completeness
   - Merged into this file
   - Moved to Archive
5. **Warnings** (if any) logged in "Warnings" section
6. **Category status** = `consolidated` (not `in-progress`)

**Lead agent consolidation step** must validate all criteria before moving category to consolidated state.
```

* [ ] **Step 3: Verify syntax**

```Shell
tail -30 stellantis-automotive-feature-classification/templates/STATE.category.md.tmpl
```

* [ ] **Step 4: Commit**

```Shell
git add stellantis-automotive-feature-classification/templates/STATE.category.md.tmpl
git commit -m "docs(template): add success criteria to STATE.category.md.tmpl"
```

***

### Task 3: Add Success Criteria to intermediate-parameter-record.json.tmpl

**Files:**

* Modify: `stellantis-automotive-feature-classification/templates/intermediate-parameter-record.json.tmpl`

**Context:** JSON schema for parameter records. Add description fields that define success for each field.

* [ ] **Step 1: Read current schema**

```Shell
cat stellantis-automotive-feature-classification/templates/intermediate-parameter-record.json.tmpl
```

* [ ] **Step 2: Add description to "required" array at top (clarify what "required" means)**

After the `"required": [...]` array, add new key `"_successMetrics": {`. Insert this JSON snippet after line 8 (after the required array):

```JSON
  "_successMetrics": {
    "parameter_id": "Must be non-empty string, unique within run",
    "category": "Must be one of categories in params.csv",
    "parameter_name": "Must match exactly one row in params.csv Optimized.Parameter.Name",
    "schema_type": "Must be one of [B/M/H, B/H, M/H, B/M, H, M, B] from params.csv",
    "presence": "Must be assigned before consolidation (one of enum, not null)",
    "status": "Must be assigned before consolidation (one of enum, not null)",
    "classification": "Must be Empty if status ≠ Success, else one of [High, Medium, Basic]",
    "decision_rule": "Must match (presence, status) combination per 10-decision-rules.md matrix",
    "traceability_blocks": "Cardinality rule: Rule-4 → zero, Rule-1/5 → ≥1, Rule-2a/2b → ≥2, Rule-3 → ≥1"
  },
```

Position it right after the required array and before additionalProperties.

* [ ] **Step 3: Validate JSON syntax**

```Shell
python3 -m json.tool stellantis-automotive-feature-classification/templates/intermediate-parameter-record.json.tmpl > /dev/null && echo "Valid JSON"
```

Expected: "Valid JSON" (no errors)

* [ ] **Step 4: Commit**

```Shell
git add stellantis-automotive-feature-classification/templates/intermediate-parameter-record.json.tmpl
git commit -m "docs(schema): add success metrics to parameter record schema"
```

***

### Task 4: Add Success Criteria to subagent-working-file.md.tmpl

**Files:**

* Modify: `stellantis-automotive-feature-classification/templates/subagent-working-file.md.tmpl`

**Context:** Subagent workspace. Add section defining what subagent should produce by end.

* [ ] **Step 1: Read current template**

```Shell
cat stellantis-automotive-feature-classification/templates/subagent-working-file.md.tmpl
```

* [ ] **Step 2: Append Success Criteria section**

Add after final section:

```Markdown

## Success Criteria for Subagent Completion

Subagent work is "complete" when ALL of:

1. **All assigned parameters** processed (none skipped or timed out without retry)
2. **For each parameter:**
   - At least one tier searched (T1 at minimum)
   - Final verdict assigned (presence + status + decision_rule)
   - Traceability block(s) recorded with evidence (chunk IDs, quotes, source sites)
3. **KB search attempts** logged (how many chunks retrieved, search terms)
4. **All tiers attempted** unless KB timeout (then mark as "inconclusive", move to next parameter)
5. **Parameter records** emit valid JSON per intermediate-parameter-record.json schema
6. **Subagent status** = `ready-for-consolidation` before exit
7. **No empty parameter slots** in final output

Lead agent will validate these before consolidating subagent's work into category STATE.
```

* [ ] **Step 3: Verify markdown syntax**

```Shell
tail -25 stellantis-automotive-feature-classification/templates/subagent-working-file.md.tmpl
```

* [ ] **Step 4: Commit**

```Shell
git add stellantis-automotive-feature-classification/templates/subagent-working-file.md.tmpl
git commit -m "docs(template): add success criteria to subagent-working-file.md.tmpl"
```

***

### Task 5: Add Success Criteria to traceability-block.md.tmpl

**Files:**

* Modify: `stellantis-automotive-feature-classification/templates/traceability-block.md.tmpl`

**Context:** Markdown version of evidence block. Add criteria for "valid traceability block".

* [ ] **Step 1: Read current template**

```Shell
cat stellantis-automotive-feature-classification/templates/traceability-block.md.tmpl
```

* [ ] **Step 2: Append Success Criteria section**

Add after final section:

```Markdown

## Success Criteria for Traceability Block

Block is valid when ALL of:

1. **Source site** present and matches one of sources downloaded in KB
2. **Chunk ID** present and valid (returned by KB retrieval, confirms document exists)
3. **Quote** present, non-empty, exactly matches text in chunk (no paraphrasing)
4. **Rationale** present, explains why this evidence supports the parameter decision
5. **Confidence** assigned (integer 0-100 or percentage)
6. **Tier** specified (T1-confirmed, T2-partial, T3-inferred, T4-not-found)
7. **Search term** logged (what query found this chunk)

Traceability blocks with missing required fields will fail consolidation validation.
```

* [ ] **Step 3: Verify markdown syntax**

```Shell
tail -20 stellantis-automotive-feature-classification/templates/traceability-block.md.tmpl
```

* [ ] **Step 4: Commit**

```Shell
git add stellantis-automotive-feature-classification/templates/traceability-block.md.tmpl
git commit -m "docs(template): add success criteria to traceability-block.md.tmpl"
```

***

### Task 6: Add Success Criteria to traceability-block.json.tmpl

**Files:**

* Modify: `stellantis-automotive-feature-classification/templates/traceability-block.json.tmpl`

**Context:** JSON schema for traceability blocks. Add success metric descriptions.

* [ ] **Step 1: Read current schema**

```Shell
cat stellantis-automotive-feature-classification/templates/traceability-block.json.tmpl
```

* [ ] **Step 2: Add \_successMetrics section after required array**

Insert this JSON after the `"required"` array:

```JSON
  "_successMetrics": {
    "source_site": "Non-empty string, URL domain or canonical name from KB download list. Must exist.",
    "chunk_id": "Valid KB chunk ID. Proves chunk exists and was indexed.",
    "quote": "Exact text match from chunk (no paraphrasing). Non-empty, ≤500 chars.",
    "rationale": "Explanation of why quote supports the parameter verdict. Non-empty, ≤200 chars.",
    "confidence": "Integer 0-100 or string percentage. Expresses strength of evidence.",
    "tier": "One of [T1-confirmed, T2-partial, T3-inferred, T4-not-found]. Must match parameter verdict.",
    "search_term": "Original query used to find this chunk. Enables reproducibility."
  },
```

Position right after the required array and before additionalProperties.

* [ ] **Step 3: Validate JSON syntax**

```Shell
python3 -m json.tool stellantis-automotive-feature-classification/templates/traceability-block.json.tmpl > /dev/null && echo "Valid JSON"
```

Expected: "Valid JSON"

* [ ] **Step 4: Commit**

```Shell
git add stellantis-automotive-feature-classification/templates/traceability-block.json.tmpl
git commit -m "docs(schema): add success metrics to traceability-block.json schema"
```

***

## Phase 2: Update Skill Instructions

### Task 7: Update stellantis-automotive-feature-classification SKILL.md

**Files:**

* Modify: `stellantis-automotive-feature-classification/stellantis-automotive-feature-classification/SKILL.md`

**Context:** Entry point skill. Already well-documented. Add "Success Metric" section and clarify context passing.

* [ ] **Step 1: Read current skill (first 200 lines)**

```Shell
head -200 stellantis-automotive-feature-classification/stellantis-automotive-feature-classification/SKILL.md
```

* [ ] **Step 2: Find section "## When to Use" and note its line number**

```Shell
grep -n "## When to Use" stellantis-automotive-feature-classification/stellantis-automotive-feature-classification/SKILL.md
```

* [ ] **Step 3: Insert "Success Metric" section after "When to Use" (before "Core Pattern")**

After the "If any of the four..." paragraph, add:

```Markdown
## Success Metric

This skill's output is successful when:

1. **Run workspace created** at `runs/<<run-id>>/` with correct directory structure
2. **STATE.md initialized** with all required header fields populated
3. **params.csv frozen** and hash recorded in STATE.md
4. **W1 workflow detected** (mode A if URLs supplied; else mode TBD)
5. **Run ID** generated and recorded in STATE.md
6. **Run status** = `active` (not `paused` or `failed`)

**Downstream:** Sub-skills receive this context and build on it. Ensure STATE.md is never inconsistent with actual workspace state.
```

* [ ] **Step 4: Find "### Roles & Responsibilities" section and add context bridge before it**

```Shell
grep -n "### Roles & Responsibilities" stellantis-automotive-feature-classification/stellantis-automotive-feature-classification/SKILL.md
```

Add before that section:

```Markdown
## Context Flow

**You receive (from user):**
- Brand, model, year, market (4 required fields) OR partial request
- Optional: workflow mode hint, KB hints, client-provided domains list

**You produce (for downstream sub-skills):**
- Run workspace (directory structure)
- STATE.md (main state file)
- Frozen params.csv (exact version used)
- KB dataset ID (passed to all downstream)
- Parameter partition plan (passed to orchestration skill)
- W1 workflow mode selection (passed to orchestration)

**Error handling:**
- **Critical:** Can't resolve 4 fields → ask user, don't proceed
- **High:** Invalid workflow mode → log, assume W1, proceed
- **Medium:** KB init fails → retry once, escalate if repeated
- **Low:** Workspace already exists → check consistency, reuse if valid
```

* [ ] **Step 5: Verify the file is still valid markdown**

```Shell
head -250 stellantis-automotive-feature-classification/stellantis-automotive-feature-classification/SKILL.md | tail -50
```

* [ ] **Step 6: Commit**

```Shell
git add stellantis-automotive-feature-classification/stellantis-automotive-feature-classification/SKILL.md
git commit -m "docs(skill): add success metric and context flow to automotive-feature-classification"
```

***

### Task 8: Update stellantis-car-trim-resolution SKILL.md

**Files:**

* Modify: `stellantis-car-trim-resolution/stellantis-car-trim-resolution/SKILL.md`

**Context:** Trim resolution skill (called by lead). Add success metric and error handling.

* [ ] **Step 1: Read first 100 lines**

```Shell
head -100 stellantis-car-trim-resolution/stellantis-car-trim-resolution/SKILL.md
```

* [ ] **Step 2: Find "## When to Use" section**

```Shell
grep -n "## When to Use" stellantis-car-trim-resolution/stellantis-car-trim-resolution/SKILL.md
```

* [ ] **Step 3: Add Success Metric and Context sections after "When to Use"**

Insert after the "When to Use" content (before next major section):

```Markdown
## Success Metric

Trim resolution is successful when:

1. **Trim ambiguity resolved** to single, named trim (or auto-selected with user confirmation)
2. **Full option set determined** (base trim options + parameter-specific options)
3. **Result recorded** in STATE.md under "Resolved full-option trim"
4. **Confidence level** recorded (e.g., "100% — exact match" or "85% — closest match from options")

If multiple valid trims exist and can't auto-select, escalate to user with visual comparison.

## Context Flow

**You receive (from lead):**
- Brand, model, year, market (guaranteed valid from preflight)
- Current STATE.md (may have trim guesses from user)
- Full params.csv (to extract relevant options)

**You produce (for lead to update STATE):**
- Resolved trim name (exact, canonical)
- Full option list (array of option names)
- Confidence level (percentage or qualitative)
- Auto-selected: yes/no flag

**Error handling:**
- **Critical:** No valid trim found → ask user to clarify (spelling, market name, year)
- **High:** Multiple valid trims (can't auto-select) → post visual comparison to user
- **Medium:** Trim options incomplete → log advisory, proceed with best-guess option set
- **Low:** Trim name formatting difference → normalize, log decision
```

* [ ] **Step 4: Verify markdown**

```Shell
head -150 stellantis-car-trim-resolution/stellantis-car-trim-resolution/SKILL.md | tail -50
```

* [ ] **Step 5: Commit**

```Shell
git add stellantis-car-trim-resolution/stellantis-car-trim-resolution/SKILL.md
git commit -m "docs(skill): add success metric and context flow to car-trim-resolution"
```

***

### Task 9: Update stellantis-source-discovery-with-client-domains SKILL.md

**Files:**

* Modify: `stellantis-source-discovery-with-client-domains/stellantis-source-discovery-with-client-domains/SKILL.md`

**Context:** Source discovery (with domains). Add success metric and error handling.

* [ ] **Step 1: Read first 100 lines**

```Shell
head -100 stellantis-source-discovery-with-client-domains/stellantis-source-discovery-with-client-domains/SKILL.md
```

* [ ] **Step 2: Add Success Metric and Context sections at top (after description, before other sections)**

Find where to insert (typically after "name:" and "description:" block):

```Markdown
## Success Metric

Source discovery with client domains succeeds when:

1. **Candidate list generated** with ≥5 unique, reachable URLs
2. **Each candidate** has: source site, snippet (excerpt), relevance score (0-100)
3. **Duplicates removed** (same domain, same content filtered)
4. **Reachability checked** (200-OK for all URLs, or marked as [stale]/[paywall]/[error])
5. **Candidate list** written to `runs/<<run-id>>/source-candidate-list.md`
6. **No fewer than 5 candidates** without escalating to discovery-without-domains

## Context Flow

**You receive (from lead):**
- Car identity: brand, model, year, market (confirmed)
- Client-provided domains list (array of domain strings)
- Existing STATE.md (run workspace initialized)

**You produce (for lead's source validation step):**
- Candidate list (markdown): source site | URL | snippet | relevance
- Reachability status per URL (OK, stale, paywall, error)
- Count of candidates (goal: ≥5)

**Error handling:**
- **Critical:** < 5 candidates found after exhausting client domains → escalate to discovery-without-domains
- **High:** 50%+ URLs unreachable → log warning, continue with reachable subset
- **Medium:** Duplicate content detected → deduplicate, keep highest-relevance instance
- **Low:** Snippet extraction fails → omit snippet, use URL title or manual description
```

* [ ] **Step 3: Verify file structure**

```Shell
head -120 stellantis-source-discovery-with-client-domains/stellantis-source-discovery-with-client-domains/SKILL.md | tail -30
```

* [ ] **Step 4: Commit**

```Shell
git add stellantis-source-discovery-with-client-domains/stellantis-source-discovery-with-client-domains/SKILL.md
git commit -m "docs(skill): add success metric and context flow to source-discovery-with-client-domains"
```

***

### Task 10: Update stellantis-source-discovery-without-client-domains SKILL.md

**Files:**

* Modify: `stellantis-source-discovery-without-client-domains/stellantis-source-discovery-without-client-domains/SKILL.md`

**Context:** Fallback source discovery (open web). Add success metric.

* [ ] **Step 1: Read first 100 lines**

```Shell
head -100 stellantis-source-discovery-without-client-domains/stellantis-source-discovery-without-client-domains/SKILL.md
```

* [ ] **Step 2: Add Success Metric and Context sections**

Insert after description:

```Markdown
## Success Metric

Source discovery without client domains succeeds when:

1. **Appends to existing candidate list** (or creates if first discovery attempt)
2. **New candidates added:** ≥5 additional URLs (cumulative list now ≥5)
3. **Each new candidate** validated for reachability + relevance
4. **Duplicates removed** (vs. previously discovered URLs)
5. **Candidate list updated** in `runs/<<run-id>>/source-candidate-list.md`
6. **Fallback flag recorded** in STATE.md event log

## Context Flow

**You receive (from lead):**
- Car identity (confirmed)
- Existing source-candidate-list.md (may be partial from with-domains attempt)
- STATE.md with event log
- Optional: list of failed domains from previous attempt

**You produce (for lead's validation step):**
- Updated candidate list (appended, deduplicated)
- Open-web search results (new candidates)
- Cumulative reachability status

**Error handling:**
- **Critical:** Still < 5 candidates after fallback → log advisory, proceed with available sources (may reduce deliverable quality)
- **High:** Search API rate-limited → retry after delay, then proceed with partial results
- **Medium:** Paywall-heavy results → log warning, filter paywalls, continue with free sources
- **Low:** Search query optimization → try alternate keywords if first attempt yields low relevance
```

* [ ] **Step 3: Verify file**

```Shell
head -120 stellantis-source-discovery-without-client-domains/stellantis-source-discovery-without-client-domains/SKILL.md | tail -30
```

* [ ] **Step 4: Commit**

```Shell
git add stellantis-source-discovery-without-client-domains/stellantis-source-discovery-without-client-domains/SKILL.md
git commit -m "docs(skill): add success metric and context flow to source-discovery-without-client-domains"
```

***

### Task 11: Update stellantis-source-validation SKILL.md

**Files:**

* Modify: `stellantis-source-validation/stellantis-source-validation/SKILL.md`

**Context:** Source validation. Add success metric and clarity.

* [ ] **Step 1: Read first 100 lines**

```Shell
head -100 stellantis-source-validation/stellantis-source-validation/SKILL.md
```

* [ ] **Step 2: Add Success Metric and Context sections**

Insert after description:

```Markdown
## Success Metric

Source validation succeeds when:

1. **All candidate URLs checked** (reachability test: HTTP HEAD or GET)
2. **Status assigned** to each URL (OK | stale | paywall | error | malformed)
3. **Validated list** written to `runs/<<run-id>>/source-candidate-list-validated.md`
4. **Count of reachable sources** recorded in STATE.md
5. **Dropped sources documented** in `.harness/Archive/sources_excluded.md` (why excluded)

## Context Flow

**You receive (from lead):**
- Candidate list from source-discovery skills (unvalidated)
- STATE.md

**You produce (for lead's download step):**
- Validated candidate list (with reachability status)
- Excluded sources list (paywall, stale, error)
- Count of valid URLs remaining

**Error handling:**
- **Critical:** > 50% URLs unreachable → log warning in STATE advisories, continue with valid subset
- **High:** URL malformed → skip, log to excluded list
- **Medium:** Timeout on single URL → mark as [timeout], continue (will retry during download)
- **Low:** Redirect chains → follow up to 5 hops, use final URL
```

* [ ] **Step 3: Verify file**

```Shell
head -120 stellantis-source-validation/stellantis-source-validation/SKILL.md | tail -30
```

* [ ] **Step 4: Commit**

```Shell
git add stellantis-source-validation/stellantis-source-validation/SKILL.md
git commit -m "docs(skill): add success metric and context flow to source-validation"
```

***

### Task 12: Update stellantis-source-download-and-ingest SKILL.md

**Files:**

* Modify: `stellantis-source-download-and-ingest/stellantis-source-download-and-ingest/SKILL.md`

**Context:** Download and ingest. Add success metric.

* [ ] **Step 1: Read first 150 lines**

```Shell
head -150 stellantis-source-download-and-ingest/stellantis-source-download-and-ingest/SKILL.md
```

* [ ] **Step 2: Add Success Metric and Context sections**

Insert after description:

```Markdown
## Success Metric

Download and ingestion succeeds when:

1. **All validated sources downloaded** (no errors on download)
2. **All downloaded docs uploaded** to KB via run_document API
3. **Ingestion polling confirms** all docs are searchable (query test: retrieve by doc ID)
4. **KB dataset ID** confirmed and recorded in STATE.md
5. **Document metadata** recorded (source site, download date, content hash)
6. **Dropped sources** (download failures) documented in `.harness/Archive/sources_excluded.md`

## Context Flow

**You receive (from lead):**
- Validated candidate list (URLs marked reachable)
- KB dataset ID (from automotive-feature-classification preflight)
- STATE.md

**You produce (for lead's orchestration step):**
- KB dataset with all docs indexed
- Document info records (doc_id, source_site, retrieval_date)
- Count of successfully ingested docs

**Error handling:**
- **Critical:** Ingestion timeout > 60 sec → retry once, then escalate (proceed with partial KB)
- **High:** 10%+ downloads fail → log failed sources, continue with successful downloads
- **Medium:** Single doc fails to ingest → log, mark in metadata, retry on next poll
- **Low:** Content extraction issues → ingest as-is, log for manual review
```

* [ ] **Step 3: Verify file**

```Shell
head -180 stellantis-source-download-and-ingest/stellantis-source-download-and-ingest/SKILL.md | tail -30
```

* [ ] **Step 4: Commit**

```Shell
git add stellantis-source-download-and-ingest/stellantis-source-download-and-ingest/SKILL.md
git commit -m "docs(skill): add success metric and context flow to source-download-and-ingest"
```

***

### Task 13: Update stellantis-lead-agent-subagent-orchestration SKILL.md

**Files:**

* Modify: `stellantis-lead-agent-subagent-orchestration/stellantis-lead-agent-subagent-orchestration/SKILL.md`

**Context:** Orchestration (most complex). Add success metric and state tracking clarity.

* [ ] **Step 1: Read first 200 lines**

```Shell
head -200 stellantis-lead-agent-subagent-orchestration/stellantis-lead-agent-subagent-orchestration/SKILL.md
```

* [ ] **Step 2: Find key sections (look for concurrency or spawn logic)**

```Shell
grep -n "concurr\|spawn\|monitor" stellantis-lead-agent-subagent-orchestration/stellantis-lead-agent-subagent-orchestration/SKILL.md | head -10
```

* [ ] **Step 3: Add Success Metric and Context sections after description**

Insert after initial description:

```Markdown
## Success Metric

Orchestration succeeds when:

1. **All subagents spawned** (max 3 concurrent)
2. **Parameter partitioning** complete (each category → list of parameters)
3. **All subagents report status** (no hung agents)
4. **Consolidation queue ordered** (serial, one category at a time)
5. **STATE.md updated** with subagent roster and timestamps
6. **No subagents in "in-progress"** (all are "ready" or "timed-out")
7. **Transition to consolidation** triggered by lead agent

## Context Flow

**You receive (from lead):**
- KB dataset ID (confirmed searchable)
- Parameter partition plan (from automotive-feature-classification)
- STATE.md (ready for subagent roster)
- Budget: 3 concurrent subagents max

**You produce (for lead's consolidation step):**
- Subagent roster in STATE.md (spawned date, status, completed date)
- .harness/SubAgent/<name>.md files (one per subagent, written by subagent)
- Consolidation queue (ordered by category)

**Error handling:**
- **Critical:** Subagent doesn't report after timeout (10 min) → mark as timed-out, log advisory, continue consolidation without its results
- **High:** Spawn fails → retry once, then escalate to user (deploy issue?)
- **Medium:** Subagent crashes mid-run → detect via status check, move to timed-out, allow consolidation to skip or retry
- **Low:** Status update delayed → poll again after 30s
```

* [ ] **Step 4: Verify file**

```Shell
head -250 stellantis-lead-agent-subagent-orchestration/stellantis-lead-agent-subagent-orchestration/SKILL.md | tail -40
```

* [ ] **Step 5: Commit**

```Shell
git add stellantis-lead-agent-subagent-orchestration/stellantis-lead-agent-subagent-orchestration/SKILL.md
git commit -m "docs(skill): add success metric and context flow to lead-agent-subagent-orchestration"
```

***

### Task 14: Update stellantis-subagent-classification-loop SKILL.md

**Files:**

* Modify: `stellantis-subagent-classification-loop/stellantis-subagent-classification-loop/SKILL.md`

**Context:** Subagent classification (core logic). Add detailed success metric and exit conditions.

* [ ] **Step 1: Read first 150 lines**

```Shell
head -150 stellantis-subagent-classification-loop/stellantis-subagent-classification-loop/SKILL.md
```

* [ ] **Step 2: Find loop/tier sections**

```Shell
grep -n "tier\|loop\|T1\|T2\|T3\|T4" stellantis-subagent-classification-loop/stellantis-subagent-classification-loop/SKILL.md | head -15
```

* [ ] **Step 3: Add Success Metric and Context sections**

Insert after description:

```Markdown
## Success Metric

Subagent classification is successful when:

1. **All assigned parameters** processed (no skipped parameters)
2. **For each parameter:**
   - Tier assignment (one of T1-confirmed, T2-partial, T3-inferred, T4-not-found)
   - Presence (Yes, No, Disputed, No Information Found)
   - Status (Success, Conflict, Unable to Decide)
   - Classification (High, Medium, Basic, Empty)
   - Decision rule (Rule-1 through Rule-5, matching presence+status)
   - Traceability block(s) with evidence (chunk ID, quote, source, confidence)
3. **Search attempts logged** (tier, search term, chunks retrieved, selected evidence)
4. **KB timeouts handled** gracefully (mark as "inconclusive", move to next parameter)
5. **Parameter records validate** against intermediate-parameter-record.json schema
6. **Subagent status** = `ready-for-consolidation` before exit
7. **Output written** to `.harness/SubAgent/<<agent-name>>.md`

## Context Flow

**You receive (from orchestration):**
- Subagent name (from file path stem)
- List of parameters to classify (one or more)
- KB dataset ID (searchable)
- Search budget (main loop: 3 iterations, fallback loop: 3 iterations)

**You produce (for lead's consolidation):**
- .harness/SubAgent/<name>.md with:
  - Contract header (JSON-formatted input parameters)
  - Scratch workspace (search attempts, decisions, evidence collection)
  - Final parameter records array (valid JSON per schema)
- Status: `ready-for-consolidation` or `timed-out`

**Error handling:**
- **Critical:** KB unreachable → try fallback KB, then mark parameters as "inconclusive" (T4)
- **High:** KB timeout on single parameter → use T1 results if available, move to next, log timeout
- **Medium:** No evidence found for a tier → skip tier, try next (T1→T2→T3→T4 progression)
- **Low:** Confidence score ambiguous → assign 50, log reasoning, proceed
- **Parameter timeout:** If > 5 min elapsed on one parameter, stop, mark as "timed-out", emit ready status
```

* [ ] **Step 4: Verify file**

```Shell
head -200 stellantis-subagent-classification-loop/stellantis-subagent-classification-loop/SKILL.md | tail -40
```

* [ ] **Step 5: Commit**

```Shell
git add stellantis-subagent-classification-loop/stellantis-subagent-classification-loop/SKILL.md
git commit -m "docs(skill): add success metric and context flow to subagent-classification-loop"
```

***

## Phase 3: Create Validation Checklist

### Task 15: Create Workflow Validation Checklist

**Files:**

* Create: `stellantis-automotive-feature-classification/WORKFLOW-VALIDATION-CHECKLIST.md`

**Context:** Consolidated checklist for lead agent to use post-consolidation and before declaring workflow complete.

* [ ] **Step 1: Create file**

```Shell
cat > stellantis-automotive-feature-classification/WORKFLOW-VALIDATION-CHECKLIST.md << 'EOF'
# Workflow W1 Validation Checklist

## Pre-Consolidation (Lead Agent Checkpoint)

Use this checklist **before** starting consolidation to verify workflow state is consistent.

- [ ] **Subagent Roster Complete**
  - All spawned subagents have entries in STATE.md subagent roster
  - All spawned subagents report final status (not "in-progress")
  - Max 3 concurrent subagents (rule enforced by orchestration)

- [ ] **Subagent Files Exist**
  - For each subagent in roster, `.harness/SubAgent/<<name>>.md` exists
  - Each file has contract header (JSON input parameters)
  - Each file has parameter records array (valid JSON)

- [ ] **Parameter Records Pre-Check**
  - Sample 3 random parameter records
  - Each record has: parameter_id, category, presence, status, classification, decision_rule
  - Each record has: traceability_blocks array with ≥1 block (Rule-4 exception: 0 blocks)

- [ ] **KB Indexing Complete**
  - KB dataset ID in STATE.md confirms searchable
  - All downloaded sources appear in KB metadata
  - Test retrieval: query one KB doc by ID → confirms searchable

- [ ] **State Consistency**
  - STATE.main.md RunStatus = `active` (or `paused` with reason if intentional)
  - All category rosters in STATE show "in-progress" or "ready"
  - Event log has entries through orchestration phase

## During Consolidation (Per-Category Checkpoint)

Use this checklist for each category as you consolidate.

- [ ] **Parameter Records Valid**
  - All records in subagent file validate against intermediate-parameter-record.json schema
  - No missing required fields
  - decision_rule matches (presence, status) per matrix in 10-decision-rules.md

- [ ] **Traceability Blocks Valid**
  - Each block has: source_site, chunk_id, quote, rationale, confidence, tier, search_term
  - Quote exactly matches text in KB chunk (test: retrieve chunk, verify quote)
  - Chunk IDs are valid KB IDs (not made-up strings)

- [ ] **All Tiers Attempted**
  - For each parameter, log shows at least T1 search attempt
  - Final tier reflects search results (T1-confirmed if found, T2-partial if partial, T3-inferred if inferred, T4-not-found if absent)
  - Tier matches classification (non-Empty only if T1-confirmed with status=Success)

- [ ] **No Orphaned Parameters**
  - All parameters assigned to category are represented in subagent output
  - No empty slots or skipped parameters

- [ ] **Consolidation Output Valid**
  - Category STATE file contains merged records from all subagents
  - Category STATE has success criteria checklist (from template) all checked
  - Category marked as "consolidated" or with clear reason if not

## Post-Consolidation (Lead Agent Final Checkpoint)

Use this checklist **after all categories consolidated** to declare workflow complete.

- [ ] **All Categories Consolidated**
  - All categories in category roster show Consolidated = total (not partial)
  - No categories in "in-progress" state
  - All subagent files moved to Archive

- [ ] **Summary Counts Accurate**
  - Presence counts: Yes + No + Disputed + No Information Found = Total parameters
  - Status counts: Success + Conflict + Unable to Decide = Total parameters
  - Classification counts: High + Medium + Basic + Empty = Total parameters (Empty ≥ Non-Success count)
  - No "Not yet classified" remaining
  - No "Failed records" unless documented in event log

- [ ] **Deliverables Generated**
  - `runs/<<run-id>>/deliverable.json` exists and valid JSON
  - `runs/<<run-id>>/deliverable.csv` exists and valid CSV
  - JSON and CSV counts match STATE.md summary counts

- [ ] **Run Status Updated**
  - STATE.main.md RunStatus = `complete`
  - Last-run date updated (UTC ISO-8601)
  - Event log final entry: consolidation-complete timestamp

- [ ] **Archive Complete**
  - All `.harness/SubAgent/` files moved to `.harness/Archive/`
  - No live subagent files remaining
  - `.harness/Archive/sources_excluded.md` documents all dropped sources

- [ ] **Advisories Logged**
  - Any subagent timeouts recorded in `.harness/advisories/`
  - Any KB timeouts recorded in `.harness/advisories/`
  - Any ambiguous parameters recorded in `.harness/advisories/`

## Quality Metrics (For Retrospective)

After workflow complete, record these metrics for optimization feedback:

| Metric | Expected | Actual | Notes |
|:-------|:---------|:-------|:------|
| Total parameters assigned | ≥5 | _ | |
| Parameters tier T1-confirmed | ≥50% | _ | |
| Parameters tier T2-partial | ≤30% | _ | |
| Parameters tier T3-inferred | ≤20% | _ | |
| Subagents spawned | 1-3 | _ | |
| Subagents timed-out | 0 | _ | |
| KB timeout incidents | 0 | _ | |
| Source download success rate | ≥80% | _ | |
| Deliverable generation time | <5 min | _ | |

---

**How to use this checklist:**
1. Print or copy to clipboard
2. Before starting consolidation: run pre-consolidation checks
3. For each category: run per-category checks
4. After all consolidated: run final checks
5. Record quality metrics
6. Fix any failed checks before declaring workflow complete
EOF
cat stellantis-automotive-feature-classification/WORKFLOW-VALIDATION-CHECKLIST.md
```

* [ ] **Step 2: Verify file created and readable**

```Shell
wc -l stellantis-automotive-feature-classification/WORKFLOW-VALIDATION-CHECKLIST.md
```

Expected: \~200+ lines

* [ ] **Step 3: Commit**

```Shell
git add stellantis-automotive-feature-classification/WORKFLOW-VALIDATION-CHECKLIST.md
git commit -m "docs: add workflow validation checklist for W1"
```

***

## Phase 4: Verify All Changes

### Task 16: Syntax Validation & Summary

**Files:**

* All modified templates and skills

**Context:** Final verification that all Markdown and JSON is valid.

* [ ] **Step 1: Validate all modified JSON templates**

```Shell
python3 << 'PYEOF'
import json
import glob

json_files = glob.glob("stellantis-automotive-feature-classification/templates/*.json.tmpl")
for f in json_files:
    try:
        with open(f) as fp:
            json.load(fp)
        print(f"✓ {f}")
    except json.JSONDecodeError as e:
        print(f"✗ {f}: {e}")
PYEOF
```

Expected: All files show ✓

* [ ] **Step 2: Validate all modified Markdown files (check for common errors)**

```Shell
for f in $(find . -name "*.md" -path "*/SKILL.md" -o -path "*WORKFLOW-VALIDATION*" -o -path "*/templates/*.md.tmpl"); do
  if [ -f "$f" ]; then
    if grep -q "<<\|TBD\|TODO\|implement later" "$f" 2>/dev/null; then
      echo "⚠ $f: contains placeholders"
    else
      echo "✓ $f"
    fi
  fi
done
```

Expected: All files show ✓

* [ ] **Step 3: Count updated files**

```Shell
echo "=== Phase 1: Templates ===" && \
ls -1 stellantis-automotive-feature-classification/templates/*.tmpl && \
echo && \
echo "=== Phase 2: Skills ===" && \
find . -name "SKILL.md" -path "*/stellantis-*/*" && \
echo && \
echo "=== Phase 3: Checklist ===" && \
ls -1 stellantis-automotive-feature-classification/WORKFLOW-VALIDATION-CHECKLIST.md
```

Expected: 8 templates + 8 SKILL files + 1 checklist = 17 files updated/created

* [ ] **Step 4: View final git status**

```Shell
git status | head -40
```

Expected: All modified files listed with "M" or "??" status

* [ ] **Step 5: Commit any final changes (if any)**

```Shell
git add -A && git status
```

* [ ] **Step 6: Create summary commit**

```Shell
git commit --allow-empty -m "docs: complete skill optimization — templates, skills, checklist

Phase 1: Added success criteria to 6 templates (STATE.main, STATE.category, parameter record schema, subagent working file, traceability blocks)
Phase 2: Added success metrics + context flow + error handling to 8 skills
Phase 3: Created workflow validation checklist for W1
Phase 4: Verified all syntax (JSON schema, Markdown)

Ready for first production run with clear metrics."
```

* [ ] **Step 7: Verify log**

```Shell
git log --oneline -10
```

Expected: Latest commit shows the optimization summary

***

## Verification

All tasks complete when:

1. ✅ All 6 templates updated with success criteria sections
2. ✅ All 8 skills updated with success metric, context flow, error handling
3. ✅ Workflow validation checklist created and committed
4. ✅ All JSON and Markdown syntax valid (no placeholders)
5. ✅ All changes committed to git

**Next:** First production run using W1 workflow. Monitor against success criteria defined in templates + skills.
