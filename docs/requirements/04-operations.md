# 04 — Operations

Purpose: define run cadence, required logging, and the README's published format. Implementation mechanics (workflow file names, schedules, runtime, triggers) are an architecture concern and intentionally not specified here.

## Run cadence

- The pipeline runs **at least daily, potentially multiple times per day.**
- The exact schedule (one run per day at a fixed UTC time, multiple runs per day, an event-driven mix) is an architecture concern and is not constrained by this document.
- Whatever cadence is chosen, the system must remain idempotent across runs: a second run on the same day with no new sources must not produce duplicate items, must not rewrite unchanged digests, and must not produce a published-dataset commit (no digest, no `index.json` mutation, no `README.md` change). Operational state and run-summary logs may still be committed; see [03-curation.md](03-curation.md) for the full zero-item-day rule and the dedup rules.

Why under-specify cadence: cadence trades freshness against cost (LLM calls, runner minutes, README churn). The right point on that curve depends on operational data the system does not yet have. Over-specifying it now would lock in a choice before it can be measured.

> NOTE: Ambiguity — the spec is silent on exact frequency. Decision: state the requirement as "at least daily, potentially multiple times per day" and defer concrete scheduling to the architecture set.

## Logging

Every run must produce a **summary log file** stored where it can be reviewed without leaving the repository (i.e., committed to the repo and human-readable).

Required content per run:

- **Per source:** outcome (`ok` / `fail`), item count fetched, and on `fail` a short reason string. Per-source failures must not break the run (see [02-sources.md](02-sources.md)).
- **Per pipeline stage:** start timestamp, finish timestamp, duration, items in, items out.
- **Filter decisions:** for every dropped candidate, a one-line reason (e.g., `duplicate URL`, `below 4-axis bar`, `non-English source`, `no perspective tag`). The intent is auditability of editorial calls, not exhaustive reasoning.
- **Final published item count** for the run.

Format requirements:

- Human-readable markdown (so the user can scan logs directly on GitHub web).
- One summary file per run.
- Sensitive values (API keys, tokens) must never appear in logs (see [05-security-and-governance.md](05-security-and-governance.md)).

Why commit the log to the repo: the runner's own console log is ephemeral and access-gated; a committed summary is durable, public, and indexable, which matches the rest of the dataset's posture.

> NOTE: Ambiguity — the spec proposes `logs/YYYY-MM-DD/` as a layout. Decision: the *requirement* is that a per-run summary log is committed and human-readable; the directory layout is an architecture concern and is not pinned here.

## README format

The README is auto-updated by the pipeline. It is the highest-traffic surface of the dataset.

Required content:

- The README shows the **last 7 days** of published items, grouped by date, most recent date first.
- Within each day group, items are listed by `first_seen_utc` descending (most recently first-published item appears at the top).
- Items older than 7 days are reachable by browsing the per-day digest folders.

Required per-item line format:

```
"<English title>" — <keyword>, <keyword>, <keyword>[, <keyword>][, <keyword>]
```

- Title is wrapped in straight double quotes.
- Title and keyword list are separated by an em-dash (`—`) with single spaces around it.
- Keywords are comma-separated, lowercase, in the order stored on the item.
- The line links to the digest file containing the item (rendered as a markdown link wrapping the whole line, or wrapping the title — pick one and apply consistently; this is an architecture call but the visible text format is fixed).

Example day group:

```
2026-05-01
"GPT-5.5 cyber capabilities match offensive-research baseline; others to follow" — gpt-5.5, cybersecurity, aisi, claude, penetration-testing
"Apple surprised by Mac Mini demand: agent workloads exceed expectations" — apple, mac-mini, ai-workload, local-ai, developer-demand
```

Why this exact line format: it is dense, scannable, and survives copy-paste into any downstream context (newsletter draft, chat, doc). Locking the visible format here protects downstream parsers that may eventually read the README directly.
