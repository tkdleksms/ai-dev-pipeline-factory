# 한국주식 투자 가이던스 (개인용) — Factory 설계 요약

## 개요
Mac Studio 로컬 LLM + 외부 API 하이브리드로 한국 주식(코스피/코스닥) 실시간 뉴스 감성을 분석하고, 텔레그램 인입까지 반영해 시장 방향(단기 1일~1주·중기 ~1달)을 '방향+확률(캘리브레이션 퍼센트)'로 산출하며, 종목 밸류에이션을 제공하는 개인용 로컬 대시보드. **설계(Stage 1~4)까지 진행, 구현(Stage 5) 생략.**

## 단계별 요약
- **Stage 0 Idea Gate**: 구체화 점수 48→73/100 (라운드 2), 가정 채움 0개
- **Stage 0 Profile**: local_web_dashboard+data_pipeline+ai_agent / risk=high / autonomy=mid
- **Stage 1 Blueprint**: 최종 점수 6→8 (2라운드), 컴포넌트 13개
- **Stage 2 PRD**: 기능 요구 11개, 스토리 4개 (critic 6→개선 반영)
- **Stage 3 Design**: 모듈 13개, 구현 단계 7개(P0~P6), ADR 7건 (`adr/` 참조, 고위험 5건)
- **Stage 4 Verify**: 테스트 18개(unit/integration/e2e/adversarial/regression), 릴리즈 게이트 8개
- **High-Risk Review**: 실행함 / overall=revise / must_fix 전부 적용(스택 Streamlit 전환, 시세 tier, 로컬LLM 결정화+골든셋, SQLite bitemporal, 보안 폭발반경0)

## Consistency Audit 결과
- 1→2: 9, blockers 0
- 2→3: 9, blockers 0
- 3→4: 7, 경고 3건 → 전부 보강 반영

## 핵심 설계 결정 (차별점)
- **정직한 게이트**: 감성 단독 ≈랜덤 → '감성+가격/거래량/변동성' 결합 + 착수 전 정량 파일럿. 미통과 시 예측 대신 '뉴스 요약·감성 추세'로 강등.
- **퍼센트는 회귀 점추정 아님** → 방향 분류 확률의 캘리브레이션으로 파생(허위정밀 회피).
- **백테스트 신뢰성**: point-in-time(bitemporal knowledge_time) + walk-forward + 한국 거래세·수수료·슬리피지 비용모델 + 룩어헤드 회귀테스트.
- **로컬 우선 하이브리드**: 대량 감성·요약 로컬, 밸류·시장종합만 API + 일일 비용캡.
- **보안 폭발반경0**: LLM 출력 자동실행 금지(human-in-loop), 텔레그램 chat_id allowlist, 시크릿 Keychain.

## 구현 결과
미진행(설계까지만). 구현 재개 시 `implementation_order` P0(파일럿 게이트)부터 시작 권장.

## Verification Evidence (증거 기반 검증)
- 빌드/테스트: 해당 없음 — **설계 단계만 수행(코드 미구현)**.
- 설계 산출물 검증: critic TCI(Stage1 8/10), consistency audit 3회(9/9/7), high-risk 적대검토 1회(revise→적용) — 전부 별도 Opus 4.8 서브에이전트로 수행.
- 미검증 항목: 실제 예측 성능·파일럿 통과 여부는 **구현 후 백테스트로만 확인 가능**(설계상 게이트로 강제).

## 다음 액션
1. 본구현 전 **법적 사전확인**(자본시장법 유사투자자문·데이터 약관) — release gate
2. **P0 파일럿** 먼저 구현해 가설(결합>벤치마크) 정량 검증 → 통과 시 확장
3. 시세 **정식/유료 API(golden)** 1개 확보 + pykrx 보조 연결
4. 로컬 한국어 금융 감성 **골든 라벨셋** 구축

> ⚠️ 투자 가이던스는 고위험 도메인입니다. 본 설계는 '정확한 예측 보장'이 아니라 '정직한 검증 게이트 + 실패 시 강등'을 핵심으로 합니다. 투자 권유 아님(개인용 참고).
