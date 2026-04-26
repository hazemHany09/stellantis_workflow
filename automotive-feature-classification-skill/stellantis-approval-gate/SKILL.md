---
name: stellantis-approval-gate
description: Use after source validation completes — to flag candidate URLs for user approval, yield, and resume on the user's explicit signal. Lead context only.
loader: lead
stage: W1-stage-3c
requires:
  - stellantis-domain-context
  - stellantis-failure-handling
provides:
  - flagged-url-set
  - approval-resume-signal
  - approved-url-set
tools: []
---

## Dependencies

- `stellantis-domain-context` — vocabulary, phase-boundary invariant.
- `stellantis-failure-handling` — URL canonicalisation rules.

Receives the validated candidate list from `stellantis-source-validation`.
Feeds the approved set to `stellantis-source-download-and-ingest`.

---

# Sub-skill — Approval gate

## Overview

The mandatory pause between source discovery and ingestion. The lead presents the validated, canonicalised candidate URL list to the user, yields, and resumes only on an explicit user signal. Approval is **URL-level** — no site-level cascade. Once the gate closes, the run never performs web search again.

## When this runs

W1 stage 3c (or its equivalent in any future gated workflow). Always after source validation, always before download. Skipped only by automated-mode workflows (currently W2 — reserved slot, not yet authored).

## Inputs

| Input | Source |
| :--- | :--- |
| Validated candidate list | In-memory output from `stellantis-source-validation` |
| `car_identity` | Main state file |
| `run_id` | Main state file |

## The two operations

The approval mechanism is conceptually mediated by an **approval store** with two operations. The store may be implemented as a structured tool (when available) or as a free-text exchange in the chat surface (current default). The contract is the same.

### `flag-for-approval` (lead → store)

For each canonicalised, validated candidate URL, post one approval entry. Required per-URL fields:

| Field | Required | Notes |
| :--- | :--- | :--- |
| `canonical_url` | yes | Already canonicalised per `stellantis-failure-handling`. |
| `title` | yes | Falls back to URL host if a real title is unavailable. |
| `excerpt` | yes | Falls back to empty string if unavailable. |
| `car_identity` | yes | Full `(brand, model, model_year, market_canonical)` tuple. |
| `run_id` | yes | The owning run. |
| `origin` | yes | `client` (user-supplied URL) or `agent` (agent-discovered). |
| `why_selected` | optional | Short reasoning. Omit if generation is too costly. |

No candidate is blocked by missing optional metadata.

### `retrieve-approved` (lead ← store)

Called once after the user signals resume. Returns the final approved URL set — only URLs the user explicitly accepted (or, in the chat-surface fallback, the URLs the user did not reject).

## Approval semantics

- **URL-level only.** Each URL is its own approval unit. There is no site-level cascade — approving one URL on `bmw.com` does not approve any other URL on `bmw.com`.
- **One client-supplied URL wins over an agent-discovered duplicate.** When canonicalisation collapses a `client` URL and an `agent` URL to the same canonical form, the `client` instance is kept and the `agent` instance is dropped before flagging.
- **Caps.** Agent-discovered URLs respect the candidate cap (default 20, user-overridable at submission). User-supplied URLs are never counted against the cap.
- **Paywalled URLs are never flagged.** They are silently dropped earlier in discovery.

## The yield / resume protocol

1. **Flag.** Post all approval entries.
2. **Pause.** Set the run state: `RunStatus = paused`, `PauseReason = awaiting-source-approval`, `PausedAt = <timestamp>`, `WorkflowStage = <stage id>`.
3. **Yield.** Return control. Do not poll, do not loop.
4. **Wait for explicit signal.** The agent resumes only on the user's "resume classification" message (or whatever explicit resume signal the surface defines). Quiescence timers are not used.
5. **Resume.** On signal: verify `RunStatus = paused`, interpret the user's reply (full approval / index list / rejection list), update state to `RunStatus = active`, and call `retrieve-approved` to fetch the approved set.

## The hard phase boundary (ARCH-4)

This skill enforces the framework's strictest invariant.

- **Before the gate:** the lead may use web search and browser rendering.
- **After the gate closes:** **no web search and no browser rendering happen for the rest of the run.** Not by the lead, not by any subagent. Evidence comes exclusively from the KB.
- The only paths to "more sources" are:
  1. A new run (full re-run workflow).
  2. A gap-fill cycle (when authored), which **re-opens** a fresh discovery → flag → approve cycle for selected parameters.

There is no hybrid mid-run re-discovery. A URL rejected at this gate cannot be re-encountered later in the same run, because there is no later web search.

## Tips & common pitfalls

- **Always canonicalise before flagging.** Duplicates that escape into the approval list waste user attention.
- **Always include `origin`.** The user UI uses it to distinguish their URLs from the agent's.
- **Never proceed on a quiescence timer.** Resume is explicit; absence of activity is not consent.
- **Never re-search the web after the gate.** If the approved set is thin, the run continues with thin coverage. Gap-fill (when authored) is the dedicated remedy.
- **A zero-approved set is legal.** The run continues; the deliverable will be Rule 4 everywhere. Do not abort.
- **The approval entries are informational; only the URL is the approval unit.** Title, excerpt, why-selected exist for the user — they don't carry decision weight in the run.

## What this skill is NOT

- Not the source-discovery skill (that produces the candidate list).
- Not the source-validation skill (that prunes the candidate list).
- Not the download or ingestion skill (those run after the gate closes).
- Not a UI specification — the chat-surface fallback and any structured store both honour the same contract.

## Maintenance

Changing the gate's contract (adding a metadata field, changing the resume signal) requires:
1. Update this skill.
2. Update `stellantis-source-validation` if it now produces additional fields.
3. Update `stellantis-source-download-and-ingest` if it consumes additional fields.
4. Apply follow-ups per `framework-maintenance/impact-checklist.md`.
