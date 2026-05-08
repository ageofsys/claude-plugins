# ageofsys-plugins — Claude 작업 지침

이 저장소는 **Github 마켓플레이스**(루트 `.claude-plugin/marketplace.json`, 마켓플레이스 이름 `ageofsys-plugins`)와 그 안의 **개별 플러그인 디렉토리**로 구성됩니다.

두 파일의 역할이 다름을 항상 인식할 것:
- **`marketplace.json`** — 카탈로그. 사용자가 플러그인을 *발견*할 때 보는 정보.
- **`<plugin>/.claude-plugin/plugin.json`** — 플러그인 매니페스트. 플러그인 자기 자신의 신원/메타데이터의 **단일 진실의 원천(single source of truth)**.

## 1. 플러그인 버전 갱신 규칙

플러그인을 수정하여 push할 때, **버전은 `<plugin>/.claude-plugin/plugin.json` 의 `version` 한 곳에서만 갱신한다.**

`marketplace.json` 의 `plugins[]` 엔트리에는 `version` 필드를 두지 않는다.

### 근거 ([공식 문서](https://code.claude.com/docs/en/plugin-marketplaces.md))
- 마켓플레이스 엔트리에서 *필수*는 `name` + `source` 둘뿐. `version`/`author`/`license`/`keywords` 등은 plugin.json에 두는 것이 정석.
- Claude Code의 버전 해석 우선순위: ① `plugin.json` → ② marketplace 엔트리 → ③ git SHA. **plugin.json이 조용히 이긴다.**
- 공식 문서는 "Avoid setting `version` in both" 라고 명시적으로 경고 — 양쪽에 두면 stale 매니페스트가 마켓플레이스 광고를 가린다.

### 적용 시점
- 기능 추가/변경/버그 수정 등 **사용자가 체감하는 변경**을 커밋·push할 때
- semver 원칙: breaking → MAJOR, 기능 추가 → MINOR, 버그 수정 → PATCH
- 문서/주석/테스트만 바뀌는 경우는 bump 생략 가능

### 체크리스트
1. 변경 범위에 맞는 새 버전 번호 결정
2. `<plugin>/.claude-plugin/plugin.json` 의 `version` 만 수정
3. `marketplace.json` 은 손대지 않는다 — 단, 플러그인의 `description` 이 카탈로그 노출용으로 바뀌어야 한다면 marketplace.json의 `description` 을 같이 갱신
4. 가능하면 `<plugin>/CHANGELOG.md` 에 항목 추가
5. 같은 커밋에 묶어서 push
6. `README.md` 의 "제공 Plugins" 표는 *버전을 표시하지 않는다* — 버전 SSOT 는 `plugin.json` 한 곳. `description` 변경이라면 그것만 동기화.

## 2. marketplace.json 엔트리 작성 규칙

각 `plugins[]` 엔트리는 **카탈로그 표시에 필요한 최소 정보**만 둔다:

```json
{
  "name": "<plugin-name>",
  "source": "./<plugin-dir>",
  "description": "<카탈로그에서 사용자에게 보일 설명>"
}
```

`version`, `author`, `license`, `keywords` 등 플러그인 자체의 메타데이터는 **plugin.json에만** 둔다.

마켓플레이스 자체의 `owner` 블록(루트 레벨)은 `ageofsys` 로 유지한다.

## 3. 플러그인 격리

플러그인의 모든 산출물(코드, 문서, 스펙, 테스트 등)은 `<plugin>/` 안에만 둔다. 마켓플레이스 루트(`./`)에는 마켓플레이스 자체에 관한 파일만 둔다 (`README.md`, `LICENSE`, `CLAUDE.md`, `.claude-plugin/marketplace.json`, `.gitignore`).

플러그인 내부 컴포넌트는 표준 디렉토리에 배치한다: `skills/`, `commands/`, `agents/`, `hooks/` (자세한 레이아웃은 루트 `README.md` 참조).

## 4. 행동 회귀 검사

각 plugin 안의 SKILL.md 의 *trigger / step / anti-pattern / degradation* 섹션을 손대면, 그 plugin 의 `docs/behavior-cases.md` (있으면) 의 해당 케이스를 함께 갱신할지 점검한다. 이 문서가 자동 eval 도입 전 단계의 수동 회귀 검사 자료다. (예: `session-management/docs/behavior-cases.md`)

## 5. 새 플러그인 추가 절차

1. `<new-plugin>/` 디렉토리 생성
2. `<new-plugin>/.claude-plugin/plugin.json` 작성 — `name`, `version`(`0.1.0` 시작), `description`, `author`, `license`, `keywords` 포함
3. 컴포넌트 파일 작성 (`skills/`, `commands/`, `agents/`, `hooks/` 중 필요한 것)
4. `.claude-plugin/marketplace.json` 의 `plugins[]` 에 **최소 정보 엔트리**(name + source + description)만 추가
5. `README.md` 의 "제공 Plugins" 표 업데이트
6. 같은 커밋으로 push
