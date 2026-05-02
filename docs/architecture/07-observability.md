# 07 — Observability

Purpose: define what gets logged where, and what the canonical committed summary log looks like. The GitHub Actions console log is supplemental; the committed `logs/YYYY-MM-DD/<run_id>-summary.md` is canonical.

## Table of contents

- [Sources of observability data](#sources-of-observability-data)
- [Canonical: committed summary log](#canonical-committed-summary-log)
- [Summary log template](#summary-log-template)
- [What each stage contributes to the summary](#what-each-stage-contributes-to-the-summary)
- [Console logs (supplemental)](#console-logs-supplemental)
- [What is intentionally NOT logged](#what-is-intentionally-not-logged)

## Sources of observability data

| Source | Persisted? | Audience | Notes |
|---|---|---|---|
| `logs/YYYY-MM-DD/<run_id>-summary.md` | yes (committed) | humans (maintainer, reviewers) | Canonical record. |
| `state/runs/<run_id>/*.done.json` | yes (committed, 30-day retention) | log-summary script, debugging | Per-stage durations, counts, status. |
| `state/runs/<run_id>/verify*.md` | yes (committed, 30-day retention) | filter quality audits | Why an item was kept or dropped. |
| GH Actions console logs | yes (90-day retention by GitHub) | live debugging | Stack traces, full HTTP errors. |
| `index.json` history via `git log` | yes (forever) | trend analysis | Each `publish(<run_id>)` commit is one publication event. |

## Canonical: committed summary log

For every run that gets past `bootstrap`, `log-summary.yml` writes one Markdown file under `logs/YYYY-MM-DD/<run_id>-summary.md`. Even runs that publish zero items get a log. *Why a log on zero-item runs*: helps the maintainer answer "did we have a quiet day, or is the pipeline broken?" without digging into Actions UI.

The log is committed in the same way as everything else (bot commit on default branch).

## Summary log template

```markdown
# Run summary — 20260502T0900Z

| Field | Value |
|---|---|
| Run ID | `20260502T0900Z` |
| Started | 2026-05-02 09:00:00 UTC |
| Finished | 2026-05-02 09:06:42 UTC |
| Total wall clock | 6m 42s |
| Final published item count | 2 |
| Status | published |

## Per-source results

The `Items` column shows items that survived all adapter-level rules. The `Skipped` column shows items returned by the source but dropped by the adapter (missing `published_at`, non-Latin title, older than 24h, etc.). A source with `Items: 0, Skipped: 0` had a quiet 24h; a source with `Items: 0, Skipped: 7` is silently malformed and the maintainer should investigate.

| Category | Source | Status | Items | Skipped | Skip reason | Notes |
|---|---|---|---|---|---|---|
| A | Hacker News | ok | 3 | 0 | — |  |
| A | TechCrunch AI | ok | 5 | 1 | missing published_at |  |
| A | The Verge AI | ok | 2 | 0 | — |  |
| A | VentureBeat AI | fail | 0 | — | — | HTTP 503 after 3 retries |
| B | Anthropic Official | ok | 1 | 0 | — |  |
| B | OpenAI Official | ok | 1 | 0 | — |  |
| B | Google DeepMind | ok | 0 | 0 | — |  |
| B | Microsoft AI | ok | 0 | 0 | — |  |
| B | Meta AI | ok | 0 | 0 | — |  |
| C | smol.ai | ok | 1 | 0 | — |  |
| C | TLDR AI | ok | 1 | 0 | — |  |
| C | The Rundown AI | ok | 1 | 0 | — |  |
| C | Ben's Bites | ok | 1 | 0 | — |  |
| D | @matthew_berman | ok | 1 | 0 | — |  |
| D | @AIExplained-Official | ok | 0 | 0 | — |  |
| D | @1littlecoder | ok | 1 | 0 | — |  |
| D | @DavidOndrej | ok | 0 | 0 | — |  |
| D | @JulianGoldieSEO | fail | 0 | — | — | feed parse error |
| D | @WorldofAI | ok | 0 | 0 | — |  |
| E | CISA KEV | ok | 0 | 0 | — |  |

**Per-source totals**: 17 ok, 2 fail, 18 raw items collected, 1 skipped.

## Per-stage timing

| Stage | Started | Finished | Duration | Items in | Items out |
|---|---|---|---|---|---|
| bootstrap | 09:00:00 | 09:00:04 | 4s | — | 1 state.json |
| fetch-A | 09:00:08 | 09:00:35 | 27s | 4 sources | 10 |
| fetch-B | 09:00:09 | 09:00:21 | 12s | 5 sources | 2 |
| fetch-C | 09:00:09 | 09:00:30 | 21s | 4 sources | 4 |
| fetch-D | 09:00:09 | 09:00:38 | 29s | 6 sources | 2 |
| fetch-E | 09:00:09 | 09:00:17 | 8s | 1 source | 0 |
| filter | 09:00:42 | 09:01:27 | 45s | 18 | 2 |
| summarize | 09:01:31 | 09:06:42 | 5m 11s | 2 | 2 |
| assemble | 09:06:46 | 09:06:52 | 6s | 2 | 1 digest, 2 index entries |
| log-summary | 09:06:55 | 09:06:58 | 3s | — | this file |

## Filter decisions

Out of 18 candidate items after fetch + dedup memory check:

| Decision | Title (truncated) | Reason |
|---|---|---|
| KEEP | Claude 4.7 with 1M context window | Vendor primary source; capability inflection (10x context); high credibility; high virality. |
| KEEP | OpenAI ships o5-preview to ChatGPT Plus | Vendor primary source; capability inflection; high credibility. |
| DROP | Yet another GPT wrapper launches | Below 4-axis bar (low credibility, low inflection). |
| DROP | Anthropic press junket recap | Duplicate URL of #349 (kept item). |
| DROP | (12 more, see verify1_filter.md / verify2_filter.md for details) | Various: low credibility, rumor, off-topic. |

Verify rounds:
- Round 1 surfaced 1 missed item ("Claude 4.7"), which was added in v2.
- Round 2 surfaced 0 issues; final == v2.

## Summarize decisions

For each kept item:
- Drafted English `key` and `content` sections per item.
- Verify round 1: 0 factual errors flagged, 0 format issues.
- Verify round 2: 0 issues.
- Final summaries written.

## Publication

- Wrote `digests/2026-05-02/20260502T0900Z.md` (2 items).
- Updated `index.json`: added entries #348, #349. New `index_max` = 349.
- Regenerated `README.md` last-7-days section (now spans 2026-04-26 to 2026-05-02).
- Commit SHA: `<filled in by log-summary.py via git rev-parse HEAD>`.

## Errors

| Stage | Source | Error |
|---|---|---|
| fetch-A | VentureBeat AI | HTTP 503 after 3 retries |
| fetch-D | @JulianGoldieSEO | XML parse error (malformed RSS) |

(Errors did not block the run; affected sources contributed 0 items.)
```

The script that generates this is `scripts/stages/log_summary.py`. It reads the `state/runs/<run_id>/` directory, walks all `*.done.json` and `verify*.md`, and renders the template. Pure assembly job, no LLM call.

## What each stage contributes to the summary

| Stage | What it must record (in its `.done.json` or in `verify*.md`) |
|---|---|
| bootstrap | `completed_at_utc`, `run_id`, `index_max`, `existing_items` count. |
| fetch-* | `started_at_utc`, `completed_at_utc`, `duration_seconds`, `sources_ok` array (each row carries `name`, `items`, `items_skipped`, `skip_reason` so silently-malformed feeds are visible — see `04-data-schemas.md` `fetch.<category>.done.json`), `sources_fail` array, `item_count`. |
| filter | `started_at_utc`, `completed_at_utc`, `duration_seconds`, `item_count_in`, `item_count_out`, decisions table. The detailed reasoning lives in `verify1_filter.md` / `verify2_filter.md`. |
| summarize | per-item start/end times (or aggregate), verify-round outcomes summary, `item_count`. |
| assemble | files written list, commit SHA, item index range. |
| log-summary | itself. (Records what it produced; closes the loop.) |

If a stage's `.done.json` is missing because the stage crashed, the log will show that stage as "MISSING" with the timestamp from when log-summary ran. Maintainer can then check Actions UI for the stack trace.

## Console logs (supplemental)

GH Actions console logs contain:
- Full Python tracebacks on crashes.
- Adapter-level HTTP debug output (only at WARN level by default; DEBUG can be enabled by re-running with `workflow_dispatch` and a debug input — see below).
- The shell scripts' stdout/stderr from each step.

GitHub retains these for 90 days by default. After that, only the committed log remains. *Why we accept this*: the committed log is sufficient for after-the-fact analysis; live debugging happens within the 90-day window.

A debug-enable input on every workflow:

```yaml
on:
  workflow_dispatch:
    inputs:
      debug:
        description: "Set to 'true' to enable verbose logging."
        required: false
        default: "false"
```

The stage scripts read `os.environ.get("STAGE_DEBUG") == "true"` (set in the workflow `env:` from the input) and lower their log level. Off by default. *Why off by default*: avoids accidentally printing HTTP request bodies that may include source URLs not yet meant to be public (rare, but possible).

## What is intentionally NOT logged

Hard rules. Violations are security incidents (see `08-security.md`):

- **API keys** — never. Not in `.done.json`, not in summary, not in verify .md, not in adapters' debug output.
- **Full HTTP request headers** — never (would leak `Authorization`).
- **Full environment variables** (`env`, `printenv`) — never.
- **Raw LLM response on prompt-injection traps** — when an adapter detects suspicious content (see `08-security.md`), the response is summarized to a short marker rather than logged verbatim. *Why*: prevents an attacker from using our committed log as an exfil channel.

The summary log uses item titles, item URLs, source names, durations, counts. That is all. Nothing in this list contains a secret unless an adapter is buggy — adapter code reviews check for this explicitly.
