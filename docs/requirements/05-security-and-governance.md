# 05 — Security and Governance

Purpose: define the repository's security posture, secret handling, model-choice externalization, and contribution policy. Implementation mechanism (workflow names, exact action versions, branch protection settings) is an architecture concern and not specified here.

## Public repository with secrets

- The repository is **public**.
- The LLM provider's API key is stored as a repository secret in the hosting platform's secrets store.
- Secrets must never be:
  - Echoed in logs (committed summary logs or runner console logs).
  - Included in committed files (digests, index, README, source configs, run logs).
  - Surfaced in PR diffs.
  - Embedded in commit messages.

Why public + secrets rather than private repo: the dataset itself is intended to be a public resource for downstream consumers, and the legacy routine was already operated as a public artifact. A private repo would defeat the purpose. The cost of this choice is the discipline required around secret hygiene.

## LLM choice is externalized in config

The choice of LLM (provider, model name, version) must live in a configuration file or repo-level environment variable, not in code.

- Swapping to a different LLM (different provider, different model size, different version) must be a single configuration change.
- The current choice (per the spec context) uses GLM-5.1 as the model under a Claude Code / Anthropic Agent SDK harness, but the requirement here is the *externalization*, not the specific model.

Why externalize: model-quality landscape changes faster than the rest of the system. Hard-coding a model would force a code change every time a better/cheaper model lands; externalizing it makes that a config edit.

> NOTE: Ambiguity — the spec mentions `data/llm.yml` as a possible location. Decision: the *requirement* is externalization (any single file or env var works); the exact location is an architecture choice.

## No external pull requests

External pull requests (PRs from forks) are **not accepted**. They must be auto-rejected.

Required behavior:

- An incoming PR from a fork (i.e., the head repository differs from the base repository) must be automatically closed with a brief comment explaining that the repository does not accept external contributions and pointing the contributor to forking as the alternative.
- The auto-rejection must be implemented in a way that does **not** check out untrusted code from the PR branch; it may only operate via the hosting platform's API.

Why no external PRs:

- Single-operator repository: there is no review capacity for community contributions.
- Security: this repo holds secrets and runs an LLM-driven pipeline against untrusted external content; accepting code from forks would dramatically expand the attack surface (workflow injection, supply-chain).

Why the "do not check out the PR branch" constraint is a hard rule: a workflow that runs on PR events with elevated permissions and checks out the fork's code is the canonical supply-chain attack vector for public repos. The rejection workflow must avoid it.

## No PR-triggered workflows beyond rejection

Beyond the external-PR rejection workflow, the system **must not** have other PR-triggered workflows. The rejection workflow is the only permitted use of `pull_request` or `pull_request_target` in this repository.

Why: same reasoning as above — the simplest secure posture is "no PR triggers except the rejection one." Adding a second PR-triggered workflow expands the attack surface (one more workflow file to keep correct under the no-checkout-of-fork-content rule), and the maintainer is the only contributor, so PR-driven automation has no audience.

If a need ever does arise for a maintainer-only PR-triggered check, it is treated as a contract change to this requirement, not a casual addition: it must be added in a separate documented change to this file, must use trigger semantics that do not grant write permissions to forked code, and must not check out fork content with elevated permissions. Until that change happens, the rule is absolute.

## Secret hygiene in practice

- Source configs, perspective configs, and any other committed file must not contain credentials.
- Run summary logs (see [04-operations.md](04-operations.md)) must redact any value sourced from a secret.
- Error messages that include request bodies or headers must scrub authorization headers before logging.
- Diffs created by the pipeline (digest writes, index writes, README writes) must be reviewed by the architecture for any field that could echo a secret; none of the per-item fields in [01-functional.md](01-functional.md) should ever carry one, but the constraint stands.

Why call this out explicitly: in a public-repo-plus-secrets posture, a single accidental log of an API key is a credential rotation event. The cost of preventing it is small; the cost of cleaning up after it is not.
