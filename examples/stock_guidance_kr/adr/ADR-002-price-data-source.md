# ADR-002: 시세 소스 tier (정식 golden + pykrx 보조)

- **상태**: accepted
- **날짜**: 2026-06-19
- **고위험 여부**: yes (High-Risk Review verdict: revise → 적용)

## 맥락 (Context)
한국 시세 데이터 확보. pykrx는 KRX 비공식 스크래퍼 — 구조변경·약관·정확성 리스크.

## 결정 (Decision)
**정식/유료 API(증권사 OpenAPI 등)=golden 기준선**, **pykrx=편의 보조**. adjusted/raw **동시 저장**, 코퍼레이트액션(배당락·분할·증자) 인지 시 교차검증 임계 **동적 완화**, 불일치는 폐기 아닌 **quarantine+검토 큐+출처 PIT 기록**. 종가는 **T+1 재확정** 잡으로 정정 반영.

## 대안 (Alternatives)
- pykrx 단독: 단일 실패점·약관 위험(기각).
- 유료 단독: 비용·개인용 과함(보조 정식소스로만).

## 결과 (Consequences)
- 안정성·정확성↑, 구현 복잡도↑(tier·교차검증·재확정). KRX rate-limit·캐시·백오프 필수.
