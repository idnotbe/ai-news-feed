# 00 — System overview

Purpose: a single, expanded picture of the components, the data flow between them, and the design principles those choices follow. Read this before any other architecture doc.

## Table of contents

- [Components](#components)
- [End-to-end data flow](#end-to-end-data-flow)
- [Design principles](#design-principles)
- [Out of scope (architecturally)](#out-of-scope-architecturally)

> DECISION: Implementation language is **Python 3.12**. Spec named "Claude Code / Anthropic Agent SDK" as the harness; both Anthropic SDKs ship in Python and TypeScript. Python is the right pick here for three reasons: (a) the data-pipeline shape matches Python's strengths (RSS/HTTP/YAML/JSON parsing libs are first-class), (b) the AX-consultant primary maintainer is more likely to read Python than TypeScript, (c) all stages can share one runtime. Cross-linked from `02-github-actions.md` "Common Python setup" and `05-llm-and-agent-runtime.md` Overview.

## Components

| Component | Lives at | Role |
|---|---|---|
| Source configs | `data/sources/*.yml`, `data/perspectives.yml` | Declare the universe of sources. Editing these is how the user adds a YouTube channel or RSS feed. |
| LLM config | `data/llm.yml` | One file: model name, base_url, max_tokens, temperature. Swapping LLMs = one-file change. |
| Adapter scripts | `scripts/adapters/*.py` (Python 3.12) | Per-source-type code that turns a config entry into normalized records. |
| Stage scripts | `scripts/stages/*.py` (Python 3.12) | The actual work each workflow runs (fetch / filter / summarize / assemble). Thin; most work lives inside the agent. |
| Agent harness | Anthropic Python SDK invoked from a stage script | Drives the LLM (GLM-5.1) for filter and summarize stages. The 2-round verify cycle is orchestrated by the stage script, not by the SDK's agent loop — see `05-llm-and-agent-runtime.md` Overview for the rationale and the explicit `> DECISION:` block disambiguating Anthropic SDK vs Claude Code CLI vs Anthropic Agent SDK. |
| Workflows | `.github/workflows/*.yml` | One workflow per stage. Triggered by cron, push, or `workflow_run`. |
| Run state | `state/runs/<run_id>/` | Per-run handoff files between stages (committed). |
| Published dataset | `index.json`, `digests/`, `README.md` | The audience-agnostic English feed Module 2 consumes. |
| Summary logs | `logs/YYYY-MM-DD/<run_id>-summary.md` | Canonical human-readable record of each run (committed). |
| State pruning | `.github/workflows/prune-state.yml` | Daily cleanup of `state/runs/` directories older than 30 days. Runs out-of-band on a daily cron, not part of the per-run chain. See `03-repo-layout-and-state.md`. |
| PR rejection | `.github/workflows/reject-external-pr.yml` | Auto-closes PRs from forks. The only PR-triggered workflow. |

## End-to-end data flow

```
data/sources/*.yml ──┐
data/llm.yml ────────┤
                     v
              [bootstrap]
                     |  state/runs/<run_id>/state.json
                     v
              push triggers fetch-* in parallel
                     |
   ┌─────────────────┼─────────────────┐
   v                 v                 v
[fetch-A]        [fetch-B] ...     [fetch-E]
   |                 |                 |
   v                 v                 v
 state/runs/<run_id>/fetch/<category>.json (one each)
 state/runs/<run_id>/fetch.<category>.done.json
                     |
                     v   filter waits for all fetch.*.done.json for this run_id
              [filter] (LLM, 2-round verify)
                     |  state/runs/<run_id>/candidates_final.json
                     v
              [summarize] (LLM, 2-round verify, parallel per item)
                     |  state/runs/<run_id>/summaries/<item_id>.md
                     v
              [assemble] (deterministic)
                     |  digests/YYYY-MM-DD/<run_id>.md
                     |  index.json (updated)
                     |  README.md (updated)
                     |  -> commit + push to default branch
                     v
              [log-summary]
                     |  logs/YYYY-MM-DD/<run_id>-summary.md
                     |  -> commit + push
                     v
              done

Out-of-band, on a separate daily cron (not part of the chain above):
  [prune-state] -> deletes state/runs/<run_id>/ directories older than 30 days.
                   See 03-repo-layout-and-state.md for the prune workflow YAML.
```

The published dataset (`index.json` + `digests/` + `README.md`) is the contract Module 2 consumes. Everything else is internal.

## Design principles

### 1. Deterministic

Every stage takes inputs identified by `run_id` from the working tree, produces outputs identified by `run_id`, and writes a `state/runs/<run_id>/<stage>.done.json` marker on success. Two replays with the same `run_id` and the same source HTML must produce the same output (LLM nondeterminism aside; we accept that and minimize blast radius via `temperature: 0` in `data/llm.yml` — see `05-llm-and-agent-runtime.md`).

*Why*: lets us re-run any single stage by hand and get a correct end state without bespoke recovery logic.

### 2. Resumable

If `summarize` crashes, re-triggering only `summarize.yml` for that `run_id` finishes the run. No earlier stage re-runs unless its done-marker is missing. The repo is the durable state; there is no in-process memory.

*Why*: GH Actions jobs are ephemeral. Anything held only in a job's RAM is lost. Writing handoff files into the repo means recovery is just "re-run the failed stage."

### 3. Public-repo-safe

The repo is public. The LLM API key (provider-agnostic GitHub Secret name `LLM_API_KEY`, currently populated with a Z.AI key for GLM-5.1, exposed to scripts as same-named env var) is the only secret. The Secret name is provider-agnostic so swapping providers is a one-file change to `data/llm.yml` plus a Secret-value rotation in GitHub Settings — no workflow file edits. No workflow runs against external PR branches with secrets. The `reject-external-pr.yml` workflow uses the `pull_request_target` no-checkout pattern from the spec.

*Why*: public dataset is a Module 1 design value (downstream Module 2 repos consume it freely). The threat model and controls are detailed in `08-security.md`.

### 4. Config-driven extensibility

Adding a YouTube channel: append one entry to `data/sources/youtube-channels.yml`. Adding a new RSS feed: append to `data/sources/rss.yml`. Swapping LLM: edit `data/llm.yml`. No code change.

*Why*: the user is the primary maintainer and is not expected to write Python to add a source.

### 5. No in-process state across stages

Stages do not pass Python objects, environment variables, or step outputs to each other. They pass files on disk in the committed working tree.

*Why*: GH Actions' inter-job artifact and cache mechanisms have retention limits, eventual consistency, and can be silently dropped. Committing files is the only mechanism with the durability guarantees we need, and it doubles as audit log.

**Concurrent-push race**: because parallel sibling workflows (the five `fetch-*.yml` files) all push to the same branch within seconds of each other, every commit-and-push step uses a **bounded fetch+rebase+retry loop** (5 attempts) before failing the workflow. The loop is defined once in `02-github-actions.md` "Common commit + push (with rebase-retry loop)" and inlined into every stage's commit step. Without this, only one of the five fetch-*.yml pushes would land per run; the other four would silently lose state and the chain would stall on `filter` waiting forever for the missing `fetch.<cat>.done.json` markers. See also `01-pipeline-stages.md` for per-stage commit step references.

### 6. Failure isolation per source

One adapter throwing does not abort its category's fetch stage. One category's fetch failing does not abort the others (separate workflows). One day producing zero items is a normal exit, not an error — `assemble` short-circuits and commits nothing.

*Why*: requirements §4 and §3 explicitly demand this. Architecturally enforced by separating concerns into separate workflows and using try/except around each adapter call.

## Out of scope (architecturally)

These are not just "non-goals for the product" (those live in requirements §9) — they are also things this architecture deliberately does *not* introduce:

- No database. The repo is the database.
- No external queue/worker (Redis, SQS, etc.). GH Actions triggers are the queue.
- No CDN or web frontend. GitHub serves `README.md` and raw files.
- No custom GitHub App. We use the default `GITHUB_TOKEN` for commits.
- No cross-repo workflow dependencies. Module 2 polls or watches this repo, but Module 1 never reaches into Module 2.

> DECISION: no database. Spec was silent on data store choice but mandated public repo + extensibility via config edits + audit-log-via-commits. Adding a DB would duplicate state and break the single-source-of-truth property.
