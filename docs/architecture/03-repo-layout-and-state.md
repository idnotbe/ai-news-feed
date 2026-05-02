# 03 — Repo layout and state

Purpose: define exactly which paths exist, which are committed, which are ignored, and how the per-run handoff state under `state/runs/<run_id>/` enables deterministic chaining.

## Table of contents

- [Full file tree](#full-file-tree)
- [Committed vs ignored](#committed-vs-ignored)
- [Why no `temp/` or `staging/`](#why-no-temp-or-staging)
- [Deterministic chaining via `state/`](#deterministic-chaining-via-state)
- [State retention](#state-retention)

## Full file tree

```
ai-news-feed/
├── README.md                                 # auto-updated; last 7 days
├── index.json                                # master index (all items, audience-agnostic)
├── .gitignore
├── requirements.txt                          # python deps
│
├── data/                                     # all configuration; editing here = adding sources / swapping LLM
│   ├── llm.yml                               # model, base_url, temperature, max_tokens
│   ├── perspectives.yml                      # closed enumeration of perspective_tags
│   └── sources/
│       ├── A-breaking.yml                    # Hacker News, TechCrunch AI, The Verge AI, VentureBeat AI
│       ├── B-vendor.yml                      # Anthropic, OpenAI, DeepMind, Microsoft AI, Meta AI
│       ├── C-curated.yml                     # smol.ai, TLDR AI, The Rundown AI, Ben's Bites
│       ├── D-youtube.yml                     # channel handles
│       └── E-security.yml                    # CISA KEV, vendor advisories
│
├── scripts/
│   ├── stages/
│   │   ├── bootstrap.py
│   │   ├── fetch.py
│   │   ├── filter.py
│   │   ├── summarize.py
│   │   ├── assemble.py
│   │   └── log_summary.py
│   ├── adapters/
│   │   ├── __init__.py
│   │   ├── base.py                           # adapter contract / abstract base
│   │   ├── rss.py                            # generic RSS / Atom (covers most categories)
│   │   ├── youtube_rss.py
│   │   ├── hn_algolia.py                     # Hacker News
│   │   ├── html_blog.py                      # vendor blogs that don't expose RSS
│   │   └── cisa_kev.py
│   └── lib/
│       ├── llm.py                            # GLM-5.1 client init (reads data/llm.yml; reads token from LLM_API_KEY env, populated from same-named GitHub Secret; default base_url targets Z.AI's Anthropic-API-compatible endpoint, see 05-llm-and-agent-runtime.md)
│       ├── dedup.py
│       ├── state_io.py                       # read/write state/runs/<run_id>/* helpers
│       └── markdown.py                       # digest + README rendering
│
├── .github/
│   └── workflows/
│       ├── bootstrap.yml
│       ├── fetch-A-breaking.yml
│       ├── fetch-B-vendor.yml
│       ├── fetch-C-curated.yml
│       ├── fetch-D-youtube.yml
│       ├── fetch-E-security.yml
│       ├── filter.yml
│       ├── summarize.yml
│       ├── assemble.yml
│       ├── log-summary.yml
│       ├── prune-state.yml
│       └── reject-external-pr.yml
│
├── digests/                                  # per-run published digest markdown
│   └── YYYY-MM-DD/
│       └── <run_id>.md                       # e.g. 20260502T0900Z.md
│
├── logs/                                     # canonical human-readable run summary
│   └── YYYY-MM-DD/
│       └── <run_id>-summary.md
│
├── state/                                    # per-run handoff between stages (committed)
│   └── runs/
│       └── <run_id>/
│           ├── state.json
│           ├── bootstrap.done.json
│           ├── fetch/
│           │   ├── A.json
│           │   ├── B.json
│           │   ├── C.json
│           │   ├── D.json
│           │   └── E.json
│           ├── fetch.A.done.json
│           ├── fetch.B.done.json
│           ├── fetch.C.done.json
│           ├── fetch.D.done.json
│           ├── fetch.E.done.json
│           ├── candidates_v1.json
│           ├── verify1_filter.md
│           ├── candidates_v2.json
│           ├── verify2_filter.md
│           ├── candidates_final.json
│           ├── filter.done.json
│           ├── summaries/
│           │   └── <item_id>.md
│           ├── summaries_v2/
│           │   └── <item_id>.md
│           ├── summaries_final/
│           │   └── <item_id>.md
│           ├── verify1_summaries.md
│           ├── verify2_summaries.md
│           ├── summarize.done.json
│           └── assemble.done.json
│
├── tests/                                    # adapter + state_io + dedup unit tests
│   └── ...
│
└── docs/
    ├── requirements/
    └── architecture/
```

## Committed vs ignored

**Committed**:
- `README.md`, `index.json`, all `digests/**`, all `logs/**`, all `state/runs/**`.
- All `data/**`, `scripts/**`, `.github/**`, `tests/**`, `docs/**`.

**`.gitignore`**:
```
# language
__pycache__/
*.pyc
.venv/
.python-version

# editor
.vscode/
.idea/

# transient
*.log
.DS_Store

# never commit secrets
.env
*.env
```

> DECISION: `state/` IS committed. Spec was silent on whether per-run state should be committed (the legacy `temp/` was always gitignored). *Why*: the entire deterministic-chain design depends on the next stage being able to *see* the previous stage's output in the working tree after a `push` event. If `state/` were ignored, we would need an alternate cross-stage transport (artifacts, cache, external store). Committing is the simplest mechanism with adequate guarantees. The cost is repo size growth, addressed below in "State retention".

> DECISION: there is **no `temp/` directory** in this repo. The legacy routine used `temp/` as in-process working memory between subagents within a single claude.ai run. Here, every stage runs in a fresh GH Actions runner with no shared filesystem; the cross-stage equivalent is `state/runs/<run_id>/`, which is committed. Within a single stage's runner, `/tmp/` (the runner's ephemeral disk) is fine for adapter scratch space. Spec was silent on whether to keep a `temp/` convention.

## Why no `temp/` or `staging/`

In the legacy claude.ai routine, all phases of one run shared a Python process and a `temp/` directory on the user's machine. In this repo:

- Each workflow is a separate runner, freshly provisioned, with no shared disk.
- The only durable cross-stage transport is the Git repo itself (or the explicit Actions artifact/cache mechanisms, which have retention limits and require extra setup).
- We chose Git as the transport. `state/runs/<run_id>/` is the result.

A `staging/` directory was considered (e.g., `staging/<run_id>/digest.md` written by `assemble`, then `publish` moves it to `digests/`). Rejected: adds an extra commit and a rename operation with no benefit, since `assemble` already verifies it has work to do before writing.

## Deterministic chaining via `state/`

The pattern in one paragraph: stage N writes its output files into `state/runs/<run_id>/...`, then writes `state/runs/<run_id>/<stage-N>.done.json`, then commits and pushes. Stage N+1 is triggered by that push, extracts `<run_id>` from the changed-file path, and as its first work step asserts the existence of all required upstream done markers under `state/runs/<run_id>/`. If any are missing, it logs `waiting for upstream: <path>` and exits 0.

Concrete first-step contract for a downstream stage:

```bash
set -euo pipefail
RUN_ID="$1"
REQUIRED=( "$2" )   # e.g. "filter.done.json"
for f in "${REQUIRED[@]}"; do
  if [ ! -f "state/runs/$RUN_ID/$f" ]; then
    echo "waiting for upstream: state/runs/$RUN_ID/$f"
    exit 0
  fi
done
```

Three properties this gives us:

1. **Resumable**: re-trigger any single stage; it picks up where the chain left off.
2. **Idempotent (per-run)**: a stage's outputs are functions of its inputs in `state/runs/<run_id>/`. Re-running overwrites in place.
3. **Audit-able**: `git log -- state/runs/<run_id>/` is the timeline of one run.

## State retention

Per-run state directories accumulate. Hourly runs × 24 × 365 = ~8,800 directories per year, each containing maybe 50–200 KB. Within GitHub repo size limits, but cluttering.

**Retention policy** (implemented as a dedicated scheduled workflow `prune-state.yml`):

> DECISION: prune is its **own** workflow file (`.github/workflows/prune-state.yml`) on a daily cron, not bolted onto `log-summary.yml`'s last step. *Why*: the log-summary path is on the hot per-run chain; a prune that occasionally takes seconds (large `git rm -r` of dozens of run directories) would slow every run. A separate daily workflow keeps the chain fast and makes the prune cadence explicit.

```yaml
# .github/workflows/prune-state.yml
name: prune-state
on:
  schedule:
    - cron: '17 3 * * *'   # daily, 03:17 UTC — off-peak vs hourly bootstrap
  workflow_dispatch: {}
permissions:
  contents: write
concurrency:
  group: prune-state
  cancel-in-progress: false
jobs:
  prune:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@<sha>
        with: { fetch-depth: 1, persist-credentials: true }
      - name: Prune old run state
        run: |
          set -euo pipefail
          git config user.name  "ai-news-feed-bot"
          git config user.email "ai-news-feed-bot@users.noreply.github.com"
          # Keep state/runs/<run_id>/ for 30 days.
          find state/runs -mindepth 1 -maxdepth 1 -type d -mtime +30 -print \
               -exec git rm -r {} + || true
          if git diff --cached --quiet; then exit 0; fi
          git commit -m "chore: prune state older than 30 days"
          # Same rebase-retry loop as every other commit + push (see
          # 02-github-actions.md "Common commit + push").
          pushed=0
          for i in 1 2 3 4 5; do
            if git pull --rebase --autostash origin "$GITHUB_REF_NAME" \
               && git push origin "HEAD:$GITHUB_REF_NAME"; then
              pushed=1; break
            fi
            sleep $((RANDOM % 5 + 1))
          done
          [ "$pushed" -eq 1 ] || { echo "prune push failed after 5 attempts"; exit 1; }
```

> DECISION: 30-day retention for `state/runs/`. Spec was silent. *Why*: 30 days covers any realistic "what went wrong last week" investigation; older debugging would use `logs/` (which is permanent and small). Git history retains the deleted state forever for true forensics.

**Pre-prune size estimate**: 30 days × 24 runs/day × ~50–200 KB per run directory ≈ **36–144 MB** of `state/runs/` on disk before the daily prune kicks in. Within GitHub repo limits (1 GB recommended, 5 GB hard) but worth knowing — if the per-run directory grows (more sources, longer verify .md files), revisit the retention window.

> NOTE: `state.json` itself grows linearly with `index.json` independent of the prune cadence — bootstrap rewrites the whole `existing_items` array on every run. At 1000 items × 1.5 sources avg ≈ 225 KB per file × hourly = ~5 MB/day of `state.json` content. Within limits, but cross-referenced from `04-data-schemas.md` `state.json` schema. Future optimization paths if size is felt: (a) windowed `existing_items` (only items from last N=14 days), (b) Bloom filter for the URL set with the LLM stage absorbing small false-positive rates. Both are deferred until size is felt.

`logs/` and `digests/` are **never pruned** — they are the canonical published record. `index.json` grows monotonically (subject to per-item updates via dedup).
