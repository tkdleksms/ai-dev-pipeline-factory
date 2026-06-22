# ADR-004: 저장 = SQLite WAL + bitemporal PIT

- **상태**: accepted
- **날짜**: 2026-06-19
- **고위험 여부**: yes (High-Risk Review verdict: accept, 완화책 포함)

## 맥락 (Context)
개인·로컬·시계열. 야간 배치(시세·공시·감성)와 대시보드 동시 접근, point-in-time·정정공시 처리 필요.

## 결정 (Decision)
**SQLite(WAL + busy_timeout)** + 분석은 **DuckDB(읽기전용/스냅샷)**. **단일 writer 직렬화**(수집 큐), 대시보드·DuckDB는 **읽기전용 연결**. **bitemporal**(valid_time, knowledge_time): 정정 시 **새 행 append + 이전 supersede**(풀스냅샷 금지). PIT 쿼리는 `knowledge_time<=T` 필터로 룩어헤드 **구조적 차단**.

## 대안 (Alternatives)
- Postgres: 개인용 과함(기각).
- Parquet 단독: 트랜잭션 약함(분석 보조로만).

## 결과 (Consequences)
- 단일 파일·간단·누수 차단. 동시 쓰기 불가(직렬화로 해결). 저장 증가는 변경분 append로 억제.
