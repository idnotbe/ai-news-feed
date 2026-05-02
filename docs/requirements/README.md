# ai-news-feed — Requirements

This folder is the contract for what the `ai-news-feed` system must do, and what it must explicitly not do. It is the single authoritative source for downstream consumers (notably any Module 2 audience-specific repackager) and for future maintainers. Implementation choices — workflow file names, schedules, schemas, runtime mechanics — live in `docs/architecture/` and may change. The requirements here are intended to be stable.

`ai-news-feed` is Module 1 of a two-module decomposition. It collects AI news from many English-language sources, deduplicates them, summarizes each item, and publishes a clean, machine-readable, audience-agnostic dataset (per-day digests, a master index, and an auto-updated README). It is explicitly not a newsletter, not a translator, not an audience-specific commentary engine, and not a website beyond its own GitHub README. Anything that turns this dataset into an email, a translated post, or a perspective-tailored brief belongs to Module 2 and is out of scope for this repo.

## Index

1. [00-overview.md](00-overview.md) — Purpose, primary user, two-module decomposition, scope boundary.
2. [01-functional.md](01-functional.md) — Per-item output contract (fields, types, JSON example).
3. [02-sources.md](02-sources.md) — Source categories, English-only rule, multi-source-per-item rule, config-only extension rule.
4. [03-curation.md](03-curation.md) — 4-axis criteria, daily cap, security escape hatch, perspective diversity, dedup rules, closed `perspective_tags` enumeration.
5. [04-operations.md](04-operations.md) — Run cadence (agnostic), logging requirements, README format.
6. [05-security-and-governance.md](05-security-and-governance.md) — Public-repo posture, secrets handling, externalized LLM choice, no external PRs.
7. [06-non-goals.md](06-non-goals.md) — Explicit non-goals.

## Reading order

New readers: read 00 first for context, then 01 for the contract, then the rest in any order. Module 2 authors only strictly need 01, 03 (perspective tags + dedup), and 06 (non-goals).
