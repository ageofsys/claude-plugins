# session-management A+B 설계 — git_state frontmatter + dead-end 회수

- 일자: 2026-05-19
- 대상 플러그인: `session-management` (현재 `0.3.0` → 본 변경으로 `0.4.0`)
- 영향 파일: `skills/session-end/SKILL.md`, `skills/session-start/SKILL.md`, `docs/behavior-cases.md`, `.claude-plugin/plugin.json`
- 비영향: `README.md`(트리거 불변), 루트 `marketplace.json`(SSOT 규칙 — 미변경)
- 리뷰: 2026-05-19 `mads:debate` (max-rounds=3, 신뢰도 높음) 채택 보강 반영. 세션 `20260519-162139-11269`

## 배경 / 동기

조사한 유사 프로젝트(softaworks `agent-toolkit/session-handoff`, IlyaGorsky `memory-toolkit`) 대비 식별된 갭 중 둘을 메운다.

- **갭 ③ — worktree/branch 분기 결정론적 감지 불가.** session-start step 2가 "시스템 컨텍스트와 비교"라는 모호 대조에 의존. 이전 세션이 어떤 worktree/branch/commit에서 끝났는지 핸드오프에 *좌표*가 없어 다른 worktree·branch에서 이어받아도 조용히 어긋난다.
- **신규 갭 — 핸드오프가 "실패한 시도"를 기록하지 않음.** 다음 세션이 이미 막힌 접근을 재시도하는 손실.

해소 원칙: 이 플러그인의 *결정론·투명* 정체성을 깨지 않는다. LLM 자동 압축·다명령 확산·자동 git 조작은 도입하지 않는다. 두 변경 모두 기존 거울-쌍 인터페이스(producer→consumer) 패턴을 재사용한다.

## A — 핸드오프 frontmatter `git_state` 블록

### A-1. 스키마 (4필드 중첩 블록)

session-end step 4c가 생성하는 frontmatter에 중첩 블록 추가:

```yaml
---
recommended_session_name: <기존>
alternative_names: [<기존 — 0~2개>]
ended_at: <기존 ISO 8601>
git_state:
  branch: <git symbolic-ref --short -q HEAD || "(detached:<sha7>)">
  commit: <git rev-parse HEAD>            # full SHA — 대조 전용
  commit_subject: <git log -1 --format=%s, 50자 이내>   # 표시 전용 — 판정 금지
  worktree_root: <git rev-parse --show-toplevel>        # 절대경로
---
```

설계 결정:
- `last_commit` 합성 문자열(`<sha7> <subject>`)은 폐기. `commit`(full SHA)은 결정론 대조 전용, `commit_subject`는 브리핑 표시 전용이며 *절대 판정에 쓰지 않는다*. sha7 충돌·문자열 split 취약성 제거.
- `branch`는 detached HEAD 시 `(detached:<sha7>)`로 기록.
- 플랫 키/단일 합성 문자열 대신 중첩 블록: 블록 전체 부재로 "0.3.0 구버전 핸드오프 또는 비-git"을 단일 판정, 결정론 대조가 문자열 split보다 견고, 기존 구조적 frontmatter 스타일과 일치.
- `repo_id`/`repo_fingerprint`는 **이번 범위 밖**. root-commit 기반 판정이 fork/template/mirror에서 오인 가능하고 모델 행동 명세의 분기 비용이 이득을 초과. 별도 사이클로 보류.

### A-2. session-end 변경 (step 4c)

- `git_state` 4필드를 채울 git 명령 명시 (A-1의 주석 명령 그대로)
- 비-git workspace → `git_state` 블록 *통째 생략*. 강제 빈 값/placeholder 금지
- step 4c 산출 보고 한 줄: "git_state: branch=… commit=<sha7>… wt=…" 또는 "git_state 생략(비-git)"

### A-3. session-start 변경 (step 2)

기존 모호 비교를 결정론적 대조로 교체. **"6 단계 절차" 헤딩·단계 수는 불변** — step 2 내부 로직만 교체(step-count drift 방지, 과거 커밋 `32bd533` 회귀 주의).

**대조 절차:**
1. frontmatter `git_state` 회수
2. 현재 `git rev-parse` 결과와 필드별 대조 후 아래 우선순위로 단일 판정 라벨 + 신호 출력

**worktree_root 검증 (cd 제안 가드 — 3조건 AND):**
- 저장 경로가 존재한다 (`[ -d <stored_root> ]`)
- `git -C <stored_root> rev-parse --show-toplevel` 이 성공한다
- 그 결과가 저장된 `worktree_root` 와 동일하다
- 셋 다 만족 → `cd <stored_root>` 제안 문자열(비실행)
- 하나라도 실패 → `cd` 제안 *없이* "경로는 있으나 git worktree 확인 실패" 또는 "현재 머신에 없음"으로 보고 (stale/이식 경로 오인 차단 — #2 이식성 우려 해소)

**commit 불일치 ancestry 4분류** (`git cat-file -e <stored_commit>` + `git merge-base --is-ancestor`):
- stored commit 로컬에 없음 → ⚠️ 미-fetch / 다른 머신 / 다른 repo 가능
- stored commit 이 현재 HEAD 의 ancestor → 정보: 이후 N commits 진행 (정상 경로)
- 현재 HEAD 가 stored commit 보다 behind → ⚠️ stale checkout / 미-pull
- diverged → ⚠️ force-push / rebase / 분기 가능성
- "정상일 수 있음" 식 뭉개기 금지 (#3 — behavior-cases §2.4 정보 조작 금지와 정합)

**branch 처리:**
- 불일치 → ⚠️ + `git checkout <stored branch>` 제안 문자열(비실행)
- `branch` 가 `(detached:…)` → `git checkout` 제안 *금지*, 브리핑에 "이전 세션 detached 상태" 별도 주의 표시

**출력 우선순위 (모든 신호 보고하되 라벨 결정 순서):**
1. worktree invalid 또는 mismatch
2. branch mismatch 또는 detached
3. commit missing / behind / diverged / ahead
- `정상` 라벨은 worktree 와 branch 가 *모두* 안전할 때만 부여

**degradation (필드 단위):**
- `git_state` 블록 전체 부재 → 기존 모호 비교 legacy fallback + "git_state 없음 — 정합성 대조 생략" 보고. 강제 생성·합성 금지
- 블록은 있으나 일부 필드 부재/파싱 실패 (예: `git_state.commit 없음`, `git_state.worktree_root 파싱 실패`) → 해당 필드만 "대조 불가" 보고, 나머지 필드는 정상 대조 (블록 전체 fallback 아님)

step 6 브리핑 `2. Git` 행에 분기 감지 라벨 반영.

## B — "시도했으나 안 된 것" 거울 참여

### B-1. 핸드오프 본문 서브섹션 (session-end step 4a)

본문 단락 *뒤에* 조건부 선택 서브섹션:

```
## 시도했으나 안 된 것
- <접근> → <왜 막혔는지>
```

규칙:
- 이번 세션에 실제 dead-end가 있을 때만 출력. 없으면 서브섹션 *통째 생략* (프로젝트 "존재하지 않는 구조는 스킵" + "노트를 작품화하지 않는다" 관례 일치)
- 길이 예산: 본문 100–300자 예산과 *별개*로 이 서브섹션 ≤200자. 항목당 1줄
- session-end 마무리 보고 형식 step 4 행에 "+ dead-end N건 기록 / 없음" 추가

### B-2. session-start 회수·노출 (거울 참여 — 독립 행)

- step 1: 핸드오프 본문에서 `## 시도했으나 안 된 것` 서브섹션 회수. 부재 시 줄 생략(`recommended_session_name` 부재 처리와 동일 패턴)
- step 6 브리핑: **독립 행** `6. 재시도 금지 접근 | ⚠️ M건 / —` 추가. 기존 `6. 진입 준비` → `7. 진입 준비` 로 리넘버 (브리핑 *표* 행만 변경, "6 단계 절차" 헤딩 불변)
- dead-end 를 **제안 시작점 판단에 반영** — "이미 막힌 접근 재시도 방지"를 제안 문장에 명시
- step 5("미해결 항목 환기")에 fold 하지 않음 — "미해결 환기" vs "재시도 방지" 는 행동 효과가 달라 독립 노출 (debate 기각 의견 반영)

## behavior-cases.md 갱신 (CLAUDE.md §4 강제)

- **3.1 session-start degradation 표**:
  - `git_state 블록 전체 부재` → step 2 legacy 모호 비교 fallback + "정합성 대조 생략" 보고
  - `git_state 일부 필드 부재/파싱 실패` → 해당 필드만 대조 불가, 나머지 정상 대조
  - `worktree_root 3조건 검증 실패` → `cd` 제안 없이 "현재 머신에 없음/worktree 확인 실패" 보고
  - `branch 불일치` → ⚠️ + `git checkout` 제안 문자열(비실행)
  - `branch = (detached:…)` → `git checkout` 제안 금지 + "detached 상태" 주의
  - `commit ancestry 4분류` 각각의 기대 라벨
  - `## 시도했으나 안 된 것 서브섹션 부재` → step 1/6 줄 생략
- **3.2 session-end degradation 표**:
  - `비-git` → `git_state` 블록 생략
  - `detached HEAD` → `branch: (detached:<sha7>)` 기록
  - `dead-end 없음` → 서브섹션 생략
- **2.1 안티패턴 표**:
  - `분기 감지 시 git checkout / cd 자동 실행` → 제안 문자열만, 실행은 사용자
  - `detached HEAD 에 git checkout 제안` → 금지
- **2.4 정보 조작 금지 표**:
  - `git_state 불일치를 브리핑에서 숨김` → ⚠️ 명시 + 제안 문자열 노출
  - `commit-only 불일치를 "정상"으로 뭉갬` → ancestry 4분류로 신호 보고

## 인터페이스 / 보고 템플릿 동기화

debate 채택: behavior-cases 뿐 아니라 양방향 표·템플릿 모두 갱신.

- **session-start SKILL.md 하단 거울 인터페이스 표(consumer 측)** 2행 추가:

  | session-end 산출 | session-start 입력 |
  |---|---|
  | frontmatter `git_state` (4필드) | step 2 — 필드별 결정론 대조 + 우선순위 라벨 + 분기 시 제안 문자열 |
  | 본문 `## 시도했으나 안 된 것` | step 1 회수 → step 6 브리핑 `6.` 행 + 제안 시작점 반영 |

- **session-end SKILL.md producer 측 서술** 갱신: step 4 산출물 목록에 `git_state` 4필드·dead-end 서브섹션을 producer로 명시
- **양쪽 SKILL 정규 보고 템플릿** 갱신:
  - session-start step 6 브리핑 표: `2. Git` 라벨 정의 확장 + 신규 `6. 재시도 금지 접근` 행 + `7. 진입 준비` 리넘버
  - session-end 마무리 보고 표: step 4 행에 git_state·dead-end 산출 한 줄
- **필드별 의미 명문화** (인터페이스 표 또는 step 2 본문에): 각 git_state 필드의 *대조 대상 / 표시 전용 / 생략 조건* — `commit`=대조, `commit_subject`=표시, `branch`=대조(detached 예외), `worktree_root`=대조(3조건 가드)

## 버전 / 릴리스

- CLAUDE.md §1: 사용자 체감 기능 추가 = MINOR → `plugin.json` `version` `0.3.0` → `0.4.0` (한 곳만)
- `marketplace.json` 미변경 (SSOT 규칙). `description` 변경 없음
- `CHANGELOG.md` 부재 시 강제 생성하지 않음. 존재하면 항목 추가
- README "제공 Plugins" 표 버전 미표시 규칙 그대로 — README 변경 없음

## 하위호환

- 0.3.0 생성 기존 핸드오프(`git_state` 없음, dead-end 서브섹션 없음)를 0.4.0 session-start가 읽어도 graceful: 블록 전체 부재 → legacy fallback, dead-end 부재 → 줄 생략. 깨지지 않음
- 0.4.0 session-end는 즉시 신규 frontmatter·서브섹션을 생산 → 다음 세션부터 거울 효과 발동

## 비목표 (이번 스펙 범위 밖)

- C(degraded-handoff 안전망 hook), D(보호 브랜치 커밋 가드) — 별도 brainstorming 사이클
- `repo_id`/`repo_fingerprint` — 별도 사이클 (fork/template 오인·분기 비용)
- LLM 자동 압축, 다명령 확산 — 의도적 비채택
- 자동 git 조작(checkout/cd 실행) — 영구 비채택, 제안 문자열까지만

## 성공 기준

1. session-end 0.4.0이 git workspace에서 `git_state` 4필드를 정확히 채운 핸드오프를 생성 (detached HEAD 시 `branch: (detached:<sha7>)`)
2. 비-git에서 `git_state` 블록을 생략하고 그 사실을 보고
3. dead-end가 있던 세션에서만 `## 시도했으나 안 된 것`이 출력, 없으면 생략
4. session-start 0.4.0이 worktree mismatch를 3조건 가드로 검증해 stale 경로에 `cd` 제안을 *내지 않음*
5. commit-only 불일치를 ancestry 4분류로 보고 (behind/diverged를 "정상"으로 뭉개지 않음)
6. 출력 우선순위(worktree→branch→commit) 적용, `정상` 라벨은 worktree+branch 모두 안전할 때만
7. 0.3.0 구버전 핸드오프(블록 전체 부재) → legacy fallback, 일부 필드 부재 → 필드 단위 degradation
8. behavior-cases.md·양방향 인터페이스 표·정규 보고 템플릿의 신규 케이스가 SKILL.md 변경과 1:1 정합
