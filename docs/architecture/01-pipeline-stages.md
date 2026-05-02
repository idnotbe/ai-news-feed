# 01 — Pipeline stages

Purpose: define every stage of the pipeline as a unit with an input contract, an output contract, an idempotency rule, and a way to determine the current `run_id`. Implementers can read this top-to-bottom and know what each stage owes.

## Table of contents

- [Conventions](#conventions)
- [Stage 0: bootstrap](#stage-0-bootstrap)
- [Stage 1: fetch (parallel, one per category)](#stage-1-fetch-parallel-one-per-category)
- [Stage 2: filter + dedup](#stage-2-filter--dedup)
- [Stage 3: summarize](#stage-3-summarize)
- [Stage 4: assemble](#stage-4-assemble)
- [Stage 5: publish (commit + push)](#stage-5-publish-commit--push)
- [Stage 6: log-summary](#stage-6-log-summary)
- [Inter-stage chaining: how a stage knows the current run_id](#inter-stage-chaining-how-a-stage-knows-the-current-run_id)

## Conventions

- `run_id` format: `YYYYMMDDTHHMMZ` (UTC, e.g. `20260502T0900Z`). Generated once by `bootstrap` and propagated via files.
- Per-run state directory: `state/runs/<run_id>/`. All inter-stage handoffs live here.
- "done marker": `state/runs/<run_id>/<stage>.done.json` containing `{ "stage": "...", "completed_at_utc": "...", "outputs": [...] }`. A stage is considered complete iff this file exists and parses.
- Every stage's first step is **upstream verification**: assert all required upstream done markers exist for the run_id; if not, exit 0 with a `waiting for upstream` log line. (Why exit 0 not fail: a missing upstream is normal in a chained-by-push system; failing would spam the Actions UI red.)
- **Every stage's commit + push step** uses the bounded fetch+rebase+retry loop defined in `02-github-actions.md` "Common commit + push (with rebase-retry loop)". This is mandatory because parallel sibling workflows (notably the five `fetch-*.yml`) push to the same branch within seconds of each other; without the loop, only one push per fan-out lands and the chain stalls. See also `00-system-overview.md` design principle #5.

## Stage 0: bootstrap

**Workflow**: `.github/workflows/bootstrap.yml`
**Trigger**: `schedule` (cron, hourly) + `workflow_dispatch` (manual).
**Concurrency**: serial; cancel-in-progress disabled (each scheduled tick is its own run).

**Input**:
- None from the repo's per-run state. Reads `data/llm.yml` and `data/sources/*.yml` to know what the run universe is.
- Reads `index.json` to populate dedup memory: per-item `{url, content_hash, id, first_seen_utc}` records (one per `sources[].url` of every existing entry, so URL → prior-item lookup is a flat dict). The flat-arrays-only shape (`existing_urls` + `existing_hashes`) was rejected because the dedup rule "same URL + same hash → skip; same URL + different hash → update" requires the URL→hash association preserved on the same record.

**What it does**:
1. Generates `run_id = strftime("%Y%m%dT%H%MZ", utcnow())`.
2. `mkdir -p state/runs/<run_id>/{fetch,summaries}`.
3. Walks `index.json`, expanding each entry into one `existing_items` record per `sources[].url` with `{url, content_hash, id, first_seen_utc}`. Multiple URL records may share the same `id` and `first_seen_utc` — that is correct (an item with two corroborating sources has two URL keys but one identity).
4. Writes `state/runs/<run_id>/state.json` with `run_at_utc`, `run_id`, `index_max`, `existing_items` (the array described above), and `source_categories`. The latter is **derived at runtime** by scanning `data/sources/*.yml` and taking the filename prefix letter (`A-breaking.yml` → `"A"`, `B-vendor.yml` → `"B"`, …). The current source set yields `["A","B","C","D","E"]`. Adding a new category is a config edit (drop a new `F-<name>.yml` into `data/sources/`) plus a new fetch workflow file; bootstrap auto-detects the new entry on its next run with no edit to `bootstrap.py`.
5. Writes `state/runs/<run_id>/bootstrap.done.json` (using the generic `<stage>.done.json` shape defined in `04-data-schemas.md`, with `outputs: ["state/runs/<run_id>/state.json"]`).
6. Commits `state/runs/<run_id>/` with message `bootstrap(<run_id>)` and pushes.

**Output contract**:
- `state/runs/<run_id>/state.json` exists and is valid JSON matching the schema in `04-data-schemas.md`.
- `state/runs/<run_id>/bootstrap.done.json` exists.

**Idempotency**: identical run_id collisions are practically impossible (minute resolution, hourly cron). If `workflow_dispatch` is triggered manually within the same minute as a cron tick, the second invocation detects an existing `state.json` for that `run_id` and exits 0 with `bootstrap already complete for run_id`. *Why*: dual-triggered bootstrap would otherwise fork the chain.

**How it knows the run_id**: it generates it. This is the only stage that does so.

## Stage 1: fetch (parallel, one per category)

**Workflows** (one per category, all identical except for the `category` env): `.github/workflows/fetch-A-breaking.yml`, `fetch-B-vendor.yml`, `fetch-C-curated.yml`, `fetch-D-youtube.yml`, `fetch-E-security.yml`.
**Trigger**: `push` on paths `state/runs/*/bootstrap.done.json`.
**Concurrency**: keyed by `<category>-<run_id>`; cancel-in-progress true (a re-run supersedes).

**Input**:
- `state/runs/<run_id>/state.json` (for `existing_items` — used downstream by filter, but also available here in case a future adapter wants pre-fetch URL skipping).
- `data/sources/<category-config>.yml` (which sources to hit in this category).
- `data/llm.yml` (only consumed by adapters that use the LLM for HTML→record extraction; see `06-source-adapters.md`).

**What it does**:
1. **Verify upstream**: assert `state/runs/<run_id>/bootstrap.done.json` exists. If not, exit 0.
2. For each source entry in the category's config, invoke its adapter. Each adapter wrapped in try/except — a failing adapter logs and is skipped (per-source failure isolation, requirements §3.5).
3. Aggregate normalized records: `{url, title, time, source, raw_text, category}`.
4. Filter by recency: only items published within the last 24h UTC.
5. Write `state/runs/<run_id>/fetch/<category>.json` (a JSON array, possibly empty).
6. Write `state/runs/<run_id>/fetch.<category>.done.json` with per-source ok/fail counts.
7. Commit (`fetch(<category>, <run_id>)`) and push.

**Output contract**:
- `state/runs/<run_id>/fetch/<category>.json` exists and is a valid JSON array (possibly empty).
- `state/runs/<run_id>/fetch.<category>.done.json` exists with `sources_ok`, `sources_fail`, `item_count`.

**Idempotency**: re-running fetch for the same `(category, run_id)` overwrites the JSON files. Records are not deduped against the previous fetch result — the new file replaces the old. *Why*: avoids "ghost item" bugs where an item present in a previous fetch attempt but absent in this one would still appear downstream.

**How it knows the run_id**: extracted from the path of the file that triggered the workflow (`state/runs/<run_id>/bootstrap.done.json`). See "Inter-stage chaining" below.

## Stage 2: filter + dedup

**Workflow**: `.github/workflows/filter.yml`
**Trigger**: `push` on paths `state/runs/*/fetch.*.done.json`.
**Concurrency**: keyed by `filter-<run_id>`; cancel-in-progress true.

**Input**:
- All `state/runs/<run_id>/fetch/*.json`.
- `state/runs/<run_id>/state.json`.
- `data/perspectives.yml` (closed enumeration of perspective tags).
- `data/llm.yml`.

**What it does**:
1. **Verify upstream**: assert `fetch.<category>.done.json` exists for every category listed in `state.json`'s `source_categories`. If any missing, exit 0 with `waiting for fetch.<missing> upstream`.
2. **Compute today's remaining budget** (per-day, NOT per-run — spec §4 mandate):
   - Read `index.json`. Count items whose `first_seen_utc` falls within the **same UTC day** as `state.run_at_utc` — call this `day_so_far`.
   - `daily_cap = 10` (from spec §4 "1–10 items per day total").
   - `remaining_budget = max(0, daily_cap - day_so_far)`.
   - If `remaining_budget == 0`, the filter still runs (for security-escape-hatch detection, see step 4) but the regular cap on `candidates_v1` is 0.
3. **Deterministic dedup pre-pass** (Python, NOT LLM): `scripts/lib/dedup.py` reads `state.existing_items` (URL→{content_hash, id, first_seen_utc} mapping; see `04-data-schemas.md` `state.json`) and partitions the fetch output into:
   - **drop** — same URL, same `content_hash` → skip entirely (dedup case 1: "already published, unchanged").
   - **update** — same URL, different `content_hash` → mark with prior `id` and `first_seen_utc` for preservation; the LLM still gets to decide whether the update is editorially worth re-publishing (dedup case 2: "same URL, content changed").
   - **new** — URL not in `existing_items` → fully new candidate; the LLM judges on the 4-axis criteria (dedup case 3).
   The LLM **never sees dropped items**. *Why deterministic*: dedup rules are crisp and mechanical (URL exact match, hash exact match); LLM judgment for these introduces a forgetting failure mode that verify rounds catch only sometimes. Code never forgets.
4. **Draft (LLM)**: agent reads the post-dedup candidate set + perspectives, applies the 4-axis criteria from requirements §4, produces `state/runs/<run_id>/candidates_v1.json` capped at `remaining_budget` items (with `reason_for_inclusion` per item). **Security escape hatch**: per spec §4, "if a critical security item appears in the source pool, it must be included even if the day is otherwise full." The agent prompt explicitly enables this branch — when the agent identifies a true critical-security item (broad-impact vulnerability, mandatory patch — category E or otherwise), it is allowed to push past `remaining_budget`. The decision must be recorded in `reason_for_inclusion` with a `security_escape_hatch: true` flag in the item record so the maintainer can audit it.
5. **Soft alert ceiling on escape-hatch overflow** (deterministic, post-LLM): the requirements (§3 "Security highlight escape hatch") mandate that a true critical security item must be included even when the day is otherwise full. We therefore **never deterministically drop** an LLM-flagged escape-hatch item. We do, however, watch for an unusual escape-hatch volume as a possible prompt-injection signal: after step 4, `scripts/stages/filter.py` checks `final_count = day_so_far + len(candidates_v1)` against `ALERT_THRESHOLD = daily_cap + 3` (default; tunable via a config constant in `scripts/lib/dedup.py`). If `final_count > ALERT_THRESHOLD`, the run is **halted pending maintainer review**: the stage writes `state/runs/<run_id>/escape_hatch_review_required.md` listing every escape-hatch item with its `reason_for_inclusion` and source URL, sets `filter.done.json`'s `escape_hatch_review_required: true`, and exits 0 *without* writing `candidates_final.json`. Downstream stages (`summarize`, `assemble`) treat the missing `candidates_final.json` as the standard "filter not yet complete" upstream-missing signal and wait. The maintainer reviews `escape_hatch_review_required.md`, removes any items that look injected, and re-triggers `filter.yml` via `workflow_dispatch -f run_id=<id>` (the rerun proceeds normally with the curated set). *Why halt rather than drop*: the spec's must-include rule for critical security advisories is hard; silently dropping a verified advisory is a worse failure mode than delaying publication by a few minutes for human review. *Why a threshold rather than no limit*: an unbounded escape hatch is exploitable under prompt-injection (T3 in `08-security.md`) — a malicious source could craft items framed as "critical security" to bypass the cap. Three escape-hatch items per UTC day is generous headroom for a real coordinated CVE day; anything beyond that warrants explicit human approval before publication.
6. **Verify round 1**: a second agent invocation (separate context, `read/analyze only` prompt) compares `candidates_v1.json` against fetch outputs and writes `state/runs/<run_id>/verify1_filter.md` listing missed important items, wrongly included items, missed dedup, and any cap-overrun without a security flag.
7. **Fix**: agent applies verify1 feedback → `state/runs/<run_id>/candidates_v2.json`.
8. **Verify round 2**: separate agent, separate context → `state/runs/<run_id>/verify2_filter.md`.
9. **Final**: agent applies verify2 feedback → `state/runs/<run_id>/candidates_final.json`.
10. Write `state/runs/<run_id>/filter.done.json` with `item_count_in`, `item_count_after_dedup`, `day_so_far`, `remaining_budget`, `item_count_out`, `security_escape_hatch_used` (bool), `escape_hatch_review_required` (bool; `true` only when the soft alert ceiling fired and the run is paused for maintainer review per step 5).
11. Commit (`filter(<run_id>)`) and push.

**Why 2-round verify here (and in summarize)**: ported from the legacy routine §2 Phase 2 / Phase 4 because the filter and summarize stages are *judgment-heavy* and silent quality regressions are the failure mode users notice latest. Deterministic stages (bootstrap, assemble, publish, log-summary) skip verification per legacy routine §0 ("결정론적 phase는 검증 라운드 생략").

**Output contract**:
- `state/runs/<run_id>/filter.done.json` exists.
- `state/runs/<run_id>/candidates_final.json` exists *unless* `filter.done.json` carries `escape_hatch_review_required: true` (the maintainer-review halt path from step 5). In the review-halt case, `candidates_final.json` is intentionally not written; downstream stages must treat that as "filter not yet complete".

**Zero-item handling**: if `candidates_final.json` is `[]`, `filter.done.json` records `item_count: 0` and the workflow still commits these markers. Downstream stages (`summarize`, `assemble`) check this count and exit 0 without writing further state. The chain terminates cleanly without committing an empty digest. *Why*: requirements §4 forbids empty publications.

**Review-halt handling**: if `filter.done.json` carries `escape_hatch_review_required: true`, downstream stages (`summarize`, `assemble`) must exit 0 with a `waiting for maintainer review` log line — no further state is written. The maintainer reviews `escape_hatch_review_required.md`, edits the candidate set as needed, and re-triggers `filter.yml` via `workflow_dispatch -f run_id=<id>`. The rerun produces a normal `candidates_final.json` and clears `escape_hatch_review_required`, which then triggers the next stage normally (push event on the rewritten `filter.done.json`).

**Idempotency**: re-running filter for the same `run_id` overwrites all state files. Adopt-the-latest semantics.

**How it knows the run_id**: extracted from the path of the triggering `fetch.<category>.done.json`.

## Stage 3: summarize

**Workflow**: `.github/workflows/summarize.yml`
**Trigger**: `push` on path `state/runs/*/filter.done.json`.
**Concurrency**: keyed by `summarize-<run_id>`; cancel-in-progress true.

**Input**:
- `state/runs/<run_id>/candidates_final.json`.
- `data/perspectives.yml`.
- `data/llm.yml`.

**What it does**:
1. **Verify upstream**: assert `filter.done.json` exists for run_id. Read it.
   - If `escape_hatch_review_required == true`, exit 0 with `waiting for maintainer review` — write nothing, including no `summarize.done.json`. The next push triggered by the maintainer's re-run of `filter.yml` will retrigger summarize.
   - Else if `item_count == 0`, write `summarize.done.json` with `skipped: true, reason: "zero items"` and exit 0.
   - Else assert `candidates_final.json` exists and proceed to step 2.
2. **Draft (parallel per item)**: for each item in `candidates_final.json`, agent produces `state/runs/<run_id>/summaries/<item_id>.md` containing frontmatter (`id`, `title`, `link`, `sources`, `keywords`, `perspective_tags`, `first_seen_utc`, `content_hash`) + body sections `## key` (3–4 line essence) and `## content` (longer English explanation). All English. No Korean output. No "consultant perspective" section.
3. **Verify round 1**: separate agent, `read/analyze only` → `state/runs/<run_id>/verify1_summaries.md`.
4. **Fix**: per-item agents apply verify1 feedback → `state/runs/<run_id>/summaries_v2/<item_id>.md`.
5. **Verify round 2**: separate agent → `state/runs/<run_id>/verify2_summaries.md`.
6. **Final**: per-item agents apply verify2 feedback → `state/runs/<run_id>/summaries_final/<item_id>.md`.
7. Write `state/runs/<run_id>/summarize.done.json` with item counts.
8. Commit (`summarize(<run_id>)`) and push.

**Output contract**:
- For every item in `candidates_final.json`, `state/runs/<run_id>/summaries_final/<item_id>.md` exists with valid frontmatter.
- `state/runs/<run_id>/summarize.done.json` exists.

**Idempotency**: re-running overwrites all summary files for the run_id.

**How it knows the run_id**: from triggering `filter.done.json` path.

## Stage 4: assemble

**Workflow**: `.github/workflows/assemble.yml`
**Trigger**: `push` on path `state/runs/*/summarize.done.json`.
**Concurrency**: keyed by `assemble-<run_id>`; cancel-in-progress false (we want every run's assemble to complete; missed assemble = lost publication).

**Input**:
- `state/runs/<run_id>/summaries_final/*.md`.
- `state/runs/<run_id>/candidates_final.json` (for canonical metadata).
- `state/runs/<run_id>/summarize.done.json` (to detect zero-item skip).
- Existing `index.json` and `README.md` in repo root.

**What it does**:
1. **Verify upstream**: assert `summarize.done.json` exists. If `skipped: true`, write `assemble.done.json` with `skipped: true` and exit 0.
2. **Build digest**: write `digests/YYYY-MM-DD/<run_id>.md` (template in `04-data-schemas.md`). Use atomic write: render to `digests/YYYY-MM-DD/<run_id>.md.tmp`, then `os.replace(..., '<run_id>.md')`.
3. **Lookup prior entry per item, preserve `first_seen_utc` and `id`**: for each item in `summaries_final/<id>.md`:
   - Build the lookup key set: every URL in `sources[*].url` plus the item's own `id` (the `id` is a fallback only; URL is the durable join key, see `04-data-schemas.md` "Update-matching rule").
   - Scan `index.json` for any existing entry whose `link` or `sources[*].url` intersects this set. **URL is the primary join key**; `id` is the fallback (vendors can edit titles, which changes the slug-based id, but URL is stable).
   - If an existing entry is found:
     - **Preserve** `id` and `first_seen_utc` from the existing entry. The summarize stage's frontmatter `first_seen_utc` is *proposed*, used only when there is no prior entry (this contract is documented in `04-data-schemas.md` summarize-frontmatter section).
     - **Append** any new `sources[]` entries from the current run. If the new run introduces a higher-priority source (per the priority rule in requirements §3), reorder so `sources[0]` is the highest priority — and update `link` to equal the new `sources[0].url`. `id` and `first_seen_utc` still do not change.
     - Update `content_hash`, `keywords`, `perspective_tags`, `digest_path` from the current run.
   - If no existing entry: this is a new item; assign the next `index` (= `index_max + 1`); use the summarize-proposed `first_seen_utc` as the canonical value going forward.
4. **Update `index.json`**: write the new/updated entries. Sort by `index` ascending. Atomic: write `index.json.tmp` then `os.replace`.
5. **Update `README.md`**: regenerate the "Last 7 days" section from `index.json`. Atomic write.
6. **Write `state/runs/<run_id>/assemble.done.json` LAST**, only after all of digest + index + README are durable on disk. *Why last*: if a crash happens between writing the digest and `index.json`, a missing `assemble.done.json` is the unambiguous "rerun assemble for this run_id" signal. With the marker written first, a partial state would falsely appear complete.
7. Commit (single commit: `assemble(<run_id>): N items`) and push.

**Output contract**:
- `digests/YYYY-MM-DD/<run_id>.md` exists.
- `index.json` updated with new items, `id` and `first_seen_utc` preserved on update.
- `README.md` regenerated.
- `state/runs/<run_id>/assemble.done.json` exists.

**Zero-item handling**: if no per-item summaries exist (because filter produced 0 items and summarize short-circuited), `assemble.done.json` is written with `skipped: true` and **no digest is written, no `index.json` mutation, no README change**. State markers and the run summary log are still committed for traceability (see Stage 6). *Why*: spec §4 forbids empty publications; a state-only commit is not a publication.

**Idempotency**: rerun is safe. Steps 2–5 are pure functions of the per-run state files plus the existing `index.json`. Items are matched by **URL first, then `id`**; reruns update in place. *Why*: lets you fix a bad summary by re-running summarize then assemble without producing duplicate digest entries.

**Recovery procedure (mid-flight crash)**: if `assemble` crashes between steps 2 and 6 (digest written but `assemble.done.json` absent, or `index.json` partially updated), the working tree may carry a partial publication commit. To recover, the operator runs `gh workflow run assemble.yml -f run_id=<id>`. The rerun's URL-first match in step 3 finds the items the partial run wrote into `index.json` and **updates them in place** rather than re-appending — no duplicate `index.json` entries appear. The preserved `first_seen_utc` from step 3 is the same value the partial run had proposed (since no prior entry existed at the time, the proposed value won), so re-running does not regress `first_seen_utc`. If a digest file from the partial run is corrupt or wrong, the maintainer reverts that one digest commit before re-running. Cross-referenced as recovery scenario #3 in `08-security.md` "Incident response".

**How it knows the run_id**: from triggering `summarize.done.json` path.

## Stage 5: publish (commit + push)

> DECISION: publish is **not a separate workflow**; it is the final step of `assemble.yml`. Spec listed publish as its own stage but separating it would require yet another `push`-triggered workflow that does only `git push` of files already committed. Combining keeps the chain shorter and means the "I shipped a digest" event corresponds to a single commit, easier to read in Git history. Spec was silent on whether publish must be its own workflow.

The commit step uses the default `GITHUB_TOKEN` and signs nothing extra. The repo's branch protection (if any) allows the bot to push to default branch — see `08-security.md`.

## Stage 6: log-summary

**Workflow**: `.github/workflows/log-summary.yml`
**Trigger**: `push` on path `state/runs/*/assemble.done.json`.
**Concurrency**: keyed by `log-<run_id>`; cancel-in-progress false.

**Input**:
- All `state/runs/<run_id>/*.done.json` (for stage durations and counts).
- `state/runs/<run_id>/fetch.*.done.json` (for per-source ok/fail).
- `state/runs/<run_id>/verify*.md` (for filter decisions).

**What it does**:
1. Aggregates the above into a single human-readable Markdown file.
2. Writes `logs/YYYY-MM-DD/<run_id>-summary.md` (template in `07-observability.md`).
3. Commits (`log(<run_id>)`) and pushes.
4. Optionally: prunes `state/runs/<run_id>/` if older than N days (separate concern; see `03-repo-layout-and-state.md`).

**Output contract**:
- `logs/YYYY-MM-DD/<run_id>-summary.md` exists, valid Markdown.

**Zero-item runs**: a zero-item run still gets a log file. The log records the empty publication outcome. Spec §4 ("on a zero-item day, the pipeline must not commit/push") refers to the **published dataset** (no empty digest, no `index.json` mutation, no README change). Operational state markers and the summary log are still committed for traceability — without them, the maintainer cannot distinguish "quiet day" from "pipeline broken."

**Idempotency**: rerun overwrites the log file.

**How it knows the run_id**: from triggering `assemble.done.json` path.

## Inter-stage chaining: how a stage knows the current run_id

A `push` event payload contains an array `.commits[]` with up to 20 commit objects (one per commit included in the push). It also carries `.head_commit`, which is **only the most recent commit** in the push. **Reading only `.head_commit` silently drops up to 19 other commits**, including the one that almost always carries the actual `state/runs/<run_id>/...done.json` we care about (e.g. when a maintainer pushes a manual fix on top of an automated commit, or when several state files land in one push because the upstream stage retried).

Every chained workflow therefore (a) accepts an explicit `run_id` via `workflow_dispatch` for manual retries, and (b) when triggered by `push`, scans the **union** of `.commits[].added[]` and `.commits[].modified[]` across all commits in the push.

```yaml
on:
  push:
    paths:
      - 'state/runs/*/<upstream>.done.json'
  workflow_dispatch:
    inputs:
      run_id:
        description: "Run ID to operate on (e.g. 20260502T0900Z). If omitted, parsed from the push event."
        required: false
        type: string

# ... permissions, concurrency placeholder (see below) ...

jobs:
  chained:
    runs-on: ubuntu-latest
    outputs:
      run_id: ${{ steps.resolve.outputs.run_id }}
    steps:
      - uses: actions/checkout@<sha>
        with: { fetch-depth: 1, persist-credentials: true }

      - name: Resolve run_id
        id: resolve
        env:
          INPUT_RUN_ID: ${{ inputs.run_id }}
        run: |
          set -euo pipefail
          # 1. Prefer the workflow_dispatch input when present (manual retry path).
          if [ -n "${INPUT_RUN_ID:-}" ]; then
            RUN_ID="$INPUT_RUN_ID"
          elif [ "${GITHUB_EVENT_NAME}" = "push" ]; then
            # 2. Scan the UNION of added+modified across ALL commits in the push,
            #    not just .head_commit. .commits[] has up to 20 entries.
            RUN_ID=$(jq -r '
              [ .commits[]?.added[]?,
                .commits[]?.modified[]? ] | .[]
            ' "$GITHUB_EVENT_PATH" \
              | grep -E '^state/runs/[^/]+/[^/]+\.done\.json$' \
              | awk -F/ '{print $3}' \
              | sort -u \
              | head -n1)
          else
            RUN_ID=""
          fi
          if [ -z "$RUN_ID" ]; then
            echo "No run_id found (no input, no matching paths in push). Nothing to do."
            exit 0
          fi
          echo "run_id=$RUN_ID" >> "$GITHUB_OUTPUT"

      - name: Verify upstream done markers
        run: |
          set -euo pipefail
          RUN_ID="${{ steps.resolve.outputs.run_id }}"
          REQUIRED=( "bootstrap.done.json" )   # adjust per stage
          for f in "${REQUIRED[@]}"; do
            if [ ! -f "state/runs/$RUN_ID/$f" ]; then
              echo "Waiting for upstream: state/runs/$RUN_ID/$f"
              exit 0
            fi
          done
```

**Why scan all commits, not just `head_commit`**: a single push can carry multiple commits when (a) a stage retries and replays its commit on top of an existing one, or (b) the maintainer manually pushes a fix on top of an automated commit. `head_commit.modified` only describes the latest commit's diff; the upstream done marker may be in an earlier commit of the same push.

**Why `sort -u | head -n1`**: if multiple distinct `run_id`s land in one push (rare but possible if the maintainer pushes batched fixes), we pick the lexicographically-first and ignore the rest. The dropped run_ids are recoverable via manual `gh workflow run filter.yml -f run_id=<id>` — see the `workflow_dispatch` input above.

**Manual-retry fallback**: when a chained workflow fails (transient LLM 5xx, runner outage), the maintainer re-triggers it via the GitHub UI or `gh workflow run <stage>.yml -f run_id=<id>`. The `inputs.run_id` path is the supported recovery surface; without it, the user would have to fabricate a fake push to re-run the stage.

> DECISION: derive run_id from the union of all commits in the push payload, with a `workflow_dispatch` input as the manual-retry override. Spec was silent on the exact mechanism. *Why*: parsing all commits is robust against the multi-commit-push failure mode (a single push of N>1 commits would otherwise drop N−1 upstream done markers if only `.head_commit` were inspected); the input is the recovery surface for any case the parser misses.

> DECISION: `workflow_run` vs `push` for chaining — use `push`. Spec listed both as acceptable. *Why*: `push` events fire even if the upstream workflow's run is filtered or restarted oddly. The thing we actually care about is "did the file land in the default branch", which is exactly what `push` signals.

### Concurrency keys

Every chained workflow's `concurrency:` group must key on the **resolved run_id**, not on `head_commit.message`. Using the commit message is wrong because (a) for filter, the triggering commit is `fetch-X(<run_id>)` where X varies by category, so the five filter invocations get five different group strings and run in parallel — doing the work 5×; (b) message-based keys are brittle if a maintainer ever edits a commit message.

The chosen pattern depends on `cancel-in-progress`:

1. **Two-job pattern** — **the** pattern for every workflow where `cancel-in-progress: true`. This applies to `fetch-A..E`, `filter`, and `summarize`. Job 1 (`resolve`) has **no concurrency gate**; it does only `actions/checkout` + the `Resolve run_id` step (cheap, ~5 sec). It exposes `outputs.run_id` via `$GITHUB_OUTPUT`. Job 2 (`work`) declares `needs: resolve` and uses `concurrency: { group: <stage>-${{ github.ref }}-${{ needs.resolve.outputs.run_id }}, cancel-in-progress: true }` — the group key is the **actual** resolved run_id, not a `'pending'` placeholder, so two pushes carrying different run_ids never collide. *Why mandatory here*: a single-job workflow gated on `${{ inputs.run_id || 'pending' }}` collapses all not-yet-resolved invocations into a single group keyed on `'pending'`. With `cancel-in-progress: true`, a still-resolving prior-hour invocation would be **silently cancelled** by a fresh hourly bootstrap (the cancel happens before the prior invocation can update the group key). The two-job split moves run_id resolution outside the concurrency gate, eliminating the collision.

2. **Single-job with input fallback** — used for `bootstrap`, `assemble`, `log-summary`, and `prune-state`, all of which use `cancel-in-progress: false`. `concurrency: { group: ${{ github.workflow }}-${{ github.ref }}-${{ inputs.run_id || 'pending' }}, cancel-in-progress: true }`-style fallback is safe here because queueing replaces cancelling: a colliding-on-`'pending'` invocation waits its turn rather than killing the prior one. No data loss, just serialization.

See `02-github-actions.md` for the per-workflow concrete bindings.
