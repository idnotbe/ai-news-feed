# 01 — Functional Requirements: Per-Item Output Contract

Purpose: define every field of a published news item. These per-item fields are the **stable contract** that Module 2 (and any future downstream consumer) consumes. Treat changes to this schema as breaking changes.

## Output language

All published text fields (`title`, `key`, `content`, `keywords`) are in **English**. This is non-negotiable for Module 1.

Why: downstream Module 2 instances may translate to other languages (Korean, etc.); having a single canonical English source avoids combinatorial source-language complexity and lets downstream translation be deterministic.

## Per-item fields

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | yes | Stable identifier for the item. **Does not change** across re-publications of the same news event, even if the title or primary source changes. Downstream consumers should join on `id`, not on `title` or `link`. |
| `title` | string (English) | yes | Original or normalized English title. |
| `link` | string (URL) | yes | Canonical link. Must equal `sources[0].url`. Surfaced as the headline link in README and digests. |
| `sources` | array of objects | yes (>=1) | All sources covering the same event. `sources[0]` is the primary; the rest are corroborating. See [02-sources.md](02-sources.md). |
| `sources[].name` | string | yes | Human-readable source name (e.g., "Anthropic News"). |
| `sources[].url` | string (URL) | yes | Source URL for this specific source. |
| `sources[].published_at` | string (ISO8601) | yes | Publication timestamp at the source, UTC. |
| `keywords` | array of strings | yes | 3–5 lowercase English keywords. Hyphens permitted (e.g., `mac-mini`). |
| `key` | string (markdown, English) | yes | Short essence: what happened, in 3–4 lines. The English equivalent of the legacy "핵심" field. |
| `content` | string (markdown, English) | yes | Longer explanation: facts, numbers, context, why it matters. More detail than `key`. |
| `perspective_tags` | array of strings | yes (>=1) | One or more values from the closed enumeration in [03-curation.md](03-curation.md). |
| `first_seen_utc` | string (ISO8601) | yes | UTC timestamp set on the **first publication** of the item. **Does not change** when the item is updated (new corroborating sources merged, `content_hash` changed, title or primary source revised). |
| `content_hash` | string | yes | SHA-256 prefix of `content`. Used for change detection across runs. |

Why a `content_hash` separate from `id`: the same news event (`id` stable) can be updated as facts emerge; the hash lets downstream consumers detect a meaningful content change without diffing the full body.

Why `link` duplicates `sources[0].url`: it is the field downstream renderers use without traversing the `sources` array. Keeping it explicit avoids ambiguity if `sources[]` is ever reordered by a consumer.

## Where the per-item contract lives

The per-item field set above is the contract Module 2 (and any other downstream consumer) reads. The aggregate index file published by the pipeline (`index.json` at the repo root) **must carry the full schema for every item**, including the markdown-bodied `key` and `content` fields — not just metadata.

Why include `key` and `content` in the index rather than only in per-day digest files: a Module 2 instance that wants to filter on `key` text or render `content` should be able to do so by reading **one** JSON file. Forcing consumers to fetch one Markdown file per item and parse YAML frontmatter to retrieve the body would make the integration burden materially higher. Per-day digest files remain the human-facing artifact; `index.json` is the machine-facing artifact and is the canonical contract.

Tradeoff: `index.json` will be larger (each item carries a few hundred to a few thousand bytes of markdown). At the dataset's expected volume (1–10 items/day, 7-day rolling README horizon, full archive on disk) this is well within an acceptable file size for a single JSON read; the simpler contract is worth the extra bytes.

## JSON example

```json
{
  "id": "2026-05-01-gpt55-cyber-baseline",
  "title": "GPT-5.5 cyber capabilities match offensive-research baseline; others to follow",
  "link": "https://www.example-vendor-blog.com/2026/05/gpt-5-5-cyber",
  "sources": [
    {
      "name": "Vendor Official Blog",
      "url": "https://www.example-vendor-blog.com/2026/05/gpt-5-5-cyber",
      "published_at": "2026-05-01T14:02:00Z"
    },
    {
      "name": "TechCrunch AI",
      "url": "https://techcrunch.com/2026/05/01/gpt-5-5-cyber-coverage",
      "published_at": "2026-05-01T16:48:00Z"
    }
  ],
  "keywords": ["gpt-5.5", "cybersecurity", "aisi", "claude", "penetration-testing"],
  "key": "Vendor reports GPT-5.5 reaches the offensive-research baseline on AISI cyber evals. Two other frontier labs are expected to clear the same bar within weeks. AISI is publishing a methodology note alongside the result.",
  "content": "On 2026-05-01, the vendor disclosed that GPT-5.5 matches the offensive-research baseline defined by the UK AI Safety Institute on a standardized cyber capability evaluation. The post links to the AISI methodology and notes that two other frontier labs have submitted models for the same evaluation, with results expected within four weeks. The practical implication for AX practitioners is that any product relying on the absence of frontier-model offensive cyber capability needs to revisit its threat model.",
  "perspective_tags": ["security", "llm", "policy"],
  "first_seen_utc": "2026-05-01T17:10:33Z",
  "content_hash": "9f4a2c81e6b0"
}
```

## Schema stability and versioning

- The field set above is the v1 contract.
- Adding a new optional field is a non-breaking change.
- Removing a field, renaming a field, changing a field's type, or changing the meaning of an existing value is a breaking change and requires a documented schema version bump that downstream consumers can detect.

> NOTE: Ambiguity — the spec does not define a `schema_version` field. Decision: do not introduce one in v1 (keeps the contract minimal and matches the spec table verbatim). When the first breaking change is needed, add `schema_version` at that point. Document this here so downstream Module 2 authors are not surprised by its absence today.
