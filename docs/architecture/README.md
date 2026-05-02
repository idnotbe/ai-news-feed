# Architecture — `ai-news-feed` (Module 1)

Purpose: explain *how* `ai-news-feed` is built so the requirements (in `docs/requirements/`) are satisfied. This is the developer-facing companion to the requirements doc; it changes as we learn, while the requirements doc is a stable contract.

## Table of contents

1. [00-system-overview.md](./00-system-overview.md) — components, data flow, design principles.
2. [01-pipeline-stages.md](./01-pipeline-stages.md) — every stage, its input/output contract, idempotency, run-id propagation.
3. [02-github-actions.md](./02-github-actions.md) — concrete `.github/workflows/*.yml` list, triggers, secrets, permissions.
4. [03-repo-layout-and-state.md](./03-repo-layout-and-state.md) — file tree, committed vs ignored paths, deterministic chaining via `state/`.
5. [04-data-schemas.md](./04-data-schemas.md) — every JSON / YAML / Markdown schema with examples.
6. [05-llm-and-agent-runtime.md](./05-llm-and-agent-runtime.md) — Anthropic Python SDK as harness (orchestrating multi-round verify in stage scripts; Claude Code CLI is *not* used — see DECISION block in that file), GLM-5.1 as LLM, config externalization.
7. [06-source-adapters.md](./06-source-adapters.md) — adding a source is a config edit; adapter contract; failure isolation.
8. [07-observability.md](./07-observability.md) — committed summary log format and what each stage writes.
9. [08-security.md](./08-security.md) — public-repo + GitHub-Secrets posture, PR rejection, secret-leak prevention, prompt-injection mitigation.

## One-page system overview

`ai-news-feed` runs as a chain of GitHub Actions workflows. Each stage is its own workflow. Each stage's "I am done" signal to the next stage is a **commit + push** of its output files into the repo. The next stage is triggered by `push` (or `workflow_run`) and its first job verifies that all upstream artifacts for the current `run_id` exist in the working tree before doing work.

```
                 cron (hourly)
                      |
                      v
              [bootstrap.yml]  <-- writes state/runs/<run_id>/state.json
                      |
                      |  (push)
                      v
   +------+------+------+------+------+
   |      |      |      |      |      |
[fetch-A][fetch-B][fetch-C][fetch-D][fetch-E]   parallel; one workflow per source category
   |      |      |      |      |      |
   +------+------+------+------+------+
                      |   each writes state/runs/<run_id>/fetch/<category>.json
                      |   and state/runs/<run_id>/fetch.<category>.done.json
                      v
   [filter.yml]   waits for ALL fetch.*.done.json for run_id; runs draft -> verify1 -> fix -> verify2 -> final
                      |   writes state/runs/<run_id>/candidates_final.json + filter.done.json
                      v
   [summarize.yml] runs draft -> verify1 -> fix -> verify2 -> final per item
                      |   writes state/runs/<run_id>/summaries/*.md + summarize.done.json
                      v
   [assemble.yml]  builds digest .md, updates index.json, updates README.md
                      |   commits + pushes
                      v
   [log-summary.yml] commits logs/YYYY-MM-DD/<run_id>-summary.md
```

### Why this shape

- **One workflow per fetch category** — failure isolation. A YouTube outage cannot block Hacker News.
- **Push as the inter-stage signal** — no shared cache, no inter-job artifacts to lose. The repo *is* the durable state. Re-running a single stage is `gh workflow run <stage>.yml`.
- **`run_id` keyed state under `state/runs/<run_id>/`** — multiple in-flight runs (e.g., the next hour starts before the previous finished) cannot collide.
- **Verify-before-work in every downstream stage** — if a stage fires from a push event for an unrelated commit, it exits 0 with `waiting for upstream` and writes nothing. Deterministic, resumable.
- **Public repo, secrets in GH Secrets, no PR-triggered execution** — cheapest secure pattern for a public-by-design dataset (see `08-security.md`).

## Conventions used in these docs

- All file paths use forward slashes.
- YAML/JSON shown is meant to be syntactically real and copy-pastable. Action SHAs appear as `@<sha>` placeholders.
- Every architectural choice has a one-line *why*.
- Where the spec is silent, decisions are tagged `> DECISION: ... because ... Spec was silent on this.` so reviewers can confirm or override.
