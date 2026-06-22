# ADR Index — stock_guidance_kr

| # | 제목 | 상태 | 고위험 |
|---|------|------|:---:|
| [ADR-001](ADR-001-dashboard-stack.md) | 대시보드 스택 = Streamlit 기본 | accepted | yes (revise→적용) |
| [ADR-002](ADR-002-price-data-source.md) | 시세 소스 tier (정식 golden + pykrx 보조) | accepted | yes (revise→적용) |
| [ADR-003](ADR-003-local-llm.md) | 로컬 LLM 감성 = Ollama 결정화+골든셋 검증 | accepted | yes (revise→적용) |
| [ADR-004](ADR-004-storage-pit.md) | 저장 = SQLite WAL + bitemporal PIT | accepted | yes (accept) |
| [ADR-005](ADR-005-security-boundary.md) | 보안 = 폭발반경 0 + Keychain | accepted | yes (revise→적용) |
| ADR-006 | 신호 = 방향 분류 + Platt 캘리브레이션 | accepted | no (stage3_design.json 참조) |
| ADR-007 | 하이브리드 라우팅 + 일일 비용캡 | accepted | no (stage3_design.json 참조) |
