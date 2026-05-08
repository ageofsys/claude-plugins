# session-management

세션 시작/종료 시 자동으로 핸드오프·메모리·git·PLANS 상태를 정리하는 짝 스킬.

## 포함 스킬

- **`session-start`** — 세션 시작 시 warm-up. 이전 세션의 핸드오프 노트, MEMORY.md, git 상태, PLANS-INDEX, 로컬 인프라를 점검하고 시작점을 한 화면 브리핑으로 제시.
- **`session-end`** — 세션 종료 시 wrap-up. 미커밋 변경 점검, 새 결정/이슈를 메모리로 기록, PLANS-INDEX 갱신, 다음 세션 핸드오프 노트 작성, 임시 자원 정리, 세션명 추천.

두 스킬은 *같은 세션 경계 인터페이스*를 공유하는 거울 동작이다. 한쪽만 사용하면 절반의 가치만 얻는다.

## 트리거

| 스킬 | 명확 트리거 | 모호 트리거 (확인 후 진입) |
|---|---|---|
| `session-start` | "세션 시작", "오늘 세션 시작", "이전 세션 이어서", "어디까지 했더라", "start session", "where did we leave off", "pick up where we left off" | ⚠️ "이어서 가자" — 직전에 활성 작업 컨텍스트가 있을 때 단일 확인 질문 후 진입 |
| `session-end` | "세션 종료", "세션 마무리", "세션 끝", "wrap up the session", "end session", "let's wrap up" | ⚠️ "오늘은 여기까지" — 진행 중 작업의 정리 신호가 없을 때 단일 확인 질문 후 진입 |

광범위한 표현("시작하자", "정리해줘") 단독으로는 트리거하지 않는다. 자세한 조건과 컨텍스트 가드 규칙은 각 SKILL.md 의 `description` 프론트매터와 `트리거 후 컨텍스트 가드` 섹션 참고.

## 메모리 저장 위치 가정

이 plugin은 다음 경로 컨벤션을 가정한다:

- 메모리 디렉토리: `~/.claude/projects/<sanitized-cwd>/memory/`
- 핸드오프 파일: `memory/project_handoff_<topic>.md`
- 인덱스: `memory/MEMORY.md`
- (선택) 플랜 인덱스: `docs/superpowers/plans/PLANS-INDEX.md`

존재하지 않는 파일은 graceful하게 스킵한다.

## 의존성

- `superpowers:writing-plans` (선택) — 신규 플랜이 필요할 때 권유

## 라이선스

MIT
