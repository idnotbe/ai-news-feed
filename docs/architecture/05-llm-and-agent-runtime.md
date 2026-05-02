# 05 — LLM and agent runtime

Purpose: explain how the Anthropic Python SDK is invoked inside a GH Action job, how GLM-5.1 is wired through the SDK, how the multi-round verify cycle is orchestrated in stage scripts, and how the LLM choice is externalized so swapping is a one-file change.

## Table of contents

- [Overview](#overview)
- [Why Anthropic SDK as harness with GLM as LLM](#why-anthropic-sdk-as-harness-with-glm-as-llm)
- [Configuration externalization (`data/llm.yml`)](#configuration-externalization-datallmyml)
- [Secret flow: GitHub Secret -> env var -> SDK](#secret-flow-github-secret---env-var---sdk)
- [Invocation inside a GH Action job](#invocation-inside-a-gh-action-job)
- [Per-stage LLM call patterns](#per-stage-llm-call-patterns)
- [Determinism and retries](#determinism-and-retries)
- [Swapping providers](#swapping-providers)

## Overview

> DECISION: We use the **Anthropic Python SDK directly** (`anthropic.Anthropic` + `client.messages.create`) and orchestrate the multi-round verify cycle in stage scripts. Spec named "Claude Code / Anthropic Agent SDK" as the harness. Note that pointing the Claude Code CLI at Z.AI is *trivial* — exactly two env vars (`ANTHROPIC_BASE_URL`, `ANTHROPIC_AUTH_TOKEN`) per https://docs.z.ai/devpack/tool/claude — so this choice is **not** about CLI redirect difficulty. *Why the SDK instead*: this pipeline is a non-interactive batch job, not an interactive coding session. The CLI's value (interactive agent loop, tool use, file editing, REPL) is irrelevant here; spawning a CLI subprocess per LLM round adds a process boundary and JSON-over-stdio plumbing for no gain. An in-process Python `messages.create` call is the simplest harness for "draft → verify → fix → verify → finalize" orchestration. The same two env vars work for both the SDK and the CLI, so swapping later (e.g., to wrap the pipeline in `claude` for ad-hoc maintainer use) is trivial. The "agent loop" — multi-turn with verification — is implemented in `scripts/stages/<stage>.py` as a sequence of `messages.create` calls with separate contexts per round. There is no SDK-managed loop; the script is the loop.

**Three things that are sometimes conflated, and what we actually use**:

| Thing | What it is | Used here? |
|---|---|---|
| Claude Code | Anthropic's CLI (a TypeScript/Node binary) that wraps an interactive agent loop with tool use, file edits, etc. | **No**, but works fine if needed: set `ANTHROPIC_BASE_URL=https://api.z.ai/api/anthropic` and `ANTHROPIC_AUTH_TOKEN=<your z.ai key>` per https://docs.z.ai/devpack/tool/claude. We don't choose it here because the pipeline is non-interactive batch — we get nothing from the CLI's interactive loop, tool use, or file editing, and a Python in-process call is simpler than a subprocess per LLM round. |
| Anthropic Python SDK | The `anthropic` PyPI package: `client.messages.create(...)`, transport-only. | **Yes.** This is the only library we depend on for LLM calls. |
| Anthropic Agent SDK / "agent loop primitives" | Higher-level loop helpers (tool use, `Stop` reasons, tool_result round-trips). Available in the SDK, but a single `messages.create` call is not "the agent loop." | **Used minimally / not at all.** Our verify rounds are independent `messages.create` invocations with separate system prompts; we do not register tools (see `08-security.md` Control 5.5). |

The actual model behind the SDK is **GLM-5.1**, accessed via an Anthropic-API-compatible base URL that Z.AI (which exposes GLM models via an Anthropic-API-compatible endpoint) exposes for exactly this kind of drop-in.

> NOTE: Implementation language is **Python 3.12**. See the table at the top of `00-system-overview.md`. Same decision is also pinned in `02-github-actions.md` "Common Python setup."

```
              ┌─────────────────────────────────────────────┐
              │   GH Actions runner (one job)               │
              │                                             │
              │   stage script (e.g. filter.py)             │
              │            │                                │
              │            v                                │
              │   anthropic.Anthropic(                       │
              │       base_url=cfg.base_url,                 │
              │       auth_token=os.environ[cfg.api_key_env])│
              │            │                                │
              │            v                                │
              │   client.messages.create(model=cfg.model,   │
              │       max_tokens=cfg.max_tokens, ...)       │
              │            │                                │
              │            v   HTTPS                        │
              │   ┌─────────────────────────────────────┐   │
              │   │ GLM endpoint (vendor)               │   │
              │   │ api.z.ai/api/anthropic              │   │
              │   └─────────────────────────────────────┘   │
              └─────────────────────────────────────────────┘
```

The Anthropic SDK is a thin transport — we use it for `messages.create` only. The 2-round verify cycle is orchestrated in the stage script (separate `messages.create` calls with fresh system prompts per round), not by SDK-managed agent-loop primitives.

## Why Anthropic SDK as harness with GLM as LLM

> DECISION: Anthropic-SDK-compatible endpoint for GLM, *not* OpenAI-compatible. Spec asked us to pick the realistic option and call out it is configurable. *Why*: (1) the harness we picked is the Anthropic Python SDK (see Overview), so the path of least resistance is to point that SDK at a `base_url` that speaks its protocol; (2) z.ai's `/api/anthropic` endpoint speaks the Anthropic API natively per https://docs.z.ai/devpack/tool/claude, so the SDK is a drop-in. The OpenAI-compatible alternative would force us to either bring in the `openai` SDK or write a translation shim. Both are worse than just configuring `base_url`.

The downside: the Anthropic-compatibility endpoint may lag the native GLM endpoint on bleeding-edge features. Acceptable: filter + summarize do not need exotic features.

## Configuration externalization (`data/llm.yml`)

```yaml
provider: glm
model: glm-5.1
base_url: https://api.z.ai/api/anthropic
# Provider-agnostic env var name; workflow files always export
# `LLM_API_KEY: ${{ secrets.LLM_API_KEY }}` regardless of which provider is
# active. Swapping providers is a one-file change to data/llm.yml plus
# rotating the LLM_API_KEY Secret value in GitHub Settings.
api_key_env: LLM_API_KEY
auth_mode: bearer        # bearer for z.ai (Authorization: Bearer ...); api_key for native Anthropic (x-api-key)
max_tokens: 4096
temperature: 0
timeout_seconds: 120
retry:
  max_attempts: 3
  backoff_seconds: 5
```

> NOTE: Verified against https://docs.z.ai/devpack/tool/claude (last checked: 2026-05-02). The `base_url` and env-var names mirror the canonical Anthropic SDK conventions because z.ai's `/api/anthropic` endpoint is Anthropic-API-compatible. The swap procedure works regardless of the exact URL because `base_url` is one config line in this file; nothing else in the repo references the URL directly.

> DECISION: `auth_mode` field added to `data/llm.yml`. *Why*: z.ai's Anthropic-compatible endpoint authenticates with `Authorization: Bearer <token>` (the Anthropic SDK reads this when constructed with `auth_token=...`). Native Anthropic uses `x-api-key` (the SDK reads this when constructed with `api_key=...`). The two are not interchangeable, so the loader must know which one to use. A single explicit field is cheaper than provider-name string matching. **Default when omitted: `bearer`** (matches the default deployment; the loader uses `cfg.get("auth_mode", "bearer")`). New `data/llm.yml` files should still set the field explicitly so a future maintainer doesn't have to remember the default.

Swapping LLM = edit this file. Examples below.

**Switch to native Anthropic Claude:**
```yaml
provider: anthropic
model: claude-sonnet-4-7
base_url: https://api.anthropic.com
api_key_env: LLM_API_KEY  # same stable env var name; only the value changes
auth_mode: api_key        # native Anthropic uses x-api-key
```
Then in GitHub Settings → Secrets and variables → Actions → `LLM_API_KEY` → Update, paste the new Anthropic key value. **No workflow file edits.**

**Switch to a self-hosted LLM behind an Anthropic-compatible proxy:**
```yaml
provider: self-hosted
model: my-local-glm
base_url: https://my-proxy.internal.example/anthropic
api_key_env: LLM_API_KEY  # unchanged
auth_mode: bearer         # adjust to api_key if your proxy authenticates with x-api-key
```

Stage scripts read this file once at startup (via `scripts/lib/llm.py`). They never hardcode a model name or URL.

```python
# scripts/lib/llm.py — single source of truth for LLM client config.
import os, yaml, anthropic

def load_client():
    with open("data/llm.yml") as f:
        cfg = yaml.safe_load(f)
    token = os.environ.get(cfg["api_key_env"])
    if not token:
        raise RuntimeError(
            f"Missing env var {cfg['api_key_env']} (configured as api_key_env in data/llm.yml)"
        )
    auth_mode = cfg.get("auth_mode", "bearer")
    common = dict(base_url=cfg["base_url"], timeout=cfg["timeout_seconds"])
    if auth_mode == "bearer":
        # z.ai and other Anthropic-compatible endpoints that take Authorization: Bearer
        client = anthropic.Anthropic(auth_token=token, **common)
    elif auth_mode == "api_key":
        # native Anthropic, which authenticates with x-api-key
        client = anthropic.Anthropic(api_key=token, **common)
    else:
        raise RuntimeError(f"Unknown auth_mode: {auth_mode!r} (expected 'bearer' or 'api_key')")
    return client, cfg

def call(client, cfg, *, system, messages, max_tokens=None):
    return client.messages.create(
        model=cfg["model"],
        max_tokens=max_tokens or cfg["max_tokens"],
        temperature=cfg["temperature"],
        system=system,
        messages=messages,
    )
```

> NOTE: If a future Anthropic SDK version drops the `auth_token=` constructor parameter, the alternative is to set the bearer header directly via `default_headers={'Authorization': f'Bearer {token}'}` on the client constructor (and pass a placeholder `api_key=`). This is a one-line change in the loader.

> DECISION: `data/llm.yml` is the single LLM config file. Spec proposed either `data/llm.yml` or repo-level env; we picked the file. *Why*: a checked-in YAML file is auditable in Git history (you can see when the model changed), survives runner restarts, and avoids action-level env duplication across 10 workflow files. The only thing that lives in env is the secret.

## Secret flow: GitHub Secret -> env var -> SDK

```
[ Settings → Secrets and variables → Actions → New repository secret ]
                       │
                       │   user pastes the Z.AI API key value into LLM_API_KEY
                       v
              [ secrets.LLM_API_KEY ]   (encrypted in GitHub backend)
                       │
                       │   workflow yaml: env block on the step that calls Python
                       v
              [ env.LLM_API_KEY ]   (process env in the runner)
                       │
                       │   scripts/lib/llm.py:  os.environ[cfg["api_key_env"]]
                       │   (cfg["api_key_env"] is "LLM_API_KEY" per data/llm.yml)
                       v
              [ anthropic.Anthropic(auth_token=...)  if auth_mode=bearer
                anthropic.Anthropic(api_key=...)     if auth_mode=api_key ]
                       │
                       v
              outbound HTTPS request, token in
                `Authorization: Bearer ...` header  (bearer mode, e.g. z.ai)
                or `x-api-key: ...` header           (api_key mode, native Anthropic)
```

The Secret name `LLM_API_KEY` and the env var name `LLM_API_KEY` are deliberately provider-agnostic and identical, so workflow files never reference a vendor-specific name. Swapping providers is `data/llm.yml` (file edit) + Secret value rotation (GitHub Settings; not a file edit).

Workflow snippet (already shown in `02-github-actions.md`, repeated here in context):

```yaml
- name: Run filter
  env:
    LLM_API_KEY: ${{ secrets.LLM_API_KEY }}    # secret -> env var, both stable names
    RUN_ID: ${{ steps.resolve.outputs.run_id }}
  run: |
    set -euo pipefail
    python scripts/stages/filter.py --run-id "$RUN_ID"
```

**Forbidden patterns** (enforced by code review and the rules in `08-security.md`):
- `set -x` anywhere in any workflow step (would echo the secret if Python error printed env).
- `env | tee something.log`, `printenv`, `echo $LLM_API_KEY`.
- Writing `data/llm.yml` with the literal key inlined instead of a `_env` reference.
- Committing any file under `state/runs/` that contains the key (none of the stages do this; the rule is the negative invariant).

## Invocation inside a GH Action job

Each stage script that uses the LLM follows the same skeleton:

```python
# scripts/stages/filter.py (sketch)
import json, pathlib, sys
from scripts.lib.llm import load_client, call
from scripts.lib.state_io import read_state, write_done

def main(run_id: str):
    state_dir = pathlib.Path(f"state/runs/{run_id}")
    fetch_files = sorted(state_dir.glob("fetch/*.json"))
    state = json.loads((state_dir / "state.json").read_text())

    client, cfg = load_client()

    # --- Draft ---
    candidates_v1 = run_filter_draft(client, cfg, fetch_files, state)
    (state_dir / "candidates_v1.json").write_text(json.dumps(candidates_v1, indent=2))

    # --- Verify round 1 ---
    verify1 = run_filter_verify(client, cfg, candidates_v1, fetch_files, state, round_number=1)
    (state_dir / "verify1_filter.md").write_text(verify1)

    # --- Fix ---
    candidates_v2 = run_filter_fix(client, cfg, candidates_v1, verify1)
    (state_dir / "candidates_v2.json").write_text(json.dumps(candidates_v2, indent=2))

    # --- Verify round 2 ---
    verify2 = run_filter_verify(client, cfg, candidates_v2, fetch_files, state, round_number=2)
    (state_dir / "verify2_filter.md").write_text(verify2)

    # --- Final ---
    candidates_final = run_filter_finalize(client, cfg, candidates_v2, verify2)
    (state_dir / "candidates_final.json").write_text(json.dumps(candidates_final, indent=2))

    write_done(state_dir / "filter.done.json", stage="filter", run_id=run_id, item_count=len(candidates_final))

if __name__ == "__main__":
    main(sys.argv[sys.argv.index("--run-id") + 1])
```

Each `run_filter_*` function constructs its system prompt and message list (with appropriate read-only / fix instructions for verify vs fix), calls `call(client, cfg, system=..., messages=...)`, and parses the response.

## Per-stage LLM call patterns

| Stage | Calls | Pattern |
|---|---|---|
| bootstrap | 0 | deterministic |
| fetch | 0 in most adapters | RSS/HTTP-only adapters do not call the LLM. The `html_blog` adapter may call the LLM if HTML→record extraction needs it. |
| filter | ~5 calls per run (draft + 2× verify + 2× fix/final) | Each call sees the entire fetch output set; sized for ~4–8K tokens of context. |
| summarize | ~5 calls per item × 2 verify cycles | Per-item parallel; thread pool inside the job. |
| assemble | 0 | deterministic |
| log-summary | 0 | deterministic |

> DECISION: filter uses 5 LLM calls (draft, verify1, fix, verify2, finalize), summarize uses ~5 per item. Spec required the 2-round verify pattern from the legacy routine but did not specify call counts. *Why*: matches legacy Phase 2 / Phase 4 structure verbatim; departing from it would re-open quality questions the legacy already answered.

## Determinism and retries

- `temperature: 0` in `data/llm.yml`. Filter and summarize prompts use structured output (JSON for filter, frontmatter+sections for summarize). Two runs of the same prompt at temp 0 are typically byte-identical with GLM-5.1 in our experience; small differences are absorbed by the verify rounds.
- Retry: `scripts/lib/llm.py` retries on `429`, `5xx`, and SDK-defined transient errors. Max 3 attempts, exponential-ish backoff (5s, 10s, 20s). Non-retryable errors (`400`, content policy) bubble up and fail the stage.
- A failed stage causes its `*.done.json` to never be written, so downstream stages naturally wait. Re-trigger the failed workflow from the Actions UI.

## Swapping providers

End-to-end checklist for swapping GLM-5.1 to a different model. **One file edit.**

1. **Rotate the Secret value** — Settings → Secrets and variables → Actions → `LLM_API_KEY` → Update. Paste the new provider's key value. The Secret *name* does not change. **No file edit.**
2. **Edit `data/llm.yml`** (the only file that changes): update `provider`, `model`, `base_url`, and `auth_mode` (`bearer` for Anthropic-compatible bearer-auth providers like z.ai; `api_key` for native Anthropic). Leave `api_key_env: LLM_API_KEY` as-is — workflow files already export this stable env-var name. Adjust `max_tokens`/`temperature` if the new provider warrants it.
3. **No workflow file edits needed.** All seven LLM-using workflows (`filter.yml`, `summarize.yml`, and the five `fetch-*.yml` files) export `LLM_API_KEY: ${{ secrets.LLM_API_KEY }}` regardless of which provider is active. The fetch workflows carry the env even when their default adapter doesn't call the LLM, because some adapters (e.g. `html_blog`) may; keeping them symmetric avoids "this fetch suddenly needs the LLM, why is the secret missing?" surprises later.
4. Commit `data/llm.yml` directly to `main`. External PRs are auto-rejected by `reject-external-pr.yml`; only the maintainer can commit.
5. Trigger `workflow_dispatch` on `bootstrap.yml` to verify a new run completes end-to-end.

> DECISION: workflow files reference the Secret by a stable provider-agnostic name (`secrets.LLM_API_KEY`), not via `data/llm.yml`. Spec required "swap = one file change", which mandates this. *Why*: GitHub Actions does not allow `secrets.*` to be looked up by a dynamic key. Of the two abstraction strategies — (a) always export a fixed env-var name like `LLM_API_KEY`, mapped from a same-named Secret, vs. (b) rename the env var on every swap — only (a) is consistent with "one file change to swap". We chose (a). The provider name is still discoverable: it is the value of `provider:` and `model:` in `data/llm.yml`, which is the single source of truth.
