---
status: not-started
progress: "Not started — awaiting Phase 0 alignment kickoff."
---

# Module 1 — LLM 하네스 & 로깅 결정 적용

## 결정 사항 (Decision Record)

이 plan은 Module 1 (ai-news-feed)에 대해 founder가 확정한 두 가지 결정을 docs에 반영한다. 코드 구현 변경은 없다 (현재 repo에 `scripts/`, `.github/workflows/`, `data/`가 아직 없으므로 순수 docs 작업).

### Decision 1 — LLM 하네스: Claude Agent SDK

GitHub Actions 안에서 LLM을 돌리는 하네스로 **Claude Agent SDK**를 사용한다.
- Python: `claude-agent-sdk`
- TypeScript: `@anthropic-ai/claude-agent-sdk`

**채택하지 않은 옵션 (이유 명시):**
- 베어 `anthropic` Python SDK — transport-only, agent loop 없음. *이 plan은 현행 `docs/architecture/05-llm-and-agent-runtime.md`의 "Anthropic Python SDK directly" 결정을 뒤집는다.*
- `anthropics/claude-code-action` GitHub Action — 호출당 컨테이너 init 비용, `@claude` mention 자동화용 (우리 use case와 다름).
- Claude Code CLI 헤드리스 (`claude --bare -p`) — subprocess 오버헤드, 세션 ID 수동 관리.

**LLM 백엔드:** GLM-5.1 via Z.AI Anthropic-compatible 엔드포인트.
- `ANTHROPIC_BASE_URL=https://api.z.ai/api/anthropic`
- `ANTHROPIC_AUTH_TOKEN=<Z.AI key>` (GitHub Secret 이름은 provider-agnostic하게 `LLM_API_KEY`)

**설정 외부화:** `data/llm.yml` 한 파일에 `provider`, `model`, `base_url`, `api_key_env`, `auth_mode`. LLM 교체 = 이 파일 1줄 + GitHub Secret 값 회전. 워크플로 파일 편집 없음.

### Decision 2 — 로깅: 4-레이어 + 옵션 1개

| Layer | 필수? | 보존 | 내용 |
|---|---|---|---|
| 1. Canonical durable record | 필수 | 영구 (commit) | `logs/YYYY-MM-DD/<run_id>-summary.md` Markdown 요약을 매 run마다 commit. per-source ok/fail/item count, per-stage 시작/종료/duration, filter decisions (drop 사유), 최종 게재 건수, errors 표. 경로는 `requirements/04-operations.md`에 고정 (Module 2가 의존할 수 있도록). |
| 2. Actions UI 인라인 렌더링 | 필수 | 90일 (GH 자동) | 각 stage가 동일 Markdown 요약을 `$GITHUB_STEP_SUMMARY` 환경 파일에도 append. Actions run 페이지에서 step 로그 옆에 렌더링. 비용: stage당 ~5줄. |
| 3. At-a-glance health | 필수 | 라이브 | `README.md` 상단에 워크플로 status badge. 최소 `bootstrap.yml` + `assemble.yml`. 비용: README에 한 줄 markdown image link × 워크플로 수. |
| 4. Native console | 자동 | 90일 (GH 자동) | GitHub이 자동 보존하는 워크플로 콘솔 로그. Stack trace, full HTTP 디버그 출력 등 raw 정보. 별도 작업 없음. |

**옵션 — 실패 시 푸시 알림 (선택):**
- Discord/Slack incoming-webhook URL을 `NOTIFY_WEBHOOK_URL` GitHub Secret으로.
- 워크플로 마지막에 `if: failure()` 스텝으로 webhook 호출.
- 비용: secret 1개 + 스텝 1개 + ~10줄.
- 페이징이 필요 없으면 생략 (매일 README가 안 바뀌면 그것 자체가 신호).

**채택하지 않은 옵션:** GH Pages 렌더링 사이트, GitHub Issues/Discussions 자동 생성, 외부 로그 aggregator (Loki/Datadog/BetterStack/Honeycomb). 이 규모(하루 24 runs)에서는 과한 surface.

---

## 실행 원칙 (Execution Principles)

복합 작업 전용. 단순 작업은 직접 처리. `pal mcp clink`와 `vibe-check`는 반드시 사용하고, "확신"을 이유로 생략 금지.

- **역할:** 메인 = 계획·위임·추적·승인만. 구현·탐색·리뷰·조사·합성은 전부 위임.
- **위임:** `subagent`는 결과만 필요할 때 (병렬은 동시 호출, 독립 작업은 `run_in_background: true`). `team`은 토론·검증·합의가 필요할 때. 항상 명시: 대상 경로, 질문, 산출물, 종료 조건. **같은 파일 동시 편집 금지.** 여러 결과의 합성도 subagent에 위임.
- **작업 기억:** 메인 스레드에는 결정·리스크·상태만. 긴 출력은 `temp/`에 저장하고 보고는 2문장 + 경로만. subagent 완료 전 선행 진행 금지.
- **vibe-check 필수:** (1) 비가역적 행동 직전, (2) 추정·가정 기반 결정 또는 설계 대안 선택 직전, (3) 3단계 이상 복합 계획 실행 직전.
- **pal mcp clink 필수:** 자기 판단 → `clink` 호출 → 차이 비교 → 합성. 호출 형식은 **`cli_name + default role + prompt + files`**. 시점: 설계/방향 확정 직전, 코드·문서 10줄+ 변경 직후, 동일 문제 해결 실패 시. 긴 맥락은 `temp/` 경로로 전달. 리뷰 목적이면 프롬프트에 반드시 `read/analyze only, do not modify files`. 민감 정보(API 키, `.env`, 자격증명, PII) 전달 금지.
- **최종안 전 2라운드 독립 검증** (독립 보고 → 교차 참조).

---

## Phase 0 — Docs-Plan Alignment (GATE)

- [ ] 0.1 현재 docs 전체 읽기: `docs/architecture/00..08`, `docs/requirements/00..06`. (subagent: `docs-reader`. 산출물: `temp/module1-decisions-phase0-snapshot.md` — 각 파일에서 SDK/로깅 관련 단락만 발췌, 인용 라인 번호 포함.)
- [ ] 0.2 Gap list 작성 — 각 결정별로 (a) 신규 요구, (b) 변경/충돌, (c) 폐기 항목 분류. 최소 다음을 다룬다:
  - **05-llm-and-agent-runtime.md**: "Anthropic Python SDK directly" 결정 → "Claude Agent SDK" 결정으로 뒤집기. `scripts/lib/llm.py` 코드 스니펫과 `messages.create` 패턴은 Agent SDK 패턴(`query()` 또는 `ClaudeSDKClient`)으로 교체.
  - **07-observability.md**: 4-레이어 모델을 명시적 표/섹션으로 추가. 기존 "Sources of observability data" 표를 4-레이어와 정합되게 재구성.
  - **04-operations.md**: `logs/YYYY-MM-DD/<run_id>-summary.md` 경로를 *fixation* (현재는 "directory layout is an architecture concern"으로 deferred — 이 결정으로 requirements 레벨로 끌어올림).
  - **02-github-actions.md**: `$GITHUB_STEP_SUMMARY` 패턴, status badge 패턴, optional webhook 스텝 패턴 추가.
  - **00-system-overview.md** / **requirements/00-overview.md**: 표층 SDK 언급 정합성 확인.
  - **08-security.md** / **05-security-and-governance.md**: `NOTIFY_WEBHOOK_URL` Secret 흐름 + 로그 누설 금지 invariant가 4-레이어 전 채널에 적용되는지 명시.
- [ ] 0.3 Impact assessment — 각 gap별 변경 scope, 영향 문서/시스템, 리스크. 산출물: `temp/module1-decisions-phase0-alignment.md`.
- [ ] 0.4 Planned doc changes 초안 — `temp/module1-decisions-phase0-drafts.md`에 변경 후 단락을 모아둔다 (live docs는 아직 안 건드림).
- [ ] 0.5 **Vibe check**: gaps / contradictions / edge cases / security / ops 체크리스트로 자기 비판 → `temp/module1-decisions-vibecheck-1.md`.
- [ ] 0.6 **PAL clink** (Codex + Gemini): "이 결정 두 개로 인해 docs 정합성에 깨지는 곳이 어디인가? 더 단순한 대안은?" `read/analyze only, do not modify files` 명시. 결과 → `temp/module1-decisions-clink-phase0.md`.
- [ ] 0.7 **GATE**: alignment 문서 + drafts 둘 다 존재해야 Phase 1로 진입.

## Phase 1 — Decision 1 적용 (LLM 하네스 → Claude Agent SDK)

- [ ] 1.1 `docs/architecture/05-llm-and-agent-runtime.md` 재작성 (subagent: `editor-llm-runtime`).
  - "Anthropic Python SDK directly" 결정 단락 → Claude Agent SDK로 교체. **이전 결정을 뒤집는 이유 (multi-round verify를 SDK가 관리 vs 우리가 직접 관리)** 를 DECISION 박스에 명시. 채택하지 않은 옵션 3개 (bare anthropic, claude-code-action, CLI headless)와 각 기각 이유 표로 정리.
  - 다이어그램의 `anthropic.Anthropic(...)` / `client.messages.create(...)` 박스를 Agent SDK 호출 패턴으로 교체.
  - "Per-stage LLM call patterns" 표 — 각 stage가 Agent SDK의 어느 entrypoint를 쓰는지 (subprocess 없는 in-process 호출) 명확화.
  - `scripts/lib/llm.py` 스니펫을 Agent SDK 기반으로 교체. `auth_mode: bearer | api_key`는 그대로 유지 (z.ai vs native Anthropic 분기 필요).
  - "Swapping providers" 체크리스트는 "1-file edit + secret rotate" 불변식을 유지.
- [ ] 1.2 `data/llm.yml` 스키마 단락에 Agent SDK가 추가로 요구하는 필드(있다면)만 보강. 없으면 그대로 유지.
- [ ] 1.3 `docs/architecture/02-github-actions.md`에서 SDK 호출 step 예시가 있다면 정합성 맞춤.
- [ ] 1.4 `docs/architecture/00-system-overview.md` + `docs/requirements/00-overview.md` SDK 언급 정합성 확인.
- [ ] 1.5 **PAL clink** (Codex + Gemini) — 변경된 `05-llm-and-agent-runtime.md` 초안에 대한 read-only 리뷰. 결과 → `temp/module1-decisions-clink-phase1.md`.

## Phase 2 — Decision 2 적용 (로깅 4-레이어 + 옵션)

- [ ] 2.1 `docs/architecture/07-observability.md` 재작성 (subagent: `editor-observability`).
  - "Sources of observability data" 표를 4-레이어 모델로 재구성 (Layer 1 Canonical, Layer 2 Step Summary, Layer 3 Status badge, Layer 4 Native console + 옵션 webhook).
  - 각 layer별 (a) 무엇을 기록하는가 (b) 누가 보는가 (c) 보존 기간 (d) 구현 비용 명시.
  - Layer 2 (`$GITHUB_STEP_SUMMARY`) — stage 스크립트가 동일 Markdown을 append하는 패턴 코드 스니펫.
  - Layer 3 — README badge markdown 예시 (`bootstrap.yml`, `assemble.yml` 최소 2개).
  - Optional webhook — `if: failure()` 스텝 패턴 + `NOTIFY_WEBHOOK_URL` Secret 정의.
  - "What is intentionally NOT logged" 불변식이 Layer 1~4 + webhook 모두에 적용됨을 명시.
- [ ] 2.2 `docs/requirements/04-operations.md` "Logging" 섹션 — 현재 "directory layout is an architecture concern"으로 deferred되어 있는 부분을 **`logs/YYYY-MM-DD/<run_id>-summary.md` 경로 고정**으로 끌어올림. *Why*: Module 2가 이 경로에 의존할 수 있도록.
- [ ] 2.3 `docs/architecture/02-github-actions.md` — `$GITHUB_STEP_SUMMARY` append 패턴, README badge, optional webhook 스텝 워크플로 단편 추가.
- [ ] 2.4 `docs/architecture/08-security.md` + `docs/requirements/05-security-and-governance.md` — `NOTIFY_WEBHOOK_URL` Secret이 `LLM_API_KEY`와 같은 흐름 (env, never logged)을 따른다고 명시. webhook payload에 secret/PII 들어가지 않음을 invariant로 추가.
- [ ] 2.5 **PAL clink** (Codex + Gemini) — 변경된 `07-observability.md` + `04-operations.md` + `02-github-actions.md` 초안에 대한 read-only 리뷰. 결과 → `temp/module1-decisions-clink-phase2.md`.

## Phase 3 — Cross-doc 정합성 sweep

- [ ] 3.1 Repo 전체 grep — `anthropic.Anthropic`, `messages.create`, `Anthropic Python SDK`, `bare SDK`, `Anthropic Agent SDK` 등 표층 언급 위치 전수 조사. (subagent: `consistency-sweeper`. 산출물: `temp/module1-decisions-phase3-grep.md`.)
- [ ] 3.2 발견된 불일치 단락을 모두 Claude Agent SDK / 4-레이어 로깅에 맞춤.
- [ ] 3.3 SSoT-스타일 사실 정의 (있다면) 동기화. 현재 repo는 `ssot/`가 없으므로 N/A 가능 — 0.1 스냅샷에서 확인.

## Phase 4 — 2-Round Independent Verification

- [ ] 4.1 **Round 1 — `verifier-1` (spawned subagent).** 다중 관점 검증 (정합성 + 보안 + ops). 자체 vibe check + PAL clink 포함. 결과 → `temp/module1-decisions-verification-round1.md`.
- [ ] 4.2 **Round 2 — `verifier-2` (spawned subagent).** Round 1과 **다른 관점** (예: red-team / 미래 Module 2 의존자 관점 / 비용 관점). 자체 vibe check + PAL clink 포함. 결과 → `temp/module1-decisions-verification-round2.md`.
- [ ] 4.3 두 라운드의 BLOCKER/HIGH 항목을 모두 drafts에 반영 → 갱신본을 `temp/module1-decisions-phase0-drafts.md`에 덮어쓰고 변경 이력을 파일 상단에 남김.

## Phase F-1 — Docs Sync (GATE)

- [ ] F-1.1 `temp/module1-decisions-phase0-drafts.md` (갱신본)을 live docs에 적용:
  - `docs/architecture/05-llm-and-agent-runtime.md`
  - `docs/architecture/07-observability.md`
  - `docs/requirements/04-operations.md`
  - `docs/architecture/02-github-actions.md`
  - `docs/architecture/08-security.md`
  - `docs/requirements/05-security-and-governance.md`
  - `docs/architecture/00-system-overview.md`, `docs/requirements/00-overview.md` (표층 정합성)
- [ ] F-1.2 Docs-plan consistency 검증 — 모든 live docs가 plan의 "결정 사항" 섹션과 정확히 일치.
- [ ] F-1.3 Cross-plan check — 다른 active plan이 없으므로 N/A. (Phase 0에서 확정.)
- [ ] F-1.4 Staleness check — 이 plan이 2주 이상 dormant였다면 drafts를 live와 재대조.

## Phase F — Commit & Push (GATE)

> NOTE: 이 repo는 현재 git이 초기화되어 있지 않다 (`Is a git repository: false`). Founder가 `git init` + remote 설정을 한 뒤에만 이 phase를 실행. 그 전까지는 F-1.1~F-1.4 적용 후 plan을 `_done/`로 옮기는 것까지만 수행하고, commit/push는 별도 후속.

- [ ] F.1 Frontmatter → `status: done`, `progress` 최종 요약. 파일을 `action-plans/_done/`로 이동.
- [ ] F.2 `git add` — 모든 변경 파일 (docs + plan 파일).
- [ ] F.3 `git commit` — 메시지에 plan 이름 + 완료 phase 명시.
- [ ] F.4 `git push`.

---

## Out of scope (이 plan에서 다루지 않음)

- 실제 `scripts/lib/llm.py` / `scripts/stages/*.py` 구현 (현재 repo에 코드 없음 — 별도 plan).
- `.github/workflows/*.yml` 워크플로 파일 작성 (별도 plan).
- `data/llm.yml` 실제 파일 생성 (별도 plan; 본 plan은 스키마 docs만 손질).
- LLM 백엔드 변경 (GLM-5.1 → 다른 모델). 본 plan은 GLM-5.1 + Z.AI 결정을 docs에 반영할 뿐.
- Module 2 (downstream 소비자) 설계.
