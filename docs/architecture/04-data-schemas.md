# 04 — Data schemas

Purpose: define every persisted file format used by the pipeline, with a JSON or Markdown example for each. These are the contracts. Any change here requires updating the corresponding stage scripts.

## Table of contents

- [Schema index](#schema-index)
- [`data/llm.yml`](#datallmyml)
- [`data/perspectives.yml`](#dataperspectivesyml)
- [`data/sources/<category>.yml`](#datasourcescategoryyml)
- [`state/runs/<run_id>/state.json`](#stateruns<run_id>statejson)
- [`state/runs/<run_id>/fetch/<category>.json`](#stateruns<run_id>fetch<category>json)
- [`state/runs/<run_id>/fetch.<category>.done.json`](#stateruns<run_id>fetch<category>donejson)
- [`state/runs/<run_id>/candidates_v1.json` / `_v2.json` / `_final.json`](#stateruns<run_id>candidates_v1json--_v2json--_finaljson)
- [`state/runs/<run_id>/summaries[/_v2/_final]/<id>.md`](#stateruns<run_id>summaries_v2_final<id>md)
- [`state/runs/<run_id>/<stage>.done.json`](#stateruns<run_id><stage>donejson)
- [`index.json`](#indexjson)
- [`digests/YYYY-MM-DD/<run_id>.md`](#digestsyyyy-mm-dd<run_id>md)
- [`README.md`](#readmemd)
- [`logs/YYYY-MM-DD/<run_id>-summary.md`](#logsyyyy-mm-dd<run_id>-summarymd)

## Schema index

| File | Format | Owner stage | Consumer stage(s) |
|---|---|---|---|
| `data/llm.yml` | YAML | hand-edited | filter, summarize |
| `data/perspectives.yml` | YAML | hand-edited | filter, summarize |
| `data/sources/<cat>.yml` | YAML | hand-edited | fetch (matching category) |
| `state/runs/<run_id>/state.json` | JSON | bootstrap | filter |
| `state/runs/<run_id>/fetch/<cat>.json` | JSON | fetch-<cat> | filter |
| `state/runs/<run_id>/fetch.<cat>.done.json` | JSON | fetch-<cat> | filter |
| `state/runs/<run_id>/candidates_*.json` | JSON | filter | filter (next round), summarize |
| `state/runs/<run_id>/summaries_final/<id>.md` | Markdown + YAML frontmatter | summarize | assemble |
| `state/runs/<run_id>/<stage>.done.json` | JSON | each stage | next stage, log-summary |
| `index.json` | JSON | assemble | downstream Module 2 |
| `digests/.../<run_id>.md` | Markdown | assemble | humans, Module 2 |
| `README.md` | Markdown | assemble | humans |
| `logs/.../<run_id>-summary.md` | Markdown | log-summary | humans |

## `data/llm.yml`

```yaml
# data/llm.yml — LLM runtime configuration. Swapping LLMs is a one-file change.
provider: glm
model: glm-5.1
# Z.AI's Anthropic-API-compatible endpoint for GLM.
# See https://docs.z.ai/devpack/tool/claude (verified 2026-05-02).
base_url: https://api.z.ai/api/anthropic
# Token flows in via env var LLM_API_KEY, sourced from the same-named GitHub
# Secret on the workflow step's env block. The Secret name is intentionally
# provider-agnostic so swapping providers is a one-file change to data/llm.yml
# (plus rotating the Secret value in GitHub Settings — not a file edit).
api_key_env: LLM_API_KEY
# bearer = Authorization: Bearer <token> (z.ai and other Anthropic-compatible
# endpoints). api_key = x-api-key header (native Anthropic).
# See 05-llm-and-agent-runtime.md DECISION block for why this field exists.
auth_mode: bearer
max_tokens: 4096
temperature: 0          # determinism
timeout_seconds: 120
# Retries for transient errors (rate limit, 5xx). Not for invalid-prompt errors.
retry:
  max_attempts: 3
  backoff_seconds: 5
```

> DECISION: `temperature: 0` by default. Spec was silent. *Why*: every percentage point of nondeterminism shows up as flaky verify-round disagreements and noisy rerun diffs. Reasoning quality at temp 0 is acceptable for filter/summarize.

## `data/perspectives.yml`

Closed enumeration of allowed `perspective_tags`. Adding a new tag requires editing this file *and* updating any downstream consumer that switches on tags.

```yaml
# data/perspectives.yml — closed enumeration. Order is documentation only.
tags:
  - developer       # IDE, devtools, frameworks, agent SDKs
  - ax              # AX consulting, enterprise transformation, change management
  - llm             # foundation model releases, capability changes, benchmarks
  - startup         # AI startup news, funding, product launches
  - security        # vulnerabilities, advisories, AI-security research
  - policy          # regulation, governance, public sector
  - research        # papers, novel techniques
  - infra           # GPUs, data center, cloud AI infra
  - product         # consumer-facing AI product launches
```

## `data/sources/<category>.yml`

Per-category source list. Adding a source = appending one entry. No code change.

Example, `data/sources/D-youtube.yml`:

```yaml
# YouTube channels. The youtube_rss adapter consumes each entry.
adapter: youtube_rss
sources:
  - handle: matthew_berman
    channel_id: UCawZsQWqfGSbCI5yjkdVkTA
  - handle: AIExplained-Official
    channel_id: UCNJ1Ymd5yFuUPtn21xtRbbw
  - handle: 1littlecoder
    channel_id: UC52X5wxOL_s5yw0dQk7NtgA
  - handle: DavidOndrej
    channel_id: UCH4kE1DUgaJ-q3vu6_3KQfg
  - handle: JulianGoldieSEO
    channel_id: <placeholder — maintainer fills in the real UC... id before deploy>
  - handle: WorldofAI
    channel_id: <placeholder — maintainer fills in the real UC... id before deploy>
```

The `<placeholder>` strings in the YAML above are not valid channel IDs and are shown here only because the canonical IDs were not at hand at doc time. The four other channels are real IDs and can be copied as-is.

Example, `data/sources/A-breaking.yml`:

```yaml
# Mixed adapters within one category.
sources:
  - adapter: hn_algolia
    query: "AI OR LLM OR Claude OR GPT OR Gemini OR agent"
    points_min: 50
  - adapter: rss
    name: TechCrunch AI
    url: https://techcrunch.com/category/artificial-intelligence/feed/
  - adapter: rss
    name: The Verge AI
    url: https://www.theverge.com/ai-artificial-intelligence/rss/index.xml
  - adapter: rss
    name: VentureBeat AI
    url: https://venturebeat.com/category/ai/feed/
```

> DECISION: each category file may use either a single top-level `adapter:` (uniform sources, e.g. all YouTube) or per-source `adapter:` keys (mixed). The adapter contract is in `06-source-adapters.md`. Spec was silent on the file shape.

## `state/runs/<run_id>/state.json`

Written by bootstrap. Read by filter (and assemble) for dedup memory + daily-cap accounting.

```json
{
  "run_at_utc": "2026-05-02T09:00:00Z",
  "run_id": "20260502T0900Z",
  "index_max": 348,
  "existing_items": [
    {
      "url": "https://www.anthropic.com/news/claude-4-7",
      "content_hash": "a1b2c3d4e5f6",
      "id": "anthropic-claude-4-7-1m-20260502",
      "first_seen_utc": "2026-05-02T09:00:00Z"
    },
    {
      "url": "https://techcrunch.com/2026/05/02/anthropic-claude-4-7/",
      "content_hash": "a1b2c3d4e5f6",
      "id": "anthropic-claude-4-7-1m-20260502",
      "first_seen_utc": "2026-05-02T09:00:00Z"
    },
    {
      "url": "https://openai.com/news/o5-preview",
      "content_hash": "9f8e7d6c5b4a",
      "id": "openai-o5-preview-20260502",
      "first_seen_utc": "2026-05-02T07:00:00Z"
    }
  ],
  "source_categories": ["A", "B", "C", "D", "E"]
}
```

> DECISION: `existing_items` is a flat array of `{url, content_hash, id, first_seen_utc}` records, **one record per source URL of every existing item** (so an item with two corroborating sources contributes two records sharing the same `id` and `first_seen_utc`). Spec was silent on the in-memory shape. *Why*: the dedup rules are URL-keyed and require URL→hash association on the same record (a flat-arrays shape with `existing_urls + existing_hashes` would lose that association). The same shape lets Stage 4 (assemble) recover prior `id` and `first_seen_utc` by URL lookup, which the spec mandates ("`first_seen_utc` does not change across updates"; see `01-pipeline-stages.md` Stage 4).

> NOTE: `state.json` size grows linearly with `index.json`. At 1000 items × 1.5 sources avg ≈ 225 KB per `state.json` file × hourly = ~5 MB/day of `state.json` content (deduplicated by Git's pack file format, but loose-object count grows). This is comfortably within GitHub repo size limits (1 GB recommended) but worth knowing. Future optimization paths if size is felt: (a) windowed `existing_items` — include only items whose `first_seen_utc` is within the last N=14 days, since older URLs are extremely unlikely to dedup-collide; (b) Bloom filter for the URL set, with the LLM stage absorbing the small false-positive rate. Both are deferred until measurement says they are needed. Cross-referenced from `03-repo-layout-and-state.md` "State retention".

### Filter input contract

The filter stage reads `state.json` and uses:
- `existing_items` for the deterministic dedup pre-pass (`scripts/lib/dedup.py`) — see `01-pipeline-stages.md` Stage 2 step 3.
- `run_at_utc` to determine the UTC day bucket for the per-day budget calculation.
- `index.json` (separately) to count items already published in the same UTC day. The **daily budget** is `max(0, 10 - day_so_far)` where `day_so_far` is the count of `index.json` entries with `first_seen_utc` in the same UTC day. Filter caps `candidates_v1` at this budget.
- **Security escape hatch** (spec §4): a critical security item (broad-impact vulnerability, mandatory patch) is allowed to push past the budget. The agent records `security_escape_hatch: true` on the candidate item; the maintainer audits via the `filter.done.json` summary and the digest review surface.
- **Soft alert ceiling on escape-hatch overflow**: a deterministic post-LLM check compares `final_count` against `ALERT_THRESHOLD = daily_cap + 3` (default `3`, defined once in `scripts/lib/dedup.py` and tunable). When `final_count > ALERT_THRESHOLD`, the filter does **not** drop items; it halts the run pending maintainer review, writes `escape_hatch_review_required.md`, sets `escape_hatch_review_required: true` in `filter.done.json`, and exits 0 without writing `candidates_final.json` (so downstream stages naturally wait). The maintainer reviews the file and re-triggers `filter.yml` via `workflow_dispatch`. See `01-pipeline-stages.md` Stage 2 step 5 and `08-security.md` Control 5.

### Daily-cap rationale

Per-run capping (10 items/run × 24 runs/day = 240 items/day) violates spec §4 "1–10 items per day total." The cap is editorial restraint at the day level; hourly cadence is a freshness mechanism, not a volume multiplier.

> NOTE: the soft alert ceiling (`daily_cap + 3`, see "Filter input contract" above) is an architectural interpretation of spec §4's "soft target … *true* emergency" wording. The spec mandates that a critical security item must be included even when the day is full; we therefore never deterministically drop an LLM-flagged escape-hatch item. Without any upper bound, however, an unbounded escape hatch is exploitable under prompt-injection (see `08-security.md` T3). Three escape-hatch items per UTC day is generous headroom for a real coordinated CVE day; beyond that, the run halts and the maintainer adjudicates.

## `state/runs/<run_id>/fetch/<category>.json`

Per-category normalized fetch output. Always a JSON array; may be empty.

```json
[
  {
    "url": "https://www.anthropic.com/news/claude-4-7",
    "title": "Claude 4.7 with 1M context window",
    "time": "2026-05-02T08:42:00Z",
    "source": "Anthropic Official",
    "category": "B",
    "raw_text": "Anthropic today announced Claude 4.7, expanding the context window to 1 million tokens for all paid tiers..."
  },
  {
    "url": "https://techcrunch.com/2026/05/02/openai-o5-preview-released/",
    "title": "OpenAI ships o5-preview to ChatGPT Plus",
    "time": "2026-05-02T07:30:00Z",
    "source": "TechCrunch AI",
    "category": "A",
    "raw_text": "OpenAI on Friday released a preview of its next reasoning model..."
  }
]
```

`raw_text` is capped at ~1500 characters per item (adapter responsibility). *Why*: keeps fetch outputs commit-friendly and limits LLM token cost downstream.

## `state/runs/<run_id>/fetch.<category>.done.json`

```json
{
  "stage": "fetch-A",
  "run_id": "20260502T0900Z",
  "category": "A",
  "completed_at_utc": "2026-05-02T09:02:13Z",
  "duration_seconds": 28,
  "sources_ok": [
    {"name": "Hacker News", "items": 3, "items_skipped": 0, "skip_reason": null},
    {"name": "TechCrunch AI", "items": 5, "items_skipped": 1, "skip_reason": "missing published_at"},
    {"name": "The Verge AI", "items": 2, "items_skipped": 0, "skip_reason": null}
  ],
  "sources_fail": [
    {"name": "VentureBeat AI", "error": "HTTP 503 after 3 retries"}
  ],
  "item_count": 10,
  "outputs": ["state/runs/20260502T0900Z/fetch/A.json"]
}
```

> The `items_skipped` and `skip_reason` fields on each `sources_ok` row let the maintainer distinguish "feed worked but had no new items in 24h" (`items: 0, items_skipped: 0`) from "feed worked but every item was malformed" (`items: 0, items_skipped: 7, skip_reason: "missing published_at"`). Without this, a silently-malformed feed records `items: 0` indistinguishable from a quiet day. Skip reasons in current adapters: `"missing published_at"` (Adapter Hard Rule 3), `"non-latin title"` (Hard Rule 4), `"older than 24h"` (recency filter).

## `state/runs/<run_id>/candidates_v1.json` / `_v2.json` / `_final.json`

```json
[
  {
    "id": "anthropic-claude-4-7-1m-20260502",
    "title": "Claude 4.7 with 1M context window",
    "primary_url": "https://www.anthropic.com/news/claude-4-7",
    "sources": [
      {"name": "Anthropic Official", "url": "https://www.anthropic.com/news/claude-4-7", "published_at": "2026-05-02T08:42:00Z"},
      {"name": "TechCrunch AI", "url": "https://techcrunch.com/2026/05/02/anthropic-claude-4-7/", "published_at": "2026-05-02T08:55:00Z"}
    ],
    "time": "2026-05-02T08:42:00Z",
    "raw_text": "Anthropic today announced ...",
    "content_hash": "a1b2c3d4e5f6",
    "reason_for_inclusion": "Vendor primary source; capability inflection (10x context); high credibility, high virality."
  }
]
```

`id` derivation: `<vendor-or-source-slug>-<title-slug>-<YYYYMMDD>`, ASCII lowercase, hyphenated, max 80 chars. Used as the filename for `summaries_final/<id>.md` and as a display key. **Note**: the `id` is *not* the join key for updates — vendors edit titles, which would change the slug. Assemble joins by URL first (`link` / `sources[*].url`) and falls back to `id`. See the "Update-matching rule" under `index.json` below for details.

## `state/runs/<run_id>/summaries[/_v2/_final]/<id>.md`

Markdown with YAML frontmatter. The frontmatter is what `assemble` consumes to write `index.json`; the body is what `assemble` interpolates into the digest.

> The `first_seen_utc` value in this frontmatter is **proposed**, not authoritative. Assemble looks up any prior entry by URL (`sources[*].url`) in `index.json`; if a prior entry exists, the proposed `first_seen_utc` is **discarded** and the prior entry's `first_seen_utc` (and `id`) are preserved. The proposed value is used only when no prior entry exists. Same rule for `id`: a vendor-edited title would change the slug-derived `id`, but the URL is stable, so the URL lookup wins. See `01-pipeline-stages.md` Stage 4.

```markdown
---
id: anthropic-claude-4-7-1m-20260502
title: "Claude 4.7 with 1M context window"
link: https://www.anthropic.com/news/claude-4-7
sources:
  - name: Anthropic Official
    url: https://www.anthropic.com/news/claude-4-7
    published_at: "2026-05-02T08:42:00Z"
  - name: TechCrunch AI
    url: https://techcrunch.com/2026/05/02/anthropic-claude-4-7/
    published_at: "2026-05-02T08:55:00Z"
keywords:
  - claude
  - long-context
  - model-release
  - anthropic
perspective_tags:
  - llm
  - developer
first_seen_utc: "2026-05-02T09:00:00Z"
content_hash: a1b2c3d4e5f6
---

## Key

Anthropic released Claude 4.7 with a 1 million token context window for all paid tiers. The expanded context is available immediately via the API and through Claude.ai for Pro and Team subscribers. No price change for existing tiers.

## Content

Anthropic announced Claude 4.7 on May 2, with the headline change being a 10x expansion of the context window from 100K to 1M tokens, matching the largest commercially available context windows. The expansion is enabled by an updated attention implementation Anthropic describes as "context-aware sparse attention," which the company claims preserves needle-in-haystack accuracy at full context length. Pricing for the new context tier is unchanged from the previous 100K offering on Pro and Team plans; API users pay the existing per-token rates with no surcharge for long contexts. The release follows OpenAI's o5-preview earlier in the week and Google's Gemini 3 announcement last month, putting Anthropic's flagship model back at parity on context length. Practical impact: workflows that previously required document chunking — full codebase review, long legal contracts, multi-document research — can now be done in a single call.
```

The `## Key` and `## Content` headings (capitalized) satisfy the `key` and `content` fields in requirements §2's table. The frontmatter field *names* (`key`, `content`, `keywords`, `perspective_tags`, etc.) remain lowercase — only the rendered Markdown headings are capitalized for visual consistency with `# Daily AI News`. Both sections are English. No `consultant perspective` section (Module 2's job).

## `state/runs/<run_id>/<stage>.done.json`

Generic shape; each stage adds its own metadata.

```json
{
  "stage": "filter",
  "run_id": "20260502T0900Z",
  "completed_at_utc": "2026-05-02T09:04:51Z",
  "duration_seconds": 45,
  "item_count": 4,
  "outputs": [
    "state/runs/20260502T0900Z/candidates_final.json"
  ],
  "skipped": false
}
```

If a stage short-circuits (zero items, missing upstream): `skipped: true` and a `reason` string.

The `filter.done.json` instance carries additional accounting fields beyond the generic shape: `item_count_in`, `item_count_after_dedup`, `day_so_far`, `remaining_budget`, `item_count_out`, `security_escape_hatch_used` (bool), and `escape_hatch_review_required` (bool — `true` only when the soft alert ceiling fired and the run is paused for maintainer review; in that case `candidates_final.json` is *not* written, downstream stages wait, and the maintainer re-triggers `filter.yml` via `workflow_dispatch` after curating the escape-hatch set). See `01-pipeline-stages.md` Stage 2.

## `index.json`

Master append-and-update index. The contract Module 2 reads.

```json
[
  {
    "index": 349,
    "id": "anthropic-claude-4-7-1m-20260502",
    "title": "Claude 4.7 with 1M context window",
    "link": "https://www.anthropic.com/news/claude-4-7",
    "sources": [
      {"name": "Anthropic Official", "url": "https://www.anthropic.com/news/claude-4-7", "published_at": "2026-05-02T08:42:00Z"},
      {"name": "TechCrunch AI", "url": "https://techcrunch.com/2026/05/02/anthropic-claude-4-7/", "published_at": "2026-05-02T08:55:00Z"}
    ],
    "keywords": ["claude", "long-context", "model-release", "anthropic"],
    "perspective_tags": ["llm", "developer"],
    "first_seen_utc": "2026-05-02T09:00:00Z",
    "content_hash": "a1b2c3d4e5f6",
    "digest_path": "digests/2026-05-02/20260502T0900Z.md",
    "key": "Anthropic released Claude 4.7 with a 1 million token context window for all paid tiers. The expanded context is available immediately via the API and through Claude.ai for Pro and Team subscribers. No price change for existing tiers.",
    "content": "Anthropic announced Claude 4.7 on May 2, with the headline change being a 10x expansion of the context window from 100K to 1M tokens... (full markdown body, multi-paragraph)"
  }
]
```

Sorted by `index` ascending. `index` is monotonic; gaps allowed if items get pruned (we do not currently prune).

> DECISION: `key` and `content` are **included in `index.json`** as full markdown strings (the same content as the corresponding digest section bodies). Spec §2 lists both as part of the per-item contract; downstream Module 2 consumers should be able to fetch the entire item set with one HTTP GET to `index.json` rather than one GET per item to `digests/.../<run_id>.md`. The size tradeoff: typical `key` is ~300 chars and `content` is ~1500 chars, so each item adds ~2 KB; 1000 items ≈ 2 MB `index.json`, still well within "fetch + parse cheaply." `digests/.../*.md` remains the human-readable rendering and is kept; the digest is for human eyeballs, `index.json` is for programmatic consumers.

### Update-matching rule (assemble)

When assemble updates an existing item: **match by `link` or any URL in `sources[*].url` first**, fall back to `id` only if no URL match is found. URL is the durable join key; `id` is a slug derived from `<vendor-slug>-<title-slug>-<YYYYMMDD>` and changes if the vendor edits the headline (which they do). On match:
- **Preserve** `id` and `first_seen_utc` from the existing entry.
- **Merge** `sources[]`: append any new sources from the current run, dedup by URL.
- **Re-prioritize**: if the merged `sources[]` would now have a different `sources[0]` per the priority rule (requirements §3), reorder so the highest-priority source is first; **update `link`** to equal the new `sources[0].url`. *Why*: spec §2 defines `link` as `sources[0].url`; the priority rule (vendor-official wins) implies `link` follows priority. `id` and `first_seen_utc` remain stable, so downstream consumers join on `id`, not on `link` (Module 2 instances that have cached the URL as a primary key would otherwise see a "different item" for the same `id`).
- **Update** `content_hash`, `keywords`, `perspective_tags`, `digest_path`, `key`, `content` from the current run.

### `id` is *not* the join key for updates

The `id` derivation is `<vendor-or-source-slug>-<title-slug>-<YYYYMMDD>`, ASCII lowercase, hyphenated, max 80 chars. It is the filename for `summaries_final/<id>.md` and a stable-enough display key. **It is not stable across vendor headline edits** (the title-slug changes if the title changes), so the spec's "stable across re-publications of the same news event" property is satisfied via the URL-keyed update path described above, not via the `id` itself. Consumers should still join on `id` for their own caches because once assemble has matched-and-preserved, the `id` is stable for the item's lifetime; the URL-first matching only happens inside assemble.

## `digests/YYYY-MM-DD/<run_id>.md`

```markdown
# Daily AI News — 2026-05-02 09:00 UTC

> Run: `20260502T0900Z` | New this run: 2 | Cumulative index: #348 to #349

---

## #348 OpenAI ships o5-preview to ChatGPT Plus

- **Sources**: OpenAI Official (primary, 2026-05-02 07:30 UTC), TechCrunch AI (2026-05-02 07:55 UTC)
- **Link**: https://openai.com/news/o5-preview
- **Keywords**: openai, o5, reasoning, chatgpt-plus
- **Perspective**: llm, product

### Key

OpenAI released o5-preview to ChatGPT Plus. The model targets agentic workflows and tool use; benchmarks are not yet published.

### Content

(longer English explanation, multi-paragraph)

---

## #349 Claude 4.7 with 1M context window

- **Sources**: Anthropic Official (primary, 2026-05-02 08:42 UTC), TechCrunch AI (2026-05-02 08:55 UTC)
- **Link**: https://www.anthropic.com/news/claude-4-7
- **Keywords**: claude, long-context, model-release, anthropic
- **Perspective**: llm, developer

### Key

(short essence)

### Content

(longer English explanation)

---
```

> DECISION: digest is **English-only**, no Korean section, no "consultant perspective" section. Spec §9 mandates this; the legacy template's `📌 핵심` / `💡 컨설턴트 관점` sections are explicitly out of scope here. Section header style uses plain English (`### Key`, `### Content`) instead of emoji-prefixed Korean labels. Frontmatter field names (`key`, `content`) stay lowercase — only the rendered headings are capitalized for visual consistency.

> NOTE on item ordering: the digest is **per-run** and lists its items in **`index` ascending order** (oldest-`index` first), as in the example above (#348 then #349). The README's "Last 7 days" section is **per-day** and lists items within each day group in **`first_seen_utc` descending order** (most recent first) — this matches the rule in `requirements/04-operations.md`. The two orderings diverge only when a single run produces multiple items: the digest preserves index-assignment order; the README cuts across runs and surfaces freshness.

## `README.md`

Auto-regenerated by `assemble` after each successful run. Last 7 days only.

```markdown
# ai-news-feed

Audience-agnostic English AI news dataset. Updated by GitHub Actions on a schedule. Module 2 (translation, audience-specific commentary, email distribution) is a separate downstream repo.

**Total cumulative**: 349 items | **Last updated**: 2026-05-02 09:00 UTC

## Last 7 days

### 2026-05-02
- "Claude 4.7 with 1M context window" — claude, long-context, model-release, anthropic ([digest](digests/2026-05-02/20260502T0900Z.md))
- "OpenAI ships o5-preview to ChatGPT Plus" — openai, o5, reasoning, chatgpt-plus ([digest](digests/2026-05-02/20260502T0900Z.md))

### 2026-05-01
- "GPT-5.5 cyber capabilities match offensive-research baseline" — gpt-5.5, cybersecurity, aisi, claude, penetration-testing ([digest](digests/2026-05-01/20260501T0900Z.md))
- "Apple surprised by Mac Mini demand: agent workloads exceed expectations" — apple, mac-mini, ai-workload, local-ai, developer-demand ([digest](digests/2026-05-01/20260501T0900Z.md))

### 2026-04-30
- (...)

(Older entries: browse the [digests/](digests/) folder.)

---

## How to add a source

Edit one file under `data/sources/`. External pull requests to this repository are auto-rejected by `reject-external-pr.yml`; fork the repository if you want to maintain your own source list.

## License

(TBD)
```

> DECISION: README shows **last 7 days** (spec §5). The legacy was 10 days; spec changed to 7. Items are listed once per day-group regardless of how many runs that day produced. **When the same item appears in multiple runs in one day** (e.g., first published at 09:00 UTC, then updated at 15:00 UTC), the README's `([digest](...))` link points to the **most recent** run's digest for that day. Spec was silent; most-recent is most useful (a reader following the link wants the freshest rendering). *Why one-link-per-day*: hourly runs would otherwise produce dozens of identical lines per day on the README.

## `logs/YYYY-MM-DD/<run_id>-summary.md`

Full template in `07-observability.md`. Brief example here:

```markdown
# Run summary — 20260502T0900Z

- **Started**: 2026-05-02 09:00:00 UTC
- **Finished**: 2026-05-02 09:06:42 UTC
- **Total duration**: 6m 42s
- **Final published count**: 2

## Per-source

| Category | Source | Status | Items | Skipped | Skip reason |
|---|---|---|---|---|---|
| A | Hacker News | ok | 3 | 0 | — |
| A | TechCrunch AI | ok | 5 | 1 | missing published_at |
| A | The Verge AI | ok | 2 | 0 | — |
| A | VentureBeat AI | fail (HTTP 503) | 0 | — | — |
| B | Anthropic Official | ok | 1 | 0 | — |
| ... | ... | ... | ... | ... | ... |

## Per-stage durations

| Stage | Duration | In | Out |
|---|---|---|---|
| bootstrap | 4s | — | 1 state.json |
| fetch (parallel) | 32s | 5 cats | 18 raw items |
| filter | 45s | 18 | 2 |
| summarize | 5m 11s | 2 | 2 |
| assemble | 6s | 2 | 1 digest, 2 index entries |

## Filter decisions

- DROP "RSS noise: yet another GPT wrapper launch" — below 4-axis bar (low credibility, low inflection).
- DROP "Anthropic press junket recap" — duplicate of #349 by URL match.
- KEEP "OpenAI o5-preview" — vendor primary source, inflection.
- KEEP "Claude 4.7 1M context" — vendor primary, inflection, virality.
```
