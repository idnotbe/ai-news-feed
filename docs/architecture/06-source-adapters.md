# 06 — Source adapters

Purpose: define the source-extension mechanism. Adding a YouTube channel or new RSS feed must be a config-only change; this doc explains the contract that makes that possible.

## Table of contents

- [Adapter contract](#adapter-contract)
- [Built-in adapters](#built-in-adapters)
- [Adding a new source by category (config-only)](#adding-a-new-source-by-category-config-only)
- [Adding a new adapter (code change)](#adding-a-new-adapter-code-change)
- [Failure isolation](#failure-isolation)
- [Adapter test fixtures](#adapter-test-fixtures)

## Adapter contract

An adapter is a Python class in `scripts/adapters/` that takes a single source-config entry and returns a list of normalized records.

```python
# scripts/adapters/base.py
from dataclasses import dataclass
from typing import Iterable

@dataclass
class FetchedRecord:
    url: str            # canonical URL of the news item
    title: str          # English title (raw, normalization happens later)
    time: str           # ISO 8601 UTC, e.g. "2026-05-02T08:42:00Z"
    source: str         # human-readable source name, e.g. "Anthropic Official"
    raw_text: str       # excerpt or summary, max ~1500 chars

class Adapter:
    """Subclass for each source type. Pure: do not write to disk; do not mutate inputs."""

    name: str = "base"

    def __init__(self, entry: dict):
        """`entry` is one item from the YAML `sources:` list."""
        self.entry = entry

    def fetch(self, *, since_utc: str, max_items: int = 50) -> Iterable[FetchedRecord]:
        """Yield FetchedRecord objects published at or after `since_utc`. Must not raise on
        empty result; must raise (with a useful message) on auth/network failure so the
        category fetch script can log and continue with the next source."""
        raise NotImplementedError
```

Hard rules an adapter must follow:

1. **Pure**: no disk writes, no commits to repo state, no side effects beyond outbound HTTP.
2. **Bounded**: cap `raw_text` at ~1500 characters and the returned list at `max_items`.
3. **UTC times only**: parse and normalize. If the source omits a publish time, **skip the item** (conservative; matches legacy routine §2 Phase 1).
4. **English-only filter**: if a record's `title` looks predominantly non-Latin script, skip it. (Coarse heuristic; true language detection lives in the filter stage.) *Why*: catches accidental ingestion via mixed-feed sources.
5. **Failure-loud, isolated**: raise on transport errors. The wrapping fetch stage catches and continues — adapters do not need their own try/except for rate limits or 5xx.

The fetch stage script wraps each adapter call. Adapters that drop items (Hard Rules 3 and 4: missing `published_at`, non-Latin title) report the count and reason via `Adapter.last_skipped` so the wrapping script can surface it in `fetch.<cat>.done.json` (so the maintainer can distinguish a quiet feed from a silently-malformed feed; see `04-data-schemas.md` `fetch.<category>.done.json`).

```python
# scripts/stages/fetch.py (sketch)
results: list[FetchedRecord] = []
sources_ok, sources_fail = [], []
for entry in load_category_config(category)["sources"]:
    adapter_name = entry.get("adapter", category_default_adapter)
    Adapter = ADAPTER_REGISTRY[adapter_name]
    inst = Adapter(entry)
    try:
        items = list(inst.fetch(since_utc=since_utc))
        results.extend(items)
        sources_ok.append({
            "name": entry.get("name") or entry.get("handle"),
            "items": len(items),
            "items_skipped": getattr(inst, "skipped_count", 0),
            "skip_reason": getattr(inst, "skip_reason", None),
        })
    except Exception as e:
        sources_fail.append({"name": entry.get("name") or entry.get("handle"), "error": str(e)[:200]})
        continue
```

The base `Adapter` class exposes `self.skipped_count: int` and `self.skip_reason: str | None`, both reset per `fetch()` call. Subclasses increment `skipped_count` and set `skip_reason` whenever they drop a per-item record under Hard Rules 3 or 4. *Why on the adapter not a counter elsewhere*: keeps the counting code next to the dropping code; impossible to miss when adding a new adapter.

`ADAPTER_REGISTRY` is a dict populated by the adapter modules' `__init_subclass__` hooks (or hand-registered in `scripts/adapters/__init__.py`).

## Built-in adapters

| Adapter | Source types it handles | Notes |
|---|---|---|
| `rss` | Generic RSS / Atom feeds | Covers TechCrunch AI, The Verge AI, VentureBeat AI, Anthropic, OpenAI, DeepMind, Microsoft AI, Meta AI, smol.ai, TLDR AI, The Rundown AI, Ben's Bites — basically most of the world. |
| `youtube_rss` | YouTube channel RSS | Special-cased only because YouTube's `feeds/videos.xml?channel_id=` URL needs assembling from a channel ID. Output `raw_text` is `description` truncated; transcripts are out of scope (per spec §3, "channel feeds, not transcripts"). |
| `hn_algolia` | Hacker News via Algolia search API | Filters by query string and minimum points. *Why API not RSS*: Algolia exposes filtering by score and date, which the HN RSS does not. |
| `html_blog` | Vendor blogs without RSS | Plain HTML parsing. *Why separate*: occasionally the LLM is invoked here for unstructured pages. The only adapter that may consume `data/llm.yml`. |
| `cisa_kev` | CISA Known Exploited Vulnerabilities feed | The single high-bar security source per spec §3.E. |

> DECISION: **no Twitter/X adapter**. Spec §3 explicitly puts X out of scope. The legacy routine had category F (X), removed in Module 1.

> DECISION: **no Korean-language adapters**. Spec §3 explicitly removes Korean sources. The legacy `temp/fetch/D_korean.json` category is gone.

## Adding a new source by category (config-only)

### Add a YouTube channel

Edit `data/sources/D-youtube.yml`. Append:

```yaml
sources:
  - handle: matthew_berman
    channel_id: UCawZsQWqfGSbCI5yjkdVkTA
  # ... existing entries ...
  - handle: NewChannelHere               # <-- new
    channel_id: UCxxxxxxxxxxxxxxx
```

Commit, push. The next bootstrap-triggered fetch run picks it up automatically.

### Add an RSS feed

Edit the appropriate category file (`A-breaking.yml`, `B-vendor.yml`, etc.). Append:

```yaml
sources:
  # ... existing entries ...
  - adapter: rss
    name: New AI Blog
    url: https://newai.example.com/feed.xml
```

### Add a Hacker News query variant

Edit `A-breaking.yml`:

```yaml
sources:
  - adapter: hn_algolia
    query: "AI OR LLM OR Claude OR GPT OR Gemini OR agent"
    points_min: 50
  - adapter: hn_algolia                  # <-- second HN query, lower bar, niche topic
    query: "MCP OR \"Model Context Protocol\""
    points_min: 20
```

### Remove a source

Delete its block. Or comment it out — the YAML parser tolerates that and Git history preserves it for re-enabling.

### Add a new source category

To introduce a category beyond A–E (e.g. `F-podcasts`):

1. Create `data/sources/F-podcasts.yml` with the per-source entries (and an `adapter:` if needed).
2. Create `.github/workflows/fetch-F-podcasts.yml` by copying `fetch-A-breaking.yml` and changing `name`, `CATEGORY`, and `STAGE`.

That is the entire change. Bootstrap **auto-detects** the new category by scanning `data/sources/*.yml` and taking the filename prefix letter (`F-podcasts.yml` → `"F"`); the new prefix lands in `state.json`'s `source_categories` array, and `filter` waits for `fetch.F.done.json` along with the others. No edit to `bootstrap.py` or `filter.py` is needed.

## Adding a new adapter (code change)

Required when a source type is fundamentally different from the existing five (e.g., a Discord channel, a podcast feed needing transcript extraction). Steps:

1. Create `scripts/adapters/<name>.py` subclassing `Adapter`.
2. Register in `scripts/adapters/__init__.py` (`ADAPTER_REGISTRY["<name>"] = MyAdapter`).
3. Add a unit test in `tests/adapters/test_<name>.py` with a recorded fixture (vcrpy or static file) so CI can verify the parsing without hitting the network.
4. Document the YAML entry shape in this file under "Built-in adapters".
5. Edit a `data/sources/<category>.yml` to actually use it.

The maintainer is the only person doing this (no external PRs).

## Failure isolation

Layers of isolation:

1. **Per-adapter** — try/except around each adapter call inside the fetch stage script. A failing adapter is recorded in `sources_fail` and the stage continues.
2. **Per-category** — each category is its own workflow. A category-A workflow crash does not affect categories B–E.
3. **Per-stage** — fetch failures of categories A–E independently feed into filter; filter waits for *all* expected `fetch.<cat>.done.json` markers before proceeding (see `01-pipeline-stages.md`). If a category's workflow crashes hard and never writes its done marker, filter waits indefinitely (until the next bootstrap run starts a new run_id).
4. **Run-level** — a totally empty source pool produces a zero-item filter result, which short-circuits summarize and assemble. `log-summary` still writes its log so the situation is visible.

> DECISION: if a fetch category's workflow never writes its done marker (e.g., GH Actions outage), the filter for that run waits forever. We do **not** add a timeout that proceeds with partial fetch input. *Why*: a partial filter result might publish "today's news" missing a critical security advisory from category E. Better to publish nothing this hour and let the next hourly bootstrap try again. This is consistent with requirements §4 ("zero items is acceptable; ... pipeline must not commit/push").

## Adapter test fixtures

To keep CI hermetic and avoid flaky tests when external sources go down:

- Each adapter has at least one recorded fixture in `tests/fixtures/adapters/<name>/`.
- Tests use `responses` or `vcrpy` to mock the HTTP call.
- A separate, **out-of-CI**, manually triggered job (`scripts/dev/check_adapters_live.py`) hits the real sources and reports drift. Not part of the publish pipeline; the maintainer runs it when a source seems silently broken.

> DECISION: live source checks are **not** in CI. Spec was silent. *Why*: making CI depend on third-party uptime would create false-red signals on the publishing pipeline, exactly the noise we want to keep out of GH Actions UI.
