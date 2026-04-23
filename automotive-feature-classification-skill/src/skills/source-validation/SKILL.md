---
name: source-validation
description: Lead-agent sub-skill for a light-weight free-text validation pass over candidate URLs before they are written into the approval-gate list. Flags obvious mismatches (wrong car, wrong year, wrong market), paywalls, dead links, and low-quality aggregators.
loader: lead
---

# Sub-skill — Source validation

## When this runs

Immediately after `source-discovery-with-client-domains` or `source-discovery-without-client-domains`, before the candidate list is posted to the user.

## Inputs

* The in-memory candidate list produced by a discovery sub-skill.
* `car_identity` from the main STATE.md.

## Tools allowed

* `browser_render_markdown` — **only** for URLs whose Google snippet is too short to judge. One call per such URL; no deep crawl. Cost-bounded — skip if the candidate list is already clean.

## Heuristics (free-text judgement)

For each candidate, the lead reviews the title + snippet (+ optional markdown render) and asks:

1. **Car match.** Does the page mention the target `brand`, `model`, and at least one of `model_year` or `market_canonical`? If neither, mark `weak-car-match` — keep but flag "why selected" accordingly.
2. **Year match.** Is the page clearly about a different model year (e.g. title says `2019` but we target `2025`)? If yes and no co-coverage, drop with reason `wrong-year`.
3. **Market match.** Explicit signal that the page is scoped to a different market (e.g. "US-only trim") — downgrade ranking but keep (markets often overlap).
4. **Paywall / login-wall.** Snippet suggests paywall per Q-SRC-4: drop silently if `origin = agent`; surface flag if `origin = client`.
5. **Aggregator / resale.** Known aggregator or classified listing hostnames (see `business-logic/04-sources.md`): drop unless `origin = client`.
6. **Dead link.** 404-like snippet or redirect chain: drop.
7. **Forum / UGC as primary source.** Soft downgrade. Keep only if it carries clear technical claims with links to primary sources.

## Outputs

Return to the discovery caller a pruned + annotated candidate list. Each surviving candidate has:

* `why_selected` — one-sentence rationale referencing the heuristic that kept it.
* `flags[]` — machine-readable flags (`weak-car-match`, `market-mismatch`, `origin-client-paywalled`, etc.).

## Success criteria

* Candidate list size ≤ input list size.
* No candidate has both a `dead-link` flag AND is kept.
* Every kept candidate has a non-empty `why_selected`.

## Failure modes

| Failure                                    | Response                                                                |
| :----------------------------------------- | :---------------------------------------------------------------------- |
| All candidates fail validation             | Return empty list; caller must escalate (ask user to provide URLs or expand `cap`). |
| Browser rendering repeatedly fails         | Continue on snippet-only judgement; note in main STATE.md event log.    |

## Business-logic trace

Q-SRC-4, Q-SRC-5, AS-SRC-A, AS-SRC-B, AS-SRC-G, `business-logic/04-sources.md` (soft guidance).
