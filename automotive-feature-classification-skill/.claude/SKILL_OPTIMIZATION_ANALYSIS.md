# Skill Optimization Analysis

## Current Frontmatter Status

**Present in all 8 skills:**
- `name` ✓
- `description` ✓
- `loader` ✓ (lead or subagent)

**Missing — CRITICAL for cross-skill navigation:**
- `requires` — explicit list of skill dependencies
- `provides` — key deliverable(s)
- `stage` — workflow stage identifier(s)

---

## Skill Dependency Graph (Logical Connections)

```
stella-automotive-feature-classification (MAIN ENTRY)
│
├─ Stage 1b: resolve-trim
│  └── stellantis-car-trim-resolution
│      ├─ requires: none
│      └─ provides: resolved_trim, variant_note
│
├─ Stage 3: source-discovery
│  ├─ Mode A (client URLs/domains)
│  │  └── stellantis-source-discovery-with-client-domains
│  │      ├─ requires: none
│  │      └─ provides: source-candidate-list.md (origin=client)
│  │
│  ├─ Mode B (open-web search)
│  │  └── stellantis-source-discovery-without-client-domains
│  │      ├─ requires: none
│  │      └─ provides: source-candidate-list.md (origin=agent)
│  │
│  └─ Stage 3b: validation (after discovery)
│     └── stellantis-source-validation
│         ├─ requires: [source-discovery-with-client-domains OR source-discovery-without-client-domains]
│         └─ provides: validated source-candidate-list.md with flags
│
├─ Stage 4-5: download-and-ingest
│  └── stellantis-source-download-and-ingest
│      ├─ requires: [source-validation]
│      └─ provides: KB dataset with ingested documents
│
└─ Stage 6: partition-and-spawn
   └── stellantis-lead-agent-subagent-orchestration
       ├─ requires: [subagent-classification-loop]
       └─ provides: partition management, subagent coordination
           │
           ├─ Stage 6a: subagent-classify (spawned)
           │  └── stellantis-subagent-classification-loop (IN SUBAGENT CONTEXT)
           │      ├─ requires: none
           │      └─ provides: per-partition classification results
           │
           └─ Stage 6b: consolidate (back to lead)
```

### Dependency Classes

**Soft Dependency (orchestrated, not loaded):**
- Main skill lazily loads sub-skills at the stage that needs them
- Not declared upfront in frontmatter

**Hard Dependency (must load before this skill runs):**
- orchestration loads subagent-classification-loop to spawn subagents

**Conditional Dependency:**
- source-validation depends on: discovery-with-domains OR discovery-without-domains (mutually exclusive)

---

## New Frontmatter Fields

### Required additions:

```yaml
---
name: <skill-name>
description: <when to use + what it does>
loader: lead | subagent
stage: <W1-stage-1b | W1-stage-3 | etc.>
requires:
  - <skill-name> | optional
  - <skill-name> | only if hard dependency
provides:
  - <deliverable type or file>
tools:
  - google_search | from "Tools allowed" section
  - browser_render_markdown
---
```

### Mapping for each skill:

| Skill | Stage(s) | Requires | Provides | Tools |
|-------|----------|----------|----------|-------|
| automotive-feature-classification | W1 entry | none (lazy loads) | orchestration, run directory | (orchestration only) |
| car-trim-resolution | W1-stage-1b | none | resolved_trim, variant_note | google_search, google_search_news |
| source-discovery-with-client-domains | W1-stage-3-modeA | none | source-candidate-list.md | google_search (site: restricted), google_search_news |
| source-discovery-without-client-domains | W1-stage-3-modeB, W2-stage-3 | none | source-candidate-list.md | google_search, google_search_news, google_search_images, google_search_videos, google_search_autocomplete |
| source-validation | W1-stage-3b | discovery-with-client-domains OR discovery-without-client-domains | annotated source-candidate-list.md | browser_render_markdown |
| source-download-and-ingest | W1-stage-4, W1-stage-5 | source-validation | KB dataset with documents | browser_render_markdown, browser_render_pdf, browser_render_content, upload_with_metadata, run_document, doc_infos, list_docs, get_metadata_summary |
| lead-agent-subagent-orchestration | W1-stage-6, W1-stage-6b | subagent-classification-loop | partition manifests, consolidated category STATE files | retrieval (via subagent), batch_update_doc_metadata, get_metadata_summary |
| subagent-classification-loop | W1-stage-6a | none | per-partition result envelope (JSON) | retrieval, retrieve_knowledge_graph, list_chunks, download_attachment, doc_infos, get_metadata_summary, set_doc_metadata, batch_update_doc_metadata (staged only) |

---

## Reference System (for shared assets)

Current: file paths scattered across documents
Proposed: centralized reference IDs

**Reference IDs to define:**

```
enums-reference
  → refers to src/references/enums.md
  → single source of truth for all enum values

lead-to-subagent-contract
  → refers to src/references/lead-to-subagent-contract.md
  → canonical shape for contract JSON

subagent-to-lead-contract (result envelope)
  → refers to src/references/subagent-to-lead-contract.md
  → canonical shape for result envelope JSON

state-main-template
  → refers to src/templates/STATE.main.md.tmpl
  → template for lead's main STATE.md

state-category-template
  → refers to src/templates/STATE.category.md.tmpl (if exists)
  → template for per-category STATE files

state-subagent-template
  → refers to src/templates/STATE.subagent.md.tmpl (if exists)
  → template for subagent working files

deliverable-schema
  → refers to src/templates/deliverable.schema.json
  → JSON schema for deliverable output

deliverable-csv-columns
  → refers to src/templates/deliverable.csv.columns.md
  → column definitions for deliverable.csv

source-candidate-list-template
  → refers to src/templates/source-candidate-list.md.tmpl
  → template for candidate list markdown

workspace-layout-reference
  → refers to src/references/workspace-layout.md
  → directory structure and file ownership
```

---

## Implementation Steps

### Phase 1: Update Frontmatter (all 8 skills)

1. Add `stage` field (from dependency graph above)
2. Add `requires` field (only non-empty for hard dependencies)
3. Add `provides` field (key deliverables)
4. Add `tools` field with allowed tools list (from "Tools allowed" sections)

### Phase 2: Remove File-Path References

1. Replace all `../../skills/XXX/SKILL.md` links with skill name references
2. Replace all `../../templates/XXX` and `src/references/XXX` with reference IDs
3. Create `REFERENCES.md` in main skill root defining all reference IDs

### Phase 3: Standardize Cross-Skill References

1. Add "Dependencies" section to each SKILL.md (structured, not prose)
2. Update WORKFLOW.md to use reference IDs instead of file paths
3. Update all contract/reference documents to use reference IDs

### Phase 4: Validation

1. Verify all 8 skills have consistent frontmatter
2. Verify no file-path references remain in skill bodies
3. Run skill-creator to validate structure compliance

---

## Estimated Impact

**Brittleness reduced:** ✓ Remove file-path coupling
**Portability improved:** ✓ Skills now self-documenting via frontmatter
**Maintenance cost:** ~2h (frontmatter + reference cleanup)
**Risk:** Low (structural only, no logic changes)
