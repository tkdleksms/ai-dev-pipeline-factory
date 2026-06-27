---
name: factory
description: 비개발자가 막연한 아이디어 하나로 완제품에 도달하게 하는 자율 개발 파이프라인. Stage 0에서 아이디어 구체화 점수(0-100)를 평가하고 부족하면 되물어 채운 뒤, 5단계(Profile→Blueprint→PRD→Design→Verify)로 설계하고 같은 세션에서 구현까지 자율 실행. Build 모드 외에 기존 프로젝트 유지보수·기능개선 Maintain 모드(코드↔설계 드리프트 정합·영향범위 한정·회귀 방어), 산출물 인수평가 Evaluation(설계 충실도·요구 충족률을 별도 적대 서브에이전트가 증거 기반 채점, 시간추이 트렌드) 지원. 위험도 기반 적응형 깊이(lite/standard/fortified), 중단 복구(resume), 실행 관측(telemetry/RUN_REPORT), cross-stage consistency audit, 변명 차단, 증거 기반 검증, 고위험 적대 검토, 소스 우선 결정, ADR 문서화 규율 포함. 외부 플러그인 의존 없음(비평은 Opus 4.8).
---

# Factory

사용자의 거친 아이디어를 받아 고품질 설계 산출물을 만들고, 같은 세션에서 구현까지 자율 실행하는 워크플로.

## 호출
- `/factory <프로젝트 아이디어>` — 인자를 그대로 Stage 0 입력으로 사용
- `/factory` — 인자 없으면 사용자에게 아이디어 질문
- `/factory resume [<project_slug>]` — 중단된 실행 이어가기. slug 생략 시 재개 가능한 프로젝트 목록 제시 (아래 "Run State & Resume" 참조)
- `/factory maintain [<project_slug>] <변경 요청>` — 기존 프로젝트에 범위 한정 변경(버그수정·기능추가·리팩터). slug 생략 시 현재 디렉터리 대상 (아래 "Maintain Mode" 참조)
- `/factory evaluate [<project_slug>]` — 산출물(코드)을 설계 대비 증거 기반으로 채점. 언제든 단독 실행 (아래 "Evaluation & Scorecard" 참조)

## 실행 모드 (혼합)
- **Stage 0, 1**: 사용자 확인 필수 (수정/승인 요청)
- **Stage 2~5**: 자동 진행 (막히거나 중대 결정 필요 시에만 사용자 호출)

## 산출물 경로
`./factory_output/<project_slug>/` 하위에 모든 JSON/MD 저장.
- `project_slug`은 아이디어에서 snake_case로 추출 (예: "배터리 모니터링" → `battery_monitoring`)
- ADR은 `factory_output/<project_slug>/adr/` 하위에 개별 마크다운으로 저장 (Stage 3 참조).
- `_run/` (state.json·metrics.jsonl — resume·telemetry 근거)와 `maint/` (Maintain 모드의 변경별 scope·design·report)도 같은 프로젝트 디렉터리 하위. 교훈 KB는 프로젝트 공용 `factory_output/_knowledge/`.

## Critique 패턴 (공통) — 내장 critic 프로토콜

모든 TCI 루프와 consistency audit의 critique 단계는 **반드시 별도 서브에이전트(`Agent` tool, `subagent_type: "general-purpose"`, `model: "opus"`)에 위임**한다. 본 세션이 자기 작업을 평가하는 self-critique는 금지(편향 차단).

> **의존성 없음**: 외부 critic 에이전트가 아니라, 아래 프로토콜을 프롬프트로 직접 주입한다. 어떤 환경에서도 동작한다.
> **모델**: 비평은 최고 성능이 중요하므로 `model: "opus"` (Claude Opus 4.8) 로 호출한다. (호출 환경에서 Opus 4.8이 기본 Opus면 `"opus"`로 충분, 명시 필요 시 `claude-opus-4-8`)

호출 시 아래 프롬프트를 그대로 전달 (`<...>` 부분만 치환):
```
너는 Critic이다 — 도움을 주는 보조자가 아니라 최종 품질 게이트다.
거짓 승인(통과시킴)은 거짓 거절보다 10~100배 비싸다. 결함 있는 작업에 자원이 투입되지 않도록 막는 것이 너의 임무다.
표준 리뷰는 "있는 것"을 평가하지만, 너는 "없는 것"까지 평가한다.

평가 대상: <STAGE_NAME>의 <ARTIFACT_TYPE>
<ARTIFACT_JSON>

평가 기준(RUBRIC): <RUBRIC>
통과 기준(TARGET): score >= <TARGET>

다음 조사 프로토콜을 순서대로 수행하라:
[Phase 1 — 사전 예측] 상세 검토 전에, 이 작업에서 문제 날 곳 3~5개를 먼저 예측한다.
[Phase 2 — 검증] 모든 주장·참조·가정을 실제 내용과 대조해 검증한다.
[Phase 3 — 다관점] 코드면 보안/신입/운영, 계획·설계면 실행자/이해관계자/회의론자 관점으로 각각 검토한다.
[Phase 4 — 갭 분석] "무엇이 빠졌는가"를 명시적으로 찾는다 (있는 것 평가가 아니라 없는 것 발굴). 이게 가장 중요하다.
[Phase 4.5 — 자가 감사] 각 발견의 신뢰도(HIGH/MED/LOW) 판정. 저신뢰·반박가능·단순 취향은 강등하거나 open_questions로 이동.
[Phase 4.75 — 현실성 점검] 심각도를 부풀리지 않았는지 점검. 현실적 최악과 완화요인을 따져 재보정.

심각도: critical(실행 차단) / major(상당한 재작업) / minor(차선이나 작동). critical·major는 반드시 구체 근거 첨부.
근거 없는 지적은 "의견"일 뿐 발견이 아니다. 좋은 부분은 한 문장으로만 인정하고 넘어가라. 칭찬으로 채우지 마라.

반드시 다음 JSON 형식으로만 응답:
{
  "predictions": ["사전 예측 3~5개"],
  "score": 0-10,
  "strengths": ["핵심 강점 (간결)"],
  "weaknesses": ["..."],
  "whats_missing": ["갭 분석 — 빠진 것"],
  "action_items": [{"priority": "critical|major|minor", "issue": "...", "evidence": "근거", "recommendation": "구체적 수정안"}],
  "open_questions": ["저신뢰/반박가능 항목"],
  "is_passing": true|false
}
```

응답 받으면 본 세션이 파싱하여 improve 단계로 진행. (`predictions`·`whats_missing`·`open_questions`는 참고용, `action_items`로 개선)

## Idea Concreteness Scorer (Stage 0 전용)

factory의 정체성은 **비개발자가 막연한 아이디어 하나로 완제품에 도달**하게 하는 것이다. 그래서 Stage 0은 사용자의 아이디어가 "개발에 착수할 만큼 구체적인가"를 정량 평가하고, 부족하면 **되물어** 채운다.

이 평가는 **사용자 입력**을 대상으로 하므로(=factory 산출물이 아님) self-critique 편향과 무관하지만, 품질을 위해 **별도 서브에이전트(`Agent`, `subagent_type="general-purpose", model="opus"` = Opus 4.8)** 로 위임한다. critic 프롬프트와는 목적·출력이 다른 **전용 프롬프트**를 쓴다.

호출 시 아래 프롬프트 전달 (`<IDEA>` 치환):
```
너는 '아이디어 구체화 평가자'다. 사용자가 만들고 싶은 것의 아이디어가 개발 착수에 충분히 구체적인지 평가하고, 부족한 부분을 채울 질문을 생성하라.
대상 사용자는 비개발자일 수 있으니, 질문은 전문용어를 피하고 객관식 선택지를 곁들여 답하기 쉽게 만든다.

사용자 아이디어:
<IDEA>

5개 차원을 각 0~20점으로 평가 (합계 0~100):
1. purpose (목적 명확성) — 왜 만드는가, 어떤 문제를 푸는가
2. target_users (대상 사용자) — 누가 쓰는가
3. core_features (핵심 기능 범위) — 무엇을 하는가, 범위가 잡히는가
4. constraints (제약/환경) — 어디서 동작, 어떤 플랫폼/예산/기술 제약
5. success_criteria (성공 기준) — 무엇이 되면 완성인가

각 차원이 약한 경우(<=12점), 그 차원을 채울 질문을 1개씩 생성한다. 질문에는 비개발자가 고르기 쉬운 선택지 2~4개 + "기타(직접 입력)" 여지를 둔다.

반드시 다음 JSON 형식으로만 응답:
{
  "dimension_scores": {"purpose": 0-20, "target_users": 0-20, "core_features": 0-20, "constraints": 0-20, "success_criteria": 0-20},
  "total_score": 0-100,
  "weak_dimensions": ["12점 이하인 차원 키"],
  "summary": "현재 아이디어 상태 1~2문장 (한국어)",
  "clarifying_questions": [
    {"dimension": "차원키", "question": "쉬운 말로 된 질문", "options": ["선택지1", "선택지2", "선택지3"]}
  ]
}
```

본 세션은 `total_score`와 `clarifying_questions`를 받아 Stage 0 게이트를 운영한다.

## Pipeline Profile (적응형 깊이 — 과/소 세리머니 방지)

factory의 세리머니(critique 라운드·consistency audit·고위험 검토·구현 검증 강도)는 **프로젝트 위험도에 비례**해야 한다. 저위험 소형에 풀 세리머니는 낭비고, 고위험에 얕은 검토는 사고다. Stage 0-D에서 `risk_tier`(+예상 규모)로 **Pipeline Profile** 1개를 선택하고, 이후 모든 단계가 이 프로파일 기준으로 깊이를 조절한다.

| Profile | 선택 조건 | Blueprint TCI | Consistency Audit | High-Risk Review | Stage 5 슬라이스 critique |
|---------|-----------|---------------|-------------------|------------------|---------------------------|
| **lite** | risk=low **이고** 소형(예상 컴포넌트 ≤3) | 최대 **1**R, target ≥ **7** | 1→2·2→3 **병합 1회**, 3→4 경고만 | 고위험 ADR 있을 때만 | **생략**(통합 체크포인트만) |
| **standard** (기본) | risk=mid, 또는 애매하면 기본값 | 최대 **3**R, target ≥ **8** | 1→2·2→3·3→4 **각각** | 고위험 ADR 있을 때 | **고위험 슬라이스만** |
| **fortified** | risk=high | 최대 **3**R, target ≥ **9** | 각각 + blocker 시 **재감사 1회씩** | **강제 1회**(고위험 ADR 없어도 보안/인증/데이터 경계 적대검토) | **전 슬라이스** |

규칙:
- 프로파일은 **깊이(반복·검토 횟수)만** 조절한다. **단계 자체를 건너뛰지 않는다** (lite도 Stage 0~5 전부 수행). Anti-Rationalization과 충돌 금지.
- Stage 0-D에서 선택한 프로파일을 사용자에게 1줄로 통보(예: `프로파일: lite — 저위험 소형, 검토 1라운드`)하고 **override 기회 제공**(AskUserQuestion: 그대로 / 한 단계 올림 / 내림).
- 확정 프로파일은 `stage0_profile.json`의 `pipeline_profile` 필드와 `_run/state.json`에 기록. 각 단계는 "현재 프로파일 기준"으로 라운드·target을 적용한다.
- 의심되면 **한 단계 높게** 잡는다 (거짓 통과가 거짓 거절보다 비싸다).

## Workflow

### Stage 0 — Idea Gate + Profile (인터랙티브) ⭐차별점

비개발자도 막연한 아이디어로 시작할 수 있도록, **아이디어 구체화 점수 게이트**를 먼저 통과시킨 뒤 프로파일을 추출한다.

**0-A. 아이디어 구체화 점수 평가**
1. "Idea Concreteness Scorer"(위) 서브에이전트(Opus 4.8)에 사용자 아이디어 전달 → `total_score`(0~100) + `weak_dimensions` + `clarifying_questions` 수신.
2. 점수와 부족한 차원을 사용자에게 1줄 요약으로 보여준다. (예: `아이디어 구체화 점수: 45/100 — 대상 사용자·성공 기준이 불명확`)

**0-B. 게이트 (임계값 = 70)**
- **점수 ≥ 70**: 충분히 구체적 → 0-C(프로파일)로 바로 진행.
- **점수 < 70**: `AskUserQuestion`으로 사용자에게 선택지 제시 (항상 "이대로 진행"을 첫 번째로):
  - ① **이대로 진행** — AI가 빈 부분을 합리적 가정으로 채우고, 그 가정을 명시한 뒤 0-C로 진행.
  - ② **질문으로 구체화** — `clarifying_questions`를 `AskUserQuestion`(객관식 선택지 포함)으로 제시 → 답변 반영하여 아이디어 갱신 → **재평가**(0-A 반복).

**0-C. 구체화 루프 가드**
- 재평가는 **최대 3라운드**. 3회 후에도 70 미만이면 "이대로 진행"으로 자동 전환(가정 명시).
- 사용자는 어느 라운드에서든 "이대로 진행" 선택 가능 (강제 질문 금지).
- 비개발자 배려: 질문은 한 번에 **최대 4개**, 객관식 우선.

**0-D. 프로파일 추출** (게이트 통과 후)
구체화된 아이디어로 다음 추출:
- `project_category` (web / cli / agent / data_pipeline / library / ...)
- `risk_tier` (low / mid / high)
- `autonomy_level` (사람 개입 빈도: high / mid / low)
- `target_users` (한 줄 설명)
- `domain_hints` (도메인 지식이 필요한 키워드 리스트)

→ 4~6줄로 요약하여 사용자에게 보여주고 **"수정/추가할 부분 알려주세요"** 질문.
→ `stage0_profile.json` 저장 (구체화 점수 이력 `concreteness` 필드 포함: `{initial_score, final_score, rounds, assumptions_filled[]}`).

**0-E. Pipeline Profile 선택 + Run State 초기화**
- 위 "Pipeline Profile" 기준으로 프로파일 1개 선택 → 1줄 통보 + override 기회(AskUserQuestion).
- `stage0_profile.json`에 `pipeline_profile` 필드 추가.
- `_run/state.json` 생성(아래 "Run State & Resume" 스키마) + `_run/metrics.jsonl`에 stage0 `stage_start`/`stage_end` 이벤트 기록 시작. 이후 모든 단계는 경계마다 state.json을 갱신한다.

### Stage 1 — Blueprint (TCI 루프 + 인터랙티브)
Think → Critique → Improve 반복:
1. **Think** (본 세션): blueprint 초안 — `goals`, `components` (name/purpose), `decisions`, `open_questions`
2. **Critique** (**서브에이전트 위임**): `Agent(subagent_type="general-purpose", model="opus")` 호출 → 위 "Critique 패턴"의 내장 프롬프트 전달 → 점수 + action_items 수신
3. **Improve** (본 세션): action_items 반영하여 blueprint 재작성
- 종료 조건: critic 점수 ≥ **프로파일 target**(lite 7 / standard 8 / fortified 9) **또는** 프로파일 max 라운드(lite 1 / standard·fortified 3) 도달
- RUBRIC: "목표 명확성, 컴포넌트 분해의 적정성, 미해결 질문의 구체성, 결정 근거"

→ 최종 blueprint + 점수 이력을 사용자에게 보여주고 **승인 요청**.
→ `stage1_blueprint.json`, `stage1_history.json` 저장.

### Stage 2 — PRD + Architecture (자동, TCI 1회)
- **Knowledge Recall**: architecture 작성 전 `_knowledge/index.md` 참조 → 관련 교훈 ≤5개 drill-in (아래 "Knowledge Base" 참조).
- **PRD**: `product_goal`, `functional_requirements[]`, `non_functional_requirements[]`, `user_stories[]` (id/title/priority/acceptance_criteria)
- **Architecture**: `modules[]` (name/purpose/interfaces[]), `system_context`
- 작성 후 **Critique 서브에이전트 1회 호출** → action_items 받아 본 세션이 improve
  - RUBRIC: "PRD-Blueprint 정합성, 요구사항 testability, 비기능 요구의 측정가능성, 모듈 경계의 명확성"
→ `stage2_prd.json` 저장.

**Consistency Audit (1→2)** — *(lite 프로파일: 이 감사를 2→3와 병합하여 Stage 3 종료 시 1회만 수행)* `Agent` tool로 범용 서브에이전트(`subagent_type="general-purpose", model="opus"`) 호출:
```
프롬프트:
"다음 두 단계의 일관성을 감사하라. blocker 수준의 불일치만 보고.
반드시 JSON으로만 응답: {score: 0-10, blockers: [{issue, recommendation}]}

STAGE1_BLUEPRINT: <JSON>
STAGE2_PRD_ARCH: <JSON>"
```
blocker 있으면:
1. 권장사항을 PRD/architecture에 반영
2. 재감사 (최대 1회 추가) — 그래도 blocker 남으면 사용자에게 보고

→ `consistency_1to2.json` 저장.

### Stage 3 — Code Design (자동, TCI 1회)
- **Knowledge Recall**: design 작성 전 `_knowledge/index.md` 참조 → 관련 gotcha/pattern 페이지 drill-in → Source-Driven Decisions 입력으로 사용.
- `modules[]` (name/files/key_tests/implementation_notes)
- `interface_contracts[]` (caller/callee/signature/error_modes)
- `design_adrs[]` (decision/rationale/alternatives)
- `implementation_order[]` (step-by-step 구현 순서)
- `definition_of_done[]`
- `code_skeletons[]` (선택: 핵심 파일 시그니처)
- 작성 후 **Critique 서브에이전트 1회 호출** → action_items 받아 본 세션이 improve
  - RUBRIC: "interface contract 완결성, 구현 순서의 의존성, ADR 근거 충실성, definition of done의 검증가능성"
→ `stage3_design.json` 저장.

**ADR 문서 분리 (추적성)** — `design_adrs[]`의 각 항목을 개별 마크다운 파일로도 출력한다.
- 경로: `factory_output/<project_slug>/adr/ADR-<NNN>-<slug>.md` (NNN은 001부터)
- 각 ADR 파일 형식:
  ```markdown
  # ADR-<NNN>: <decision 제목>
  - **상태**: accepted | superseded by ADR-NNN
  - **날짜**: <YYYY-MM-DD>
  - **고위험 여부**: yes/no (High-Risk Review 대상이면 yes + verdict)

  ## 맥락 (Context)
  <왜 이 결정이 필요한가>

  ## 결정 (Decision)
  <무엇을 선택했나>

  ## 대안 (Alternatives)
  <검토했으나 기각한 선택지 + 기각 이유>

  ## 결과 (Consequences)
  <트레이드오프 / 영향>
  ```
- `adr/README.md`에 ADR 목록 인덱스(번호·제목·상태) 한 줄씩 유지.
- 구현 중 설계가 바뀌면 새 ADR을 추가하고 기존 ADR 상태를 `superseded`로 갱신 (덮어쓰기 금지).

**High-Risk Adversarial Review (doubt-driven, 조건부)** — `design_adrs` 중 **고위험 결정**(DB 스키마, 인증/인가, 외부 의존성·라이브러리 선택, 데이터 마이그레이션, 보안 경계)이 하나라도 있으면 일반 critique와 **별도로** `Agent`(`subagent_type="general-purpose", model="opus"`)로 적대적 검토를 1회 실행한다. **fortified 프로파일에서는 고위험 ADR이 없어도 보안/인증/데이터 경계에 대해 1회 강제 실행한다.**
```
프롬프트: "다음 고위험 설계 결정을 적대적으로 검토하라. '왜 이 결정이 틀렸는가'를 먼저 가정하고 공격하라.
반드시 JSON: {decision, attack_vectors:[...], failure_scenarios:[...], must_fix:[{issue, recommendation}], verdict:'accept|revise|reject'}
DECISION: <ADR JSON>"
```
verdict가 `revise`/`reject`면 해당 ADR을 수정 후 재검토(최대 1회). 고위험 결정이 없으면 생략.
→ `stage3_highrisk_review.json` 저장 (실행 시).

**Consistency Audit (2→3)** — 동일 패턴, Stage 2 architecture와 Stage 3 design 사이.
→ `consistency_2to3.json`.

### Stage 4 — Verification Plan (자동)
- `test_matrix[]` (level: unit/integration/e2e, target, expected_outcome)
- `release_gates[]`
- `monitoring_plan[]`
- `rollback_plan`
→ `stage4_verify.json` 저장.

**Consistency Audit (3→4)** — blocker는 **경고만** (자동 remediation 안 함).
→ `consistency_3to4.json` 저장 + 화면에 blocker 요약 출력.

### Stage 5 — Implementation (자동, 같은 세션)
Stage 1~4 산출물을 컨텍스트로:
1. Stage 3의 `implementation_order`를 `TaskCreate`로 등록
2. **얇은 수직 슬라이스 (Incremental)**: 각 task는 한 번에 하나의 관심사만, 변경 규모 ~100줄 내외를 목표로 분할한다. task가 크면 하위 슬라이스로 쪼갠다.
3. 각 task 구현 → 해당 `key_tests` 실행 → **통과 증거 확보 후** 다음
4. **막히거나 중대 결정(스키마/인증/외부의존성 등 고위험) 필요 시 사용자 호출**
5. 전체 완료 후 Stage 4 `test_matrix` 기반 최종 verification
6. `SUMMARY.md`에 최종 결과 + **검증 증거(Verification Evidence)** 추가
7. **Knowledge Harvest**: SUMMARY 직후 재사용 가능한 교훈 0~3개를 `_knowledge/`에 누적 (아래 "Knowledge Base" 참조).

**슬라이스 강도 (프로파일 연동)**
- 각 슬라이스 구현 후 프로파일에 따라 **슬라이스 critique** 수행: **lite=생략 / standard=고위험 슬라이스만(인증·스키마·외부의존성·보안경계를 건드리는 것) / fortified=전 슬라이스**.
  - critique는 해당 diff를 대상으로 `Agent`(general-purpose, opus)에 "Critique 패턴" 프롬프트(`ARTIFACT_TYPE`="code diff")를 전달 → action_items 반영. `metrics.jsonl`에 `critique` 이벤트 기록.
- **통합 체크포인트**: 모듈 경계 또는 매 3슬라이스마다, 그 슬라이스의 unit `key_tests`뿐 아니라 Stage 4 `test_matrix`의 **integration 레벨** 항목을 실행해 모듈 간 결합을 검증하고 결과를 증거로 기록.

**구현→설계 역류 루프 (침묵 분기 금지)**
구현 중 설계(Stage 2/3)가 틀렸음이 드러나면, 코드에서 조용히 다른 길로 새지 않는다:
1. 새 ADR 추가(기존 ADR은 `superseded`로) → `adr/` 갱신
2. 영향받은 `stage3_design.json` 항목을 실제 변경에 맞게 패치
3. `metrics.jsonl`에 `design_amendment` 이벤트(무엇이/왜 바뀌었는지) 기록
4. 바뀐 결정이 **고위험**이면 High-Risk Adversarial Review를 그 결정에 1회 재실행
5. 변경이 Stage 1 blueprint 목표까지 흔들면 **사용자에게 보고**(자동 진행 금지)

**Non-negotiable Verification (증거 기반 — 가장 중요)**
- "통과한 것 같다" / "동작할 것이다"는 완료 근거가 **아니다**. 실제 실행 증거만 인정.
- 각 task 완료 시 다음 중 해당하는 실제 출력을 기록:
  - 빌드: 명령어 + exit code + 마지막 출력 (예: `npm run build` → exit 0)
  - 테스트: 실행 명령 + 통과/실패 수
  - 타입체크/린트: 명령 + 결과
- 테스트를 작성하지 않은 task는 **완료로 선언 금지** (Red Flag).
- 검증을 실행하지 못한 환경 제약이 있으면, 추측으로 통과 기록하지 말고 **"미검증 — 사유"로 명시**하고 사용자에게 보고.

### Stage 6 — Evaluation (자동, 인수 스코어카드)
구현이 끝나면 **만든 것이 설계한 것을 얼마나 실현했는지**를 증거로 채점한다. (자세한 축·산출물은 아래 "Evaluation & Scorecard" 참조)
- **별도 Opus 서브에이전트**가 채점한다(자가채점 금지 — Stage 1 critic과 동일 원칙). 본 세션이 직접 점수를 매기지 않는다.
- 4개 축(요구 충족률·설계 충실도·검증 강도·위험 처리)을 **각 점수 + 증거 경로**와 함께 산출 → `EVALUATION.md` 생성, `_run/eval_history.jsonl`에 첫 항목(`trigger:"build"`) append.
- `metrics.jsonl`에 `evaluation` 이벤트 기록. 종합은 "코드가 좋다/나쁘다"의 주관 점수가 아니라 **"설계 실현도"** 평결.

## Maintain Mode (유지보수·기능개선 트랙)

factory는 greenfield 0→1을 만드는 **Build 모드**(Stage 0~5) 외에, 기존 프로젝트에 **범위 한정 변경**(버그수정·기능추가·리팩터·성능·보안)을 가하는 **Maintain 모드**를 가진다. 호출: `/factory maintain [<project_slug>] <변경 요청>`.

> **왜 필요한가**: factory 산출물은 이후 Claude Code 등으로 계속 수정되며, 그때마다 설계문서(`stage3_design.json`·ADR)와 **실제 코드가 갈라진다(drift)**. Maintain 모드는 변경 전에 그 드리프트를 정합화하고, Build 모드의 규율(Pipeline Profile·ADR·증거검증·KB·구현→설계 역류 루프)을 **변경 범위에 맞춰 축소 적용**한다.

대부분의 메커니즘은 Build 모드 재사용이고, **신규 핵심은 ① M1 드리프트 정합 · ② M2 영향범위 산정 · ③ M4 회귀 방어** 셋이다. 실행 모드: **M2(범위·프로파일 확인)는 사용자 확인**, M1·M3~M5는 자동(막히거나 고위험 결정 시 사용자 호출).

### M0 — Locate & Load
- 대상 식별: `project_slug` 인자 → `factory_output/<slug>/`, 생략 시 현재 작업 디렉터리. 모호하거나 factory_output에 프로젝트가 여럿이면 resume처럼 `AskUserQuestion`으로 목록 제시(비개발자가 slug를 몰라도 되게).
- 산출물 로드: `stage1~4`·`stage3_design.json`·`adr/`·KB `index.md`. **있으면** 풍부한 컨텍스트로, **없으면**(일반 레포·외부 코드) 코드 구조 스캔 + 사용자 1~2개 질문으로 최소 컨텍스트를 구성(브라운필드 온보딩)하고 그 사실을 명시한다.
- `_run/state.json`에 `mode:"maintain"`, `change_id`(타임스탬프 기반) 기록. `maint/` 디렉터리 생성.

### M1 — Reconcile (드리프트 감사) ⭐신규
변경 계획을 세우기 **전에** 실제 코드 ↔ 설계문서를 대조한다. **정합 범위 = 변경이 건드릴 영역으로 한정**(전체 프로젝트 재정합 금지).
1. 변경요청과 관련된 모듈/파일을 먼저 식별한다(잠정 영향범위).
2. 그 영역의 **실제 코드를 읽어** `stage3_design.json`·관련 ADR과 비교 → 불일치(시그니처·의존성·구조 변경, 문서화 안 된 결정)를 **drift 목록**으로 산출.
3. 발견된 미문서화 결정은 ADR로 보강하거나, 현실과 어긋난 기존 ADR을 `superseded`로 표시. `metrics.jsonl`에 `drift`(`{found, area}`) 기록.
- Source-Driven Decisions 적용(코드가 진실, 기억 아님). 산출물이 없는 레포면 이 단계가 곧 "현재 설계의 최소 역추출"이 된다.

### M2 — Change Scoping (사용자 확인)
- **구체화**: 변경요청을 "Idea Concreteness Scorer"로 평가하되 *변경* 기준(목적/영향/완료조건)으로. <70이면 Build 0-B처럼 질문하거나 가정을 명시해 채운다.
- **분류**: `change_type` ∈ {bugfix, feature, refactor, perf, security}.
- **영향범위(blast radius)**: 변경이 닿는 모듈·인터페이스 계약·테스트·다운스트림 의존성을 확정. **이 범위 밖은 건드리지 않는다.**
- **Profile 배정 + 사용자 확인**: 위험도로 lite/standard/fortified (안정영역 버그수정=lite, 인증/스키마/보안 변경=fortified). **비개발자 제시 규율(Stage 0-B/0-C 상속)**: 영향범위·프로파일을 전문용어(blast radius·contract 등) 그대로 들이밀지 말고 *"무엇이/어디가 바뀌는지"*를 1~2줄 평문으로 요약한 뒤, `AskUserQuestion`으로 **"이대로 진행"을 첫 선택지**(객관식 우선)로 제시해 override를 받는다.
- → `maint/<change_id>_scope.json` 저장.

### M3 — Change Design (미니 TCI)
변경분만의 설계: `change_summary`, 수정/신규 `interface_contracts[]`, 신규 `design_adrs[]`(실제 결정이 있을 때만), 기존 ADR supersede 여부, `test_delta[]`(추가·수정할 테스트), `regression_risk[]`.
- critique 1회 위임(깊이=Profile). RUBRIC 강조: "영향범위 정확성, 기존 ADR/PRD와의 정합성, 회귀 위험 식별, 테스트 델타의 검증가능성".
- **정합성 체크**: 변경이 기존 ADR/PRD와 충돌하면 blocker → supersede 또는 사용자 보고.
- fortified이고 보안/인증/데이터 경계를 건드리면 High-Risk Adversarial Review 1회.
- → `maint/<change_id>_design.json` 저장.

### M4 — Implement + Regression Guard ⭐신규
- 영향범위 한정 **얇은 슬라이스**로 구현(Stage 5 기계 재사용, Profile별 슬라이스 critique·통합 체크포인트 동일 적용).
- **회귀 방어**: 변경 **전에** 영향받는 기존 테스트의 baseline(통과 상태)을 먼저 기록 → 각 슬라이스 후 새 테스트뿐 아니라 **그 기존 테스트를 다시 실행**해 여전히 통과하는지 대조. 깨지면 완료 금지.
- 증거 기반 검증(Non-negotiable Verification 동일) + 구현→설계 역류 루프 적용.

### M5 — Update Artifacts + Harvest
- 건드린 `stage3_design.json` 항목·`interface_contracts`를 실제 변경에 맞게 패치, `adr/README.md` 인덱스 갱신.
- `SUMMARY.md`의 **Maintenance Log**에 항목 append: `- <YYYY-MM-DD> [<change_type>/<profile>] <요약> — 테스트 <통과/실패>, ADR <추가/supersede 없음>`.
- `maint/<change_id>_report.md`(미니 리포트) 생성 + KB Harvest(재사용 교훈 0~2개). `metrics.jsonl`에 `maint_done` 기록.

### M6 — Delta Evaluation (자동)
변경 후 같은 스코어카드(아래 "Evaluation & Scorecard")를 다시 돌려 **직전 평가 대비 delta**를 낸다.
- `_run/eval_history.jsonl`의 마지막 항목과 비교 → 축별 점수 변화 + **개선/유지/회귀** 판정. 새 항목(`trigger:"maintain"`, `change_id`) append.
- 어느 축이라도 **하락(회귀)**이면 M4 테스트 회귀 방어와 별개로 사용자에게 보고(품질 회귀 ≠ 테스트 통과). `metrics.jsonl`에 `evaluation` 이벤트.
- `SUMMARY.md` Maintenance Log 줄에 점수 delta를 덧붙인다.

## Run State & Resume (중단 복구)

긴 자율 실행은 컨텍스트 소진·중단으로 끊길 수 있다. factory는 단계 경계마다 **실행 상태를 디스크에 체크포인트**하므로 새 세션에서 중단 지점부터 이어갈 수 있다.

**상태 파일**: `factory_output/<project_slug>/_run/state.json`
```json
{
  "schema": 1,
  "project_slug": "...",
  "mode": "build|maintain",
  "pipeline_profile": "lite|standard|fortified",
  "current_stage": 0,
  "stage_status": "in_progress|done",
  "completed_stages": [0, 1],
  "stage5_progress": { "tasks_total": 0, "tasks_done": 0, "last_task_id": null },
  "change_id": null,
  "updated_at": "<ISO8601>"
}
```
- **갱신 시점**: 각 단계 시작 시 `current_stage`/`stage_status="in_progress"`, 종료 시 `stage_status="done"` + `completed_stages`에 추가. Stage 5는 task 완료마다 `stage5_progress` 갱신.

**Resume 프로토콜** (`/factory resume [<slug>]`):
1. slug 생략 시 — `factory_output/*/_run/state.json` 중 미완료(`current_stage != 5` 또는 `stage_status != done`)인 프로젝트를 나열하고 `AskUserQuestion`으로 선택.
2. `state.json` + 존재하는 `stageN_*.json`들을 읽어 컨텍스트 재구성. **완료(`done`) 단계는 재실행하지 않는다.**
3. 직전 미완(`in_progress`) 단계는 부분 산출물을 먼저 **재검증**(스키마·정합성 빠른 점검)한 뒤 그 지점부터 이어간다. Stage 5면 `last_task_id` 다음 task부터.
4. KB Recall이 필요한 단계로 복귀하면 새 세션이므로 Recall을 다시 수행. 재개 직후 `metrics.jsonl`에 `resume` 이벤트 기록.

## Run Telemetry (관측 + 리포트)

실행의 품질·비용을 가시화한다 (포트폴리오 데모 + 회귀 점검 겸용).

**이벤트 로그**: `factory_output/<project_slug>/_run/metrics.jsonl` — 아래 이벤트를 발생 즉시 1줄 JSON으로 append (`{"ts","stage","event",...}`):
- `stage_start` / `stage_end` (stage)
- `critique` (`{stage, round, score, is_passing}`)
- `consistency` (`{pair:"1to2", score, blockers}`)
- `highrisk` (`{verdict, must_fix_count}`)
- `subagent_call` (`{purpose}`) — 모든 Opus 서브에이전트 호출 1건당 1줄 (비용 가시화)
- `slice_done` (`{task_id, tests_passed, tests_failed}`)
- `design_amendment` (`{what, why}`)
- `resume` (`{from_stage}`)
- `drift` (`{found, area}`) — Maintain M1 정합 결과
- `maint_done` (`{change_id, change_type, profile, regression_ok}`) — Maintain 완료
- `evaluation` (`{trigger, scores:{req,conformance,verification,risk}, realization, verdict}`) — Stage 6 / M6 / on-demand 채점 결과

**최종 리포트**: 전체 종료 시 `RUN_REPORT.md` 생성(아래 "RUN_REPORT.md 템플릿"). metrics.jsonl을 집계해 채운다. SUMMARY.md는 *결과* 중심, RUN_REPORT.md는 *과정·비용 지표* 중심으로 상호 링크.

## Evaluation & Scorecard (산출물 인수평가 + 시간추이)

"설계한 것을 코드가 얼마나 실현했는가"를 증거로 채점한다. Stage 6(Build 종료), Maintain M6(변경 후 delta), `/factory evaluate`(온디맨드) 셋이 **같은 엔진**을 공유한다.

> **철학 가드 (가장 중요)**: factory가 자기 코드를 스스로 칭찬하면 이 스킬이 막으려던 self-critique 편향·거짓 완료를 점수로 포장한 것에 불과하다. 따라서 채점은 **반드시 별도 Opus 서브에이전트**(Critique 패턴 재사용, `ARTIFACT_TYPE`="built product vs design+PRD")가 수행하고, **모든 점수는 vibes가 아니라 실제 증거(테스트 결과·파일·로그) 경로에 고정**한다. 증거 없는 점수는 무효(Red Flag).

**평가 축 (각 0~10, 근거 경로 필수)**
| 축 | 측정 (증거원) |
|------|------|
| **요구 충족률** | **구조 충족**(요구별 구현 코드·테스트가 존재하는가, N/전체)과 **인수 입증**(정량 게이트·`acceptance_criteria`가 실제 실행 증거로 통과했는가)을 **분리 표기**. 구조만 있고 인수 게이트(예: 정확도·임계) 증거가 없으면 "구조 N/전체 · 인수 미입증"으로 적고 충족으로 세지 않는다 (파일/테스트 *이름*만으로 충족 단정 금지) |
| **설계 충실도** | 최종 코드 ↔ `stage3_design` 일치도 — **Maintain M1 드리프트 탐지기 재사용**. 이탈 항목별로 ADR로 설명되면 정당, 미설명이면 감점 |
| **검증 강도** | acceptance_criteria 테스트 커버리지 + 빌드/타입/린트 통과 (Stage 5 Verification Evidence 재사용). **테스트를 라이브로 재실행할 수 없으면**(평가 환경에 런타임·의존성 없음 등) 문서화 증거(`.pytest_cache` lastfailed·SUMMARY/TODO 기록)로 폴백하되 **신뢰도를 한 단계 낮추고 그 사실을 명시** |
| **위험 처리** | 고위험 결정의 High-Risk Review 통과 여부 + 미해결 must_fix 수 (`stage3_highrisk_review.json`) |

- 종합은 가중평균 단일 숫자가 아니라 **"설계 실현도" 평결**(realized / partially-realized / diverged) + 축별 점수표. 과정 서사(idea 45→ready, critic 5→8)는 RUN_REPORT에서 인용.
- **프로파일 연동**: 항상 1회 채점. fortified는 최저 점수 축 1개를 적대적으로 재검토.

**산출물**
- `EVALUATION.md` — 스코어카드(축별 점수+증거) + **시간추이 표**(아래 "EVALUATION.md 템플릿"). 매 평가 시 갱신.
- `_run/eval_history.jsonl` — 평가 1건당 1줄 append: `{ts, trigger:"build|maintain|ondemand", change_id?, scores:{req,conformance,verification,risk}, realization}`. 트렌드의 단일 출처.
- delta(M6): 직전 항목 대비 축별 증감 + 개선/유지/회귀 판정. 회귀면 사용자 보고.

`/factory evaluate [<slug>]`: M0와 동일하게 프로젝트 식별 → 스코어카드 1회 실행(`trigger:"ondemand"`). **`_run/`이 없으면**(텔레메트리 도입 전/외부 프로젝트) **먼저 생성**한 뒤 `eval_history.jsonl`을 시작한다. 산출물 없는 외부 레포면 설계 기준이 약하므로 그 한계를 명시.

## Source-Driven Decisions (추정 금지 — 소스 우선)
라이브러리·프레임워크·API·외부 의존성 사용을 **결정하거나 코드를 작성하기 전에**, 기억에 의존하지 말고 실제 소스를 확인한다. (Stage 2 architecture·Stage 3 design·Stage 5 implementation 전반에 적용)

확인 우선순위:
1. **설치된 관련 스킬** 먼저 참조 (예: `vercel:nextjs`, `vercel:ai-sdk`, `vercel:shadcn` 등 권위 있는 소스 스킬)
2. **실제 설치본** 확인: `node_modules`의 타입 정의(`.d.ts`), `package.json`의 정확한 버전
3. 위로 불충분하면 **공식 문서 조회** (WebFetch 등)

규칙:
- API 시그니처·메서드명·옵션 이름·버전을 **확인 없이 단정하지 않는다**. (Anti-Rationalization의 "라이브러리 버전/API는 알고 있으니" 항목과 연동)
- 확인 결과와 기억이 다르면 **항상 실제 소스를 따른다**.
- 고위험 결정(High-Risk Adversarial Review 대상)에는 이 소스 확인을 **필수 전제**로 한다.
- 확인한 핵심 사실(버전·시그니처 등)은 해당 ADR 또는 design_notes에 근거로 기록한다.

## Knowledge Base (cross-project 복리 누적)
factory 실행이 끝날 때마다 재사용 가능한 교훈을 누적하고, 새 실행은 그것을 참조한다. karpathy "LLM Wiki" 패턴의 적용 — 프로젝트 간 지식이 휘발되지 않고 복리로 쌓인다.

**위치**: `factory_output/_knowledge/` (프로젝트 디렉터리들과 나란히, 프로젝트 무관 공용)
- `index.md` — 전 페이지 카탈로그 (페이지당 1줄). `log.md` — 시간순 이력(append-only). `pages/` — 개별 교훈(lesson/gotcha/pattern).

**컨텍스트 폭증 방지 (필수 규율)**: 누적은 디스크에, 컨텍스트엔 **`index.md` + 관련 페이지 ≤5개만** 올린다. KB 전체를 통째로 읽지 않는다 → 컨텍스트는 KB 크기와 무관하게 거의 상수. index.md가 커지면 카테고리 분할, Lint로 낡은 페이지 supersede.

**① Recall (참조)** — Stage 0-D(domain_hints 확정 후), Stage 2 architecture·Stage 3 design 시작 시 수행:
- `_knowledge/index.md`만 먼저 읽고 → 현재 프로젝트와 관련된 페이지 **≤5개**만 drill-in.
- 발견한 교훈은 Source-Driven Decisions의 입력으로 쓰고, 적용 시 design_notes/ADR에 출처(`_knowledge/pages/<page>`) 기록.
- KB가 없거나(첫 실행) 관련 페이지가 없으면 조용히 생략.

**② Harvest (수확)** — 전체 종료 시 SUMMARY 작성 직후 수행:
- 이번 실행에서 얻은 **재사용 가능한** 교훈만 추출 (라이브러리 함정, 검증된 결정 패턴, 보안/아키텍처 원칙). 프로젝트 한정 trivia는 제외.
- 한 실행당 **0~3개** 페이지로 제한(노이즈 방지). 기존 페이지와 겹치면 새로 만들지 말고 갱신.
- 각 페이지는 `_knowledge/README.md`의 형식(frontmatter 포함)으로 `pages/`에 저장 → `index.md` 카탈로그 갱신 → `log.md`에 `harvest` 항목 append.

**③ Lint (정기 점검)** — KB 페이지가 대략 15개↑이거나 사용자가 요청하면: 모순·낡은 사실·고아 페이지·중복을 점검해 갱신/supersede(삭제 대신 `status: superseded by ...`). `log.md`에 `lint` 항목 기록.

## Anti-Rationalization (단계 건너뛰기 변명 차단)
자율 실행 중 아래 핑계가 떠오르면 **모두 거부**한다. 변명에 굴복하는 것이 factory의 가장 흔한 실패다.

| 변명 | 반박 |
|------|------|
| "이건 간단하니 Stage 1/Blueprint 건너뛰자" | ❌ 간단해 보이는 것이 가장 자주 잘못 설계된다. 전 단계 필수. |
| "critic 호출 생략하고 내가 직접 평가하자" | ❌ self-critique 편향. 무조건 별도 서브에이전트(general-purpose + opus). |
| "테스트/검증은 나중에, 일단 구현 완료 선언" | ❌ Stage 4 검증 증거 없이 Stage 5 완료 선언 금지. |
| "consistency audit은 시간 낭비" | ❌ 단계 간 불일치는 구현에서 폭발한다. audit 필수. |
| "빌드/테스트 돌려본 셈 치고 통과로 기록" | ❌ 실제 실행 출력(로그) 없이 '통과' 기록 금지. |
| "라이브러리 버전/API는 알고 있으니 확인 불필요" | ❌ 추정 금지. 불확실하면 실제 파일/문서 확인 후 결정. |
| "lite 프로파일이니 단계/audit 통째로 건너뛰자" | ❌ 프로파일은 **깊이만** 줄인다. lite도 Stage 0~5와 audit을 전부 수행(횟수만 축소). |
| "유지보수 변경이니 드리프트 감사/설계 갱신은 생략하자" | ❌ 코드↔설계 정합 없이 낡은 설계 위에 계획하면 변경이 깨진다. M1·M5 필수(범위만 한정). |
| "평가는 내가(본 세션) 직접 점수 매기자" / "증거 없이 대충 9점" | ❌ 자가채점은 self-critique 편향. 채점은 별도 Opus 서브에이전트 + 모든 점수에 증거 경로 고정. |

## Red Flags (이러면 잘못 가고 있다 — 즉시 교정)
- Blueprint `components`가 1개뿐 → 분해 부족, 재작성
- PRD 요구사항에 측정 불가 표현("빠르게", "잘", "쉽게") → 정량 기준으로 재작성
- `acceptance_criteria`가 비어있거나 "동작함" 수준 → 검증 가능한 형태로 재작성
- 단일 task/커밋이 ~500줄 초과 또는 여러 관심사 혼재 → 얇은 수직 슬라이스로 분할
- Stage 5에서 테스트 없이 "완료" → 미완료로 간주
- critic 점수가 라운드마다 동일 → 개선 정체, 즉시 종료하고 사용자 보고
- `_run/state.json` 갱신 없이 다음 단계 진행 → resume 불가, 즉시 체크포인트
- 프로파일이 risk_tier와 불일치(risk=high인데 lite) → 재선택 (의심되면 한 단계 높게)
- Maintain에서 M2 영향범위 **밖** 파일을 수정 → 범위 이탈, 멈추고 범위 재산정(또는 사용자 보고)
- Maintain에서 변경 전 baseline 없이 회귀 테스트 통과 주장 → M4 회귀 방어 위반, baseline부터 기록
- Evaluation 점수에 증거 경로가 없음(주관적 인상) → 무효, 증거 고정 후 재채점
- Evaluation을 본 세션이 자가채점 → 별도 Opus 서브에이전트로 재실행

## Rules
- **모든 critique는 `Agent` tool로 범용 서브에이전트(`general-purpose` + `model="opus"`)에 위임** (TCI 내부 + cross-stage consistency 둘 다). 위 "Critique 패턴"의 내장 프롬프트 사용. self-critique 금지, 외부 플러그인 의존 없음.
- Think/Improve/Implement는 본 세션이 직접 수행 (컨텍스트 공유)
- 각 단계 시작 시 한 줄 진행 보고: `[Stage 2] PRD 작성 중...`, `[Stage 2 Critique] critic 서브에이전트 호출 중...`
- 단계 경계마다 `_run/state.json` 갱신, 주요 이벤트는 `_run/metrics.jsonl`에 append (resume·telemetry 근거)
- 모든 단계는 **현재 Pipeline Profile**의 라운드·target·검토 강도를 적용한다 (lite/standard/fortified)
- 사용자 확인 필요 단계에서 "수정/추가할 부분 알려주세요" 명시
- 산출물 JSON: 키는 영문, 값은 한국어 허용
- 모든 단계 같은 세션에서 진행 — 별도 executor handoff 없음
- TCI 루프에서 점수가 향상되지 않으면 즉시 종료 (무한 루프 방지)
- critic 응답이 JSON 파싱 실패하면 한 번 더 호출, 두 번째도 실패하면 사용자에게 보고

## Output
- **단계 종료 시**: 핵심 결정 3~5줄 요약 + 저장된 파일 경로
- **전체 종료 시**: `SUMMARY.md`(결과) + `RUN_REPORT.md`(과정·비용 지표) + `EVALUATION.md`(인수 스코어카드·시간추이) 경로 + 구현 결과 + 다음 액션 제안

## SUMMARY.md 템플릿
```markdown
# <project_name>

## 개요
<Stage 2 product_goal>

## 단계별 요약
- **Stage 0 Idea Gate**: 구체화 점수 <initial>→<final>/100 (라운드 <N>), 가정 채움 <count>개
- **Stage 0 Profile**: <category/risk/autonomy>, 파이프라인 프로파일 <lite|standard|fortified>
- **Stage 1 Blueprint**: 최종 점수 <N>, 컴포넌트 <count>개
- **Stage 2 PRD**: 기능 요구 <N>개, 스토리 <N>개
- **Stage 3 Design**: 모듈 <N>개, 구현 단계 <N>개, ADR <N>건 (`adr/` 참조)
- **Stage 4 Verify**: 테스트 <N>개, 게이트 <N>개
- **High-Risk Review**: <실행 여부 / verdict / must_fix 수> (고위험 ADR 있을 경우)
- **Stage 6 Evaluation**: 설계 실현도 <realized|partial|diverged>, 요구 충족 <N/전체> (`EVALUATION.md` 참조)

## Consistency Audit 결과
- 1→2: <score>, blockers: <count>
- 2→3: <score>, blockers: <count>
- 3→4: <score>, blockers: <count>

## 구현 결과
<완료된 task / 남은 task / 발견된 이슈>

## Verification Evidence (증거 기반 검증)
- 빌드: <명령어 → exit code / 출력 요약>
- 테스트: <명령어 → 통과 N / 실패 N>
- 타입체크·린트: <명령어 → 결과>
- 미검증 항목: <항목 + 사유 (있을 경우)>

## Knowledge Base
- Recall: 참조한 교훈 페이지 <목록 또는 없음>
- Harvest: 누적한 교훈 <목록 또는 없음> (`factory_output/_knowledge/`)

## 다음 액션
<권장 후속 작업>

## Maintenance Log
<!-- Maintain 모드(/factory maintain) 실행마다 한 줄씩 append. 최신이 위로. -->
- <YYYY-MM-DD> [<change_type>/<profile>] <변경 요약> — 테스트 <통과 N/실패 N>, ADR <추가/supersede 없음>
```

## RUN_REPORT.md 템플릿
`_run/metrics.jsonl`을 집계해 생성 (과정·비용 가시화 — 포트폴리오 데모용). SUMMARY.md(결과)와 상호 링크.
```markdown
# Run Report — <project_name>

> 프로파일: <lite|standard|fortified> · 생성: <YYYY-MM-DD>

## 파이프라인 지표
| 단계 | TCI 라운드 | 최종 점수 | 비고 |
|------|-----------|-----------|------|
| Stage 1 Blueprint | <N>/<max> | <score> | |
| Stage 2 PRD/Arch  | 1         | <score> | |
| Stage 3 Design    | 1         | <score> | High-Risk: <verdict 또는 미실행> |

## Consistency Audit
| 쌍   | 점수 | blockers | 재감사 |
|------|------|----------|--------|
| 1→2  | <score> | <count> | <Y/N> |
| 2→3  | <score> | <count> | <Y/N> |
| 3→4  | <score> | <count> | (경고만) |

## 비용 (Opus 서브에이전트 호출)
- 총 **<N>회** — idea scorer <n> / critique <n> / consistency <n> / high-risk <n> / slice critique <n>

## Stage 5 구현
- 슬라이스 <done>/<total> · 통합 체크포인트 <n>회
- 테스트: 통과 <N> / 실패 <N>
- design_amendment: <n>건 (<요약>)

## 이력
- 구체화 점수 <initial>→<final> (라운드 <N>), 가정 채움 <n>개
- resume: <발생 횟수 또는 없음>

→ 결과 요약은 `SUMMARY.md` 참조.
```

## EVALUATION.md 템플릿
별도 Opus 서브에이전트의 증거 기반 채점 결과. 매 평가(Stage 6 / Maintain M6 / on-demand)마다 갱신하며 시간추이를 누적한다. 점수는 모두 증거 경로에 고정.
```markdown
# Evaluation — <project_name>

> 채점: 별도 Opus 서브에이전트 · 최근 평가: <YYYY-MM-DD> (<build|maintain|ondemand>)
> 설계 실현도: **<realized | partially-realized | diverged>**

## 스코어카드 (최근)
| 축 | 점수 | 근거 (증거 경로) |
|------|------|------|
| 요구 충족률 | <N/전체> | <충족 입증 테스트/로그 + 미충족 목록> |
| 설계 충실도 | <0~10> | <코드↔stage3_design 이탈 목록 + ADR 설명 여부> |
| 검증 강도 | <0~10> | <테스트 커버리지 + 빌드/타입/린트 결과> |
| 위험 처리 | <0~10> | <High-Risk verdict + 미해결 must_fix 수> |

## 시간추이 (eval_history.jsonl 집계)
| 시점 | trigger | 요구 | 충실도 | 검증 | 위험 | 판정 |
|------|---------|------|--------|------|------|------|
| v1 (빌드) | build | <..> | <..> | <..> | <..> | 기준선 |
| <날짜> <변경요약> | maintain | <..>(<±Δ>) | <..> | <..> | <..> | <개선/유지/회귀> |

## 과정 서사 (참고 — RUN_REPORT 인용)
- 아이디어 구체화 45→ready, 설계 critic <초기>→<최종>

## 권고
- <최저 점수 축에 대한 구체적 개선 제안>
```
