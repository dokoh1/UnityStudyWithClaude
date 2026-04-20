---
name: help-interview
description: 면접 준비 스킬들의 사용법과 예시를 보여준다. 사용 가능한 전문가 에이전트 목록, 모드, 사용 예시를 안내한다.
---

# 면접 준비 에이전트 사용 가이드

아래 내용을 그대로 출력한다.

---

## 사용 가능한 에이전트

| 커맨드 | 분야 |
|--------|------|
| `/unity-expert` | Unity 게임 엔진 |
| `/csharp-expert` | C# 프로그래밍 |
| `/os-expert` | 운영체제 |
| `/algo-expert` | 알고리즘 |
| `/ds-expert` | 자료구조 |

## 모드

| 모드 | 설명 |
|------|------|
| `quiz [주제] [문제수]` | 퀴즈 (주제/문제수 생략 시 랜덤 5문제) |
| `interview [주제]` | 모의 면접 (주제 생략 시 랜덤) |
| `learn [주제]` | 개념 학습 (주제 생략 시 랜덤) |
| `solve [난이도]` | 코딩 문제 풀이 (algo만) |
| `implement [자료구조]` | 자료구조 직접 구현 (ds만) |
| `code-review` | 코드 리뷰 (csharp만) |
| `save` | 세션 결과를 .md 파일로 저장 |

## 사용 예시

```
/os-expert quiz                 ← OS 랜덤 주제 퀴즈 5문제
/os-expert quiz 데드락 3        ← OS 데드락 퀴즈 3문제
/csharp-expert interview        ← C# 랜덤 주제 모의 면접
/algo-expert learn              ← 알고리즘 랜덤 주제 학습
/algo-expert learn 그래프       ← 그래프 개념 학습
/ds-expert implement BST        ← BST 직접 구현
/algo-expert solve medium       ← 알고리즘 중급 문제 풀이
/unity-expert save              ← 세션 결과 저장
```

## 저장 위치

`save` 실행 시 `~/study-notes/[분야]/YYYY-MM-DD_주제_모드.md`로 저장됩니다.
오답 노트가 자동으로 포함됩니다.

인자: $ARGUMENTS
