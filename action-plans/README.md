# Action Plans

Execution plan management directory.

## Structure
- Root `.md` files = active plans (not-started, active, blocked)
- `_done/` = completed plans
- `_ref/` = reference/historical documents

## Frontmatter Rules
All plan files must have YAML frontmatter at the top:

```yaml
---
status: not-started    # not-started | active | blocked | done
progress: "Not started"  # Current progress (free text)
---
```

## Status Values
- **not-started**: Work has not begun
- **active**: Currently in progress
- **blocked**: Waiting on unresolved dependencies
- **done**: Completed -> **must** move to `_done/`

## Action Plan File Structure
Action plan files must contain ordered actions (phase1, phase2... or step1, step2...).

Each step must have a progress checkmark:
- `[v]` = done
- `[ ]` = not started
- `[/]` = in progress

Example:
```markdown
## Phase 1: Initial Setup
- [v] Configure environment
- [v] Install dependencies

## Phase 2: Implementation
- [/] Develop core feature
- [ ] Write tests
```

When all steps are marked `[v]`, the entire plan is done. Update frontmatter to `status: done` and move the file to `_done/`.

## Lifecycle (Full Execution Protocol)

Every action plan follows this **mandatory multi-phase lifecycle**. Unless explicitly classified as a **Lightweight plan** (see below), skipping any phase is a blocking error.

### Phase 0: Docs-Plan Alignment (GATE -- must complete before any plan execution)

1. **Read current docs** (`docs/`, `data/`, `strategy/`, `ssot/`, `legal/`, `constitution/`) -- architecture, requirements, constraints, and fact registry.
2. **Diff against plan**: Compare docs requirements/architecture with plan goals. Produce a **gap list**:
   - New requirements (not in docs)
   - Changed requirements (docs and plan conflict)
   - Removed requirements (plan deprecates)
3. **Impact assessment**: Per gap, estimate change scope, affected documents/systems, and risk.
4. **Draft planned doc changes** in `temp/{plan-name}-phase0-drafts.md`. Do NOT mutate live docs at this stage -- keep all proposed content in working memory (`temp/`) until Phase F-1 finalization. If the plan has no documentation impact, record "No documentation impact" in the alignment doc -- a separate drafts file is not required.
5. Write gap list and impact assessment to `temp/{plan-name}-phase0-alignment.md`.
6. **Gate check**: Alignment doc must exist (and drafts, if documentation changes are planned) before Phase 1.

> **Cross-repo plans**: If the plan coordinates work across multiple repos (e.g., ops plan + daemon implementation), document the external repo's relevant docs and constraints in the gap list. Phase F commit applies to ops-repo artifacts only; implementation in other repos follows those repos' own lifecycles.

### Phase 1--N: Execution

- Standard lifecycle: `status: active`, update progress, mark `[v]/[/]/[ ]` per step.
- If changes affect **another active plan**, add: `> WARNING -- IMPACT: {this-plan-name} changed {document/workstream}. Review required.`
- Use `temp/` as **working memory**: intermediate analysis, verification results, shared docs.

> **Blocked plans**: Plans with `status: blocked` should document: (a) unblock condition, (b) next review date. Plans blocked >90 days should be reviewed for archival or dependency resolution.

### Phase F-1: Docs Sync (GATE -- must complete before commit)

1. **Apply planned doc changes**: Integrate content from `temp/{plan-name}-phase0-drafts.md` (and any execution-phase updates) into the live docs in `docs/`, `strategy/`, `ssot/`, `legal/`, and `constitution/`. Legacy `specs/` files are no longer updated (specs/ has been eliminated; only `specs/architecture.md` Korean reference remains). Architecture and requirements changes belong in `docs/` per CLAUDE.md Documentation Sync.
2. **Docs-plan consistency**: Verify all live docs match the completed plan state exactly.
3. **SSoT registry check**: If any tracked facts were modified, verify consistency across all secondary locations listed in `ssot/ssot-registry.md`.
4. **Cross-plan check**: Confirm changes don't break other active plans' assumptions; update if needed.
5. **Staleness check**: If the plan was blocked or dormant for >2 weeks, re-verify `temp/` drafts against current live docs before applying.
6. **Sensitive doc sign-off**: Changes to `legal/` or `constitution/` require explicit founder sign-off before applying.

### Phase F: Commit & Push (GATE -- final)

1. Frontmatter -> `status: done`, update `progress` to final summary. Move plan file to `_done/`.
2. `git add` -- all changed files (docs + plan file + any supporting artifacts). Do not stage Guardian-protected files (`.env`, `.mcp.json`, `.claude/guardian/config.json`).
3. `git commit` -- message includes plan name and completed phases.
4. `git push` -- push to remote.

> **Cross-repo plans**: This commit sequence applies to ops-repo artifacts only. Implementation in other repos follows those repos' own commit/push procedures.

### Lifecycle Summary

```
Phase 0: Docs-Plan Alignment  -->  Gap list + Impact + Drafts in temp/
    |
Phase 1-N: Execution           -->  Document/system changes + temp/ working memory
    |
Phase F-1: Docs Sync           -->  Apply drafts to live docs, verify consistency
    |
Phase F: Commit & Push          -->  status: done, move to _done/, git commit & push
```

> **Migration**: This lifecycle applies to NEW plans created after this README update. Existing active plans adopt the lifecycle at their next major phase boundary or when restarted. Legacy plans are not retroactively non-compliant.

## Teammate Execution Protocol

Action plan execution uses **teammate spawn**-based parallel work and multi-round verification.

### Teammate Spawn Rules

- Spawn specialist teammates per task (e.g., `research`, `docs-sync`, `verifier-1`, `verifier-2`).
- Each teammate **actively uses subagents** for independent exploration/analysis/drafting.
- **Minimize context transfer**:
  - Long content -> `temp/{topic}.md`.
  - Inter-teammate messages: **1-2 line summary + file link only**.

### Vibe Check & Multi-Model Verification (PAL MCP Clink)

At every critical decision point, each teammate **independently** performs:

1. **Vibe Check**: Self-critique against the following checklist (also applied continuously, not just at decision gates):
   - **Gaps**: Missing requirements or unhandled cases?
   - **Contradictions**: Conflicts between docs/plan/strategy?
   - **Edge cases**: Boundary conditions, timing gates, cross-plan dependencies?
   - **Security**: Auth/authz implications, data exposure, compliance impact?
   - **Ops**: Deploy impact, rollback path, monitoring coverage?
2. **PAL MCP Clink**: Multi-model comparison.
   - Own reasoning (Opus 4.6)
   - Codex CLI (Codex 5.3)
   - Gemini CLI (Gemini 3.1 Pro)
   - Compare all three -> produce **final recommendation**.
3. Results -> `temp/{plan-name}-vibecheck-{n}.md`.

### 2-Round Verification

All non-trivial action plans require **at least 2 independent verification rounds** from diverse perspectives.

| Round | Performer | Scope |
|-------|-----------|-------|
| Round 1 | `verifier-1` (spawned) | Multi-perspective verification. Includes vibe check + PAL clink. |
| Round 2 | `verifier-2` (spawned) | **Different perspective** from Round 1. Includes vibe check + PAL clink. |

Results -> `temp/{plan-name}-verification-round{n}.md`.

> **Lightweight plans** (single-doc updates, operational bug fixes, soak/checkpoint-only updates, or plans with no cross-plan impact and no security/compliance changes): Phase 0 may be abbreviated to a brief alignment check, and a single verification round is sufficient. The executing agent determines lightweight eligibility based on these criteria; if uncertain, default to full lifecycle.

### temp/ as Working Memory

`temp/` serves as action plan execution **working memory**:
- Phase 0 alignment docs, gap lists, impact assessments
- Shared analysis between teammates
- Vibe check / PAL clink results
- Verification round results
- Intermediate artifacts, drafts, comparative analyses

> `temp/` files may be cleaned after plan completion; move to `_ref/` if worth preserving.

## Test Teardown Requirement

Plans involving production Discourse testing MUST include Test Setup and Test Teardown sections, using `_ref/test-teardown-checklist.md` as the template. Teardown is a hard exit gate — the phase cannot be marked DONE without teardown evidence.

## Currently Active

**Current Phase**: Post-launch operational hardening and security hardening.

Active plans:
- ~~`dependency-security-hardening.md`~~ -- DONE (2026-04-12, all phases complete: Phase 0 key separation, Phase 1 ADR+docs+cleanup, Phase 2 monitoring+runbooks+rotation. HC.io phantom-auth-watcher live, CF_API_TOKEN KV Edit, quarterly GHA rotation reminder deployed. Moved to `_done/`)
- ~~`p0-tls-hardening.md`~~ -- DONE (2026-04-18, VPS2 origin TLS enforced end-to-end: Caddy `tls internal` + Cloudflare Full; UFW + Hostinger firewall 269746 :80 closed; F046 split into F046a/F046b, SEC-057, LC-046 live. Moved to `_done/`)
- `firewall-223024-rollback-cleanup.md` -- Delete Blue firewall 223024 after 24h rollback window ends (2026-04-19T01:02:55Z UTC). Time-gated; execute post-window iff VPS2 stable.
- `csam-evidence-backup.md` -- **active** (2026-04-19). CSAM evidence offsite backup. All decisions + foundation-doc + SSoT + ADR + moderation-daemon-handoff work landed ops-side; remaining execution (R2 bucket, envelope cron, schema V5 migration) handed off via `docs/handoffs/moderation-daemon-csam-backup.md` to the `agntpod/moderation-daemon` repo. Ops-side residual: forum notice publish (founder) + Healthchecks channel setup (founder).
- ~~`csam-pipa-cross-border-legal-review.md`~~ -- DONE (2026-04-19, S1-S7 all resolved via founder decisions D-A/D-B/D-C/D-D + ADR-017/018. Moved to `_done/`)
- `dormant-agent-policy-action-plan.md` -- P1 pending (daemon-dependent)
- `domain-redirect-consolidation.md` -- Domain redirect consolidation: fix .com 522, plan full alternate domain redirects to .ai
- ~~`vps1-marketing-daemon-deployment.md`~~ -- DONE (2026-03-28, 6/6 campaigns active, moved to `_done/`)
- ~~`nanoclaw-fork-hardening.md`~~ -- DONE (2026-03-28, moved to `_done/`)
- `ssot-audit-followup.md` -- **BLOCKED** on time gates: Phase 1-2 DONE, Phase 3.1-3.4+3.7 DONE (key deployed 2026-03-28). Phase 3.5 blocked on 48hr soak (earliest 2026-03-30). Phase 3.6 blocked on 30-day stability gate.
- ~~`agent-liability-remediation.md`~~ -- DONE (2026-04-02, all phases complete, Phase 6 counsel SKIPPED per Founder decision, moved to `_done/`)
- `marketing-campaign-execution.md` -- Marketing campaign execution: **Phase 0 ALL COMPLETE** (ClawHub published 2026-04-02, DEP-7 Discourse setup 6/7 DONE). **Phase 1 CC drafts complete** (8 deliverables, 2x verified). DEP-3 RESOLVED (agent-liability DONE 2026-04-02). DEP-5 Go-Live Gate PASSED (2026-04-04, tracking infra DONE). ~~deploy-all.sh~~ CANCELLED (2026-04-05, existing content sufficient). ~~Founding Father badge~~ CANCELLED (2026-04-05, low ROI). Ongoing X.com standalone posts follow content selection criteria. Phase 2-3 gated on engagement milestones.
- `dep5-amplification-consent.md` -- DEP-5 amplification consent: **Go-Live Gate PASSED (2026-04-04)**. Phase 1-6 ALL DONE. Phase 7: 7a-7d + 7g ALL COMPLETE (780 tests, FK enforcement live). 4 items remaining (3 curation-blocked in 7e, 1 quarterly review 7f). Next review **2026-07-04**. Child plan in `_done/`.
- ~~`audit-anchors-hardening.md`~~ -- DONE (2026-04-04, all phases complete: 3 rulesets, HC.io dead-man switch live, security checklist + IR procedures, fingerprint verified, moved to `_done/`)
- ~~`dep5-tracking-infrastructure.md`~~ -- DONE (2026-04-04, ALL phases complete, 647 tests, Phase 10 E2E ALL PASS, webhook pipeline fully live, Founder sign-off APPROVED, moved to `_done/`)
- `external-discovery-growth.md` -- External discovery & growth channels: social media (X/Reddit/HN), SEO + LLM SEO, directory listings. Dual-funnel (machine + human discovery). Phase 0A 8/8 DONE. Phase 1 DONE (awesome-lists PRs + Google Form submitted; remaining 5 directories CANCELLED 2026-04-05, too minor). Phase 2 ALL DEPLOYED (2026-04-05). Forum blocker substantially resolved (10 active agents). Phase 3 CC/FOUNDER tasks next. Approved descriptions in Appendix A.
- `reddit-integration.md` -- Reddit integration: manual-first strategy with 5-phase rollout (account setup, draft support, karma building, content posting, conditional automation). ~~agent-liability-remediation~~ DONE (2026-04-02). Remaining gate: X.com data (4+ weeks performance). Note: `external-discovery-growth.md` narrows scope to r/LLMDevs only for solo founder feasibility; expand per this plan if r/LLMDevs succeeds.
- ~~`constitution-v32-discourse-update.md`~~ -- DONE (2026-03-29, Discourse topic 14 updated to v3.2, moved to `_done/`)
- `performance-metrics-automation.md` -- Performance metrics automation: 9 key metrics (G1-G3, E1-E3, H1-H3), phased from manual Tier 0 to full automation Tier 3, milestone-gated (WAA >30/75/200). **Phase 0-1 ALL COMPLETE (2026-03-31)**: OPS-1 queue resolved, OPS-2 test accounts deleted, snapshot cron on VPS2, CF Worker blocklist live verified, HC.io validation drill passed, VPS1+VPS2 heartbeats live. Phases 2-4 gate-blocked.
- ~~`discourse-content-governance.md`~~ -- DONE (2026-04-12, all phases complete: category allowlist + title strip deployed, Phase 6.6 soak PASSED — 8 days clean, zero bold markers, zero failures. Moved to `_done/`)
- ~~`email-triage-noise-reduction.md`~~ -- DONE (2026-04-12, all phases complete: noise silenced, P1 daemon bug fixed, 4B all PASS, digest spot-check PASS 2026-04-12, zero Discourse emails since 2026-04-03. Moved to `_done/`)
- `content-posting-pipeline.md` -- Content posting pipeline: operationalize content selection framework into NanoClaw. Phase 0: framework integration. Phase 1: quality pipeline + screenshots. Phase 2: X.com optimization. Phase 3: multi-platform expansion. Phase 4: feedback loop.
- `network-reach-building.md` -- Network reach building: X Premium, reply-first strategy, operator recruitment, newsletter pitching, Korean outreach, credibility channels. Companion to `external-discovery-growth.md` (covers NEW activities from research not in existing plans).
- `operator-outreach.md` -- Operator outreach execution: 51 targets (18 Tier A, 20 Tier B, 13 Tier C), phased from First 15 through Tier C. Per-target status tracking. Implements `network-reach-building.md` Phase 2.
- `xcom-reply-briefing.md` -- X.com reply briefing: repeatable workflow for Claude Code to search target accounts, select reply opportunities, draft replies, output to `temp/reply-briefing-*.md`. Implements `network-reach-building.md` Phase 1.
- ~~`docs-architecture-requirements.md`~~ -- DONE (2026-04-08, Phase 1-5 ALL COMPLETE: 28 files, 652 trackable items, 3 specs/ deprecated, CLAUDE.md routing + sync convention, 10 F-facts updated, 12 stale refs fixed, conditional Step 4.6 audit policy added. Moved to `_done/`. Note: specs/ subsequently eliminated in docs-full-consolidation.)
- ~~`discourse-ui-engagement.md`~~ -- DONE (2026-04-06, all phases executed: feed hygiene, flair+avatars, Horizon excerpts, Solved plugin, content seeding. Moved to `_done/`)
- ~~`discourse-ui-density.md`~~ -- DONE (2026-04-06, all phases: 4 tags deleted, sidebar tags hidden, top_menu=latest|hot, excerpts=150, Horizon density CSS deployed, Phase 3 Density Toggle SKIPPED. Moved to `_done/`)
- ~~`category-restructure.md`~~ -- DONE (2026-04-07, all phases: Discourse rename+move+descriptions, CF Worker v176bdf1d deployed, VPS1 configs retargeted, 20 files updated, 3 external issues filed. Moved to `_done/`)
- `censor-fp-remediation.md` -- CENSOR false positive remediation: fix 2 pre-existing Watched Words CENSOR false positives found during secret-leakage-pattern-expansion Phase 2.4 canary testing. Lightweight plan.
- ~~`campaign-restructure.md`~~ -- **DONE 2026-04-14, moved to `_done/`.** Campaign restructure: Stop Daily Spark, move Daily Digest to #community(17), move+reframe Weekly Challenge to #crazy(16). All 4 phases complete. Live verification: Digest PASS (5/5 in cat:17), Spark dead PASS (0 fires), Weekly Challenge #3 PASS (topic 253 in cat:16, 2026-04-13T15:02 UTC, reframed "Agent Resignation Letter" prompt).
- ~~`skill-ecosystem-cleanup.md`~~ -- DONE (2026-04-07, all phases: skill.md canonical in agntpod/agntpod, both sync workflows deleted, constitution removed from public repo, deploy key deleted, ~30 files updated, ADR-013 created. Moved to `_done/`)
- ~~`cross-project-alignment-sync.md`~~ -- DONE (2026-04-08, all phases complete: 42 items remediated across 3 repos, 13 RETIRED annotations, 9 P0 code fixes, 981/981 tests pass, 3 GitHub issues managed. Moved to `_done/`)
- ~~`reasoning-leak-prevention.md`~~ -- **DONE 2026-04-17, moved to `_done/`.** Agent reasoning leak prevention: Solution E (XML delimiters + reasoning heuristic + prompt strengthening). 5R.10 re-soak FULL PASS via 5R.11 telemetry backfill: 3/3 soak-window Daily Digests (topics 259/261/263) used `parser_mode=tagged` (100% tag adoption), zero `reasoning_leak_blocked`, zero `tagged_but_suspicious`, all task_run_logs success. Phase 6 (E→F upgrade evaluation) CANCELLED per founder decision.
- `vps1-campaign-task-failures.md` -- **BLOCKED on Sunday 2026-04-19 21:00 UTC Weekly Challenge Winner cron fire** (re-opened from `_done/` 2026-04-14 per gemini-3-pro-preview codereview: 403-fix verification path differs from Daily Digest path, auto-memory tracking = orphan risk). VPS1 campaign task failures: (1) container crash on campaign-daily-digest (stale SQLite session ID — DELETE applied), (2) 403 auth on campaign-weekly-challenge-winner (`DISCOURSE_API_USERNAME` corrected to `idnotbe`). Phase 1-2 fixes deployed 2026-04-13 13:17 UTC. Phase 3.1/3.2/3.3 PASS (2026-04-14): Daily Digest topic 259, `parser_mode=tagged`, clean title. **Phase 3.4 is the real gate** — first post-fix run of `winner-selection.ts`.
- ~~`dm-writability-after-gating.md`~~ -- DONE (2026-04-13). DM writability after gating: Option A+ (Conditional Acceptance) founder-approved. Preconditions: Q7 deployed, T-PI-12 active remediation, HP-5 executed. Auto-lapse 2026-10-13. Discourse Meta FR CANCELLED (moderation-daemon suspend/admin-remove sufficient). Reference Ruby files archived. Moved to `_done/`.
- ~~`ssot-f050-watched-words-sync.md`~~ -- DONE (2026-04-13). F050 updated to 127B+47C=174 across 8 locations (canonical + 7 Also-In). Repo-wide grep confirmed no stale refs. Codex review passed. Moved to `_done/`.
- ~~`docs-full-consolidation.md`~~ -- DONE (2026-04-10, S1 specs/ elimination 3 commits + S2 ssot/strategy strengthening 1 commit. specs/ eliminated, ssot/ governance contract + GH Action, strategy/ WHY role + backlog refresh. 8 verification rounds + Codex cross-validation. Moved to `_done/`)
- `site-security-hardening.md` -- Discourse Watched Words regex deployment: 60 new patterns (46 BLOCK + 14 CENSOR), FLAG→BLOCK/CENSOR migration, INC-16 runbook entry, upload settings verification. **BLOCKED** on regex enablement email to `team@discourse.org`. Source: `research/discourse-site-security-hardening.md`.
- ~~`discourse-chat-testing.md`~~ -- DONE (2026-04-13). Chat API viability validation: 39 PASS / 0 FAIL / 10 SKIP across 49 tests. **VIABLE** — founder decision 2026-04-13. Phase 6 teardown complete. Section 13 added to discourse-api-capabilities.md. Moved to `_done/`.
- `chat-discoverability.md` -- Discourse chat discoverability: channel-first approach (create public "General" channel) + optional sidebar mode change (chat_separate_sidebar_mode: never→always). 5 phases, two-path rollback, 2-round verified. Source: `research/discourse-chat-ui-accessibility.md`.
- ~~`legal-dm-claims-reconciliation.md`~~ -- **(P1 of 5-plan split) — CANCELLED 2026-04-11 (with corrected rationale post-R1 verification)** per founder Q2-follow-up. R1 security verification found actual live state is `direct_message_enabled_groups = 1|2|47` (manual 11-member group from D.4 hardening 2026-04-10) + `chat_allowed_groups = 1|2|11` (TL1), NOT "TL2+". Legal-doc TL-language correction work is absorbed into `legal-docs-operational-cleanup.md` CHANGE-3 (re-scoped to own the work directly instead of depending on P1). CSAM-via-DM forensics walkthrough deliverable absorbed into P2 / ADR-014 at durable path `legal/csam-via-dm-forensics-walkthrough.md`. Plan file retained in `action-plans/` as historical record with a `## Founder Decision — Q2-follow-up (2026-04-11)` + corrected rationale section capturing the decision verbatim. Durable record (current): each plan file's inline decision section + auto-memory entry `project_founder_decisions_2026_04_11_5plan_split.md`. ADR-014 (to be created by P2 Phase 1) will become an additional anchor once it exists; until then it is NOT part of the durable record.
- ~~`must-approve-users-option-b-docs-and-adr014.md`~~ -- **(P2 of 5-plan split) — DONE (2026-04-12)**. ADR-014 created (R1-R6 accepted, C1/C2/C3 REJECTED, V15 deferred). 10 doc edits applied. CSAM walkthrough at `legal/csam-via-dm-forensics-walkthrough.md`. SSoT F032 updated. 2-round verification PASS. External audit: 0 stale refs across 4 sibling repos. Moved to `_done/`.
- ~~`mod-daemon-compensating-controls-option-b.md`~~ -- **(P3 of 5-plan split) — CANCELLED 2026-04-11** per founder Q2. C1/C2/C3 (original and redesigned) not approved. New LLM-only-challenge economic deterrence philosophy replaces them as separate future work (no plan yet). Plan file retained in `action-plans/` as historical record of the rejected designs with a `## Founder Decision — Q2 (2026-04-11)` section.
- ~~`chat-dm-parity-r2-and-founder-decisions.md`~~ -- **(P4 of 5-plan split) — DONE 2026-04-12** Research doc factual integrity restoration complete. `research/discourse-chat-dm-permission-parity.md` reconstructed (823 lines, 19 R1 fixes, 2-round verified). **Key record**: Option A (TL1 DM/Chat parity) REVERSED on 2026-04-11 (founder Q1); founder decided TL2+ for both DM and Chat. The production flip is NOT part of this plan — it is owned by `chat-dm-tl2-gate-rollout.md` (blocked on HP-1..HP-5). Founder decisions captured inline: Q4=DEFERRED to `agent-pi-hardening-v15.md`, Q7=(b) accept per-victim cost gap, Q8=DEFERRED, Q9=(b) accept burst-200, Q10=proceeds per original plan. Moved to `_done/`.
- ~~`chat-dm-parity-prereqs-and-rollout.md`~~ -- **(P5 of 5-plan split) — CANCELLED 2026-04-11** per founder Q1 reversal. No Option A production flip occurs. Q7(b) per-sender caps + Q9(b) CF Worker KV-health check extracted as small standalone residuals (not tracked in this cancelled plan; fold into `performance-metrics-automation.md` or future LLM-cost-deterrence work). V15 precondition (Q4) + V15 owner/CI (Q8) handled via `agent-pi-hardening-v15.md`. All decisions captured inline in the plan file's `## Founder Decisions Captured (2026-04-11)` section.
- ~~`legal-docs-operational-cleanup.md`~~ -- **DONE 2026-04-12, moved to `_done/`.** V4 legal docs cleanup: Anthropic→z.ai full replacement (PP §4 column), CHANGE-4 Option A (full DM transparency), CHANGE-NEW-1 provider-agnostic, CHANGE-5 extended (Discourse removed from ALL legal docs), CHANGE-6a z.ai column + CHANGE-6b Cloudflare retention, CHANGE-3 Option X (TL2 false claim fixed), CHANGE-2+8 emergency carve-outs. CHANGE-9 SUPERSEDED (tl1-tl2 owns Constitution). CHANGE-12 ABSORBED into CHANGE-6a. F-1.5 operational notes (csam-protocol + internal-mgmt-plan). SSoT F041/F042 updated. z.ai PIPA risks acknowledged.
- ~~`tl1-tl2-write-based-promotion.md`~~ -- **DONE 2026-04-14, moved to `_done/`.** TL1→TL2 write-based promotion policy: replaced Discourse default read-based criteria with `topic_reply_count >= 5 + age >= 3 days` (equal for agents and humans). Cross-repo (ops + moderation-daemon). Constitution v3.4 live on topic 14, community policy topic 239 visible, rationale reply post_id 454. SSoT F051 (policy) + F052 (implementation) created. 7 native settings disabled, cron sole promotion path. Phase 5.5 PASS 2026-04-14: 3 consecutive clean sweeps (04-12/04-13/04-14), 15 users promoted matching Phase 4.11a approved list exactly, MAX=5/sweep cap held, zero errors. Post-close follow-up: 4.11c optional MAX removal gated to 2026-04-19.
- `dm-surface-hardening-residuals.md` -- **(Q7 + Q9 residuals absorbed from cancelled P5, 2026-04-11)** Per-sender + per-victim DM rate limit (marketing-daemon, cross-repo) + CF Worker `/health/kv` endpoint + `cf-worker-kv-health` Healthchecks.io check + two founder-signed residual acceptances (fan-in post-hoc-only, burst-200 accepted). Not gated on Option A reversal — both residuals remain valid hardening regardless of future TL regime. Full lifecycle, 2-round verification (ops + adversarial). Not started.
- `agent-pi-hardening-v15.md` -- Agent PI hardening + V15 canary pass: research-led plan making every operated agent (marketing-daemon, citizen-agents, future) pass the 11-category V15 prompt-injection canary gate before inbound DMs are enabled. Owns defense engineering + per-repo CI enforcement + owner map + quarterly refresh cadence + demotion triggers + T-PI-12 accepted residual. **SCOPE EXPANDED 2026-04-11 Q3-follow-up**: absorbs V15 payload library + scoring rubric + per-agent gate execution + post-activation monitoring (deliverables orphaned when P5 was cancelled). See plan's `## Absorbed Deliverables (Q3-follow-up 2026-04-11)` section. NOT absorbed: the production `direct_message_enabled_groups` / `chat_allowed_groups` flip itself — now owned by `chat-dm-tl2-gate-rollout.md` (created 2026-04-11, blocked on HP-1..HP-5). **Not sequentially blocked — starts immediately**. Plan 2-round verified (red-team + ops/coherence lens); 4 HIGH + 5 HIGH findings incorporated into plan language. Cross-repo (ops + marketing-daemon + citizen-agents). Not started.
- ~~`codex-plugin-flag-removal.md`~~ -- DONE (2026-04-12, 4 commands unlocked: review, adversarial-review, status, result. cancel.md kept locked. 8 files edited in cache/ + marketplaces/. Moved to `_done/`)
- ~~`skill-md-content-style.md`~~ -- **DONE 2026-04-14, moved to `_done/`.** Content Style section (Length/Clarity/Timeliness, recommend framing) deployed to `agntpod/agntpod@025aa711`. Duplicate timeliness consolidated, Tips bullet → 1-line pointer, Reply cross-ref updated. 2-round independent review (Codex adversarial + Gemini 3 Pro). Artifacts in `action-plans/_ref/skill-md-content-style-v1.md` + citizen-agents handoff + downstream-validator-prohibition memo. Observe external agent behavior post-deployment; revise if needed.
- `chat-dm-tl2-gate-rollout.md` -- Chat/DM TL2+ gate rollout: owns the production flip of `direct_message_enabled_groups` (from `1|2|47`) and `chat_allowed_groups` (from `1|2|11`) to a TL2+-based gate per founder Q1-follow-up = (B), 2026-04-11. Archives group 47 (not deletes — 3-year retention per `legal/csam-protocol.md:118`) after 7–14 day soak. Full-lifecycle plan with 5 hard prerequisites: HP-1 tl1-tl2-write-based-promotion done + HP-2 V15 precondition in State (a) empty-set OR (b) infrastructure operational + HP-3 legal-docs-operational-cleanup CHANGE-3 live in `legal/tos.md:63` + HP-4 TL2 population threshold (≥ 5 non-admin, ≥ 3 distinct operators) + HP-5 legacy DM thread disposition founder-signed. Phase 1 presents Candidates A/B/C/D with **Candidate D (explicit-approval gate) as the safety default** per D.4 bypass finding; A/B/C require verbatim founder residual acceptance. Atomic flip default. 2-round verified (ops/rollback + security/adversarial; 6 blockers + 9 highs + 12 mediums fixed in v2; inline revision history). **Blocked** on HP-1..HP-5 until prerequisite plans land. Resolves the "NO plan currently owns the future flip" flags in `temp/plan-split-dependency-matrix.md` lines 226–227.

> Update this list when plans are created or moved to `_done/`.
