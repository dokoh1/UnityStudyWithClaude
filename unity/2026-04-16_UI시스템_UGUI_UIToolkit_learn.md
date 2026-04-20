# Unity 학습 기록 - 2026-04-14~16 (Q13)

## 세션 정보
- 분야: Unity
- 주제: UGUI vs UI Toolkit → RectTransform/Anchor → UI 최적화 → Mask → 3D Text vs Canvas World Space → Canvas가 필요한 이유
- 모드: learn
- 날짜: 2026-04-14 ~ 2026-04-16

---

## Unity UI의 역사

```
1세대: IMGUI (OnGUI) → 에디터 전용, 게임 UI에 부적합
2세대: UGUI (2014~) → 현재 주력, Canvas 기반
3세대: UI Toolkit (2021~) → 웹 패러다임 (UXML+USS+C#)
```

---

## UGUI vs UI Toolkit 핵심 비교

| | UGUI | UI Toolkit |
|---|---|---|
| 구조 | Canvas + GameObject | UIDocument + VisualElement |
| UI 요소 | GameObject (무거움) | VisualElement (가벼움) |
| 레이아웃 | Anchor + Layout Group | Flexbox (CSS) |
| 스타일 | Inspector 수동 | USS (CSS 비슷) |
| 가상화 | 수동 구현 | ListView 내장 |
| 성능 (대량) | 느림 (Canvas 재빌드) | 빠름 (변경분만) |
| World Space | 지원 | 제한적 |
| 에셋스토어 | 풍부 | 아직 작음 |
| 학습 곡선 | 낮음 | 중간 (웹 지식 필요) |

### 선택 기준
- 기존 프로젝트 / World Space / 에셋 활용 → UGUI
- 신규 프로젝트 / 대규모 UI / 에디터 확장 → UI Toolkit
- 실무에서는 하이브리드: 런타임은 UGUI, 에디터는 UI Toolkit

---

## RectTransform과 Anchor 시스템

### Anchor vs Position

**Anchor = "부모의 어느 지점을 기준으로?"**
**Position = "그 기준에서 얼마나 떨어져?"**

```
일반 Position만 쓰면:
  1920x1080: 버튼 (100, 100) → 왼쪽 아래 구석
  1280x720: 여전히 (100, 100) → 상대적으로 더 왼쪽 아래

Anchor를 쓰면:
  Anchor (1, 1) = 오른쪽 위 기준
  Position (-100, -30) = 기준에서 떨어진 거리
  → 어떤 해상도에서도 오른쪽 위 구석!
```

비유: Anchor = "호텔 꼭대기 남동쪽 구석 방" (건물 크기 바뀌어도 기준 유지), Position = "거기서 10m 떨어진 방"

### Anchor Preset (Shift/Alt/Shift+Alt)

- **클릭만**: Anchor만 변경
- **Shift 클릭**: Anchor + Pivot 함께 변경 (회전/스케일 기준점 일치)
- **Alt 클릭**: Anchor + Position 함께 변경 (해당 위치로 이동)
- **Shift+Alt 클릭**: Anchor + Pivot + Position 전부 변경 (가장 확실, 초보자 추천)

### Stretch Anchor
Anchor Min과 Max가 다르면 부모 크기에 따라 늘어남. Inspector에서 Width/Height 대신 Left/Right/Top/Bottom 표시.

### Pivot
UI 자체의 기준점. 회전, 크기 변경의 중심. (0,0)=좌하, (0.5,0.5)=중앙, (1,1)=우상.

---

## Canvas가 왜 필요한가

### UI는 3D 렌더링과 다르다
- 카메라 독립 (카메라 움직여도 UI는 제자리)
- 원근법 없음 (항상 같은 크기)
- 항상 맨 앞 (3D에 가려지면 안 됨)
- 해상도 대응 (픽셀/비율 기준)

### Canvas의 역할
1. **배칭** — 같은 Atlas/Material UI를 하나의 메시로 합침 → Draw Call 감소
2. **좌표 변환** — RectTransform → 화면 픽셀 좌표
3. **렌더링 순서** — Sorting Order로 앞뒤 결정
4. **이벤트 처리** — GraphicRaycaster로 클릭 감지
5. **재빌드 관리** — 변경된 Canvas만 재빌드

Canvas 없이 UGUI 컴포넌트(Image, Text, Button)는 렌더링 불가.

---

## Canvas Scaler

**Scale With Screen Size** (가장 흔함):
- Reference Resolution 기준 비례 스케일
- Match=0.5 (Width/Height 중간) 권장
- 모든 해상도에서 적절히 스케일

---

## Safe Area 대응

노치, 펀치홀, 둥근 모서리 피하기. `Screen.safeArea`로 안전 영역 얻어서 RectTransform의 Anchor 조정. 기능적 UI는 Safe Area 안에, 배경은 밖까지 확장.

---

## World Space UI의 문제

- 각 Canvas가 별도 Draw Call → 100개면 100 Draw Call
- 메모리 오버헤드 (Canvas 내부 구조)
- **대안**: Screen Space + WorldToScreenPoint 변환 (모든 HP바를 같은 Canvas에)
- 거리 기반 LOD (가까운 적만 UI 표시)

---

## Layout Group 중첩 문제

Layout Group은 자식 변경 시 재계산. 중첩되면 지수적 비용. **해결**: 1단만 사용, 고정은 Anchor로, 대량 추가 시 enabled 끄고 한 번에 재계산.

---

## Mask vs RectMask2D

| | Mask | RectMask2D |
|---|---|---|
| 방식 | Stencil Buffer | Rect 클리핑 |
| 모양 | 임의 (원, 별 등) | 사각형만 |
| 성능 | 느림 (추가 Pass) | 빠름 |
| 용도 | 원형 프로필 등 | ScrollView 등 대부분 |

**원칙**: 사각형이면 RectMask2D, 특수 모양일 때만 Mask.

---

## 텍스트 표시 3가지 방법

### 1. UI Text (Canvas 안)
- TextMeshProUGUI + CanvasRenderer
- Canvas가 수집해서 배칭
- 화면 고정 UI (HUD, 메뉴)

### 2. 3D Text (Canvas 없이)
- TextMeshPro + MeshRenderer
- 일반 3D 오브젝트로 렌더링
- 월드에 물리적으로 존재하는 텍스트 (간판, 데미지 숫자)
- 가벼움, 배칭 가능

### 3. Canvas World Space Text
- Canvas를 World Space로 설정
- CanvasRenderer로 UI 렌더링
- 복합 UI 가능 (텍스트+이미지+슬라이더)
- Canvas 재빌드 오버헤드

### 3D Text vs Canvas World Space — 근본이 다름

```
3D Text: MeshRenderer (3D 오브젝트) → 가벼움, 텍스트만
Canvas WS: CanvasRenderer (UI 시스템) → 무거움, 복합 UI 가능

적 100마리 이름표:
  3D Text 100개 → 배칭 가능 → Draw Call 수 개
  Canvas 100개 → 각각 독립 → Draw Call 100개!
```

**선택**: 텍스트만 → 3D Text, 복합 UI → Canvas WS, 수가 많으면 → Screen Space 변환

---

## UI 성능 문제 & 최적화 총정리

### 1. Canvas 과도한 재빌드
→ Canvas 분리 (정적/동적)

### 2. Raycast Target 과다
→ 상호작용 없는 UI는 Raycast Target = false

### 3. Layout Group 중첩
→ 1단만, 고정은 Anchor로

### 4. 문자열 업데이트 스팸
→ 값 변경 시만 + TMP SetText (GC 0)

### 5. 과도한 Overdraw
→ 불필요한 배경 제거, 투명 최소화

### 6. Mask 성능 함정
→ RectMask2D 우선, Mask 밖 자식도 렌더링됨 주의

### 7. World Space Canvas 오버헤드
→ Screen Space 변환 또는 거리 기반 LOD

### 8. 모바일 해상도 대응
→ Canvas Scaler (Scale With Screen Size, Match=0.5) + Safe Area

### 9. Pixel Perfect 남용
→ 정적 UI만 ON, 움직이는 UI는 OFF

### 10. EventSystem 중복
→ 씬 전환 시 중복 체크

### 11. 큰 텍스처
→ Sprite Atlas + 적절한 해상도 + 압축 (모바일 ASTC)

### 12. ScrollView 대량 아이템
→ Pool 패턴 (보이는 것만 생성, 재활용)

---

## UI 최적화 체크리스트

```
구조: □ Canvas 분리 □ Canvas Scaler □ Safe Area □ Layout Group 최소
컴포넌트: □ Raycast Target □ TMP □ Sprite Atlas □ RectMask2D
런타임: □ 문자열 최소화 □ Pool □ 거리 기반 World UI
이미지: □ 적절한 해상도 □ 압축 □ 9-slice □ Mipmap OFF
모바일: □ 터치 44x44+ □ Pixel Perfect OFF □ Safe Area
```

---

## 면접 답변 예시

```
Q: "UGUI와 UI Toolkit의 차이를 설명해주세요"

A: "UGUI는 Canvas 기반으로 모든 UI가 GameObject이며
    RectTransform/Anchor로 해상도 대응합니다.
    장점은 직관적이고 에셋이 풍부하다는 점,
    단점은 Canvas 재빌드 비용과 대량 UI 성능 문제입니다.

    UI Toolkit은 UXML/USS/C#으로 웹 개발 패러다임을 따르고
    VisualElement(GameObject 아님)으로 메모리 효율적입니다.
    Flexbox 레이아웃, 데이터 바인딩, Virtualization 내장으로
    대량 UI에서 극적 성능 차이를 보입니다.

    선택은 프로젝트 성격에 따릅니다. 기존 프로젝트나
    World Space UI에는 UGUI, 신규 대규모 UI나 에디터 확장에는
    UI Toolkit. 하이브리드 접근도 흔합니다.

    UGUI 최적화는 Canvas 분리, Raycast Target 최소화,
    TMP SetText, Sprite Atlas가 핵심입니다."
```

---

## 면접 레벨별 기대

| 레벨 | 기대 |
|---|---|
| 주니어 | UGUI 기본 (Canvas, Image, Button, RectTransform, Anchor) |
| 미들 | Canvas 분리, Anchor 활용, UI Toolkit 인지, Mask/RectMask2D |
| 시니어 | 최적화 전략, 하이브리드 접근, 성능 프로파일링, Pool 패턴 |
| 리드 | UI 아키텍처 설계, 팀 가이드라인, 기술 선택 결정 |

---

## 오답 노트

1. **Anchor ≠ Position** — Anchor는 "부모의 어느 지점 기준", Position은 "기준에서 떨어진 거리". 해상도 대응의 핵심.

2. **Canvas는 필수** — UI는 3D와 다른 렌더링 필요 (카메라 독립, 원근 없음). Canvas가 배칭/좌표변환/이벤트 관리. 없으면 UGUI 렌더링 불가.

3. **3D Text ≠ Canvas World Space Text** — 3D Text는 MeshRenderer(3D 오브젝트), Canvas WS는 CanvasRenderer(UI 시스템). 렌더링 파이프라인이 근본적으로 다름. 성능 차이 큼.

4. **Canvas 재빌드가 가장 큰 병목** — UI 하나 변경 시 같은 Canvas 전체 재빌드. 정적/동적 Canvas 분리가 핵심.

5. **Raycast Target 기본값이 true** — 대부분의 UI에서 불필요하지만 기본 ON. 수동으로 OFF 필요.

6. **Mask는 Stencil Buffer 사용** — 무거움. 사각형이면 RectMask2D로 교체.

7. **Layout Group 중첩은 지수적 비용** — 1단만 사용. 고정 레이아웃은 Anchor로.

8. **TMP SetText는 GC 0** — `scoreText.text = "Score: " + score`는 매 프레임 GC. SetText 사용.

9. **World Space Canvas 100개 = Draw Call 100개** — 대량이면 Screen Space + WorldToScreenPoint 변환.

10. **Shift+Alt 클릭이 가장 확실** — Anchor + Pivot + Position 한 번에 설정.
