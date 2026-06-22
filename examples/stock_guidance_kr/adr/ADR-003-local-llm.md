# ADR-003: 로컬 LLM 감성 = Ollama 결정화 + 골든셋 검증

- **상태**: accepted
- **날짜**: 2026-06-19
- **고위험 여부**: yes (High-Risk Review verdict: revise → 적용)

## 맥락 (Context)
Mac Studio 로컬 LLM으로 한국어 금융 뉴스 감성/요약. 일반 감성≠금융 감성, 비결정성은 PIT/재현성 붕괴.

## 결정 (Decision)
Ollama 한국어 모델. **완전 결정화**(temperature=0, seed 고정, top_p/top_k 명시, repeat_penalty+num_predict — _knowledge/ollama-repetition-loop). 투입 전 **골든 라벨셋(한국 금융 뉴스 수백건)으로 정확도/F1 측정 + 사전식 baseline 비교**, 미달 시 감성을 **'참고 태그'로 강등**(의사결정 신호 금지). 요약은 **추출형(원문 인용)**, **수치는 모델 생성 금지**(원문 파싱), 근거 스팬 첨부. 모델명+양자화+버전을 결과에 PIT 기록(drift 추적).

## 대안 (Alternatives)
- 디코딩만 고정: 결정성 불충분(기각).
- 전량 API 감성: 비용·로컬 활용 목적과 배치(기각).

## 결과 (Consequences)
- 재현성·신뢰성↑, 골든셋 구축 비용 발생. 성능 미달 시 자동 강등으로 과신 방지.
