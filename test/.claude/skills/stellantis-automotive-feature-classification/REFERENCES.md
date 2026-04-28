# Reference Index

Canonical definitions for all shared assets referenced across the 8-skill framework. Skills and workflows use **reference IDs** (not file paths) to decouple from directory structure.

**Usage:** When a skill or document mentions a reference, use the ID (e.g., `enums-reference`) not the file path.

---

## Core References (Loaded by lead and subagent)

### enums-reference
- **Path:** `src/references/enums.md`
- **Purpose:** Single source of truth for all enum values (levels, categories, result types)
- **Who reads it:** Lead (preflight), all subagents (on spawn)
- **Format:** Markdown lists with category headers

### lead-to-subagent-contract
- **Path:** `src/references/lead-to-subagent-contract.md`
- **Purpose:** Canonical shape for contract JSON passed from lead to subagent
- **Who reads it:** Lead (stage 6), subagents (stage 6a on spawn)
- **Format:** Contract schema + JSON example

### subagent-to-lead-contract
- **Path:** `src/references/subagent-to-lead-contract.md`
- **Purpose:** Canonical shape for result envelope JSON returned by subagent
- **Who reads it:** Lead (consolidation), subagents (on spawn to understand expected output)
- **Format:** Result envelope schema + JSON example + traceability block schema

---

## State & Workspace References

### state-main-template
- **Path:** `src/templates/STATE.main.md.tmpl`
- **Purpose:** Template for lead's main STATE.md (run-level metadata, event log, summary counts)
- **Who reads it:** Lead (preflight stage 1)
- **Format:** Markdown template with placeholder sections

### state-category-template
- **Path:** `src/templates/STATE.category.md.tmpl`
- **Purpose:** Template for per-category consolidated records (`.harness/Category/<category>.md`)
- **Who reads it:** Lead (consolidation, stage 6b)
- **Format:** Markdown template with per-parameter record layout

### state-subagent-template
- **Path:** `src/templates/STATE.subagent.md.tmpl`
- **Purpose:** Template for subagent working files (`.harness/SubAgent/<agent-name>.md`)
- **Who reads it:** Lead (stage 6, pre-spawn), subagents (on spawn to populate)
- **Format:** Markdown with contract block + iteration sections + result envelope block

### workspace-layout-reference
- **Path:** `src/references/workspace-layout.md`
- **Purpose:** Directory structure, file ownership matrix, permissions (read/write per role)
- **Who reads it:** Lead (preflight)
- **Format:** Directory tree + ownership table

---

## Deliverable References

### deliverable-schema
- **Path:** `src/templates/deliverable.schema.json`
- **Purpose:** JSON schema for final deliverable.json (validation + field definitions)
- **Who reads it:** Lead (stage 7, emit-deliverable)
- **Format:** JSON Schema

### deliverable-csv-columns
- **Path:** `src/templates/deliverable.csv.columns.md`
- **Purpose:** Column definitions, enums, and formatting rules for deliverable.csv
- **Who reads it:** Lead (stage 7)
- **Format:** Markdown table + rules

### source-candidate-list-template
- **Path:** `src/templates/source-candidate-list.md.tmpl`
- **Purpose:** Template for source-candidate-list.md (discovery output, user-facing)
- **Who reads it:** Discovery skills (stage 3)
- **Format:** Markdown with table layout

---

## Workflow References

### W1-workflow-definition
- **Path:** `workflows/W1-normal-pipeline-mode-a/WORKFLOW.md`
- **Purpose:** Authoritative stage-by-stage definition of W1 (Normal Pipeline, Mode A + B)
- **Who reads it:** Lead (entire run), orchestration layer
- **Format:** Markdown with stage tables and detailed action steps

### W2-workflow-definition
- **Path:** `workflows/W2-discovery-fallback/WORKFLOW.md` (if exists)
- **Purpose:** Workflow for Mode B (open-web) discovery without user-supplied URLs
- **Who reads it:** Lead (if Mode A yields nothing, or as fallback)
- **Format:** Markdown with stage tables

---

## File Ownership & Contracts

### file-ownership-matrix
- **Path:** `src/references/file-ownership-matrix.md`
- **Purpose:** Table of all framework files: who reads, who writes, when
- **Who reads it:** Lead (preflight), developers (understanding permissions)
- **Format:** Markdown table (file path | lead read/write | subagent read/write | purpose)

---

## Summary

**Total reference IDs:** 11

| ID | File | Stage Used | Role |
|----|----|-----------|------|
| enums-reference | `src/references/enums.md` | preflight + stage 6a | lookup |
| lead-to-subagent-contract | `src/references/lead-to-subagent-contract.md` | stage 6 | contract shape |
| subagent-to-lead-contract | `src/references/subagent-to-lead-contract.md` | stage 6a + 6b | result shape |
| state-main-template | `src/templates/STATE.main.md.tmpl` | stage 1 | initialize |
| state-category-template | `src/templates/STATE.category.md.tmpl` | stage 6b | consolidate |
| state-subagent-template | `src/templates/STATE.subagent.md.tmpl` | stage 6 | pre-spawn |
| workspace-layout-reference | `src/references/workspace-layout.md` | stage 1 | understand structure |
| deliverable-schema | `src/templates/deliverable.schema.json` | stage 7 | validate output |
| deliverable-csv-columns | `src/templates/deliverable.csv.columns.md` | stage 7 | assemble CSV |
| source-candidate-list-template | `src/templates/source-candidate-list.md.tmpl` | stage 3 | format output |
| W1-workflow-definition | `workflows/W1-normal-pipeline-mode-a/WORKFLOW.md` | W1 entry | orchestration |
