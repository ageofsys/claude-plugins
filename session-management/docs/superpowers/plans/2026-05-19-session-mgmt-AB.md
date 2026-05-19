# session-management A+B Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** session-management 0.4.0에 핸드오프 frontmatter `git_state` 4필드 결정론 대조(A)와 "시도했으나 안 된 것" 거울-쌍 회수(B)를 추가한다.

**Architecture:** 이 플러그인의 산출물은 markdown SKILL.md 행동 명세이며 실행 코드가 아니다. 자동 테스트 러너가 없고 `docs/behavior-cases.md`가 수동 회귀 하니스다. 따라서 TDD를 다음으로 사상한다 — (red) behavior-cases.md에 기대 케이스 행을 먼저 적고 현재 SKILL.md가 그것을 미충족함을 확인, (green) SKILL.md를 편집, (verify) 두 파일을 나란히 재독해 같은 계약을 표현하는지 확인, (commit). CLAUDE.md §4를 지키기 위해 behavior-cases 변경과 그것이 문서화하는 SKILL 변경은 항상 같은 커밋에 묶는다. 커밋은 사용자 선호(테마별 논리 단위 분할)에 따라 A-producer / A-consumer / B-producer / B-consumer / version 5개로 나눈다. 각 중간 커밋은 거울-쌍 하위호환 설계상 graceful한 상태로 끝난다.

**Tech Stack:** Markdown (SKILL.md 명세), YAML frontmatter, git. 빌드·테스트 도구 없음 — 검증은 두 파일 1:1 정독.

**Spec:** `session-management/docs/superpowers/specs/2026-05-19-session-mgmt-AB-design.md` (커밋 `fb13a90`)
**Branch:** `feature/session-mgmt-AB-git-state-deadend` (이미 체크아웃됨)
**모든 경로는 저장소 루트 `/Users/ageofsys/workspaces/my/claude-plugins/` 기준.**

---

## File Structure

| 파일 | 책임 | 변경 |
|---|---|---|
| `session-management/skills/session-end/SKILL.md` | producer — wrap-up 7단계 | step 4a(B), step 4c(A), 마무리 보고 표 |
| `session-management/skills/session-start/SKILL.md` | consumer — warm-up 6단계 | step 1(B 회수), step 2(A 대조), step 6 브리핑 표, 거울 인터페이스 표 |
| `session-management/docs/behavior-cases.md` | 수동 회귀 하니스 | §2.1/§2.4 안티패턴, §3.1/§3.2 degradation 행 추가 |
| `session-management/.claude-plugin/plugin.json` | 매니페스트 (버전 SSOT) | `version` `0.3.0`→`0.4.0` |

비변경(확인만): `README.md`, 루트 `.claude-plugin/marketplace.json`.

---

## Task 1: A-producer — session-end가 git_state 4필드를 생산

**Files:**
- Modify: `session-management/skills/session-end/SKILL.md` (step 4c frontmatter 블록, 마무리 보고 표 step 4 행)
- Modify: `session-management/docs/behavior-cases.md` (§3.2 session-end degradation, §2.1 안티패턴)

- [ ] **Step 1 (red — 기대 명세 작성): behavior-cases.md에 session-end 측 케이스 추가**

`session-management/docs/behavior-cases.md` 의 `### 3.2 session-end` 표(현재 마지막 행이 `| 사용자가 이미 `/rename` 한 흔적 | 7 | 스킵 + "이미 명명됨" 보고 |`)에 다음 행들을 표 끝에 추가:

```markdown
| 비-git workspace | 4c | `git_state` 블록 통째 생략 (강제 빈 값 금지) + "git_state 생략(비-git)" 보고 |
| detached HEAD | 4c | `branch: (detached:<sha7>)` 로 기록 |
| dead-end 없음 | 4a | `## 시도했으나 안 된 것` 서브섹션 통째 생략 |
```

(`dead-end 없음` 행은 Task 3에서 SKILL이 충족시키지만, 같은 §3.2 표라 한 번에 적어 둔다 — Task 3 커밋 전까지는 미충족 상태로 남는 것을 Task 3 Step 2에서 확인한다.)

`### 2.1 자동 비가역 작업 금지` 표 끝(현재 마지막 행 `| `/rename <name>` 시도 ... |`)에 추가:

```markdown
| `git_state` 에 placeholder/빈 값 강제 주입 (비-git인데) | 블록 자체를 생략 — 값을 지어내지 않음 |
```

- [ ] **Step 2 (red verify): 현재 session-end SKILL.md가 git_state를 생산하지 않음을 확인**

Run: `grep -n "git_state" session-management/skills/session-end/SKILL.md`
Expected: 출력 없음 (exit 1). git_state 개념이 아직 없음을 확인 = red.

- [ ] **Step 3 (green): session-end SKILL.md step 4c frontmatter 블록 교체**

`session-management/skills/session-end/SKILL.md` 의 `#### 4c. 핸드오프 파일 생성` 아래 frontmatter yaml 블록을 찾는다. 현재:

```yaml
  ---
  recommended_session_name: <4b의 1순위>
  alternative_names: [<대안 1>, <대안 2>]   # 선택 — 0~2개
  ended_at: <ISO 8601 종료 시각>
  ---
```

다음으로 교체:

```yaml
  ---
  recommended_session_name: <4b의 1순위>
  alternative_names: [<대안 1>, <대안 2>]   # 선택 — 0~2개
  ended_at: <ISO 8601 종료 시각>
  git_state:                                 # git workspace 일 때만 — 비-git이면 블록 통째 생략
    branch: <git symbolic-ref --short -q HEAD || "(detached:<sha7>)">
    commit: <git rev-parse HEAD>             # full SHA — 대조 전용
    commit_subject: <git log -1 --format=%s, 50자 이내>   # 표시 전용, 판정 금지
    worktree_root: <git rev-parse --show-toplevel>        # 절대경로
  ---
```

그리고 같은 4c 의 `수행:` 리스트(현재 `- frontmatter 아래에 4a 본문` 위)에 다음 항목을 `- 파일 frontmatter:` 다음 줄들 뒤, `- frontmatter 아래에 4a 본문` 앞에 추가:

```markdown
- `git_state` 4필드를 위 주석의 git 명령으로 채운다. 비-git workspace 면 `git_state` 블록을 통째 생략한다 (빈 값·placeholder 금지). detached HEAD 면 `branch` 를 `(detached:<sha7>)` 로 기록한다.
```

- [ ] **Step 4 (green): 마무리 보고 표 step 4 행 갱신**

같은 파일 `## 마무리 보고 형식` 의 표에서 현재 행:

```markdown
| 4. 핸드오프 | ✅ + 다음 시작점 한 줄 |
```

교체:

```markdown
| 4. 핸드오프 | ✅ + 다음 시작점 한 줄 + git_state(branch/commit7/wt) 또는 "git_state 생략(비-git)" |
```

- [ ] **Step 5 (verify): SKILL ↔ behavior-cases 1:1 정합 재독**

Run: `grep -n "git_state\|detached" session-management/skills/session-end/SKILL.md session-management/docs/behavior-cases.md`
Expected: session-end SKILL.md 의 4c·보고표, behavior-cases §3.2·§2.1 에 git_state·detached 가 일관 표현됨. `비-git → 블록 생략`, `detached → (detached:<sha7>)` 가 양쪽에서 같은 의미인지 눈으로 확인.

- [ ] **Step 6: Commit**

```bash
git add session-management/skills/session-end/SKILL.md session-management/docs/behavior-cases.md
git commit -m "$(cat <<'EOF'
session-end: 핸드오프 frontmatter에 git_state 4필드 생산 (A-producer)

step 4c가 branch/commit/commit_subject/worktree_root 기록.
비-git→블록 생략, detached→(detached:sha7). behavior-cases §3.2/§2.1 동기화.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 2: A-consumer — session-start step 2 결정론 대조

**Files:**
- Modify: `session-management/skills/session-start/SKILL.md` (step 2 전체, step 6 브리핑 표 `2. Git` 라벨, 거울 인터페이스 표)
- Modify: `session-management/docs/behavior-cases.md` (§3.1 session-start degradation, §2.4 정보 조작 금지)

- [ ] **Step 1 (red): behavior-cases.md에 session-start 측 git_state 케이스 추가**

`session-management/docs/behavior-cases.md` `### 3.1 session-start` 표 끝(현재 마지막 행 `| 모든 단계 ❌ | 6 | 한 줄 브리핑이라도 항상 출력 |`)에 추가:

```markdown
| `git_state` 블록 전체 부재 (0.3.0 구버전/비-git) | 2 | legacy 모호 비교 fallback + "git_state 없음 — 정합성 대조 생략" 보고 |
| `git_state` 일부 필드 부재/파싱 실패 | 2 | 해당 필드만 "대조 불가" 보고, 나머지 필드는 정상 대조 (블록 전체 fallback 아님) |
| worktree_root 3조건 검증 실패 (경로없음/git -C 실패/결과≠저장값) | 2 | `cd` 제안 없이 "현재 머신에 없음/worktree 확인 실패" 보고 |
| branch 불일치 | 2 | ⚠️ + `git checkout <stored branch>` 제안 문자열(비실행) |
| branch = (detached:…) | 2 | `git checkout` 제안 금지 + 브리핑에 "이전 세션 detached 상태" 주의 |
| stored commit 로컬에 없음 | 2 | ⚠️ 미-fetch/다른 머신/다른 repo 가능 |
| stored commit 이 현재 HEAD 의 ancestor | 2 | 정보: 이후 N commits 진행 (정상 경로) |
| 현재 HEAD 가 stored commit 보다 behind | 2 | ⚠️ stale checkout/미-pull |
| stored commit 과 diverged | 2 | ⚠️ force-push/rebase/분기 가능성 |
```

`### 2.4 정보 조작 금지` 표 끝(현재 마지막 행 `| 추천 세션명을 "현재 세션명" 처럼 표시 | ... |`)에 추가:

```markdown
| `git_state` 불일치를 브리핑에서 숨김 | ⚠️ 명시 + 제안 문자열 노출 |
| commit-only 불일치를 "정상" 으로 뭉갬 | ancestry 4분류로 신호 보고 (`정상` 라벨은 worktree+branch 모두 안전할 때만) |
```

`### 2.1 자동 비가역 작업 금지` 표 끝에 추가:

```markdown
| 분기 감지 시 `git checkout` / `cd` 자동 실행 | 제안 문자열만, 실행은 사용자 |
| detached HEAD 에 `git checkout` 제안 | 금지 — "detached 상태" 주의만 |
```

- [ ] **Step 2 (red verify): 현재 session-start step 2가 결정론 대조를 안 함을 확인**

Run: `grep -n "git_state\|merge-base\|ancestry\|symbolic-ref" session-management/skills/session-start/SKILL.md`
Expected: 출력 없음 (exit 1). step 2가 아직 모호 비교임을 확인 = red.

- [ ] **Step 3 (green): session-start SKILL.md step 2 본문 교체**

`session-management/skills/session-start/SKILL.md` 의 `### 2. Git 상태 점검` 섹션 전체(`### 2. Git 상태 점검` 헤더부터 다음 헤더 `### 3. PLANS-INDEX 다음 작업 식별` 직전까지)를 다음으로 교체:

````markdown
### 2. Git 상태 점검

목표: 이전 세션 종료 시점과 현재 작업 트리가 어긋나 있지 않은지 *결정론적으로* 확인한다. **이 절은 step 2 내부 로직만 교체한다 — "6 단계 절차" 헤딩·단계 수는 불변** (step-count drift 회귀 주의).

수행:
- `git status -s` — 미커밋 잔류 변경 점검 (있으면 *왜 남아 있는지*가 핸드오프에 적혀 있어야 정상)
- `git log -5 --oneline` — 마지막 커밋들이 핸드오프와 일치하는지 확인
- 핸드오프 frontmatter 의 `git_state` 회수 후 현재 `git rev-parse` 결과와 필드별 대조:
  - **worktree_root** — `cd` 제안은 3조건 AND 일 때만: ① `[ -d <stored_root> ]` ② `git -C <stored_root> rev-parse --show-toplevel` 성공 ③ 그 결과 == 저장된 `worktree_root`. 셋 다 만족 → `cd <stored_root>` 제안 문자열(비실행). 하나라도 실패 → `cd` 제안 *없이* "경로는 있으나 git worktree 확인 실패" 또는 "현재 머신에 없음" 보고
  - **branch** — 불일치 → ⚠️ + `git checkout <stored branch>` 제안 문자열(비실행). `branch` 가 `(detached:…)` → `git checkout` 제안 *금지* + "이전 세션 detached 상태" 별도 주의
  - **commit** — `git cat-file -e <stored_commit>` + `git merge-base --is-ancestor` 로 4분류: (a) 로컬에 없음 → ⚠️ 미-fetch/다른 머신/다른 repo 가능 (b) stored 가 현재 HEAD 의 ancestor → 정보: 이후 N commits 진행(정상) (c) 현재 HEAD 가 behind → ⚠️ stale checkout/미-pull (d) diverged → ⚠️ force-push/rebase/분기 가능성. **"정상일 수 있음" 식 뭉개기 금지**. `commit_subject` 는 표시 전용 — 절대 판정에 쓰지 않는다
- 출력 우선순위 (모든 신호 보고하되 종합 라벨 결정 순서): ① worktree invalid/mismatch ② branch mismatch/detached ③ commit missing/behind/diverged/ahead. `정상` 라벨은 worktree 와 branch 가 *모두* 안전할 때만 부여 (commit ahead/behind/diverged 여도 worktree+branch 안전하면 라벨은 정상, ⚠️ 신호는 별도 표시)
- 자동 git 조작(`checkout`/`cd` 실행) 절대 금지 — `/rename`·PLANS-불일치 선례와 동일하게 *제안 문자열까지만*

환경 처리:
- 비-git workspace → 보고 후 스킵 ("git 저장소 아님 — 정합성 비교 생략"). 다른 단계는 계속.
- `git` 명령 실패 → ⚠️ 보고 + 다른 단계 계속. 사용자에게 점검 권고.
- `git_state` 블록 전체 부재 (0.3.0 구버전 핸드오프/비-git) → 기존 모호 비교 legacy fallback + "git_state 없음 — 정합성 대조 생략" 보고. 강제 생성·합성 금지.
- `git_state` 블록은 있으나 일부 필드 부재/파싱 실패 → 해당 필드만 "대조 불가" 보고, 나머지 필드는 정상 대조 (블록 전체 fallback 아님).

산출: "브랜치: X, 미커밋 N건, 마지막 커밋: …" + git_state 대조 라벨/신호 한 줄(분기 시 제안 문자열 포함), 또는 "비-git — 스킵" / "git_state 없음 — 대조 생략".
````

- [ ] **Step 4 (green): step 6 브리핑 표 `2. Git` 라벨 확장**

같은 파일 `## 시작 브리핑 형식` 의 표에서 현재 행:

```markdown
| 2. Git | ✅/⚠️/❌ + 브랜치/미커밋/최근 커밋 |
```

교체:

```markdown
| 2. Git | ✅/⚠️/❌ + 브랜치/미커밋/최근 커밋 + git_state 대조(worktree→branch→commit 우선순위, 분기 시 제안 문자열) |
```

- [ ] **Step 5 (green): 거울 인터페이스 표에 git_state 행 추가**

같은 파일 `## session-end 와의 인터페이스` 표(헤더 `| session-end 산출 | session-start 입력 |`)에서 `| 갱신된 `MEMORY.md` 인덱스 | 1단계 — 시스템 컨텍스트로 자동 주입 |` 행 바로 아래(표 끝)에 추가:

```markdown
| frontmatter `git_state` (4필드) | 2단계 — 필드별 결정론 대조 + 우선순위 라벨 + 분기 시 제안 문자열 |
```

- [ ] **Step 6 (verify): 1:1 정합 재독**

Run: `grep -n "git_state\|merge-base\|ancestry\|worktree_root\|detached" session-management/skills/session-start/SKILL.md session-management/docs/behavior-cases.md`
Expected: step 2·브리핑표·인터페이스표 와 behavior-cases §3.1·§2.4·§2.1 의 git_state 케이스가 같은 계약(3조건 가드, 4분류, 우선순위, 라벨 규칙)을 표현. 눈으로 대조.

- [ ] **Step 7: Commit**

```bash
git add session-management/skills/session-start/SKILL.md session-management/docs/behavior-cases.md
git commit -m "$(cat <<'EOF'
session-start: step 2 git_state 결정론 대조 (A-consumer)

worktree 3조건 가드, commit ancestry 4분류, 출력 우선순위,
필드 단위 degradation. 인터페이스/브리핑 표 + behavior-cases §3.1/§2.4/§2.1 동기화.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 3: B-producer — session-end가 dead-end 서브섹션을 생산

**Files:**
- Modify: `session-management/skills/session-end/SKILL.md` (step 4a, 마무리 보고 표 step 4 행)
- (behavior-cases §3.2 `dead-end 없음` 행은 Task 1 Step 1에서 이미 추가됨 — 여기서 SKILL이 충족시킨다)

- [ ] **Step 1 (red verify): 현재 session-end가 dead-end 서브섹션을 생산하지 않음을 확인**

Run: `grep -n "시도했으나 안 된 것\|dead-end" session-management/skills/session-end/SKILL.md`
Expected: 출력 없음 (exit 1) = red. (behavior-cases.md 에는 Task 1 에서 `dead-end 없음` 행이 이미 있으나 SKILL 이 아직 미충족 — 이 Task 가 green 으로 만든다.)

- [ ] **Step 2 (green): step 4a 본문 작성 규칙에 dead-end 서브섹션 추가**

`session-management/skills/session-end/SKILL.md` `#### 4a. 노트 본문 작성` 의 `수행:` 리스트. 현재:

```markdown
수행:
- 이번 세션의 *결과* (한두 줄), 다음 세션의 *시작점* (한두 줄), *주의사항/블록커* 가 있으면 추가
- 본문은 너무 길게 쓰지 않는다 — 100~300자가 이상적
```

교체:

````markdown
수행:
- 이번 세션의 *결과* (한두 줄), 다음 세션의 *시작점* (한두 줄), *주의사항/블록커* 가 있으면 추가
- 본문은 너무 길게 쓰지 않는다 — 100~300자가 이상적
- 이번 세션에 *실제로 시도했으나 폐기된 접근*(dead-end)이 있으면 본문 단락 뒤에 조건부 서브섹션을 둔다. 없으면 서브섹션 통째 생략 (존재하지 않는 구조는 만들지 않는다):

  ```
  ## 시도했으나 안 된 것
  - <접근> → <왜 막혔는지>
  ```

  이 서브섹션은 본문 100~300자 예산과 *별개*로 ≤200자, 항목당 1줄. 다음 세션이 같은 막힌 길을 재시도하지 않게 하는 것이 목적
````

- [ ] **Step 3 (green): 마무리 보고 표 step 4 행에 dead-end 표기 추가**

같은 파일 `## 마무리 보고 형식` 표의 (Task 1에서 한 번 갱신된) `4. 핸드오프` 행:

```markdown
| 4. 핸드오프 | ✅ + 다음 시작점 한 줄 + git_state(branch/commit7/wt) 또는 "git_state 생략(비-git)" |
```

교체:

```markdown
| 4. 핸드오프 | ✅ + 다음 시작점 + git_state(branch/commit7/wt) 또는 "git_state 생략" + dead-end N건 기록/없음 |
```

- [ ] **Step 4 (verify): 1:1 정합 재독**

Run: `grep -n "시도했으나 안 된 것" session-management/skills/session-end/SKILL.md session-management/docs/behavior-cases.md`
Expected: session-end 4a 의 서브섹션 규칙과 behavior-cases §3.2 `dead-end 없음 → 서브섹션 생략` 행이 같은 의미. 눈으로 확인.

- [ ] **Step 5: Commit**

```bash
git add session-management/skills/session-end/SKILL.md
git commit -m "$(cat <<'EOF'
session-end: 핸드오프 본문에 dead-end 서브섹션 생산 (B-producer)

step 4a에 조건부 "## 시도했으나 안 된 것" (있을 때만, ≤200자).
보고표 step4 행에 dead-end 표기 추가.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 4: B-consumer — session-start가 dead-end를 회수·노출 (거울 참여)

**Files:**
- Modify: `session-management/skills/session-start/SKILL.md` (step 1 회수, step 6 브리핑 표 행 추가+리넘버, 거울 인터페이스 표)
- Modify: `session-management/docs/behavior-cases.md` (§3.1 dead-end 서브섹션 부재 행)

- [ ] **Step 1 (red): behavior-cases.md §3.1에 dead-end 회수 케이스 추가**

`session-management/docs/behavior-cases.md` `### 3.1 session-start` 표 끝(Task 2에서 추가한 commit 관련 행들 뒤)에 추가:

```markdown
| `## 시도했으나 안 된 것` 서브섹션 부재 | 1, 6 | step 1 회수 줄 생략, step 6 브리핑 `6. 재시도 금지 접근` 행 `—` 표시 |
| dead-end 서브섹션 존재 | 1, 6 | step 1 회수 → step 6 `6. 재시도 금지 접근 | ⚠️ M건`, 제안 시작점에 "막힌 접근 재시도 방지" 명시 |
```

- [ ] **Step 2 (red verify): 현재 session-start가 dead-end를 회수/노출하지 않음을 확인**

Run: `grep -n "시도했으나 안 된 것\|재시도 금지" session-management/skills/session-start/SKILL.md`
Expected: 출력 없음 (exit 1) = red.

- [ ] **Step 3 (green): step 1 핸드오프 로드에 dead-end 회수 추가**

`session-management/skills/session-start/SKILL.md` `### 1. 메모리 인덱스 + 핸드오프 노트 로드` 의 `수행:` 리스트에서 현재 줄:

```markdown
- 핸드오프 노트의 **이번 세션 결과 / 다음 시작점 / 주의사항** 세 줄 추출
```

바로 다음 줄에 추가:

```markdown
- 핸드오프 본문에 `## 시도했으나 안 된 것` 서브섹션이 있으면 회수 (step 6 브리핑 `6. 재시도 금지 접근` 행 + 제안 시작점 반영). 부재 시 줄 생략 (`recommended_session_name` 부재 처리와 동일 패턴). step 5("미해결 항목 환기")에 fold 하지 않는다 — "미해결 환기"와 "재시도 방지"는 행동 효과가 다르다
```

- [ ] **Step 4 (green): step 6 브리핑 표에 행 추가 + 진입 준비 리넘버**

같은 파일 `## 시작 브리핑 형식` 의 표. 현재 마지막 두 행:

```markdown
| 5. 미해결 항목 | ✅/⚠️ + 환기할 N건 |
| 6. 진입 준비 | ✅ |
```

교체 (브리핑 *표* 행만 변경 — "## 6 단계 절차" 헤딩·단계 수는 불변):

```markdown
| 5. 미해결 항목 | ✅/⚠️ + 환기할 N건 |
| 6. 재시도 금지 접근 | ⚠️ M건 / — (이전 세션 dead-end) |
| 7. 진입 준비 | ✅ |
```

같은 `## 시작 브리핑 형식` 블록 안의 산문 예시에 `**제안 시작점:**` 문장이 있으면 그 뒤에 다음 문구가 포함되도록 한 줄 보강(없으면 `**선택지:**` 줄 앞에 추가):

```markdown
> dead-end 가 회수되면 제안 시작점에 "이미 막힌 접근(<요약>) 재시도 금지" 를 명시한다.
```

- [ ] **Step 5 (green): 거울 인터페이스 표에 dead-end 행 추가**

같은 파일 `## session-end 와의 인터페이스` 표에서 Task 2가 추가한 `| frontmatter `git_state` (4필드) | ... |` 행 바로 아래에 추가:

```markdown
| 본문 `## 시도했으나 안 된 것` | 1단계 회수 → 6단계 브리핑 `6. 재시도 금지 접근` 행 + 제안 시작점 반영 |
```

- [ ] **Step 6 (verify): 1:1 정합 + 리넘버 무결성 재독**

Run: `grep -n "재시도 금지 접근\|진입 준비\|시도했으나 안 된 것" session-management/skills/session-start/SKILL.md session-management/docs/behavior-cases.md`
Expected: 브리핑 표가 `5. 미해결 / 6. 재시도 금지 접근 / 7. 진입 준비` 순서이고 중복/누락 없음. behavior-cases §3.1 의 두 dead-end 행이 step 1·6 동작과 일치. "## 6 단계 절차" 헤딩이 그대로 6단계인지(절차 수 불변) 확인.

- [ ] **Step 7: Commit**

```bash
git add session-management/skills/session-start/SKILL.md session-management/docs/behavior-cases.md
git commit -m "$(cat <<'EOF'
session-start: dead-end 회수·브리핑 노출 (B-consumer)

step 1 회수 + 브리핑 신규 행 "6. 재시도 금지 접근" + 진입준비 7로 리넘버
+ 인터페이스 표 + 제안 시작점 반영. behavior-cases §3.1 동기화.
6단계 절차 헤딩 불변(step-count drift 방지).

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 5: 버전 bump + 최종 교차 정합 점검

**Files:**
- Modify: `session-management/.claude-plugin/plugin.json` (`version`)
- 확인만: 루트 `.claude-plugin/marketplace.json`, `README.md`

- [ ] **Step 1: plugin.json 버전 0.3.0 → 0.4.0**

`session-management/.claude-plugin/plugin.json` 의 `"version": "0.3.0"` 를 `"version": "0.4.0"` 으로 변경. 다른 키 불변.

- [ ] **Step 2 (verify): marketplace.json·README 가 버전을 중복 보유하지 않음을 확인**

Run: `grep -n "version\|0\.3\.0\|0\.4\.0" .claude-plugin/marketplace.json README.md`
Expected: `marketplace.json` 에 `version` 키 없음(SSOT 규칙), `README.md` "제공 Plugins" 표에 버전 컬럼 없음. 둘 다 변경 불필요 확인. 만약 버전 흔적이 있으면 CLAUDE.md §1 위반이므로 별도 보고.

- [ ] **Step 3 (verify): 전체 교차 정합 최종 정독**

Run: `grep -n "git_state\|시도했으나 안 된 것\|재시도 금지 접근" session-management/skills/session-end/SKILL.md session-management/skills/session-start/SKILL.md session-management/docs/behavior-cases.md`
Expected 체크리스트(눈으로 확인):
- session-end producer(4c git_state, 4a dead-end) ↔ session-start consumer(step2 대조, step1 회수) 가 같은 필드명·서브섹션명 사용
- behavior-cases §3.1/§3.2/§2.1/§2.4 의 모든 신규 행이 SKILL 동작과 1:1
- 거울 인터페이스 표 2행이 실제 step 동작과 일치
- 브리핑 표 7행, "6 단계 절차" 헤딩 6단계 유지 (drift 없음)

- [ ] **Step 4: Commit**

```bash
git add session-management/.claude-plugin/plugin.json
git commit -m "$(cat <<'EOF'
session-management 0.3.0 → 0.4.0 (A+B: git_state + dead-end)

CLAUDE.md §1: 기능 추가 MINOR, plugin.json 단일 SSOT.
marketplace.json/README 미변경 확인.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Self-Review

**1. Spec coverage** (스펙 §별 → 태스크 매핑):
- A-1 스키마(4필드, detached, 중첩블록) → Task 1 Step 3 ✓
- A-2 session-end step 4c → Task 1 Step 3-4 ✓
- A-3 session-start step 2(3조건 가드/4분류/우선순위/필드단위 degradation/legacy fallback) → Task 2 Step 3 ✓
- B-1 session-end step 4a 서브섹션 → Task 3 Step 2 ✓
- B-2 session-start step 1 회수 + 브리핑 독립 행 + 리넘버 + 제안 반영 + step5 fold 금지 → Task 4 Step 3-5 ✓
- behavior-cases §3.1/§3.2/§2.1/§2.4 → Task 1 Step 1, Task 2 Step 1, Task 4 Step 1 ✓
- 인터페이스/보고 템플릿 양방향 동기화 → Task 1 Step 4, Task 2 Step 4-5, Task 3 Step 3, Task 4 Step 4-5 ✓
- 버전/릴리스(plugin.json SSOT, marketplace/README 미변경) → Task 5 ✓
- 하위호환(블록 전체 부재 fallback / 필드 단위 degradation / dead-end 부재 줄 생략) → Task 2 Step 3, Task 4 Step 3, behavior-cases 행 ✓
- 성공 기준 1–8 → 각 Task verify step 에서 검증 ✓
- 비목표(C/D/repo_id/LLM압축/자동 git) → 어느 Task 도 구현 안 함 ✓ (스펙 비목표 절과 일치)

갭 없음.

**2. Placeholder scan:** "TBD/TODO/적절히 처리/유사함" 없음. 모든 편집 step 이 교체 전/후 실제 markdown 텍스트 포함. ✓

**3. Type consistency:** 필드명 일관 — `git_state.branch/commit/commit_subject/worktree_root` 가 Task 1(생산)·Task 2(소비)에서 동일. 서브섹션명 `## 시도했으나 안 된 것` 이 Task 3(생산)·Task 4(소비)에서 동일. 브리핑 행명 `6. 재시도 금지 접근` 이 Task 4 Step 4·Step 5·behavior-cases 에서 동일. `commit`=대조 / `commit_subject`=표시전용 구분이 Task 1·Task 2 에서 일관. ✓

이상 없음.
