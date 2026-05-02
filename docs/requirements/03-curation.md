# 03 — Curation

Purpose: define how candidate items are judged, how many are published, what diversity is enforced, and how duplicates and updates are handled.

## 4-axis importance criteria

Every candidate item is judged on four axes:

1. **Credibility** — is the source authoritative, is the claim verifiable, is the framing free of obvious hype?
2. **Inflection-point-ness** — does this change a baseline assumption (a model crosses a capability threshold, a regulator establishes a precedent, a vendor changes a default)?
3. **Virality** — is this widely covered or rapidly spreading among practitioners (proxy for "the user will hear about it elsewhere if we don't surface it")?
4. **Practicality** — does this affect what an AX practitioner can ship, deploy, or recommend in the near term?

**Weighting (highest to lowest): credibility > inflection > virality > practicality.**

Why this order: a high-credibility inflection event is the system's reason for existing; virality and practicality are tiebreakers that prevent the system from being academic-only. Credibility is first because a viral but uncredible item is worse than no item.

> NOTE: Ambiguity — the spec lists axes and weight order but does not define a numeric scoring scheme. Decision: leave scoring (numeric vs. ordinal vs. LLM-judged) to the architecture set. The requirement is only that all four axes are evaluated and that ties are broken in the stated order.

## Daily volume

- Soft target: **1–10 items per day** (cumulative over the UTC day, not per pipeline run).
- Prefer fewer high-quality items over padding to hit the upper bound.
- When the pipeline runs multiple times per day (an architecture detail), the cap applies to the cumulative day total across all runs, **not** to each run individually. The filter stage must read the day's already-published items and budget the remaining headroom accordingly.
- A zero-item day is acceptable. On a zero-item day, the pipeline must make **no published-dataset changes**: no new digest file, no `index.json` mutation, no `README.md` change. Operational state and run-summary logs may still be committed for traceability.

Why no published-dataset changes on a zero-item day: silence is a valid signal; an empty digest would dilute the README and the per-day archive. Operational logs are exempt because the run did happen and the trace is useful when investigating why a day produced nothing.

## Security highlight escape hatch

If a critical security item (broad-impact vulnerability, mandatory patch) appears in the source pool, it must be included even if the day's quota is otherwise full. The 1–10/day cap is a soft target; a true emergency security item is allowed to push the count above 10.

To bound prompt-injection risk, the implementation enforces a soft alert ceiling on items published per UTC day even with the security escape hatch active; the architecture documents the exact value. **Items beyond this ceiling are not dropped** — doing so would violate the must-include rule above. Instead, the run halts pending maintainer review: the pipeline lists the over-quota escape-hatch items with their inclusion reasons and source URLs, blocks publication, and waits for a human to re-trigger the filter stage after curating the set. A real coordinated CVE day can still publish every advisory; an attacker producing fake "critical security" items just routes the maintainer's attention to the alert list.

Why: the cost of delaying publication for a few minutes of human review is small; the cost of silently dropping an actionable security advisory is not. The alert ceiling exists because an unbounded escape hatch is exploitable by a malicious or compromised source crafting items framed as "critical security" to bypass the cap; surfacing those for human review is the bound, not silent suppression.

## Perspective diversity

A typical day's published set should represent multiple `perspective_tags`. The pipeline should encourage diversity across tags (e.g., not all `llm`, not all `startup`) but **must not** turn this into a hard quota that overrides the 4-axis quality bar.

Operational interpretation: when two candidate items are roughly equal on the 4-axis bar and one would broaden the day's tag coverage, prefer the diversifying one. When the diversifying item is materially below the bar, drop it.

Why a soft preference rather than a quota: a forced tag quota would push low-credibility items into the digest on slow days, which violates the credibility-first weight order.

## Closed enumeration of `perspective_tags`

`perspective_tags` is a closed enumeration. Downstream filters (Module 2) depend on a fixed vocabulary, so adding a new tag is a contract change, not a daily editorial choice.

Initial enumeration:

- `developer` — items most useful to engineers building software, including SDK changes, dev-tool releases, framework updates.
- `ax` — items most useful to AI Transformation consultants and their client-facing work.
- `llm` — frontier-model news: capability updates, model releases, eval results.
- `startup` — funding, launches, market moves at the company level.
- `security` — vulnerabilities, advisories, security-relevant model behavior, AI-safety evaluations with operational impact.
- `policy` — regulatory action, legislation, government guidance, AI governance bodies.
- `research` — academic and industrial research papers, methodology, novel results without immediate product impact.
- `infra` — hardware, datacenters, accelerators, inference-serving platforms.
- `product` — end-user product launches and significant product changes.

A given item carries one or more tags; tags are not mutually exclusive (a frontier-model security eval could be `llm` + `security` + `policy`).

Why a closed set: an open vocabulary would let the producer drift over time and silently break Module 2 filters. The set is intentionally small enough to stay coherent.

> NOTE: Ambiguity — the spec lists the enumeration but invites refinement. Decision: keep the spec's nine tags as v1 (no additions, no removals). Future additions go through a documented contract change.

## Deduplication and update rules

Three cases must be handled:

1. **Same URL, no change.** A candidate whose URL exactly matches an already-published item's `link` (or any existing `sources[].url`) and whose content hash matches what is already published is a duplicate. **Skip.**
2. **Same URL, content changed.** Same URL match, but the recomputed `content_hash` differs from the stored one. Treat as an **update**: overwrite the existing item's `content`, `key`, `keywords`, `content_hash`, and (if the title changed) `title`. Preserve `id` and `first_seen_utc`. Append any newly observed corroborating sources to `sources[]`.
3. **Same topic, different URL.** Different URL, but the source-merging logic concludes the two URLs describe the *same news event* (e.g., a vendor blog post and a TechCrunch write-up of it). Merge into a single item: pick the primary per the priority in [02-sources.md](02-sources.md), put it at `sources[0]`, append the others. If the source-merging logic cannot confirm they are the same event, **publish them as separate items** — the system does not dedup by topic alone.

Why "do not dedup by topic alone": two outlets writing about "GPT-5.5" on the same day might be covering two different sub-stories (a release and an eval). Topic-level merging without event-level confirmation would silently drop one story.
