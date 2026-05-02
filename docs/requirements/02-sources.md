# 02 — Sources

Purpose: define which sources Module 1 ingests, how new sources are added, and how multiple sources covering the same event are combined into a single item.

## Language rule: English-only

Module 1 ingests **English-language sources only**. Korean-language outlets (e.g., AI타임스, 디지털데일리) and other non-English outlets are excluded.

Why: Module 1's output contract is English-only ([01-functional.md](01-functional.md)). Ingesting non-English sources would force Module 1 to translate at the input boundary, which belongs to Module 2 (or to a separately scoped translation step), not to the upstream canonical dataset.

## Source categories

The initial source pool is grouped into five categories. The category boundaries are editorial; they exist to make the source list reviewable, not to drive different code paths.

- **A. Breaking English news.** General-tech outlets that surface AI news quickly. Examples: Hacker News (AI-relevant items), TechCrunch AI, The Verge AI, VentureBeat AI.
- **B. Vendor official blogs.** First-party announcements from frontier-model and platform vendors. Examples: Anthropic News, OpenAI News, Google DeepMind Blog, Microsoft AI Blog, Meta AI Blog.
- **C. Curated dailies.** Human-curated AI newsletters whose own selection is itself a useful signal. Examples: smol.ai (Swyx), TLDR AI, The Rundown AI, Ben's Bites.
- **D. YouTube channel feeds (RSS).** Channel-level RSS feeds (titles and metadata only — transcripts are not in scope here). Example channels: @matthew_berman, @AIExplained-Official, @1littlecoder, @DavidOndrej, @JulianGoldieSEO, @WorldofAI.
- **E. Security advisories with broad blast radius.** Critical vulnerability and patch notices for widely-deployed software whose AX impact is real. Examples: CISA KEV entries, vendor security advisories. **Inclusion bar is very high** — only items that broadly affect AX consultants or their clients qualify.

Out of scope (explicitly removed from the legacy routine):

- Korean-language news (incompatible with English-only output).
- X/Twitter (was never implemented in the legacy routine and is not in scope here).

## Adding a new source is a config-only change

Adding a new source — a new YouTube channel, a new RSS feed, a new vendor blog — must be possible by editing a configuration file alone. No code change, no doc change beyond the source list, no schema change.

Why: the source list will churn (channels go quiet, new newsletters launch, vendors split blogs). Forcing code changes for every source change would make the maintenance cost of breadth too high. This requirement is a hard constraint on the architecture; the architecture set must satisfy it.

> NOTE: Ambiguity — the spec proposes per-category YAML files under `data/sources/` but the requirements doc must stay implementation-agnostic. Decision: state only the *behavioral* requirement (config-only addition) here; the file layout is an architecture concern.

## Multi-source per item

Multiple sources covering the same news event must be merged into a single published item.

- The merged item has `sources[]` containing every source URL that covers the event, each with its own `name`, `url`, and `published_at`.
- The **primary** source is `sources[0]`. Selection priority for the primary, highest first:
  1. Vendor official blog (category B)
  2. English breaking news (category A)
  3. Curated daily (category C)
  4. YouTube channel feed (category D)
  - Category E (security) is treated as primary whenever the item is a security item, regardless of where else it appears.
- The item's top-level `link` field equals `sources[0].url`.

Why prioritize vendor official as primary: it is the most authoritative form of the news for almost every event class; secondary outlets typically derive from it.

Items that are URL-different but topically the same are *only* merged when the source-merging logic confirms they describe the same event. Same topic but different events (e.g., two different vendors releasing two different models on the same day) remain separate items. See [03-curation.md](03-curation.md) for the exact dedup rules.

### Stability of `link` and `sources[0]` across updates

When an item is updated and a higher-priority source appears that did not exist at first publication (e.g., day 1 only TechCrunch covered the event; day 2 the vendor official blog post is published, which outranks TechCrunch per the priority above):

- `sources[0]` is updated to the newly-arrived higher-priority source.
- The top-level `link` field is updated to match the new `sources[0].url` (the equality constraint in [01-functional.md](01-functional.md) is preserved).
- `id` and `first_seen_utc` **are preserved** — they are the durable join keys for downstream consumers across updates.

Implication for downstream consumers: **join on `id`, not on `link`**. A consumer that caches items keyed by URL will see what looks like a "different item" for the same `id` after a primary-source promotion. Joining on `id` avoids this.

## Per-source failure isolation

A failing source must not block the run.

- A YouTube channel returning HTTP 5xx, an RSS feed timing out, or a vendor blog being unreachable must be logged and skipped.
- The pipeline continues with whatever sources succeeded.
- The published output for the day contains items derived only from successful sources; the failure is recorded in the run summary log (see [04-operations.md](04-operations.md)).

Why: with a long-tail source list (especially YouTube channels), a single flaky source would otherwise produce a zero-item day. The cost of skipping one source is far lower than the cost of an empty digest.

### Distinguish "no new items" from "items dropped due to missing publish time"

Source adapters skip individual items that lack a usable publish timestamp (this is required for ordering and dedup). A feed in which **every** item lacks a publish time will therefore yield zero items even though it is reachable and well-formed in every other respect.

The failure-isolation contract requires that this case be **observable**:

- Per-source run state must distinguish three outcomes: (a) source reachable with new items, (b) source reachable with no new items in the lookback window, (c) source reachable but all candidate items skipped due to missing `published_at` (or other adapter-level filter).
- Outcome (c) must surface in the run summary log (see [04-operations.md](04-operations.md)) — counts of skipped items per source, with a short reason — so a silently-malformed feed does not look healthy.

Why: a feed that has flipped to a malformed shape upstream (e.g., a YouTube edge case where `<published>` is missing) would otherwise look identical in the log to a quiet-news day on that source. The maintainer needs to be able to tell the two apart to know whether the adapter needs attention.
