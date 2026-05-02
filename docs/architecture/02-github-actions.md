# 02 — GitHub Actions

Purpose: enumerate every workflow file, its trigger, inputs, outputs, secrets, permissions, and runner. Action versions appear as `@<sha>` placeholders — pin to commit SHAs at implementation time.

## Table of contents

- [Workflow inventory](#workflow-inventory)
- [Run-id convention](#run-id-convention)
- [Shared snippets](#shared-snippets)
- [bootstrap.yml](#bootstrapyml)
- [fetch-A-breaking.yml (pattern for all fetch-*.yml)](#fetch-a-breakingyml-pattern-for-all-fetch-yml)
- [filter.yml](#filteryml)
- [summarize.yml](#summarizeyml)
- [assemble.yml](#assembleyml)
- [log-summary.yml](#log-summaryyml)
- [reject-external-pr.yml](#reject-external-pryml)

## Workflow inventory

| File | Trigger | Purpose |
|---|---|---|
| `bootstrap.yml` | `schedule` (cron, hourly) + `workflow_dispatch` | Generate run_id and seed `state/runs/<run_id>/state.json`. |
| `fetch-A-breaking.yml` | `push` on `state/runs/*/bootstrap.done.json` | Fetch category A sources. |
| `fetch-B-vendor.yml` | same | Fetch category B (vendor blogs). |
| `fetch-C-curated.yml` | same | Fetch category C (curation dailies). |
| `fetch-D-youtube.yml` | same | Fetch category D (YouTube channel RSS). |
| `fetch-E-security.yml` | same | Fetch category E (security advisories). |
| `filter.yml` | `push` on `state/runs/*/fetch.*.done.json` | Dedup + 4-axis filter, 2-round verify. |
| `summarize.yml` | `push` on `state/runs/*/filter.done.json` | Per-item English summary, 2-round verify. |
| `assemble.yml` | `push` on `state/runs/*/summarize.done.json` | Build digest, update index.json + README, commit, push. |
| `log-summary.yml` | `push` on `state/runs/*/assemble.done.json` | Write canonical run summary log. |
| `prune-state.yml` | `schedule` (cron, daily 03:17 UTC) + `workflow_dispatch` | Delete `state/runs/<run_id>/` directories older than 30 days. See `03-repo-layout-and-state.md`. |
| `reject-external-pr.yml` | `pull_request_target: opened` | Auto-close PRs from forks. Only PR-triggered workflow. |

## Run-id convention

- Format: **`YYYYMMDDTHHMMZ`** (UTC, e.g. `20260502T0900Z`).
- Generated **once** by `bootstrap.yml` via `date -u +%Y%m%dT%H%MZ`.
- Embedded in **every per-run path**: `state/runs/<run_id>/...`, `digests/YYYY-MM-DD/<run_id>.md`, `logs/YYYY-MM-DD/<run_id>-summary.md`.
- Each downstream workflow extracts run_id from the file path of the commit that triggered it (see `01-pipeline-stages.md` "Inter-stage chaining").

> DECISION: minute resolution (`HHMM`) instead of second resolution. Spec was silent on the exact format beyond "UTC timestamp". *Why*: hourly cron + manual workflow_dispatch will not collide at minute resolution; second resolution would just lengthen filenames.

## Shared snippets

These appear in most workflow files. Defined once here so docs stay readable.

### Common `permissions` block

```yaml
permissions:
  contents: write     # commit + push state files and outputs
  actions: read       # for workflow_run trigger metadata if used
  pull-requests: none
  id-token: none
```

*Why minimal*: principle of least privilege. Only `contents: write` is needed (and only for stages that write). The `reject-external-pr.yml` workflow uses a different, more restricted set.

### Common checkout step

```yaml
- name: Checkout
  uses: actions/checkout@<sha>
  with:
    fetch-depth: 1
    persist-credentials: true   # so we can git push at end of stage
```

### Common Python setup

> DECISION: implementation language is **Python 3.12**. Spec was silent. *Why*: Anthropic SDK and most adapter HTTP/parsing libs are first-class in Python; AX-consultant primary maintainer is more likely to read Python than TypeScript.

```yaml
- name: Setup Python
  uses: actions/setup-python@<sha>
  with:
    python-version: '3.12'
    cache: pip
- name: Install deps
  run: pip install -r requirements.txt
```

### Common commit + push (with rebase-retry loop)

Every stage that pushes uses **this** pattern. *Why a loop*: the five `fetch-*.yml` workflows triggered by a single bootstrap push run in parallel and all attempt to push within seconds of each other. Without rebase-on-conflict, the first push wins and the other four hit non-fast-forward rejection — silently breaking the chain (no `fetch.<cat>.done.json` lands for those four, so `filter.yml` waits forever). The same race recurs at every fan-in/fan-out boundary in the chain.

```yaml
- name: Commit and push
  env:
    STAGE: ${{ env.STAGE }}      # e.g. "fetch-A"
    RUN_ID: ${{ steps.resolve.outputs.run_id }}
  run: |
    set -euo pipefail
    git config user.name  "ai-news-feed-bot"
    git config user.email "ai-news-feed-bot@users.noreply.github.com"
    # add only the paths this stage owns
    git add state/runs/"$RUN_ID"/
    if git diff --cached --quiet; then
      echo "No changes to commit."
      exit 0
    fi
    git commit -m "$STAGE($RUN_ID)"
    # Bounded fetch+rebase+retry. Required because parallel sibling workflows
    # (e.g. fetch-A..E) push to the same branch within the same few seconds.
    pushed=0
    for i in 1 2 3 4 5; do
      if git pull --rebase --autostash origin "$GITHUB_REF_NAME" \
         && git push origin "HEAD:$GITHUB_REF_NAME"; then
        pushed=1
        break
      fi
      sleep $((RANDOM % 5 + 1))
    done
    if [ "$pushed" -ne 1 ]; then
      echo "ERROR: failed to push after 5 rebase-retry attempts."
      exit 1
    fi
```

*Why scoped `git add`*: avoids accidentally committing untracked junk that landed in the runner's workspace.

*Why `--autostash`*: defensive — the working tree should be clean (we just committed), but a leftover untracked or modified file from a prior step would otherwise abort `git pull --rebase`.

*Why bounded retries with final `exit 1`*: silent push failures are the worst outcome (downstream stages wait indefinitely). Five attempts cover the realistic 5-way fan-in case; exhausting them means something is actually wrong, and we want the workflow to go red so the maintainer notices.

> DECISION: bounded retry with hard fail. Spec was silent on commit conflicts. *Why*: an unbounded loop could pin a runner for the full 6-hour limit during an upstream incident; a single attempt would silently lose state on every busy run.

**This pattern is referenced from**:
- `00-system-overview.md` design principle #5 ("the repo is the durable state").
- `01-pipeline-stages.md` — every stage's "commit + push" step.
- This file — every workflow snippet below uses this pattern (shown inline once, then assumed).

## bootstrap.yml

```yaml
name: bootstrap
on:
  schedule:
    - cron: '0 * * * *'   # every hour on the hour, UTC
  workflow_dispatch: {}
permissions:
  contents: write
concurrency:
  group: bootstrap
  cancel-in-progress: false
jobs:
  bootstrap:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@<sha>
        with: { fetch-depth: 1, persist-credentials: true }
      - uses: actions/setup-python@<sha>
        with: { python-version: '3.12', cache: pip }
      - run: pip install -r requirements.txt
      - name: Generate run_id and state.json
        id: bootstrap
        run: |
          set -euo pipefail
          RUN_ID=$(date -u +%Y%m%dT%H%MZ)
          echo "run_id=$RUN_ID" >> "$GITHUB_OUTPUT"
          mkdir -p "state/runs/$RUN_ID/fetch" "state/runs/$RUN_ID/summaries"
          python scripts/stages/bootstrap.py --run-id "$RUN_ID"
      - name: Commit and push
        env:
          RUN_ID: ${{ steps.bootstrap.outputs.run_id }}
        run: |
          set -euo pipefail
          # Uses the "Common commit + push (with rebase-retry loop)" pattern
          # documented above. Inlined here for readability of the full bootstrap.yml.
          git config user.name  "ai-news-feed-bot"
          git config user.email "ai-news-feed-bot@users.noreply.github.com"
          git add "state/runs/$RUN_ID/"
          if git diff --cached --quiet; then exit 0; fi
          git commit -m "bootstrap($RUN_ID)"
          pushed=0
          for i in 1 2 3 4 5; do
            if git pull --rebase --autostash origin "$GITHUB_REF_NAME" \
               && git push origin "HEAD:$GITHUB_REF_NAME"; then
              pushed=1; break
            fi
            sleep $((RANDOM % 5 + 1))
          done
          [ "$pushed" -eq 1 ] || { echo "push failed after 5 attempts"; exit 1; }
```

**Inputs**: none.
**Outputs**: `state/runs/<run_id>/state.json`, `bootstrap.done.json`.
**Secrets**: none (bootstrap does not call the LLM).
**Runner**: `ubuntu-latest`.
**Why hourly cron**: matches the legacy claude.ai routine cadence. Cron is UTC-fixed, no surprises with DST.

## fetch-A-breaking.yml (pattern for all fetch-*.yml)

This workflow uses the **two-job pattern** required for any `cancel-in-progress: true` workflow. Job 1 (`resolve`) has no concurrency gate; it only resolves `run_id` and exposes it as a job output. Job 2 (`work`) declares `needs: resolve` and gates concurrency on the **actual resolved run_id**, not on a `'pending'` placeholder. See `01-pipeline-stages.md` "Concurrency keys" for why a single-job pattern with `${{ inputs.run_id || 'pending' }}` would silently cancel a still-resolving prior-hour run when a fresh hourly bootstrap fires.

```yaml
name: fetch-A-breaking
on:
  push:
    paths:
      - 'state/runs/*/bootstrap.done.json'
  workflow_dispatch:
    inputs:
      run_id:
        description: "Run ID to operate on. If omitted, parsed from the push event."
        required: false
        type: string
permissions:
  contents: write
env:
  CATEGORY: A
  STAGE: fetch-A
jobs:
  resolve:
    runs-on: ubuntu-latest
    # No concurrency gate here. This job is cheap (~5 sec) and exists only to
    # produce the resolved run_id BEFORE the work job's concurrency check runs.
    outputs:
      run_id: ${{ steps.resolve.outputs.run_id }}
    steps:
      - uses: actions/checkout@<sha>
        with: { fetch-depth: 1, persist-credentials: false }
      - name: Resolve run_id
        id: resolve
        env:
          INPUT_RUN_ID: ${{ inputs.run_id }}
        run: |
          set -euo pipefail
          # Prefer the workflow_dispatch input. Else scan UNION of added/modified
          # across .commits[] (NOT just .head_commit; see 01-pipeline-stages.md
          # "Inter-stage chaining").
          if [ -n "${INPUT_RUN_ID:-}" ]; then
            RUN_ID="$INPUT_RUN_ID"
          else
            RUN_ID=$(jq -r '
              [ .commits[]?.added[]?, .commits[]?.modified[]? ] | .[]
            ' "$GITHUB_EVENT_PATH" \
              | grep -E '^state/runs/[^/]+/bootstrap\.done\.json$' \
              | awk -F/ '{print $3}' \
              | sort -u | head -n1)
          fi
          if [ -z "$RUN_ID" ]; then echo "No run_id; exit."; exit 0; fi
          echo "run_id=$RUN_ID" >> "$GITHUB_OUTPUT"

  work:
    needs: resolve
    if: needs.resolve.outputs.run_id != ''
    runs-on: ubuntu-latest
    # Group key is the ACTUAL resolved run_id (never 'pending'). Two pushes
    # carrying different run_ids land in different groups and never collide.
    concurrency:
      group: fetch-A-${{ github.ref }}-${{ needs.resolve.outputs.run_id }}
      cancel-in-progress: true
    steps:
      - uses: actions/checkout@<sha>
        with: { fetch-depth: 1, persist-credentials: true }
      - uses: actions/setup-python@<sha>
        with: { python-version: '3.12', cache: pip }
      - run: pip install -r requirements.txt
      - name: Verify upstream
        env:
          RUN_ID: ${{ needs.resolve.outputs.run_id }}
        run: |
          set -euo pipefail
          test -f "state/runs/$RUN_ID/bootstrap.done.json" \
            || { echo "waiting for bootstrap"; exit 0; }
      - name: Run fetch
        env:
          LLM_API_KEY: ${{ secrets.LLM_API_KEY }}
          RUN_ID: ${{ needs.resolve.outputs.run_id }}
        run: |
          set -euo pipefail
          python scripts/stages/fetch.py --run-id "$RUN_ID" --category "$CATEGORY"
      - name: Commit and push
        env:
          RUN_ID: ${{ needs.resolve.outputs.run_id }}
        run: |
          set -euo pipefail
          # Uses the "Common commit + push (with rebase-retry loop)" pattern.
          # CRITICAL for fetch-*.yml: all five fetch workflows push to the same
          # branch within seconds of each other; without the rebase-retry loop,
          # four out of five would hit non-fast-forward and the chain would stall.
          git config user.name  "ai-news-feed-bot"
          git config user.email "ai-news-feed-bot@users.noreply.github.com"
          git add "state/runs/$RUN_ID/"
          if git diff --cached --quiet; then exit 0; fi
          git commit -m "$STAGE($RUN_ID)"
          pushed=0
          for i in 1 2 3 4 5; do
            if git pull --rebase --autostash origin "$GITHUB_REF_NAME" \
               && git push origin "HEAD:$GITHUB_REF_NAME"; then
              pushed=1; break
            fi
            sleep $((RANDOM % 5 + 1))
          done
          [ "$pushed" -eq 1 ] || { echo "push failed after 5 attempts"; exit 1; }
```

**Inputs**: `state/runs/<run_id>/state.json`, `data/sources/<category>.yml`.
**Outputs**: `state/runs/<run_id>/fetch/A.json`, `state/runs/<run_id>/fetch.A.done.json`.
**Secrets**: `LLM_API_KEY` (provider-agnostic GitHub Secret name; populated with the Z.AI API key by default. Exposed to the script as env var `LLM_API_KEY`; the loader at `scripts/lib/llm.py` reads it and passes it to the SDK with the right kwarg based on `data/llm.yml`'s `auth_mode`. Only adapters that need LLM-driven extraction read it; most don't).
**Runner**: `ubuntu-latest`.

The other four `fetch-*.yml` files (`fetch-B-vendor.yml`, `fetch-C-curated.yml`, `fetch-D-youtube.yml`, `fetch-E-security.yml`) are byte-identical to this one except for the `name`, `CATEGORY`, and `STAGE` env vars. They all use the same two-job pattern. *Why duplicate-with-trivial-diffs instead of one matrix workflow*: spec §3 explicitly asks for "one workflow per source category" so that one category's failure does not abort the others. A matrix in a single workflow shares failure semantics and makes per-category re-trigger awkward.

## filter.yml

`filter.yml` is the **canonical example** of the two-job pattern. `summarize.yml` and the five `fetch-*.yml` files all follow the same skeleton; only the trigger paths, the regex inside `Resolve run_id`, the `Verify upstream` step body, and the script invocation differ.

CRITICAL: filter is fan-in. Five `fetch-*.yml` workflows each push their `fetch.<cat>.done.json`, so filter would receive five separate push triggers in close succession. The two-job pattern (a) collapses the five push-triggered invocations into one work-job per run_id (because they all resolve to the same run_id and share a concurrency group), and (b) does **not** silently cancel a still-running prior-hour work-job when a fresh hour's bootstrap fires (because the prior hour's group key is the prior hour's run_id, not `'pending'`).

```yaml
name: filter
on:
  push:
    paths:
      - 'state/runs/*/fetch.*.done.json'
  workflow_dispatch:
    inputs:
      run_id:
        description: "Run ID to operate on. If omitted, parsed from the push event."
        required: false
        type: string
permissions:
  contents: write
env:
  STAGE: filter
jobs:
  resolve:
    runs-on: ubuntu-latest
    # No concurrency gate. Cheap (~5 sec) — exists only to compute run_id
    # BEFORE the work job's concurrency check evaluates the group key.
    outputs:
      run_id: ${{ steps.resolve.outputs.run_id }}
    steps:
      - uses: actions/checkout@<sha>
        with: { fetch-depth: 1, persist-credentials: false }
      - name: Resolve run_id
        id: resolve
        env:
          INPUT_RUN_ID: ${{ inputs.run_id }}
        run: |
          set -euo pipefail
          # Prefer the workflow_dispatch input. Else scan UNION of added/modified
          # across .commits[] (NOT just .head_commit; see 01-pipeline-stages.md
          # "Inter-stage chaining").
          if [ -n "${INPUT_RUN_ID:-}" ]; then
            RUN_ID="$INPUT_RUN_ID"
          else
            RUN_ID=$(jq -r '
              [ .commits[]?.added[]?, .commits[]?.modified[]? ] | .[]
            ' "$GITHUB_EVENT_PATH" \
              | grep -E '^state/runs/[^/]+/fetch\.[A-Z]\.done\.json$' \
              | awk -F/ '{print $3}' \
              | sort -u | head -n1)
          fi
          if [ -z "$RUN_ID" ]; then echo "No run_id; exit."; exit 0; fi
          echo "run_id=$RUN_ID" >> "$GITHUB_OUTPUT"

  work:
    needs: resolve
    if: needs.resolve.outputs.run_id != ''
    runs-on: ubuntu-latest
    # Group key uses the ACTUAL resolved run_id. Two pushes carrying different
    # run_ids land in DIFFERENT groups; cancel-in-progress: true correctly
    # cancels only same-run_id duplicates (the desired collapse-the-fan-in
    # behavior). It does NOT cancel a still-running prior-hour invocation.
    concurrency:
      group: filter-${{ github.ref }}-${{ needs.resolve.outputs.run_id }}
      cancel-in-progress: true
    steps:
      - uses: actions/checkout@<sha>
        with: { fetch-depth: 1, persist-credentials: true }
      - uses: actions/setup-python@<sha>
        with: { python-version: '3.12', cache: pip }
      - run: pip install -r requirements.txt
      - name: Verify upstream (all fetch categories complete)
        env:
          RUN_ID: ${{ needs.resolve.outputs.run_id }}
        run: |
          set -euo pipefail
          CATS=$(jq -r '.source_categories[]' "state/runs/$RUN_ID/state.json")
          for C in $CATS; do
            if [ ! -f "state/runs/$RUN_ID/fetch.$C.done.json" ]; then
              echo "waiting for fetch.$C"
              exit 0
            fi
          done
      - name: Run filter (draft + 2-round verify + final)
        env:
          LLM_API_KEY: ${{ secrets.LLM_API_KEY }}
          RUN_ID: ${{ needs.resolve.outputs.run_id }}
        run: |
          set -euo pipefail
          python scripts/stages/filter.py --run-id "$RUN_ID"
      - name: Commit and push
        env:
          RUN_ID: ${{ needs.resolve.outputs.run_id }}
        run: |
          set -euo pipefail
          # Uses the "Common commit + push (with rebase-retry loop)" pattern.
          git config user.name  "ai-news-feed-bot"
          git config user.email "ai-news-feed-bot@users.noreply.github.com"
          git add "state/runs/$RUN_ID/"
          if git diff --cached --quiet; then exit 0; fi
          git commit -m "$STAGE($RUN_ID)"
          pushed=0
          for i in 1 2 3 4 5; do
            if git pull --rebase --autostash origin "$GITHUB_REF_NAME" \
               && git push origin "HEAD:$GITHUB_REF_NAME"; then
              pushed=1; break
            fi
            sleep $((RANDOM % 5 + 1))
          done
          [ "$pushed" -eq 1 ] || { echo "push failed after 5 attempts"; exit 1; }
```

**Inputs**: all `state/runs/<run_id>/fetch/*.json` + `state.json`.
**Outputs**: `candidates_v1.json`, `verify1_filter.md`, `candidates_v2.json`, `verify2_filter.md`, `candidates_final.json`, `filter.done.json` (all under `state/runs/<run_id>/`). On the maintainer-review halt path (see `01-pipeline-stages.md` Stage 2 step 5), `candidates_final.json` is intentionally omitted and `escape_hatch_review_required.md` is written instead; `filter.done.json` carries `escape_hatch_review_required: true` so downstream `summarize.yml` knows to wait.
**Secrets**: `LLM_API_KEY` (provider-agnostic GitHub Secret name; populated with the Z.AI API key by default).
**Runner**: `ubuntu-latest`.

## summarize.yml

Same two-job skeleton as `filter.yml`. The deltas vs filter are:
- **trigger paths**: `state/runs/*/filter.done.json`
- **regex inside `Resolve run_id`**: `^state/runs/[^/]+/filter\.done\.json$`
- **`Verify upstream` step**: checks for `filter.done.json` (and short-circuits if `item_count: 0`)
- **script invoked**: `scripts/stages/summarize.py`

Full YAML below — readers can copy directly:

```yaml
name: summarize
on:
  push:
    paths:
      - 'state/runs/*/filter.done.json'
  workflow_dispatch:
    inputs:
      run_id:
        description: "Run ID to operate on."
        required: false
        type: string
permissions:
  contents: write
env:
  STAGE: summarize
jobs:
  resolve:
    runs-on: ubuntu-latest
    outputs:
      run_id: ${{ steps.resolve.outputs.run_id }}
    steps:
      - uses: actions/checkout@<sha>
        with: { fetch-depth: 1, persist-credentials: false }
      - name: Resolve run_id
        id: resolve
        env:
          INPUT_RUN_ID: ${{ inputs.run_id }}
        run: |
          set -euo pipefail
          if [ -n "${INPUT_RUN_ID:-}" ]; then
            RUN_ID="$INPUT_RUN_ID"
          else
            RUN_ID=$(jq -r '
              [ .commits[]?.added[]?, .commits[]?.modified[]? ] | .[]
            ' "$GITHUB_EVENT_PATH" \
              | grep -E '^state/runs/[^/]+/filter\.done\.json$' \
              | awk -F/ '{print $3}' \
              | sort -u | head -n1)
          fi
          if [ -z "$RUN_ID" ]; then echo "No run_id; exit."; exit 0; fi
          echo "run_id=$RUN_ID" >> "$GITHUB_OUTPUT"

  work:
    needs: resolve
    if: needs.resolve.outputs.run_id != ''
    runs-on: ubuntu-latest
    concurrency:
      group: summarize-${{ github.ref }}-${{ needs.resolve.outputs.run_id }}
      cancel-in-progress: true
    steps:
      - uses: actions/checkout@<sha>
        with: { fetch-depth: 1, persist-credentials: true }
      - uses: actions/setup-python@<sha>
        with: { python-version: '3.12', cache: pip }
      - run: pip install -r requirements.txt
      - name: Verify upstream
        env:
          RUN_ID: ${{ needs.resolve.outputs.run_id }}
        run: |
          set -euo pipefail
          test -f "state/runs/$RUN_ID/filter.done.json" \
            || { echo "waiting for filter.done.json"; exit 0; }
          # If filter halted pending maintainer review, do not proceed. The
          # rerun of filter.yml after curation will re-emit a push that
          # retriggers this workflow with escape_hatch_review_required=false
          # and a real candidates_final.json.
          REVIEW=$(jq -r '.escape_hatch_review_required // false' \
            "state/runs/$RUN_ID/filter.done.json")
          if [ "$REVIEW" = "true" ]; then
            echo "filter is paused for maintainer review of escape-hatch overflow; waiting"
            exit 0
          fi
          # Short-circuit if filter produced zero items.
          ITEMS=$(jq -r '.item_count // .item_count_out // 0' \
            "state/runs/$RUN_ID/filter.done.json")
          if [ "$ITEMS" -eq 0 ]; then
            echo "filter produced 0 items; summarize.py will write skipped done.json"
          fi
      - name: Run summarize (draft + 2-round verify + final, parallel per item)
        env:
          LLM_API_KEY: ${{ secrets.LLM_API_KEY }}
          RUN_ID: ${{ needs.resolve.outputs.run_id }}
        run: |
          set -euo pipefail
          python scripts/stages/summarize.py --run-id "$RUN_ID"
      - name: Commit and push
        env:
          RUN_ID: ${{ needs.resolve.outputs.run_id }}
        run: |
          set -euo pipefail
          # Uses the "Common commit + push (with rebase-retry loop)" pattern,
          # inlined here so a maintainer copy-pasting summarize.yml has the
          # loop literally on the page.
          git config user.name  "ai-news-feed-bot"
          git config user.email "ai-news-feed-bot@users.noreply.github.com"
          git add "state/runs/$RUN_ID/"
          if git diff --cached --quiet; then exit 0; fi
          git commit -m "$STAGE($RUN_ID)"
          pushed=0
          for i in 1 2 3 4 5; do
            if git pull --rebase --autostash origin "$GITHUB_REF_NAME" \
               && git push origin "HEAD:$GITHUB_REF_NAME"; then
              pushed=1; break
            fi
            sleep $((RANDOM % 5 + 1))
          done
          [ "$pushed" -eq 1 ] || { echo "push failed after 5 attempts"; exit 1; }
```

The script `scripts/stages/summarize.py` is responsible for:
- short-circuiting (writing `summarize.done.json` with `skipped: true`) if `filter.done.json` reports `item_count: 0`,
- parallelizing per-item summarize calls (Python `concurrent.futures.ThreadPoolExecutor`, since most time is LLM I/O wait),
- running the 2-round verify cycle.

**Outputs**: `state/runs/<run_id>/summaries/<id>.md`, `summaries_v2/`, `summaries_final/`, `verify1_summaries.md`, `verify2_summaries.md`, `summarize.done.json`.

> DECISION: parallelize per-item summarize **inside one job** rather than spawning a matrix of jobs. Spec was silent. *Why*: per-item LLM calls are tens-of-seconds each; a matrix would multiply checkout/setup overhead. One job × thread pool is faster and cheaper.

## assemble.yml

Trigger and concurrency:

```yaml
on:
  push:
    paths:
      - 'state/runs/*/summarize.done.json'
  workflow_dispatch:
    inputs:
      run_id:
        description: "Run ID to operate on."
        required: false
        type: string
permissions:
  contents: write
concurrency:
  group: assemble-${{ github.ref }}-${{ inputs.run_id || 'pending' }}
  cancel-in-progress: false   # never cancel in-flight assemble; missed = lost publication
```

The `Resolve run_id` step uses the same union-of-`.commits[]` parser, regex narrowed to `^state/runs/[^/]+/summarize\.done\.json$`. The job runs `scripts/stages/assemble.py`, which:
- short-circuits if `summarize.done.json` reports `skipped: true`,
- writes `digests/YYYY-MM-DD/<run_id>.md` **atomically** (write to a `.tmp` filename, `os.replace` to final name),
- updates `index.json` atomically (same temp+rename pattern),
- regenerates `README.md` atomically,
- writes `assemble.done.json` **last**, only after all of digest + index + README are durable on disk,
- the final commit step adds *both* `state/runs/<run_id>/` and `digests/`, `index.json`, `README.md` so that one commit represents the publication.

*Why atomic write + done-marker-last*: a crash between writing the digest and updating the index would otherwise leave the working tree half-published (digest committed but `index.json` stale). Writing the done marker last means a missing/stale `assemble.done.json` is the unambiguous signal "rerun assemble for this run_id."

```yaml
- name: Commit and push (publication)
  env:
    RUN_ID: ${{ steps.resolve.outputs.run_id }}
  run: |
    set -euo pipefail
    # Uses the "Common commit + push (with rebase-retry loop)" pattern.
    # Publication commit touches digests/, index.json, README.md — all of which
    # may have been modified by a parallel log-summary or prune push that beat
    # us to the branch. The rebase-retry covers that.
    git config user.name  "ai-news-feed-bot"
    git config user.email "ai-news-feed-bot@users.noreply.github.com"
    git add "state/runs/$RUN_ID/" "digests/" "index.json" "README.md"
    if git diff --cached --quiet; then exit 0; fi
    git commit -m "publish($RUN_ID): N items"
    pushed=0
    for i in 1 2 3 4 5; do
      if git pull --rebase --autostash origin "$GITHUB_REF_NAME" \
         && git push origin "HEAD:$GITHUB_REF_NAME"; then
        pushed=1; break
      fi
      sleep $((RANDOM % 5 + 1))
    done
    [ "$pushed" -eq 1 ] || { echo "push failed after 5 attempts"; exit 1; }
```

## log-summary.yml

Single-job pattern (because `cancel-in-progress: false` — queueing replaces cancelling, so a `'pending'` group key cannot lose a run). Full YAML below — readers can copy directly:

```yaml
name: log-summary
on:
  push:
    paths:
      - 'state/runs/*/assemble.done.json'
  workflow_dispatch:
    inputs:
      run_id:
        description: "Run ID to operate on."
        required: false
        type: string
permissions:
  contents: write
concurrency:
  group: log-${{ github.ref }}-${{ inputs.run_id || 'pending' }}
  cancel-in-progress: false
env:
  STAGE: log-summary
jobs:
  log:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@<sha>
        with: { fetch-depth: 1, persist-credentials: true }
      - uses: actions/setup-python@<sha>
        with: { python-version: '3.12', cache: pip }
      - run: pip install -r requirements.txt
      - name: Resolve run_id
        id: resolve
        env:
          INPUT_RUN_ID: ${{ inputs.run_id }}
        run: |
          set -euo pipefail
          if [ -n "${INPUT_RUN_ID:-}" ]; then
            RUN_ID="$INPUT_RUN_ID"
          else
            RUN_ID=$(jq -r '
              [ .commits[]?.added[]?, .commits[]?.modified[]? ] | .[]
            ' "$GITHUB_EVENT_PATH" \
              | grep -E '^state/runs/[^/]+/assemble\.done\.json$' \
              | awk -F/ '{print $3}' \
              | sort -u | head -n1)
          fi
          if [ -z "$RUN_ID" ]; then echo "No run_id; exit."; exit 0; fi
          echo "run_id=$RUN_ID" >> "$GITHUB_OUTPUT"
      - name: Verify upstream
        env:
          RUN_ID: ${{ steps.resolve.outputs.run_id }}
        run: |
          set -euo pipefail
          test -f "state/runs/$RUN_ID/assemble.done.json" \
            || { echo "waiting for assemble"; exit 0; }
      - name: Render run summary log
        env:
          RUN_ID: ${{ steps.resolve.outputs.run_id }}
        run: |
          set -euo pipefail
          python scripts/stages/log_summary.py --run-id "$RUN_ID"
      - name: Commit and push
        env:
          RUN_ID: ${{ steps.resolve.outputs.run_id }}
        run: |
          set -euo pipefail
          # Uses the "Common commit + push (with rebase-retry loop)" pattern,
          # inlined here so a maintainer copy-pasting log-summary.yml has the
          # loop literally on the page.
          git config user.name  "ai-news-feed-bot"
          git config user.email "ai-news-feed-bot@users.noreply.github.com"
          # log-summary writes a logs/YYYY-MM-DD/<run_id>-summary.md file plus
          # may touch state/runs/<run_id>/ if it records its own done marker.
          git add "logs/" "state/runs/$RUN_ID/"
          if git diff --cached --quiet; then exit 0; fi
          git commit -m "$STAGE($RUN_ID)"
          pushed=0
          for i in 1 2 3 4 5; do
            if git pull --rebase --autostash origin "$GITHUB_REF_NAME" \
               && git push origin "HEAD:$GITHUB_REF_NAME"; then
              pushed=1; break
            fi
            sleep $((RANDOM % 5 + 1))
          done
          [ "$pushed" -eq 1 ] || { echo "push failed after 5 attempts"; exit 1; }
```

## reject-external-pr.yml

Quoting the spec §8 stub verbatim, with `permissions` tightened:

```yaml
name: Reject external PRs
on:
  pull_request_target:
    types: [opened]
jobs:
  close:
    if: github.event.pull_request.head.repo.full_name != github.repository
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
    steps:
      - uses: actions/github-script@<sha>
        with:
          script: |
            await github.rest.issues.createComment({
              ...context.issue,
              body: 'This repository does not accept external pull requests. Please fork and self-host if you want to modify it.'
            });
            await github.rest.pulls.update({
              owner: context.repo.owner,
              repo: context.repo.repo,
              pull_number: context.issue.number,
              state: 'closed'
            });
```

**Critical safety properties** (see `08-security.md` for full rationale):
- `pull_request_target` runs in the **base repo's context** with the base repo's secrets, but we never `actions/checkout` the PR branch, so attacker code cannot execute.
- Only the GitHub API is invoked; no script from the PR is ever read.
- `permissions` is minimal: only `pull-requests: write` (needed to close the PR).
- The `if:` guard ensures we only close PRs from forks; internal branches (which only the maintainer can create) are unaffected.

> DECISION: this is the **only** workflow that triggers on a PR-related event. All other workflows trigger on `push` (or `schedule`). Spec §8 explicitly recommended this posture.

> DECISION: drop `contents: read` from this workflow's `permissions:` block. *Why*: the workflow performs no `actions/checkout` and reads no repo content; it only calls the GitHub REST API. Granting `contents: read` is harmless but violates principle of least privilege and exceeds the spec §8 stub minimum (which lists only `pull-requests: write`). When omitted, the default `read-all` repo policy is overridden by this stricter per-job declaration: only `pull-requests: write` is granted; everything else is implicitly `none`.
