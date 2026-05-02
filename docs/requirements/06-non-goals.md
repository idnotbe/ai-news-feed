# 06 — Non-Goals

Purpose: state explicitly what `ai-news-feed` (Module 1) does not do. Each non-goal exists to keep the upstream dataset reusable and the system's surface area small. Anything in this list that becomes a need belongs in Module 2 or in a new module — not in this repo.

## Module 1 does not translate

- Module 1 produces only English output (see [01-functional.md](01-functional.md)).
- Translation to Korean or any other non-English language is out of scope and must not be added to this repo.

Why: a downstream consumer (Module 2) targets a specific language audience; mixing translation into the upstream feed would couple the canonical dataset to a single audience and defeat reuse.

## Module 1 does not send email

- No SMTP, no transactional-email-API integration, no recipient list, no send schedule.
- The pipeline publishes files into the repository; how anyone receives those files is not Module 1's concern.

Why: email cadence and audience belong to the audience-specific repackager, not to the canonical dataset.

## Module 1 does not generate audience-specific commentary

- No "AX 관점", no "for executives", no "for engineers", no per-audience framing of the same event.
- The published item carries the neutral fields defined in [01-functional.md](01-functional.md) — `key`, `content`, `perspective_tags` — and nothing more.
- `perspective_tags` is a *classification* (so Module 2 can filter), not a commentary; it does not editorialize.

Why: every audience-specific overlay is a Module 2 concern. Adding any here would force every downstream consumer to live with one specific consultant's framing.

## Module 1 does not accept external pull requests

- PRs from forks are auto-rejected (see [05-security-and-governance.md](05-security-and-governance.md)).
- No exceptions, including for "small fixes."

Why: single-operator repo, secrets present, and untrusted code in PRs is a known supply-chain vector. The cost of an "occasional good external PR" is dwarfed by the cost of the wrong one.

## Module 1 does not ingest Korean-language sources

- Korean-language outlets (AI타임스, 디지털데일리, etc.) are excluded from the source pool.
- This is enforced at the source-config level, not at the post-fetch filter level.

Why: same reasoning as the English-only output rule — translation belongs to Module 2. Ingesting non-English content here would either drop the content (waste) or force inline translation (out of scope).

## Module 1 is not a website

- The only user-facing surface is the GitHub-rendered `README.md` and the per-day digest files browseable in the repository's file tree.
- No static-site generator, no custom domain, no JavaScript-rendered front end, no hosted dashboard.

Why: the dataset is the product. A website on top would be either a thin GitHub-rendering shim (no value over GitHub itself) or a separate downstream concern (out of scope, like Module 2).

## Out of scope (legacy items explicitly removed)

- The legacy routine had an X/Twitter source category. It was never implemented and is not in scope here.
- The legacy routine ingested Korean-language news. That category is removed.
- The legacy routine produced a "consultant perspective" overlay in the same artifact as the news. That overlay is removed from Module 1 and belongs to Module 2.

Why list the removals: future maintainers reading the legacy routine alongside this repo will otherwise wonder where these went. They are gone on purpose.
