# ADR-001: 대시보드 스택 = Streamlit 기본

- **상태**: accepted
- **날짜**: 2026-06-19
- **고위험 여부**: yes (High-Risk Review verdict: revise → 적용)

## 맥락 (Context)
개인 1인·로컬 구동·"한눈에 보는 시각 대시보드" 요구. 사용자가 "개발이 더 쉬운 형태"를 명시.

## 결정 (Decision)
대시보드는 **Streamlit 기본**(단일 Python 의존성, 차트 한 줄, 빠른 반복). React+FastAPI 승급은 **실측 트리거**(초당 갱신·다중 패널 동기화·복잡 상호작용)가 실제 발생할 때만.

## 대안 (Alternatives)
- React+FastAPI: 개인 1인엔 빌드체인·2언어·CORS 등 과설계(초기 기각, 승급 경로로만 유지).
- Next.js: JS 생태계이나 데이터층(Python) 브리지 필요.

## 결과 (Consequences)
- 장점: 개발 단순·단일 언어·유지보수 비용↓.
- 단점: 고도 커스텀 인터랙션 한계 → 승급 트리거로 관리. 금액은 Decimal 처리로 정밀도 보존.
