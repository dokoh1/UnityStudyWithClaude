# Unity 학습 기록 - 2026-04-14 (Q12)

## 세션 정보
- 분야: Unity
- 주제: MonoBehaviour 생명주기 → Awake/OnEnable/Start → FixedUpdate/Update/LateUpdate → OnDisable/OnDestroy → Script Execution Order
- 모드: learn
- 날짜: 2026-04-14

---

## 전체 생명주기 흐름

```
[오브젝트 생성]
    ↓
Awake()
    ↓
OnEnable()
    ↓
Start()  ← 첫 프레임 직전
    ↓
[게임 루프]
  FixedUpdate() → Collision/Trigger 이벤트
  Update()
  LateUpdate()
  렌더링
    ↓
[비활성화 시]
OnDisable()
    ↓
[파괴 시]
OnDestroy()
    ↓
[앱 종료 시]
OnApplicationQuit()
```

---

## 초기화 단계

### Awake()
- **호출 시점**: 오브젝트 생성 시, 비활성 상태여도 호출, 1회만
- **용도**: 자기 자신 초기화, GetComponent 캐싱, 필드 할당
- **주의**: 다른 오브젝트 참조 금지 (아직 초기화 안 됐을 수 있음)

### OnEnable()
- **호출 시점**: Awake 직후 + SetActive(true)/enabled=true 될 때마다
- **용도**: 이벤트 구독(+=), 외부 시스템 등록, 활성화 시 상태 리셋
- **중요**: OnDisable과 반드시 짝으로!

### Start()
- **호출 시점**: 모든 Awake 완료 후, 첫 Update 직전, 1회만
- **용도**: 다른 오브젝트 참조 (안전), 본격 게임 로직
- **특징**: IEnumerator 반환으로 코루틴 가능

### 세 함수 비교

| | Awake | OnEnable | Start |
|---|---|---|---|
| 호출 횟수 | 1회 (생성 시) | 여러 번 | 1회 (첫 활성 시) |
| 비활성도 호출? | ✅ | ❌ | ❌ |
| 순서 | 가장 먼저 | Awake 직후 | 모든 Awake 후 |
| 다른 오브젝트? | ❌ 위험 | 상황별 | ✅ 안전 |
| 코루틴? | ❌ | ❌ | ✅ |

**용도 정리**: Awake = 자기 초기화, OnEnable = 이벤트 구독, Start = 남 참조

---

## 게임 루프 단계

### FixedUpdate() — 물리 전용
- 고정 시간 간격 (기본 0.02초 = 50Hz)
- PC 성능 무관
- 한 프레임에 0~N번 호출될 수 있음
- **용도**: Rigidbody 조작, AddForce, velocity
- **시간**: `Time.fixedDeltaTime` (항상 0.02)

### Update() — 일반 로직
- 매 프레임 호출
- 프레임 간격 불규칙 (16ms ~ 33ms)
- **용도**: 입력 감지, 게임 로직, Transform 이동
- **시간**: `Time.deltaTime` (프레임마다 다름)

### LateUpdate() — 모든 Update 후
- Update 전부 끝난 후 호출
- 렌더링 직전
- **용도**: 카메라 추적, 다른 오브젝트 최종 상태 읽기

### 한 프레임 내 순서
```
(accumulatedTime 충족 시 반복)
FixedUpdate → 물리 → Collision/Trigger
Update
LateUpdate
렌더링
```

---

## 충돌 이벤트

```
OnCollisionEnter/Stay/Exit  (Solid Collider)
OnTriggerEnter/Stay/Exit    (Trigger)
```

- FixedUpdate 직후 호출
- 두 오브젝트 중 하나는 Rigidbody 필요

---

## 종료 단계

### OnDisable()
- **호출 시점**: SetActive(false), enabled=false, 파괴 직전
- **용도**: 이벤트 구독 해제(-=), 외부 시스템에서 제거

### OnDestroy()
- **호출 시점**: Destroy() 호출, 씬 전환, 앱 종료
- **용도**: 최종 정리, 네이티브 리소스 해제
- **주의**: 앱 강제 종료 시 호출 안 될 수 있음

### OnApplicationQuit() / OnApplicationPause()
```csharp
void OnApplicationQuit() { SaveData(); }
void OnApplicationPause(bool pause) { if (pause) SaveData(); }
```
모바일 백그라운드 전환 대응.

---

## 여러 오브젝트의 호출 순서

### 전체 원칙

```
씬 로드 시:
  1단계: 모든 오브젝트의 Awake (순서 보장 X)
  2단계: 모든 오브젝트의 OnEnable
  3단계: 모든 오브젝트의 Start
  4단계: 게임 루프 시작
```

**중요**: 같은 단계 내에서 오브젝트 간 순서는 보장 안 됨!

### 문제 예시

```csharp
// 위험: GameManager.Awake보다 먼저 실행될 수 있음
void Start()
{
    GameManager.Instance.RegisterPlayer(this);
}
```

---

## Script Execution Order

### 설정 방법

**Edit > Project Settings > Script Execution Order**

```
-100  GameManager   ← 가장 먼저
 -50  AudioManager
   0  (기본)
  50  UIManager
 100  Analytics    ← 가장 나중
```

숫자가 작을수록 먼저 실행.

### 코드로 설정

```csharp
[DefaultExecutionOrder(-100)]
public class GameManager : MonoBehaviour { }
```

### 실무 원칙

- 핵심 매니저: -200 ~ -100
- 시스템 매니저: -50
- 기본 스크립트: 0
- 후처리: 50

**주의**: 남용하면 의존성 복잡. 이벤트/초기화 패턴 우선, Script Execution Order는 최후 수단.

---

## 실무 활용 패턴

### 패턴 1: 안전한 초기화
```csharp
void Awake()     { rb = GetComponent<Rigidbody>(); }  // 자기 자신
void OnEnable()  { GameEvents.OnX += Handler; hp = 100; }  // 구독+리셋
void Start()     { target = GameManager.Instance.Player; }  // 남 참조
void OnDisable() { GameEvents.OnX -= Handler; }  // 해제 (짝)
void OnDestroy() { SaveStats(); }  // 최종 정리
```

### 패턴 2: Object Pooling과 OnEnable
풀에서 꺼낼 때 OnEnable에서 상태 리셋. Destroy가 아니라 SetActive(false)로 반환.

### 패턴 3: 카메라 추적은 LateUpdate
Update에서 하면 1프레임 딜레이 → 떨림. LateUpdate에서 플레이어 최종 위치 따라감.

### 패턴 4: 입력은 Update, 물리는 FixedUpdate
```csharp
void Update()      { if (Input.GetKeyDown(Space)) jumpQueued = true; }
void FixedUpdate() { if (jumpQueued) { Jump(); jumpQueued = false; } }
```

### 패턴 5: 이벤트 구독 패턴
```csharp
void OnEnable()  { Event += Handler; }
void OnDisable() { Event -= Handler; }  // 반드시 짝!
```

---

## 자주 하는 실수

1. **Awake에서 다른 오브젝트 참조** — Start에서 해야 안전
2. **OnEnable/OnDisable 짝 불일치** — 메모리 누수!
3. **Update에서 물리 조작** — 프레임레이트 의존
4. **OnDestroy에서 정적 참조 해제 실수** — `if (Instance == this)` 체크
5. **FixedUpdate에서 GetKeyDown** — 입력 놓칠 수 있음

---

## 숨겨진 생명주기 메서드

- **Reset()** — 에디터에서 Component 처음 추가 시
- **OnValidate()** — Inspector 값 변경 시 (유효성 검사)
- **OnDrawGizmos() / OnDrawGizmosSelected()** — Scene View 시각화
- **OnApplicationPause(bool)** — 모바일 백그라운드
- **OnApplicationFocus(bool)** — 포커스 변화
- **OnBecameVisible() / OnBecameInvisible()** — 카메라 가시성

---

## 면접 답변 예시

```
Q: "MonoBehaviour 생명주기 호출 순서와 용도를 설명해주세요"

A: "초기화, 게임 루프, 종료 단계로 나뉩니다.

    초기화는 Awake → OnEnable → Start 순서입니다.
    Awake는 자기 자신 초기화와 GetComponent 캐싱,
    OnEnable은 이벤트 구독 (오브젝트 활성화마다 호출),
    Start는 모든 Awake가 끝난 후라 다른 오브젝트 참조가 안전합니다.

    게임 루프에서 FixedUpdate는 고정 시간으로 물리 처리,
    Update는 매 프레임 일반 로직과 입력 감지,
    LateUpdate는 카메라 추적처럼 다른 오브젝트의 최종 상태를
    읽어야 할 때 씁니다.

    종료는 OnDisable로 이벤트 해제, OnDestroy로 최종 정리입니다.
    OnEnable/OnDisable은 반드시 짝으로 써서 메모리 누수를 방지합니다.

    같은 단계 내 오브젝트 간 순서는 보장 안 되므로
    Script Execution Order로 제어하지만,
    이벤트나 초기화 패턴으로 해결하는 게 우선입니다."
```

---

## 면접 레벨별 기대

| 레벨 | 기대 |
|---|---|
| 주니어 | Awake/Start/Update 차이, 기본 용도 |
| 미들 | OnEnable/OnDisable 짝, FixedUpdate vs Update, 이벤트 구독 패턴 |
| 시니어 | Script Execution Order, Object Pooling과 OnEnable, 복잡한 초기화 |
| 리드 | 대규모 프로젝트 생명주기 관리, 의존성 설계 |

---

## 오답 노트

1. **Awake는 비활성 상태에서도 호출됨** — 단, OnEnable은 활성화될 때만.

2. **Start는 "모든 Awake가 끝난 후"** — 다른 오브젝트 참조는 Start에서.

3. **OnEnable/OnDisable은 반드시 짝** — 이벤트 누수, 파괴된 객체 접근 방지.

4. **FixedUpdate는 한 프레임에 0~N번** — 고사양에서 안 호출되거나, 저사양에서 여러 번.

5. **LateUpdate는 Update 전부 끝난 후** — 카메라 추적의 표준 위치.

6. **입력은 Update, 물리는 FixedUpdate** — GetKeyDown은 Update에서만 확실.

7. **같은 단계 내 오브젝트 순서 비결정** — Script Execution Order로 제어 가능하지만 남용 금지.

8. **OnDestroy는 보장 안 될 수 있음** — 앱 강제 종료 시. OnApplicationPause도 활용.

9. **OnEnable이 풀링에서 중요** — 오브젝트를 Destroy 대신 SetActive(false)로 반환하고 OnEnable에서 상태 리셋.

10. **OnValidate는 에디터 전용** — Inspector 값 변경 시 호출. 유효성 검사에 활용.
