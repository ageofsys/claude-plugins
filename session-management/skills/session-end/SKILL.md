---
name: session-end
description: Use this skill whenever the user signals the end of a working session — explicit phrases like "세션 종료", "세션 종료한다", "세션 마무리", "세션 끝", "wrap up the session", "end session", "let's wrap up", or any clearly equivalent wording. Performs a structured wrap-up — uncommitted changes review with commit suggestions, capture of new decisions/issues into the project memory, PLANS-INDEX progress sync, handoff note for the next session, cleanup of background processes and temporary cloud resources, MEMORY.md index refresh, and session name recommendation (so the session is searchable in the future). Do not skip this skill even when the session feels small; missed handoffs cost more than the few minutes of structured wrap-up. If memory or plans files for this project don't exist yet, the skill degrades gracefully — only running the steps that apply. Ambiguous phrases like "오늘은 여기까지" (could mean true session-end OR a mid-task context switch) trigger the skill but require a single confirm question before running the wrap-up — see `트리거 후 컨텍스트 가드` section.
---

# Session-end Wrap-up

당신이 한 세션의 마무리를 진행 중일 때 이 스킬을 사용한다. 핵심 목적은 **다음 세션이 매끄럽게 이어지도록** 현재 세션의 상태/결정/잔여 작업을 정리하는 것이다. 사용자가 매번 직접 chasing 하지 않고도 일관된 핸드오프가 만들어지는 게 가치다.

## 운영 원칙

- **묻기 전에 점검부터.** 사용자가 "세션 종료" 라고 했을 때 가장 짜증나는 응답은 "어떻게 정리할까요?" 다. 기본 7단계를 수행한 뒤 *결과*만 보고하고, 의사결정이 필요한 시점에만 질문한다.
- **단계별 결과 요약**을 사용자에게 보여준다. 각 단계가 ✅/⚠️/❌ 중 무엇인지 한 줄씩.
- **읽기 → 분석 → 제안 → 사용자 확인 → 실행** 순서. 자동 커밋·자동 push 같은 비가역 작업은 사용자 승인 후에만.
- **존재하지 않는 파일/구조는 스킵.** `docs/superpowers/plans/PLANS-INDEX.md` 가 없는 프로젝트라면 3단계는 자동 생략하고 그렇게 보고한다. 강제 생성하지 않는다.
- 결과가 짧으면 짧게, 길면 섹션 헤더로 정리. 정리 노트 자체를 작품화하지 않는다 — 다음 세션의 *입력 자료* 일 뿐이다.

## 트리거 후 컨텍스트 가드 (over-action 방지)

트리거 표현 일부는 *세션 단위 종료* 의도가 아니라 *현재 작업의 일시 중단·전환*을 가리킬 수 있다. wrap-up 7단계는 메모리·PLANS·핸드오프 노트까지 갱신하므로, 의도가 어긋난 채 진입하면 미완성 상태가 다음 세션의 *시작점*으로 잘못 박제된다.

### 모호 트리거 매핑

| 트리거 | 명확한 세션 종료 | 모호 — 확인 필요 |
|---|---|---|
| "세션 종료", "세션 종료한다", "세션 마무리", "세션 끝", "wrap up the session", "end session", "let's wrap up", "이번 세션 정리" | ✅ 즉시 진입 | — |
| **"오늘은 여기까지"** | — | ⚠️ 직전 발화에 *진행 중 작업의 명시적 정리 신호* (예: "마지막 커밋 정리하고", "내일 이어서") 가 동반되지 않으면 **단일 확인 질문 후 진입**: "세션 종료 wrap-up 진행할까요? / 잠깐 자리 비우는 건가요?" |

### 운영 규칙

- 명확한 표현 → 7단계 즉시 진입.
- 모호 표현이고 **직전 대화에 진행 중인 작업의 정리 신호가 없으면** → 한 줄 확인 질문 후 사용자가 session-end 의사를 밝힐 때만 진입. 자동 진입 금지.
- 사용자가 "잠깐 자리 비움" 고르면 skill 종료. 메모리·PLANS·handoff에는 손대지 않는다.
- 확인 질문은 **한 번만**.

## 7 단계 절차

### 1. 미커밋 변경 점검 + 커밋 제안

목표: 작업한 내용이 git 에 안전하게 보존되도록 한다.

수행:
- `git status -s` 와 `git diff --stat` 로 변경 사실 파악
- 의미 있는 변경이면 변경 그룹별로 커밋 메시지 초안 제시 (한 번에 묶어도 되는지 vs 분리할지 판단)
- 비밀/대용량/실수 commit 위험이 보이면(예: `.env`, 키, 빌드 산출물) 명시적으로 경고
- **자동 커밋 금지** — 사용자가 동의하면 그때 실행

산출: "변경 N건, 커밋 제안 메시지: ..." 또는 "정리할 변경 없음".

### 2. 새 결정/Issue 메모리 기록

목표: 코드/git 으로 환원되지 않는 *판단 정보* 가 다음 세션까지 살아 있게 한다.

수행:
- 이번 세션 대화에서 등장한 **새로운 결정**(예: "X 패턴으로 가기로 함", "Y 는 보류"), **발견된 Issue**(코드 결함·인프라 이상·사용자 환경 제약), **사용자 선호**(예: "FullAccess 는 지금만, 나중에 좁힌다") 식별
- 기존 메모리에 보강할 부분과 새 파일 필요한 부분을 구분
- 메모리 디렉토리는 `~/.claude/projects/<sanitized-cwd>/memory/` (Claude Code 표준)
- 새 파일은 `project_*.md` / `feedback_*.md` / `reference_*.md` 명명 규칙 따름. 프론트매터(name/description/type) 포함
- 중복 제거 — 같은 사실이 여러 메모리에 흩어지면 통합

산출: "새 메모리 N개, 기존 갱신 M개" + 파일 경로 리스트.

### 3. PLANS-INDEX 진척 갱신

목표: 프로젝트의 작업 인덱스가 현실과 어긋나지 않게 한다.

수행:
- `docs/superpowers/plans/PLANS-INDEX.md` 존재 시 열기 (없으면 스킵, 보고)
- 이번 세션에서 완료된 플랜 → ✅ 마커
- 이번 세션에서 *새로* 만든 플랜 → 인덱스에 항목 추가
- 이전엔 ⬜ 였는데 부분 진행됨 → 진행 중 마커 + 짧은 비고
- 가능한 한 변경 최소 — 인덱스가 진실의 출처

산출: "PLANS-INDEX 변경 X줄" 또는 "변경 없음".

### 4. 다음 세션 핸드오프 노트

목표: 다음 세션을 처음 여는 시점에 "어디부터 시작할지"가 즉시 보이게 한다. 동시에 step 7의 추천 세션명을 *지금* 계산하여 frontmatter에 박아 둔다 — 다음 세션이 그 추천명을 그대로 회수해서 "지난 핸드오프 추천명"으로 표시할 수 있게.

#### 4a. 노트 본문 작성

수행:
- 이번 세션의 *결과* (한두 줄), 다음 세션의 *시작점* (한두 줄), *주의사항/블록커* 가 있으면 추가
- 본문은 너무 길게 쓰지 않는다 — 100~300자가 이상적

#### 4b. 추천 세션명 후보 계산 (step 7과 공유)

수행:
- 이번 세션 대화·노트 본문의 어휘를 기준으로 추천 세션명 1개 (선호) ~ 3개 (대안 포함) 생성
- 형식 가이드는 step 7 의 *이름 형식 예시* 와 동일
- 사용자가 이미 `/rename` 한 흔적이 보이면 그 이름을 추천명으로 사용 (재계산 X) — step 7 도 스킵

#### 4c. 핸드오프 파일 생성

수행:
- 위치: `memory/project_handoff_<session-topic>.md`
- 파일 frontmatter:
  ```yaml
  ---
  recommended_session_name: <4b의 1순위>
  alternative_names: [<대안 1>, <대안 2>]   # 선택 — 0~2개
  ended_at: <ISO 8601 종료 시각>
  ---
  ```
- frontmatter 아래에 4a 본문
- 사용자에게 인라인 표시 (지금 화면에 한 번)

산출: 메모리 파일 경로 + 인라인 표시 + 추천 세션명 1줄.

### 5. 백그라운드 프로세스/임시 자원 정리

목표: 다음 세션을 시작했을 때 *유령 자원* 이 남아 혼란을 일으키지 않게 한다.

수행:
- 이번 세션에서 띄운 백그라운드 프로세스(Bash run_in_background, Monitor) 미종료 시 종료
- AWS/클라우드에서 검증용으로 만든 임시 메시지·임시 파일 정리 (예: SQS test 메시지 purge, 다운로드한 검증 파일 `/tmp/*.mp3` 등)
- 영구 자원(큐 자체, 버킷 자체)은 **절대 삭제 금지** — purge 와 delete 를 혼동하지 않는다
- 삭제할지 모호하면 묻는다

산출: 정리한 임시 자원 리스트 + 보존된 영구 자원 명시.

### 6. MEMORY.md 인덱스 갱신

목표: 메모리 인덱스가 실제 메모리 파일과 일치하게 한다.

수행:
- `~/.claude/projects/<sanitized-cwd>/memory/MEMORY.md` 의 항목과 실제 파일 비교
- 새로 추가된 메모리 → 인덱스에 한 줄 추가
- 삭제된 메모리 → 인덱스에서 제거
- 항목 설명이 변경된 의미를 반영하지 못하면 갱신
- **인덱스 자체는 콘텐츠 메모리가 아님** — 한 줄짜리 포인터만 유지

산출: "인덱스 N줄 추가/M줄 변경" 또는 "갱신 불필요".

### 7. /rename 출력

목표: step 4c 에서 frontmatter 에 박아 둔 추천 세션명을 사용자에게 슬래시 명령 한 줄로 제시한다. **재계산하지 않는다** — 후보는 4b 에서 한 번만 만든다.

수행:
- step 4c 의 `recommended_session_name` 을 그대로 사용 — `/rename <이름>` 형태로 출력
- 자동 적용 X — `/rename` 은 슬래시 명령이라 Claude 가 직접 실행 못 함
- 사용자가 이번 세션 중 이미 `/rename` 한 흔적이 보이면 (대화에 `<command-name>/rename</command-name>` 등) 단계 스킵 + 보고

이름 형식 예시 (4b 계산 시 적용한 기준 — 참조용):
- 길이 60자 이내 — 검색·세션 리스트에서 한눈에 들어옴
- 우선 키워드 (조합): (a) 작업 ID/번호 (`#6`, `R3`, `P14` 등), (b) 핵심 도메인 (`broadcast/mic`, `eligibility`, `device-lifecycle`), (c) 결과 동사 (`재현`, `fix`, `조사`, `완료`)
- ❌ "디버깅 작업", "버그 수정", "오늘 한 일 정리" — 검색 키워드 약함
- ❌ "세션 마무리" — 의미 없음
- ✅ `#6 broadcast/mic LIVEKIT_CONNECT_FAILED n=2 재현 + R3 fix`
- ✅ `Phase 1 grant/fencing 완료`
- ✅ `3-DEVICE 재설계 + FE inventory zone 신규`

산출: `/rename <이름>` 한 줄. 또는 "이미 명명됨 — 스킵".

## 마무리 보고 형식

7 단계 끝에 다음 형식으로 한 화면 요약을 남긴다:

```
## 세션 종료 요약

| 단계 | 결과 |
|---|---|
| 1. 커밋 | ✅/⚠️/❌ + 한 줄 |
| 2. 메모리 | ✅/⚠️/❌ + 한 줄 |
| 3. PLANS-INDEX | ✅/⚠️/❌ + 한 줄 |
| 4. 핸드오프 | ✅ + 다음 시작점 한 줄 |
| 5. 임시 자원 | ✅/⚠️/❌ + 한 줄 |
| 6. MEMORY.md | ✅/⚠️/❌ + 한 줄 |
| 7. 세션명 | ✅/⚠️ + `/rename <추천명>` 한 줄 (또는 이미 명명됨 표시) |

다음 세션 시작점: <한 문장>
```

## 안티패턴 (피할 것)

- **"마무리 작업 시작합니다" 만 출력하고 멈추는 패턴.** 사용자는 결과를 원한다 — 점검부터 진행하고 *그 다음에* 보고.
- **불필요한 메모리 양산.** 같은 결정이 여러 파일에 흩어지면 다음 세션이 더 헷갈림. 1개의 정확한 메모리 ≫ 5개의 부분 메모리.
- **모든 변경을 한 커밋으로 묶기.** 보안 키 변경 + 코드 리팩터 + 문서 수정이 한 커밋이면 review/revert 가 어려움. 의미 단위로 분리.
- **사용자가 명시적으로 답한 결정을 다시 묻기.** 본 세션 대화에서 이미 결정된 사항(예: "FullAccess 부여, 보안 cleanup 별도 처리")은 그대로 반영.
- **존재하지 않는 파일을 강제 생성.** `PLANS-INDEX.md` 없는 프로젝트면 그냥 스킵하고 보고.

## 참고 — 사용자 의도 신호

다음 표현이 등장하면 트리거: "세션 종료", "세션 종료한다", "세션 마무리", "세션 끝", "오늘은 여기까지", "wrap up the session", "end session", "let's wrap up", "이번 세션 정리". 사용자가 "정리해줘" 만 말하면 트리거하지 않음 — 너무 광범위. 명시적인 *세션 단위 종료* 의도가 있을 때만 동작.
