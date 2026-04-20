# Unity 학습 기록 - 2026-04-02 (Part 2)

## 세션 정보
- 분야: Unity
- 주제: 카메라 추적 → Rigidbody Interpolation → SmoothDamp/Lerp 원리 → Cinemachine 전체
- 모드: learn
- 날짜: 2026-04-02 ~ 2026-04-03

---

## 이야기의 시작: 카메라가 떨린다

코드 하나가 주어졌다.

```csharp
public class CameraFollow : MonoBehaviour
{
    public Transform target;
    public Vector3 offset = new Vector3(0, 5, -10);

    void Update()
    {
        transform.position = target.position + offset;
    }
}
```

> "카메라가 캐릭터를 따라갈 때 미세하게 떨립니다. 원인과 해결 방법은?"

**답: Update()에서 카메라를 이동하면 캐릭터 이동과 카메라 이동의 타이밍이 어긋나서 떨림이 발생한다. LateUpdate()로 변경해야 한다.**

원리는 이렇다. 같은 프레임 내에서 `CameraFollow.Update()`가 `Player.Update()`보다 먼저 실행될 수 있다. 그러면 카메라는 플레이어의 "이전 프레임" 위치를 참조해서 이동하고, 그 직후 플레이어가 새 위치로 이동한다. 결과적으로 1프레임의 위치 차이가 떨림으로 보인다.

`LateUpdate()`는 모든 `Update()`가 완료된 후에 실행된다. 따라서 플레이어가 이번 프레임에서 최종 위치를 결정한 뒤에 카메라가 따라가므로, 항상 동기화된다.

---

## 첫 번째 꼬리질문: "LateUpdate로 바꾸면 모든 떨림이 해결되는가?"

아니다. **플레이어가 어떻게 이동하느냐에 따라 다르다.**

### Transform 이동 (Update에서) → LateUpdate로 해결 ✅

```
Update()에서 transform 이동 → LateUpdate()에서 카메라 추적
→ 같은 프레임 내에서 완결 → 떨림 없음
```

### Rigidbody 이동 (FixedUpdate에서) → LateUpdate만으로는 부족

FixedUpdate는 0.02초마다 고정 간격으로 실행되고, Update/LateUpdate는 프레임마다 불규칙하게 실행된다. 이 타이밍 불일치가 문제를 만든다.

```
프레임 1: FixedUpdate 2번 → 플레이어 2칸 이동 → LateUpdate → 카메라 2칸 점프
프레임 2: FixedUpdate 1번 → 플레이어 1칸 이동 → LateUpdate → 카메라 1칸 점프
프레임 3: FixedUpdate 3번 → 플레이어 3칸 이동 → LateUpdate → 카메라 3칸 점프

카메라가 매 프레임 다른 거리를 점프 → 떨림으로 보임
```

### 해결: Rigidbody Interpolation

Rigidbody의 Inspector에서 Interpolate를 설정하면, 이전 FixedUpdate와 현재 FixedUpdate 사이를 부드럽게 보간해준다. Update/LateUpdate에서 `transform.position`을 읽으면 보간된 부드러운 값이 나오므로 떨림이 사라진다.

```
Interpolation = None:
  FixedUpdate 위치: 1 ---- 2 ---- 3 ---- 4
  Update에서 보이는: 1  1  2  2  3  3  4  4 → 계단식, 떨림

Interpolation = Interpolate:
  FixedUpdate 위치: 1 ---- 2 ---- 3 ---- 4
  Update에서 보이는: 1  1.5  2  2.5  3  3.5  4 → 부드러움
```

### "FixedUpdate에서 카메라를 이동하면 안 되나?"

되긴 하지만 권장하지 않는다. FixedUpdate는 초당 50회(기본)인데, 화면 렌더링은 초당 60회, 144회 등으로 더 빠를 수 있다. 렌더링 사이에 카메라 위치가 갱신 안 되는 프레임이 생겨서 끊겨 보인다. 특히 고주사율 모니터에서 심하다.

---

## 두 번째 꼬리질문: "카메라를 부드럽게 따라가게 하려면?"

원래 코드는 즉시 따라가기(Snap)다. 부드럽게 만드는 방법은 세 가지가 있다.

### 방법 1: Vector3.Lerp

```csharp
void LateUpdate()
{
    Vector3 desired = target.position + offset;
    transform.position = Vector3.Lerp(transform.position, desired, smoothSpeed * Time.deltaTime);
}
```

### 방법 2: Vector3.SmoothDamp (가장 권장)

```csharp
private Vector3 velocity = Vector3.zero;

void LateUpdate()
{
    Vector3 desired = target.position + offset;
    transform.position = Vector3.SmoothDamp(transform.position, desired, ref velocity, 0.3f);
}
```

### 방법 3: Cinemachine (실무에서 가장 많이 사용)

코드 없이, Unity의 Cinemachine 패키지로 해결. 디자이너가 파라미터를 직접 조정할 수 있어 협업에 유리하다.

---

## 깊이 파기: Lerp와 SmoothDamp는 왜 저런 원리로 되어있는가?

### Lerp의 수학

Lerp의 공식은 단순하다:

```
result = A + (B - A) * t
```

하지만 Unity에서 "매 프레임 현재 위치를 시작점으로 Lerp"하면 **지수적 감쇠(Exponential Decay)**가 된다.

```
매 프레임 "남은 거리의 10%"를 이동:

프레임 1: 거리 100 → 10 이동 → 남은 90
프레임 2: 거리  90 →  9 이동 → 남은 81
프레임 3: 거리  81 →  8 이동 → 남은 73
...
프레임 50: 거리  1 →  0.1 이동 → 거의 안 움직임
```

수학적으로: `n프레임 후 남은 거리 = 초기 거리 × (1 - t)^n`

**Lerp의 문제점:**

1. **프레임레이트 의존** — 60fps에서는 1초에 60번 계산해서 빨리 도착하고, 30fps에서는 30번만 계산해서 느리게 도착한다. `Time.deltaTime`을 곱하면 대략 보정되지만 완벽하지 않다.

2. **시작이 갑작스러움** — 첫 프레임부터 최고 속도다. "남은 거리의 비율"이라서 처음에 거리가 크면 이동량도 크다. 카메라가 "탁" 하고 출발하는 느낌이 난다.

### SmoothDamp의 수학: 스프링-댐퍼 시스템

SmoothDamp는 물리학의 **스프링-댐퍼(Spring-Damper)** 시스템에서 왔다.

```
실생활 비유: 자동차 서스펜션

  스프링만: 통통 계속 튀어다님 (진동)
  스프링 + 댐퍼(쇼크 업소버): 부드럽게 안정됨

  스프링 = 목표 위치로 당기는 힘 (복원력)
  댐퍼 = 움직임을 저항하는 힘 (감쇠력)
  둘의 균형 = 오버슈트 없이 부드럽게 도달
```

미분 방정식: `F = -kx - cv` (k: 스프링 강도, x: 목표까지 거리, c: 댐핑 계수, v: 현재 속도)

실제 동작:

```
시작 (거리 100, 속도 0):
  스프링 힘 큼, 댐퍼 힘 0 → 천천히 가속 시작

중간 (거리 50, 속도 높음):
  스프링 힘 중간, 댐퍼 힘 큼 → 가속이 줄어듦

거의 도착 (거리 5, 속도 중간):
  스프링 힘 작음, 댐퍼 힘 > 스프링 힘 → 감속하며 부드럽게 정지
```

### 핵심 차이: 속도를 기억하는가

```
Lerp:
  매 프레임 독립적으로 계산
  이전 프레임의 움직임을 모름
  → 갑자기 방향이 바뀌면 V자로 꺾임 (부자연스러움)

SmoothDamp:
  ref velocity로 이전 속도를 기억
  관성이 있음!
  → 갑자기 방향이 바뀌어도 U자 곡선으로 부드럽게 전환
```

**smoothTime 파라미터**: "대략 이 시간 안에 목표에 도달한다"는 의미. 0.05초면 거의 즉시, 0.3초면 적당히 부드러움, 1.0초면 매우 느리게. 값이 클수록 "무거운" 느낌, 작을수록 "민첩한" 느낌.

### 상황별 카메라 추적 최적 조합

| 플레이어 이동 방식 | 카메라 추적 방법 |
|---|---|
| Transform (Update) | LateUpdate + SmoothDamp |
| Rigidbody + Interpolate 켜짐 | LateUpdate + SmoothDamp |
| Rigidbody + Interpolate 꺼짐 | FixedUpdate + SmoothDamp (차선) |
| 실무 (어떤 방식이든) | Cinemachine |

---

## Cinemachine: 카메라 추적 도구가 아닌 카메라 시스템 프레임워크

"카메라가 플레이어를 따라가게 해주는 툴"이 아니라, **영화 촬영 감독을 Unity에 넣은 것**이다. 영화 촬영에 필요한 모든 것 — 카메라 위치, 피사체 추적, 장면 전환, 카메라 흔들림, 구도, 돌리/크레인 샷, 멀티 카메라 전환 — 을 Cinemachine이 전부 처리한다.

### 핵심 구조: Brain + Virtual Camera

실제 카메라(Main Camera)는 1개이고, 거기에 CinemachineBrain이 붙어있다. Virtual Camera는 여러 개를 만들 수 있으며, 각각 다른 카메라 설정을 가진다. Brain은 Priority가 가장 높은 Virtual Camera의 설정값을 Main Camera에 복사한다.

실제 카메라가 여러 개 있는 게 아니다. 가상 카메라의 "설정값"만 Main Camera에 적용하는 것이다. 전환할 때는 VCam1의 위치/회전에서 VCam2의 위치/회전으로 부드럽게 블렌드된다.

---

### 기능 1: Follow (추적)

카메라가 대상을 따라가는 방식. 여러 알고리즘이 내장되어 있다.

- **Transposer** — 고정 오프셋 유지. X/Y/Z Damping을 축별로 설정 가능. 가장 기본.
- **Orbital Transposer** — 타겟 주위를 공전. 마우스/스틱으로 회전. 다크소울, 엘든링 같은 3인칭 카메라.
- **Framing Transposer** — Dead Zone/Soft Zone으로 화면 내 타겟 위치 제어. 2D 플랫포머, 메트로배니아에 적합.
- **Tracked Dolly** — 미리 정한 경로(Path) 위를 카메라가 이동. 컷씬, 리플레이 카메라.

### 기능 2: Look At (주시)

카메라가 어디를 바라볼지 결정. Follow와 독립.

- **Composer** — 타겟을 화면의 특정 위치에 유지. Screen X/Y로 삼분할법(Rule of Thirds) 구도 쉽게 설정.
- **Group Composer** — 여러 타겟을 동시에 화면에 담고 자동 줌 조절. 스매시브라더스 같은 격투 게임.

### 기능 3: Noise (카메라 흔들림)

Perlin Noise 기반의 자연스러운 **지속적** 흔들림. 핸드헬드, 걷기 바운스, 호러 분위기 등에 사용. 프리셋이 제공되어 Inspector에서 고르기만 하면 된다. 직접 구현하면 `Random.insideUnitSphere`라 부자연스럽고 축별 제어가 불가능하다.

### 기능 4: Camera Blend (카메라 전환)

상황이 바뀔 때 카메라가 자연스럽게 전환. Priority만 바꾸면 Brain이 자동 블렌드.

```csharp
dialogCam.Priority = 20;  // 대화 시작 → 자동 블렌드
dialogCam.Priority = 0;   // 대화 종료 → 자동 복귀
```

특정 VCam 간 전환에만 다른 블렌드를 적용 가능 (보스 등장은 Cut, 대화는 EaseInOut 등).

### 기능 5: Confiner (영역 제한)

카메라가 맵 밖을 보여주지 않게 Collider로 제한. 2D 플랫포머, 탑다운 RPG에서 필수.

### 기능 6: Impulse (충격 반응)

폭발, 착지, 피격 등 **일회성 충격**에 반응하는 카메라 쉐이크. Noise(지속적)와 구분된다. 거리 감쇠, 방향성, 동시 발생 합성이 자동으로 처리된다.

### 기능 7: State-Driven Camera

Animator 상태에 따라 자동 카메라 전환. 걷기 → 달리기 → 에임 → 등반 → 사망 등 각 상태에 다른 VCam을 매핑. 코드 한 줄 없이 Animator 상태만으로 카메라가 자동 전환된다.

### 기능 8: Timeline 연동 (컷씬)

Unity Timeline과 결합하면 코드 없이 컷씬 제작. 카메라 전환 타이밍, 블렌드, 앵글을 타임라인에서 드래그로 설정. 프로그래머 없이 연출가가 직접 컷씬을 만들 수 있다.

### 기능 9: CinemachinePath (경로 시스템)

카메라나 오브젝트가 따라가는 경로. 베지어 곡선 지원. 컷씬 카메라 이동, 레이싱 리플레이, NPC 순찰 등에 활용.

### 기능 10: ClearShot (자동 장애물 회피)

여러 VCam 중 "타겟이 가장 잘 보이는" 카메라를 자동 선택. 벽에 가려지면 다른 앵글로 자동 전환.

---

## Cinemachine 면접 레벨별 기대 수준

### 주니어: "Cinemachine을 써봤다"

기본 셋업을 할 수 있는가.

```
기대 답변:
  "Main Camera에 CinemachineBrain을 붙이고,
   Virtual Camera를 만들어서 Follow와 Look At에
   타겟을 넣으면 따라갑니다."
```

알아야 할 것:
- Brain과 Virtual Camera의 관계 (실제 카메라 1개, 가상 카메라 여러 개)
- Follow(위치)와 Look At(회전)의 차이
- Damping 값의 역할

나올 수 있는 꼬리질문:
- "Virtual Camera와 실제 카메라의 관계는?" → VCam은 설정값만 가지고, Brain이 Main Camera에 복사
- "Cinemachine을 왜 쓰나요?" → 검증된 기능, 디자이너 협업 가능

---

### 미들: "장르별 카메라 설정과 블렌드 전환을 할 수 있다"

게임 장르별로 적절한 카메라 설정을 선택하고, 여러 카메라 사이를 전환할 수 있는가.

장르별 선택:
- 3인칭 액션 (다크소울) → Orbital Transposer + Composer + Cinemachine Collider
- 2D 플랫포머 (셀레스테) → Framing Transposer + Dead Zone/Soft Zone + Confiner2D
- 탑다운 (디아블로) → Transposer (높은 오프셋) + Hard Look At
- FPS → 게임플레이는 직접 구현, 연출/컷씬만 Cinemachine

카메라 블렌드:
- Priority만 바꾸면 자동 전환
- Custom Blends로 전환마다 다른 블렌드 방식/시간 설정 가능
- 코드는 `cam.Priority = 20;` 한 줄

나올 수 있는 꼬리질문:
- "Dead Zone과 Soft Zone의 차이?" → Dead Zone 안이면 카메라 고정, Soft Zone이면 천천히 따라감, 밖이면 즉시
- "카메라가 벽을 뚫는 문제?" → Cinemachine Collider Extension으로 Pull Forward
- "블렌드 중에 또 다른 카메라로 전환하면?" → Brain이 현재 블렌드 상태에서 새 블렌드를 자연스럽게 이어감

---

### 시니어: "State-Driven + Timeline으로 시스템 구축, Impulse로 피드백 설계"

카메라 시스템 전체를 설계하고 게임 시스템과 통합할 수 있는가.

**State-Driven Camera 시스템:**

Animator 상태와 카메라를 연결하되, 모든 상태를 Animator에 넣으면 복잡해지므로 커스텀 상태 시스템과 연동하는 방식을 설계한다. 카메라 설정은 ScriptableObject(Camera Profile)로 데이터화해서 디자이너가 Inspector에서 직접 조정할 수 있게 한다.

```
상태별 카메라 프로필 예시:
  Idle   → 거리 5m, FOV 60
  Sprint → 거리 7m, FOV 75 (속도감)
  Aim    → 어깨 오프셋, FOV 40 (줌)
  Death  → 서서히 줌아웃 + 위에서 내려다봄
```

**Impulse 기반 피드백 시스템:**

단순 카메라 흔들림이 아닌, 게임 전체 피드백 체계로 설계한다.

```
이벤트별 피드백 강도 체계:
  약한 피격  → 흔들림 약, 히트스톱 없음
  강한 피격  → 흔들림 중, 히트스톱 약간
  폭발      → 흔들림 강, 히트스톱 + 화면 효과
  보스 스킬  → 흔들림 최강, 히트스톱 + 크로매틱 어버레이션
```

Impulse Channel로 카테고리를 분리하고 (플레이어 액션 / 환경 / UI), 각 채널의 감도를 VCam별로 다르게 설정할 수 있다. 거리 감쇠도 자동 적용된다.

**Timeline + Cinemachine 컷씬:**

Timeline에 Cinemachine Track, Animation Track, Audio Track, Signal Track을 배치해서 코드 없이 컷씬을 만든다. 컷씬 시작 시 플레이어 입력 비활성화, HUD 숨기기를 하고, 종료 시 원래 상태로 복구한다.

나올 수 있는 꼬리질문:
- "컷씬 스킵은?" → `PlayableDirector.Stop()` + 스킵 전용 블렌드나 페이드로 시각적 끊김 방지
- "여러 Impulse 동시 발생 시?" → 자동 합성되지만, 최대 제한 매니저를 위에 두는 게 좋음
- "빠른 상태 전환 시 카메라가 정신없으면?" → 최소 유지 시간을 두어 0.1초 미만의 상태 전환은 카메라 전환을 트리거하지 않게

---

### 리드: "디자이너 워크플로우 설계 + Extension 커스텀"

팀 전체의 카메라 작업 효율을 높이는 시스템과 프로세스를 설계할 수 있는가.

**디자이너 워크플로우:**

카메라 파라미터를 코드가 아닌 데이터(ScriptableObject)로 관리한다. 프로그래머는 시스템 구조를 만들고, 디자이너는 프로필 값을 조정하고, 연출가는 Timeline에서 컷씬을 만든다. 각자 맡은 영역만 건드리고, 코드 수정 없이 카메라 느낌을 변경할 수 있는 구조.

```
역할 분리:
  프로그래머    → 시스템 구조, Extension, 에디터 도구
  레벨 디자이너 → Confiner 영역, Dolly Track 경로
  게임 디자이너 → 카메라/Impulse 프로필 값 조정
  연출 담당     → Timeline 컷씬, 블렌드 커브
```

**Cinemachine Extension 커스텀:**

CinemachineExtension을 상속해서 프로젝트 전용 기능을 만든다.

```csharp
// 예: 지형에 맞춰 카메라 높이를 자동 조정
public class CameraTerrainAdjuster : CinemachineExtension
{
    protected override void PostPipelineStageCallback(
        CinemachineVirtualCameraBase vcam,
        CinemachineCore.Stage stage,
        ref CameraState state,
        float deltaTime)
    {
        if (stage != CinemachineCore.Stage.Body) return;
        // 카메라 아래 지형을 감지해서 높이 보정
    }
}
```

Cinemachine 파이프라인은 4단계 (Body → Aim → Noise → Finalize)로 구성되며, 커스텀 Extension은 원하는 단계에 끼어들 수 있다.

Extension 예시:
- 카메라-플레이어 사이 벽을 반투명으로 만드는 WallFader
- 속도에 비례해 FOV를 변화시키는 SpeedBasedFOV
- 물속에서 카메라 틸트 + 포스트프로세싱 연동

나올 수 있는 질문:
- "Cinemachine 도입 시 고려할 점?" → 학습 곡선, Cinemachine과 커스텀 코드의 경계 설정, 데이터 주도 설계
- "Cinemachine을 쓰면 안 되는 경우?" → FPS 에임, VR, 특수 물리 카메라 등 정밀 제어가 필요한 경우. 이때도 컷씬만 Cinemachine 사용하는 공존 구조 가능
- "Extension 작성 시 주의점?" → 파이프라인 단계 이해 필수, 매 프레임 호출이므로 GC 할당 금지

---

## 전체 여정 정리

```
카메라 떨림 문제 (Update에서 추적)
  → LateUpdate로 변경
    → "Rigidbody 이동이면?"
      → Rigidbody Interpolation
        → "부드럽게 따라가려면?"
          → Lerp (지수적 감쇠, 관성 없음)
          → SmoothDamp (스프링-댐퍼, 관성 있음)
            → "실무에서는?"
              → Cinemachine
                → Follow/LookAt/Blend 기본
                → Noise/Impulse 피드백
                → State-Driven 상태 카메라
                → Timeline 컷씬
                → Extension 커스텀
                → 디자이너 워크플로우 설계
```

| 깊이 | 내용 | 면접 레벨 |
|------|------|----------|
| LateUpdate 사용 | 기본 타이밍 이해 | 주니어 |
| Rigidbody Interpolation | 물리/렌더링 타이밍 불일치 | 주니어~미들 |
| Lerp vs SmoothDamp 원리 | 수학적 배경과 트레이드오프 | 미들 |
| Cinemachine 기본 | Follow, LookAt, Blend | 주니어 |
| 장르별 카메라 설정 | 상황에 맞는 설정 선택 | 미들 |
| State-Driven + Impulse 시스템 | 게임 시스템과 통합 | 시니어 |
| Timeline 컷씬 시스템 | 연출 파이프라인 구축 | 시니어 |
| Extension 커스텀 + 워크플로우 | 팀 효율 설계 | 리드 |

---

## 오답 노트

이 세션에서 틀린 문제는 없었으나, 학습 중 정리가 필요했던 개념:

1. **Lerp가 매 프레임 호출될 때 지수적 감쇠가 되는 이유** — 시작점이 매 프레임 "현재 위치"로 갱신되기 때문. 고정된 A→B 보간이 아니라 "남은 거리의 비율"을 반복 적용하는 것.

2. **SmoothDamp의 `ref velocity`가 핵심인 이유** — 이전 프레임의 속도를 기억함으로써 관성이 생기고, 방향 전환 시 V자가 아닌 U자 곡선이 된다. Lerp와의 가장 큰 차이.

3. **Cinemachine은 카메라 추적 도구가 아님** — 카메라 시스템 전체를 관리하는 프레임워크. Follow 외에도 Blend, Impulse, State-Driven, Timeline 연동, Extension 등 영화 촬영에 필요한 모든 것을 포함.
