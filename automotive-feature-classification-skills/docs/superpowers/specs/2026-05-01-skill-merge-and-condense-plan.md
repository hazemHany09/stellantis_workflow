# Skill merge & condense plan — automotive-feature-classification-skills

Status: **proposal / not yet executed**
Goal: collapse 17 skill files into **3 (aggressive)** or **4 (recommended)**, and shrink each one by removing prose that restates frontmatter, ownership matrices, invariants, or other content already canonicalised elsewhere — **without changing any decision rule, hard rule, ownership boundary, retry budget, or contract**.

This document supersedes earlier drafts that proposed a 7-skill target.

---

## 1. Diagnosis

### 1.1 Chaotic loading

The entry skill [stellantis-automotive-feature-classification/SKILL.md](../../../stellantis-automotive-feature-classification/SKILL.md) declares **15 `requires:`** entries in YAML, telling the loader to pull every operational sub-skill at preflight. Its body simultaneously says "load sub-skills lazily." YAML wins. The lead's context is saturated before stage 1.

It also pulls `stellantis-subagent-classification-loop` (`loader: subagent`) into the lead's context — a skill the lead never executes.

### 1.2 Total context burden

| Skill                                                  | Lines | Actor             | Stage             |
| ------------------------------------------------------ | ----: | ----------------- | ----------------- |
| stellantis-automotive-feature-classification           |   283 | lead              | entry             |
| stellantis-domain-context                              |    91 | shared            | always-on         |
| stellantis-decision-rules                              |   178 | shared            | classification    |
| stellantis-output-contract                             |   133 | shared            | emission          |
| stellantis-workflow-modes                              |    76 | lead              | dispatch          |
| stellantis-failure-handling                            |   130 | shared            | any               |
| stellantis-knowledge-base-protocol                     |   114 | shared            | any               |
| stellantis-run-workspace                               |   191 | shared            | any               |
| stellantis-car-trim-resolution                         |   108 | lead              | W1-stage-1b       |
| stellantis-source-discovery-with-client-domains        |    67 | lead              | W1-stage-3-modeA  |
| stellantis-source-discovery-without-client-domains     |    66 | lead              | W1-stage-3-modeB  |
| stellantis-source-validation                           |    77 | lead              | W1-stage-3b       |
| stellantis-approval-gate                               |    76 | lead              | W1-stage-3c       |
| stellantis-source-download-and-ingest                  |   379 | lead              | W1-stage-4/5      |
| stellantis-lead-agent-subagent-orchestration           |   192 | lead              | W1-stage-6        |
| stellantis-subagent-classification-loop                |   158 | subagent          | W1-stage-6a       |
| stellantis-subagent-doc-deep-dive                      |   140 | subagent          | W1-stage-6b       |
| **Total**                                              | **2 459** |               |                   |

### 1.3 Redundancy hot-spots (concrete)

- **Lead-is-sole-spawner rule** stated in 5 places: entry skill (twice), `run-workspace`, `lead-agent-subagent-orchestration`, and the README architectural rules.
- **Writer-ownership table** in 3 places: entry skill ("Roles & Responsibilities" + matrix), `run-workspace` ("Writer ownership"), and prose inside `lead-agent-subagent-orchestration`.
- **Pause/resume contract** in 2 places: entry skill and `run-workspace`.
- **STATE.md schema** enumerated twice: entry skill and `run-workspace`.
- **URL canonicalisation** in `failure-handling` and re-summarised in `approval-gate` and `source-validation`.
- **Concurrency caps** restated in entry skill, `run-workspace`, `failure-handling`, and `lead-agent-subagent-orchestration`.
- **Tool allow-list** appears as both a per-skill `tools:` frontmatter list AND as a per-actor matrix in the entry skill body.
- **"Hard rule:" prose blocks** in the entry skill restate what the actor × tool matrix already says.
- **"What this skill is NOT" + "Maintenance" footers** appear in every shared skill — pure signposting, no rules.
- The 379-line `source-download-and-ingest` carries embedded YAML/JSON envelope examples that belong under `templates/`.

---

## 2. Constraints — load-bearing, must not change

The merge preserves **every** behavioural rule. Specifically:

1. The five decision rules and the master decision matrix, including `confidence` (`consensus / single-source / vague-only / silent-all`), the seven `source_type` categories, the `forum_or_ugc` demotion, the consensus promotion rule, and the inverse-retrieval prerequisite for Rule 4 (with the verbatim query patterns).
2. The eight master invariants.
3. The four hard constants (scope / required inputs / output triple / phase boundary).
4. The vocabulary contract (`category`, `source site`, `parameter`, `level`, `applicable level set`, `Presence/Status/Classification`, `clear/vague/silent`, `active source`, `KB`, `run workspace`, `full-option trim`).
5. The complete failure-class table — download retries 1/2/4 s × 4 attempts; 60-min ingestion ceiling; rate-limit detection strings; access-denied detection strings; subagent re-spawn-once policy; `sources_excluded` footer schema.
6. KB capability categories, ingestion-wait protocol, retrieval iteration budget (3 main + 3 fallback), "no web search inside classification" hard rule.
7. Lead-is-sole-spawner; download/ingest subagents are content-blind (size only); R2 reads only `target_doc.local_path`; lead is sole reader of `params.csv`.
8. Concurrency caps — ≤ 3 classification subagents, ≤ 5 download subagents, serial consolidation.
9. Complete `.harness/` layout and writer-ownership table.
10. Per-parameter record schema (incl. `confidence`, `inverse_retrieval_attempted`), traceability-block cardinality table, deliverable filename convention, dual JSON+CSV emission, CSV via Python script.
11. All workflow-modes composition rules; W1 mode A/B mutually exclusive; new run = new run id; no silent chaining; W2..W8 reserved.
12. All tool allow-lists per actor.
13. Partition rule: max 15 parameters per R1 subagent, never crossing categories.
14. Cache-check directive on `.harness/downloads/` before any `fetch_url` / `ragflow_download` call.

The plan touches **structure and prose redundancy only**. Section §6 is a verbatim-preservation checklist run before any deprecated skill is deleted.

---

## 3. Target architecture

Two tiers: a **foundation** every actor loads, and a **role-runbook** loaded per actor type. No stage-by-stage lazy loading because YAML `requires:` tooling cannot reliably express it; a single role-runbook per actor is simpler and cheaper.

### 3.1 Recommended target — 4 skills

| #  | Merged skill                          | Replaces (count)                                                | Loaded by    | ~Lines |
| -: | ------------------------------------- | --------------------------------------------------------------- | ------------ | -----: |
| 1  | **stellantis-core-contracts**         | domain-context + decision-rules + output-contract + failure-handling + run-workspace (writer-ownership only) (5) | shared (everyone) | ~480 |
| 2  | **stellantis-lead-runbook**           | automotive-feature-classification (entry) + workflow-modes + run-workspace (rest) + knowledge-base-protocol (lead-side) + car-trim-resolution + source-discovery-with-client-domains + source-discovery-without-client-domains + source-validation + approval-gate + source-download-and-ingest + lead-agent-subagent-orchestration (11) | lead | ~700 |
| 3  | **stellantis-subagent-classification** | subagent-classification-loop + subagent-doc-deep-dive + knowledge-base-protocol (retrieval-side) (2½) | R1 + R2 subagents | ~260 |
| 4  | **stellantis-subagent-download-ingest** | source-download-and-ingest (subagent-side carve-out only) (½) | download/ingest subagents | ~120 |
| —  | serper-api-mastery                    | (kept; out of scope)                                            | lead         |   244 |

Plus the entry skill becomes a **3-line shim** with the user-facing description and a single `requires:` line pointing at `stellantis-lead-runbook`. Tooling that scans frontmatter still finds the entry; the runbook does the work.

#### Why this split

- The lead is one actor. Its work is end-to-end sequential. Splitting "discovery vs ingest vs orchestration" into separate files only forces the lead to re-load context between adjacent stages. A single runbook with `## Stage N` headings is shorter overall and follows the agent's actual linear path.
- R1 (KB retrieval) and R2 (single-file read) emit the **same** record shape, follow the **same** decision rules, write the **same** envelope, and differ only in their input source and tool list. A single subagent-classification skill with two role sections (`## Role: R1 — KB retrieval`, `## Role: R2 — single-file read`) deduplicates the envelope, advisory, partition-summary, and decision-rule invocation prose. The subagent reads the section that matches the `role` field in its contract; the other section is at most ~100 lines of unread context, less than the duplication it eliminates.
- Download/ingest subagents are different work — content-blind I/O, no decision rules, no schema records. They stay in their own narrow runbook (~120 lines) so they don't drag classification context into a fetch-and-upload worker.

### 3.2 Aggressive alternative — 3 skills

Collapse #3 and #4 into **stellantis-subagent-runbook** with three role sections (`## Role: download-ingest`, `## Role: R1`, `## Role: R2`). Saves one file at the cost of every subagent loading ~120 unread lines from the other roles. Recommended only if the deployment has a tool-budget penalty per skill file load.

| #  | Skill                                 | Loaded by    | ~Lines |
| -: | ------------------------------------- | ------------ | -----: |
| 1  | stellantis-core-contracts             | shared       |   ~480 |
| 2  | stellantis-lead-runbook               | lead         |   ~700 |
| 3  | stellantis-subagent-runbook           | every subagent | ~380 |

### 3.3 Aggregate impact (recommended 4-skill target)

| Metric                                            | Today  | After merge |
| ------------------------------------------------- | -----: | ----------: |
| SKILL.md files in package                         |     17 |           4 |
| Lines preloaded into lead at preflight            | ~2 000 |     ~1 180 |
| Lines pulled into a R1 subagent on spawn          |   ~700 |       ~740 |
| Lines pulled into a R2 subagent on spawn          |   ~680 |       ~740 |
| Lines pulled into a download/ingest subagent      |   ~580 |       ~600 |

R1/R2/download numbers are roughly flat — those actors already load only what they need, so dedup gives only the prose-condensation win. The big lever is the **lead**, which today over-loads by ~70 % at preflight and shrinks to a single linear runbook.

---

## 4. Merge map (every source skill → its destination)

| Source skill                                            | Destination                                | Section in destination |
| ------------------------------------------------------- | ------------------------------------------ | ---------------------- |
| stellantis-domain-context                               | stellantis-core-contracts                  | §1 Mission & vocabulary; §2 Master invariants |
| stellantis-decision-rules                               | stellantis-core-contracts                  | §3 Decision rules & matrix; §4 Source-type & confidence; §5 Inverse retrieval |
| stellantis-output-contract                              | stellantis-core-contracts                  | §6 Per-parameter record; §7 Deliverable & header; §8 sources_excluded footer |
| stellantis-failure-handling                             | stellantis-core-contracts                  | §9 Failure classes & retries; §10 URL canonicalisation |
| stellantis-run-workspace (ownership table only)         | stellantis-core-contracts                  | §11 Roles, ownership & concurrency |
| stellantis-run-workspace (layout, pause/resume, STATE)  | stellantis-lead-runbook                    | §1 Workspace layout; §2 STATE.md schema; §3 Pause/resume |
| stellantis-workflow-modes                               | stellantis-lead-runbook                    | §4 Workflow identification |
| stellantis-knowledge-base-protocol (lead-side)          | stellantis-lead-runbook                    | §5 KB lifecycle: create / upload / wait / delete |
| stellantis-knowledge-base-protocol (retrieval-side)     | stellantis-subagent-classification         | §R1.3 KB retrieval protocol |
| stellantis-automotive-feature-classification (preflight) | stellantis-lead-runbook                   | §6 Preflight (10 steps) |
| stellantis-car-trim-resolution                          | stellantis-lead-runbook                    | §7 Trim resolution (stage 1b) |
| stellantis-source-discovery-with-client-domains         | stellantis-lead-runbook                    | §8 Source discovery — Mode A |
| stellantis-source-discovery-without-client-domains      | stellantis-lead-runbook                    | §9 Source discovery — Mode B |
| stellantis-source-validation                            | stellantis-lead-runbook                    | §10 Source validation (stage 3b) |
| stellantis-approval-gate                                | stellantis-lead-runbook                    | §11 Approval gate (stage 3c) |
| stellantis-source-download-and-ingest (lead-side)       | stellantis-lead-runbook                    | §12 Download (stage 4); §13 Ingestion wait (stage 5) |
| stellantis-source-download-and-ingest (subagent-side)   | stellantis-subagent-download-ingest        | entire body |
| stellantis-lead-agent-subagent-orchestration            | stellantis-lead-runbook                    | §14 Partitioning (stage 6); §15 R1 spawn & consolidate; §16 R2 spawn & consolidate; §17 Final emission |
| stellantis-subagent-classification-loop                 | stellantis-subagent-classification         | §R1 (full content) |
| stellantis-subagent-doc-deep-dive                       | stellantis-subagent-classification         | §R2 (full content) |

Templates and YAML/JSON examples extracted from `source-download-and-ingest` and `lead-agent-subagent-orchestration` move into [stellantis-automotive-feature-classification/templates/](../../../stellantis-automotive-feature-classification/templates/) and [references/](../../../stellantis-automotive-feature-classification/references/) — they are referenced by name from the skill body, not inlined.

---

## 5. Condensation rules (apply to every merged skill)

Each rule is mechanical and removes prose only — no logic changes.

1. **Single source of truth per concept.** A constraint lives in exactly one place. Every other skill links to that anchor. Concepts and their canonical homes:
   - Vocabulary, invariants, decision rules, output schema, failure policy, ownership matrix, concurrency caps, tool allow-list → `stellantis-core-contracts`.
   - `.harness/` tree, STATE.md schema, pause/resume → `stellantis-lead-runbook §1–§3`.
2. **Drop "What this skill is NOT" sections.** Pure signposting, zero rules.
3. **Drop per-skill "Maintenance" footers.** One line at the top of each skill pointing at [framework-maintenance/impact-checklist.md](../../../framework-maintenance/impact-checklist.md) is enough.
4. **Drop "Used by" / "Consumed by" reverse-reference lists.** Forward-references in `requires:` plus `grep` cover this.
5. **Drop prose that restates `provides:` / `requires:` from the frontmatter.**
6. **Collapse "Tips & common pitfalls" sections.** Drop any pitfall that restates a master invariant. Keep only operational pitfalls (e.g. "don't escalate one URL failure into a run abort"). Bullet lists, no headings.
7. **Replace prose hard-rule blocks with one row in the matrix.** The "Hard rule: classification subagents never use open-web search ..." paragraphs become a single ❌ in the actor × tool matrix.
8. **Move embedded examples to `templates/` or `references/`.** Skill body keeps a one-line pointer.
9. **Tables, not paragraphs, for any rule with > 2 conditions.** The `failure-class` and decision-matrix tables are good models.
10. **Sentence-level cuts.** Drop adjectives that signal but don't constrain: "deliberately", "strictly", "obviously", "of course", "as noted above". They double word count without changing meaning.

Applying §5 alone — without any merging — shaves ~25 % off each existing skill. Combined with the merging in §4, total package goes from ~2 460 lines to ~1 560 lines while preserving every rule.

---

## 6. Logic-preservation checklist

Each merged file must verify the union of these statements is preserved verbatim, or as a paraphrase that does not weaken or strengthen the rule:

- [ ] Five decision rules with exact `(Presence, Status, Classification)` triples and traceability cardinalities.
- [ ] Master decision matrix as one six-row table.
- [ ] Classification-level hierarchy (Basic ⊂ Medium ⊂ High).
- [ ] Seven `source_type` categories with authority columns.
- [ ] `confidence` rules including `forum_or_ugc` demotion and same-source-type-conflict advisory.
- [ ] Inverse-retrieval queries listed verbatim (`"<feature_name>" "not available"` etc., 6 patterns).
- [ ] Eight master invariants.
- [ ] Four hard constants.
- [ ] Failure-class table including verbatim rate-limit markers (`"just a moment"`, `"cf-challenge"`, `"ray id:"`, `"checking your browser"`, `"enable javascript and cookies"`, `"cf_chl"`, `"attention required"`, `"ddos protection"`) and access-denied markers (`"403 forbidden"`, `"access denied"`, `"not authorized"`, `"401 unauthorized"`, `"you don't have permission"`, `"permission denied"`, `"forbidden"`).
- [ ] Retry budgets — download 1/2/4 s × 4 attempts; retrieval 3 retries; subagent re-spawn × 1; ingestion 60-min ceiling; rate-limit auto-flag at `Attempts ≥ 3`; access-denied **never** retried.
- [ ] URL canonicalisation steps (5 steps in order).
- [ ] KB ingestion-wait protocol (6 steps in order); "no partial-ingestion classification" rule.
- [ ] Required document metadata (10 fields incl. optional `cycle`, `origin`).
- [ ] `.harness/` directory tree exactly as drawn — `downloads/`, `DownloadAgent/`, `Category/`, `SubAgent/`, `DeepDiveAgent/`, `Archive/`, `advisories/`, plus root files (`STATE.md`, `params.csv`, `event-log.md`, `document-promise-board.md`, `partition-summaries.md`, `source-candidates.md`, `source-approved.md`, `sources_excluded.md`).
- [ ] Writer-ownership table — every row.
- [ ] "Lead is sole spawner" — stated once in `core-contracts §11` and not restated elsewhere.
- [ ] Content-blind property of download/ingest subagents (size only, no content read).
- [ ] R2 single-file restriction (`target_doc.local_path` only).
- [ ] No subagent reads `params.csv`; lead embeds slice in contract.
- [ ] Partition rule: max 15 parameters per R1 subagent, never crossing categories.
- [ ] Concurrency caps: ≤ 3 classification, ≤ 5 download/ingest, serial consolidation.
- [ ] Per-parameter record fields including `confidence` and `inverse_retrieval_attempted`.
- [ ] Traceability-block cardinality table per rule.
- [ ] Deliverable filename convention `<brand>-<model>-<year>-<market>-<run-id>.{json,csv}`.
- [ ] CSV via Python script procedure (4 steps).
- [ ] Run header fields (12 entries).
- [ ] Workflow-modes composition rules (6 rules) and W2..W8 reserved slots.
- [ ] Tool allow-lists per actor (lead, download/ingest subagent, R1, R2).
- [ ] Pause / resume contract steps.
- [ ] Cache-check directive on `.harness/downloads/`.
- [ ] Three-block working-file structure (contract / scratch / result envelope) for each subagent type.
- [ ] STATE.md T1/T2 tier split (resume-critical vs telemetry).

A merged file that fails any item is rolled back before the deprecated skill is deleted.

---

## 7. Migration steps

Reviewable units; package stays bisectable.

1. **Create `stellantis-core-contracts/SKILL.md`.** Copy domain-context + decision-rules + output-contract + failure-handling, plus the writer-ownership / concurrency / tool-allow-list tables from run-workspace and the entry skill. Apply §5 condensation rules. Add a deprecation banner to each of the 5 source skills: *"Superseded by `stellantis-core-contracts/SKILL.md` §N."*
2. **Create `stellantis-subagent-classification/SKILL.md`.** Two role sections (`## Role: R1`, `## Role: R2`). Move the KB retrieval protocol (allowed tools, iteration budget, no-web-fallback) from `knowledge-base-protocol` into §R1.3. Add deprecation banners.
3. **Create `stellantis-subagent-download-ingest/SKILL.md`.** Carve out the subagent-side content from the 379-line `source-download-and-ingest` (the content-blind fetch + upload protocol, the rate-limit / access-denied detection, the result-envelope shape). Add deprecation banner to the original.
4. **Create `stellantis-lead-runbook/SKILL.md`** with §1–§17 mirroring §4's section table. Apply §5 condensation. Move embedded examples to `templates/` and `references/`.
5. **Reduce the entry skill** to a 3-line shim:
   ```yaml
   ---
   name: stellantis-automotive-feature-classification
   description: Use when user names a specific car (brand, model, year, market) and asks about features, parameters, options, capabilities, or levels.
   requires:
     - stellantis-lead-runbook
   ---
   ```
   Body: a single sentence pointing at `stellantis-lead-runbook` for the full pipeline.
6. **Run §6 logic-preservation checklist** end-to-end. Every box ticked → proceed. Any miss → patch the destination skill and re-check.
7. **Grep for old skill names** (`stellantis-domain-context`, `stellantis-decision-rules`, etc.). Each remaining hit is either an intentional pointer to the deprecation banner or a cross-reference bug — fix.
8. **Delete deprecated skill directories.** Update [README.md](../../../README.md) skill index. Append [framework-maintenance/change-log.md](../../../framework-maintenance/change-log.md) entry per [framework-maintenance/impact-checklist.md](../../../framework-maintenance/impact-checklist.md).

Each step compiles to a working set of skills. Step *N+1* starts only after step *N* is reviewed.

---

## 8. What this plan does not change

- No edit to `business-logic/` documents.
- No edit to `artifacts/params.csv`.
- No tool gain or loss for any actor.
- No new workflow. W2..W8 stay reserved.
- No change to deliverable schema, run header, or `sources_excluded` footer.
- No change to subagent contract envelope shape.
- `serper-api-mastery` stays untouched (generic external skill).

---

## 9. Open questions for the maintainer

1. **3 vs 4 skills.** Recommended: 4. Choose 3 (single subagent-runbook) only if the deployment penalises every additional skill file load. Both options preserve identical logic.
2. **Entry-skill shim vs full merge into lead-runbook.** Recommended: keep the shim because external tooling and the user-facing trigger description live there. The shim adds three lines; full merge would force every loader to read `stellantis-lead-runbook`'s name, which is less intuitive.
3. **Workspace layout location.** Layout drawing lives in `stellantis-lead-runbook §1` (lead writes most of it). Subagents reference only the working-file three-block structure, which is in `stellantis-core-contracts §11`. Confirm this split is acceptable — the alternative is duplicating the tree drawing.
4. **Optional helper script.** A `tools/check-skill-merge.ps1` that diffs section anchors against §6 is cheap to add. Skip unless review surfaces drift.

---

## 10. Risks

| Risk                                                                | Mitigation |
| ------------------------------------------------------------------- | ---------- |
| Dedup pass silently weakens a rule.                                 | §6 checklist before deletion. |
| Consumer references an old skill name post-merge.                   | Step 7 grep audit. |
| Loader tooling ignores body-level lazy loading.                     | Skills are merged so each actor needs only one runbook — no body-level lazy loading required. |
| 379-line `source-download-and-ingest` carries undocumented edge cases. | Read it line-by-line during step 3; preserve every "if X then Y" branch verbatim, even if relocated. |
| Subagent contracts (`lead-to-subagent-contract`, `subagent-to-lead-contract`) drift after orchestration is folded into `stellantis-lead-runbook`. | Pin them in [stellantis-automotive-feature-classification/references/](../../../stellantis-automotive-feature-classification/references/); do not edit them in step 4. |
| `stellantis-lead-runbook` at ~700 lines becomes hard to navigate.   | 17 numbered top-level sections (`## §1` … `## §17`) and a TOC at the top. The lead reads it once; navigation is by anchor. |
| Two role sections in `stellantis-subagent-classification` get out of sync. | Common content (envelope shape, decision-rule invocation, advisory formats) lives in a `## Common` section above the role splits. |

---

## 11. Out of scope

- The `STATE.md` enrichment design tracked in [2026-04-30-state-md-enrichment-design.md](2026-04-30-state-md-enrichment-design.md). That spec describes content changes inside `STATE.md`; this plan only restructures skill loading. They are independent and can ship in either order.
