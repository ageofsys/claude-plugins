# ageofsys-plugins

ageofsys의 개인용 [Claude Code](https://docs.claude.com/en/docs/claude-code) plugin marketplace.

## 설치

```bash
# Claude Code 안에서 marketplace 등록 (한 번)
/plugin marketplace add ageofsys/claude-plugins

# 원하는 plugin 설치
/plugin install <plugin-name>@ageofsys-plugins
```

## 제공 Plugins

| Plugin | 설명 | 버전 |
|---|---|---|
| [`session-management`](./session-management) | 세션 시작/종료 시 자동으로 핸드오프·메모리·git·PLANS 상태를 정리하는 짝 스킬 | 0.2.0 |

## 저장소 구조

```
.
├── .claude-plugin/
│   └── marketplace.json        # marketplace 매니페스트 (plugin 목록)
├── README.md
└── <plugin-name>/              # plugin 단위 디렉토리
    ├── .claude-plugin/
    │   └── plugin.json         # plugin 매니페스트
    ├── README.md
    └── skills/                 # (또는 commands/, agents/, hooks/)
        └── <skill-name>/
            └── SKILL.md
```

## Plugin 추가 절차

1. 새 디렉토리 생성: `<new-plugin>/`
2. `<new-plugin>/.claude-plugin/plugin.json` 작성 (기존 plugin 참고)
3. `<new-plugin>/skills/...`, `<new-plugin>/commands/...` 등 컴포넌트 작성
4. `.claude-plugin/marketplace.json` 의 `plugins` 배열에 항목 추가
5. semver 기준 버전 부여 (`0.1.0` 시작)
6. PR or main 직접 push

## 라이선스

MIT
