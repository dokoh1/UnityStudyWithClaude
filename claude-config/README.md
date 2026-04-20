# Claude Code 설정 파일

이 폴더에는 Claude Code에서 사용하는 커스텀 설정 파일들이 포함되어 있습니다.

## 구조

```
claude-config/
├── settings.json           # 글로벌 설정
├── settings.local.json     # 로컬 권한 설정
├── commands/               # 슬래시 커맨드 (구 방식)
│   ├── unity-expert.md
│   ├── csharp-expert.md
│   ├── os-expert.md
│   ├── algo-expert.md
│   ├── ds-expert.md
│   └── help-interview.md
└── skills/                 # 스킬 (현재 방식)
    ├── unity-expert/SKILL.md
    ├── csharp-expert/SKILL.md
    ├── os-expert/SKILL.md
    ├── algo-expert/SKILL.md
    ├── ds-expert/SKILL.md
    ├── help-interview/SKILL.md
    ├── game-design/SKILL.md
    └── game-review/SKILL.md
```

## 설치 방법

```bash
# 커맨드 복사
cp commands/*.md ~/.claude/commands/

# 스킬 복사
cp -r skills/* ~/.claude/skills/
```

## 스킬 목록

| 스킬 | 설명 |
|------|------|
| unity-expert | Unity 게임 엔진 면접 준비 에이전트 |
| csharp-expert | C# 프로그래밍 면접 준비 에이전트 |
| os-expert | 운영체제 면접 준비 에이전트 |
| algo-expert | 알고리즘 면접 준비 에이전트 |
| ds-expert | 자료구조 면접 준비 에이전트 |
| help-interview | 면접 준비 스킬 사용 가이드 |
| game-design | 게임 기획 파트너 에이전트 |
| game-review | 게임 기획서 평가 에이전트 |

## 주의

- `.credentials.json`은 개인 인증 정보라 포함하지 않았습니다
- `history.jsonl`, `sessions/` 등 대화 기록도 제외
- `settings.local.json`의 권한 설정은 본인 환경에 맞게 수정 필요
