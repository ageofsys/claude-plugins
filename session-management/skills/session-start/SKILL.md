---
name: session-start
description: Use this skill whenever the user signals the start of a working session — explicit phrases like "세션 시작", "세션 시작한다", "오늘 세션 시작", "이어서 가자", "어디까지 했더라", "이전 세션 이어서", "start session", "let's start the session", "where did we leave off", "pick up where we left off", or any clearly equivalent wording. Performs a structured warm-up — loads MEMORY.md and the latest handoff note, checks git/branch state and uncommitted leftovers, surfaces the next plan from PLANS-INDEX, verifies local infra/background processes, and presents a one-screen briefing of where to start. Mirror counterpart of the session-end skill. Degrades gracefully when memory/plans/handoff files don't exist — only runs the steps that apply. Do not trigger on vague phrases like "시작하자" alone — requires explicit *session-level* start intent.
---

# Session-start Warm-up

당신이 한 세션을 새로 여는 시점에 이 스킬을 사용한다. 핵심 목적은 **이전 세션의 끝을 다시 입력**으로 가져와, 사용자가 따로 설명하지 않아도 *어디서부터 이어가면 되는지* 즉시 파악·제시하는 것이다. `session-end` 가 출력으로 만든 흔적(핸드오프 노트, 메모리, PLANS-INDEX, 커밋)을 입력으로 다시 회수하는 거울 동작이다.

## 운영 원칙

- **묻기 전에 읽기부터.** 사용자가 "세션 시작"이라고 했을 때 가장 짜증나는 응답은 "오늘 뭐 할까요?" 다. 메모리·git·PLANS 를 먼저 읽고 *후보 시작점*을 제시한 뒤, 사용자가 confirm/redirect 하게 한다.
- **단계별 결과 요약**을 사용자에게 보여준다. 각 단계가 ✅/⚠️/❌ 중 무엇인지 한 줄씩.
- **읽기 → 종합 → 제시 → 사용자 확인 → 작업 진입** 순서. 자동으로 코드를 수정하거나 빌드를 돌리지 않는다 — warm-up 은 *상태 복원* 까지만.
- **존재하지 않는 파일/구조는 스킵.** 핸드오프 노트가 없으면 그렇게 보고하고, 메모리만으로 시작점을 추론한다. 강제 생성 금지.
- 사용자가 명시한 다른 작업 의도가 있으면 (예: "세션 시작하고 P14 부터") 그 의도를 우선하되, 1~3 단계 요약은 그래도 보여준다 — 컨텍스트가 누락된 채 작업에 들어가면 다음 어긋남이 생긴다.

## 6 단계 절차

### 1. 메모리 인덱스 + 핸드오프 노트 로드

목표: 이전 세션이 남긴 *판단 정보* 와 *시작점 한 줄*을 회수한다.

수행:
- `~/.claude/projects/<sanitized-cwd>/memory/MEMORY.md` 의 인덱스 확인 (대부분 시스템 컨텍스트로 이미 주입되어 있음)
- 가장 최근 `project_handoff_*.md` 식별 (파일명 날짜·`mtime` 기준)
- 핸드오프 노트의 **이번 세션 결과 / 다음 시작점 / 주의사항** 세 줄 추출
- 같은 토픽의 다른 `project_*.md` 도 훑어 *결정/제약*이 살아 있는지 확인 (특히 "보류", "TODO", "다음 세션에" 표현)

산출: "최근 핸드오프: `project_handoff_<topic>.md` — 다음 시작점: …" 또는 "핸드오프 없음, MEMORY.md 만 참조".

### 2. Git 상태 점검

목표: 이전 세션 종료 시점과 현재 작업 트리가 어긋나 있지 않은지 확인한다.

수행:
- `git status -s` — 미커밋 잔류 변경 점검 (있으면 *왜 남아 있는지*가 핸드오프에 적혀 있어야 정상)
- `git log -5 --oneline` — 마지막 커밋들이 핸드오프와 일치하는지 확인
- 현재 브랜치 명시 (예: `livekit-azure`). 시스템 컨텍스트의 git 상태와 비교해 *분기 또는 누락된 커밋이 있는지* 감지
- 핸드오프엔 "정리된 상태" 인데 잔류 변경이 있으면 ⚠️ 로 보고 — 다른 환경에서의 변경일 수 있음

산출: "브랜치: X, 미커밋 N건, 마지막 커밋: …" + 정합성 한 줄.

### 3. PLANS-INDEX 다음 작업 식별

목표: "다음에 무엇을 할지"를 인덱스에서 자동 추론한다.

수행:
- `docs/superpowers/plans/PLANS-INDEX.md` 존재 시 열기 (없으면 스킵, 보고)
- 첫 번째 `⬜ 미작성` 또는 `🟡 진행중` 항목 찾기
- 핸드오프의 "다음 시작점" 과 PLANS-INDEX 의 다음 항목이 **일치하는지** 교차 검증
  - 일치 → ✅ 그 항목을 시작점으로 제시
  - 어긋남 → ⚠️ 두 후보를 모두 보여주고 사용자에게 선택 요청
- 신규 플랜이 필요해 보이면 (`session-end` 시 인덱스에 등록된 빈 항목 등) `superpowers:writing-plans` 스킬을 사용자에게 권유

산출: "다음 후보 플랜: `<plan-id> — <title>` (status: ⬜/🟡)" 또는 "PLANS-INDEX 없음 — 핸드오프만 사용".

### 4. 환경/백그라운드 프로세스 점검

목표: 코드 작업에 들어가기 전 *로컬 인프라가 살아 있는지* 1초 안에 확인.

수행:
- 프로젝트가 명시한 로컬 의존(예: `docker compose ps` 로 PostgreSQL/MongoDB/Redis) 상태 빠르게 점검 — CLAUDE.md 에 시작 명령이 있으면 그것 기준
- 이전 세션의 `Bash run_in_background` / `Monitor` 잔류 프로세스가 있는지는 *현재 세션 컨텍스트* 에는 없으므로, 의심되면 `lsof -i :PORT` 등 가벼운 확인
- 모두 다운된 상태이면 사용자에게 "**`docker compose up -d` 실행할까요?**" 단일 질문 — 자동 실행 금지
- 환경 검증 자체가 작업 의도가 아니라면 ✅/⚠️ 만 남기고 진행

산출: "로컬 인프라: ✅ 모두 up / ⚠️ 일부 down (재기동 제안)".

### 5. 미해결 결정/Issue 환기

목표: 이전 세션이 "보류" 한 결정이 *다시 살아 나야* 한다 — 그렇지 않으면 침묵 속에 잊힌다.

수행:
- 메모리(`feedback_*.md` / `project_*.md`) 에서 "보류", "TODO", "다음 세션에", "추후 결정" 같은 표현 검색
- 핸드오프 노트의 **주의사항/블록커** 섹션이 있으면 인용
- "이번 세션에서 다룰지 / 계속 보류" 를 사용자에게 한 번 묻는 항목으로 정리 (자동 결정 금지)

산출: "환기할 미해결 항목 N건" 또는 "없음".

### 6. 시작 브리핑 출력

목표: 사용자가 *한 화면* 을 보고 바로 작업에 진입할 수 있게 한다.

수행:
- 아래 "시작 브리핑 형식" 으로 단일 메시지 출력
- 마지막 줄에 **"다음 작업으로 진입할까요? (y / 다른 지시)"** 한 줄 — 사용자의 confirm 또는 redirect 를 기다림
- 사용자가 redirect 하면 그 의도를 따르되, 1~3 단계의 요약은 살린 채 진입

산출: 브리핑 메시지.

## 시작 브리핑 형식

6 단계 끝에 다음 형식의 단일 메시지를 남긴다:

```
## 세션 시작 브리핑

| 단계 | 결과 |
|---|---|
| 1. 핸드오프 | ✅/⚠️/❌ + 한 줄 (다음 시작점) |
| 2. Git | ✅/⚠️/❌ + 브랜치/미커밋/최근 커밋 |
| 3. PLANS-INDEX | ✅/⚠️/❌ + 다음 후보 플랜 |
| 4. 로컬 인프라 | ✅/⚠️/❌ + 한 줄 |
| 5. 미해결 항목 | ✅/⚠️ + 환기할 N건 |
| 6. 진입 준비 | ✅ |

**제안 시작점:** <한 문장>
**선택지:** (1) 제안대로 진행 (2) 다른 작업으로 redirect (3) 보류 결정 먼저 정리

다음 작업으로 진입할까요?
```

## 안티패턴 (피할 것)

- **"세션 시작했습니다, 무엇을 할까요?" 만 출력하기.** 사용자는 *이미 적힌 시작점* 을 다시 설명하고 싶지 않다 — 핸드오프부터 읽고 후보를 *제시* 하라.
- **메모리를 무비판적으로 인용.** 메모리는 시점 기록이다. 1~2주 지난 메모리는 git/코드와 충돌 가능 — 의심되면 현재 상태로 *검증 후* 인용.
- **PLANS-INDEX 와 핸드오프 어긋남을 사용자에게 숨기기.** 어긋남 자체가 신호다 — 두 후보를 *모두* 보여주고 사용자가 고르게 한다.
- **자동으로 인프라 기동/빌드/테스트 실행.** warm-up 은 *상태 복원* 까지. 실행은 사용자 confirm 후.
- **session-end 에서 정리한 임시 자원을 다시 만들기.** 이전 세션이 cleanup 한 자원(SQS test 메시지 등)을 자동 재생성하지 않는다.

## 참고 — 사용자 의도 신호

다음 표현이 등장하면 트리거: "세션 시작", "세션 시작한다", "오늘 세션 시작", "이어서 가자", "이전 세션 이어서", "어디까지 했더라", "어디서부터 시작?", "start session", "let's start the session", "where did we leave off", "pick up where we left off", "resume session". "시작하자" 만은 트리거하지 않음 — 너무 광범위. 명시적인 *세션 단위 시작* 의도가 있을 때만 동작.

## session-end 와의 인터페이스

| session-end 산출 | session-start 입력 |
|---|---|
| `project_handoff_*.md` (다음 시작점) | 1단계 — 가장 먼저 읽음 |
| 새/갱신된 `project_*.md` / `feedback_*.md` | 1단계 — 결정/제약 환기 |
| 갱신된 `PLANS-INDEX.md` | 3단계 — 다음 후보 플랜 |
| 커밋된 변경 / 정리된 미커밋 | 2단계 — 정합성 비교 |
| 정리된 임시 자원 / 종료된 백그라운드 프로세스 | 4단계 — 빈 상태 가정으로 시작 |
| 갱신된 `MEMORY.md` 인덱스 | 1단계 — 시스템 컨텍스트로 자동 주입 |

두 스킬은 같은 *세션 경계 인터페이스* 를 공유한다. 한쪽이 빠지면 다른 쪽의 가치가 절반 이하로 떨어진다.
