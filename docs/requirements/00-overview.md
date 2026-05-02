# 00 — Overview

Purpose: define the role of `ai-news-feed` (Module 1), its primary user, and where its responsibility ends.

## Primary user

The primary user is an AX (AI Transformation) Consultant who needs a daily, deduplicated, English-language signal of what changed in AI. The user is a single-operator persona; the system is not designed for community contributions (see [05-security-and-governance.md](05-security-and-governance.md)).

> NOTE: Ambiguity — the spec describes one specific user but the dataset is meant to be reusable by any number of downstream consumers. Decision: treat the AX Consultant as the *editorial* persona (i.e. their judgment defines "important enough to publish"), but treat the *output* as audience-agnostic. This keeps the dataset reusable while keeping curation decisions coherent.

## Two-module decomposition

The legacy "Daily AI News" routine (a multi-agent in-process pipeline run from claude.ai) is being split into two independent public GitHub repositories:

- **Module 1 — `ai-news-feed` (this repo).** The upstream, canonical dataset. Collects, dedupes, summarizes, and publishes English-only AI news as per-day digest files, a master `index.json`, and an auto-updated `README.md`.
- **Module 2 — separate downstream repo (not in scope).** Consumes Module 1's output and repackages it for a specific audience: filtering by `perspective_tags`, translating to non-English languages, adding audience-specific commentary (e.g., "AX 관점"), and delivering by email or other channels on its own cadence.

Why split: the upstream dataset must be reusable by zero, one, or many downstream repackagers. Mixing audience-specific transforms into Module 1 would couple the dataset to a single audience and prevent reuse.

## Scope boundary with Module 2

Module 1 owns:

- Source ingestion (English sources only).
- Deduplication and source-merging.
- Importance filtering against the 4-axis criteria.
- Summarization into the per-item English fields defined in [01-functional.md](01-functional.md).
- Publication of the dataset (digests + index + README).
- Operational logging.

Module 1 does **not** own (these are Module 2 concerns):

- Translation to any non-English language.
- Any audience-specific commentary or perspective overlay.
- Email rendering, recipient management, or send scheduling.
- Filtering for a specific reader profile beyond the closed `perspective_tags` enumeration that Module 2 uses as input.

The contract between modules is: Module 2 reads Module 1's published output (digest files and `index.json`) using the field schema in [01-functional.md](01-functional.md). Module 1 makes no API calls to Module 2 and has no awareness of any specific Module 2 instance.

Why a hard boundary: it lets multiple Module 2 instances exist (different languages, different audiences) without forcing changes to Module 1.
