# ADR-005: 보안 = 폭발반경 0 + Keychain

- **상태**: accepted
- **날짜**: 2026-06-19
- **고위험 여부**: yes (High-Risk Review verdict: revise → 적용)

## 맥락 (Context)
외부 텍스트(뉴스·텔레그램)가 로컬 LLM 경로로 유입. 프롬프트 인젝션은 완전 차단 불가가 정설. 개인용이라도 시크릿·로컬 API 노출면 존재.

## 결정 (Decision)
방어를 '격리'가 아닌 **'폭발 반경 0'** 으로 재정의:
- LLM 출력은 **자동 실행/주문/도구 호출에 직결 금지**(human-in-the-loop), 출력은 **JSON enum 스키마 강제 + 검증**.
- 텔레그램은 **숫자 chat_id 화이트리스트**(username 위조 방지), 쓰기/주문성 명령 금지(무상태 조회만).
- 시크릿(API 키·봇 토큰)은 **macOS Keychain**(.env 평문 지양), 프로젝트는 백업(iCloud/Time Machine) 제외.
- 대시보드는 **localhost 바인딩 + Host 헤더 검증**(DNS rebinding 방어), 상태변경은 CSRF/토큰.

## 대안 (Alternatives)
- '시스템 프롬프트 분리=차단'으로 신뢰: 과신(기각).

## 결과 (Consequences)
- 인젝션 발생해도 피해 경로 차단. 자동매매 불가(설계상 의도). 운영 편의 일부 감소(Keychain·확인 단계).
