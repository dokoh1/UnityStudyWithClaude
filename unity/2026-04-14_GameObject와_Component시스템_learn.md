# Unity 학습 기록 - 2026-04-14 (Q11)

## 세션 정보
- 분야: Unity
- 주제: GameObject/Component → 컴포지션 → GetComponent 성능 → Tag/Layer → Prefab Variant → Null 체크 심화
- 모드: learn
- 날짜: 2026-04-14

---

## GameObject와 Component의 본질

### GameObject는 "빈 껍데기"

- GameObject는 아무것도 안 함, Component를 담는 컨테이너일 뿐
- GameObject의 실제 내용: 이름, 태그, 레이어, 활성 상태, Transform + Component 리스트
- **Transform만이 유일한 필수 Component** (Destroy 불가)

### Component의 역할

Component = 기능의 단위
- Transform: 위치/회전/크기
- MeshRenderer: 렌더링
- Collider/Rigidbody: 물리
- Animator: 애니메이션
- AudioSource: 소리
- MonoBehaviour: 커스텀 스크립트

GameObject는 Component를 조합해서 의미를 가짐.

---

## 컴포지션 기반 설계

### 상속 vs 컴포지션

```
상속: "A는 B다" (is-a)
컴포지션: "A는 B를 가진다" (has-a)
```

### 상속 기반의 문제

- 다중 상속 불가 (C#)
- "날 수 있는 적 + 폭발 가능한 적" 조합 어려움
- 코드 중복 발생

### 컴포지션의 해결

Unity 방식:
```
GameObject "Enemy":
├── Transform
├── FlyingMovement    ← 날 수 있음
├── SwimmingMovement  ← 수영 가능
├── HealthSystem
└── Explosive         ← 폭발 가능

→ 필요한 기능만 조립
→ 런타임에 추가/제거 가능
→ 다중 상속 문제 없음
```

### 장점
1. **유연성** — 기능을 자유롭게 조합
2. **재사용성** — 하나의 Component를 여러 곳에서
3. **단일 책임 원칙** — 각 Component가 하나의 역할
4. **협업 용이** — 디자이너가 조립 가능

### 단점
1. 참조 분산 → GetComponent 성능 이슈
2. 의존성 추적 어려움 → RequireComponent로 일부 해결
3. 초보자 혼란

---

## GetComponent의 성능 이슈 (깊이)

### Unity 내부 구조

C# Component는 C++ 네이티브 객체의 래퍼:
```
C# 객체 (매니지드):
  IntPtr m_CachedPtr  ← C++ 객체 포인터

C++ 객체 (네이티브):
  실제 Unity 엔진 데이터
```

### GetComponent 내부 동작

```
C# GetComponent<T>()
  → P/Invoke (매니지드↔네이티브 전환)
  → C++ 엔진에서 Component 리스트 선형 검색
  → 타입 계층 확인
  → 결과를 C# 객체로 래핑하여 반환
```

### 비용 분석 (1회 호출)

| 단계 | 비용 |
|---|---|
| P/Invoke (매니지드↔네이티브 전환) | ~50ns |
| typeof(T) 리플렉션 | ~20ns |
| 컴포넌트 리스트 순회 | ~30~200ns |
| 결과 래핑 | ~50ns |
| **총합** | **~150~400ns** |

**진짜 비용은 P/Invoke 오버헤드** (선형 검색이 아님 — 컴포넌트 보통 5~10개라 작음)

### 매 프레임 호출 시 재앙

```
1000개 오브젝트 × 60fps = 60,000회/초
60,000 × 300ns = 18ms
→ 60fps 예산 16.67ms 초과!
→ 프레임 드랍!
```

---

## TryGetComponent가 왜 더 빠른가

### Unity의 "Fake Null" 문제

```
Destroy(obj) 호출 시:
  1. C++ 객체 파괴 (메모리 해제)
  2. C# 객체는 살아있음 (GC 수거 전까지)
  3. m_CachedPtr는 유효하지 않은 값
  → 이게 "fake null"
```

Unity의 UnityEngine.Object는 `==` 연산자를 오버로드해서 네이티브 생존 여부 체크:
- `if (obj == null)` → 커스텀 오버로드 호출 → 네이티브 체크 포함 → ~20ns 추가

### 비교

**GetComponent + null 체크:**
1. P/Invoke GetComponent 호출
2. 결과 래핑
3. `== null` 체크 (커스텀 오버로드 → 네이티브 재확인)
4. 총: ~170ns

**TryGetComponent:**
1. P/Invoke TryGetComponent 호출
2. 네이티브에서 검색 + 존재 플래그 반환
3. out 파라미터로 값 전달
4. 별도 null 체크 불필요
5. 총: ~130ns

**~20~30% 빠름**. 하지만 캐싱에 비하면 여전히 ~2000배 느림.

---

## 실무 6가지 Rule (자세히)

### Rule 1: Update/FixedUpdate에서 GetComponent 금지

이유: 매 프레임 × 오브젝트 수만큼 P/Invoke 오버헤드 누적.
1000개 × 60fps = 18ms/프레임 낭비.

### Rule 2: 같은 GameObject의 Component는 SerializeField 또는 Awake 캐싱

```csharp
// 방법 A: Awake 캐싱
private Rigidbody rb;
void Awake() { rb = GetComponent<Rigidbody>(); }

// 방법 B: SerializeField (추천)
[SerializeField] private Rigidbody rb;
```

### Rule 3: 다른 GameObject의 Component는 SerializeField가 최선

GameObject.Find, FindObjectOfType은 씬 전체 검색으로 훨씬 비쌈.

```csharp
// ❌ Find 계열 금지
GameObject.Find("Player").GetComponent<Health>();

// ✅ Inspector에서 연결
[SerializeField] private Health playerHp;

// ✅ 이벤트 기반 (더 좋음)
player.OnHealthChanged += UpdateBar;
```

### Rule 4: 동적으로 찾아야 할 때만 TryGetComponent

상대가 뭐 가졌는지 모를 때 (충돌 상대 등):
```csharp
void OnCollisionEnter(Collision col)
{
    if (col.gameObject.TryGetComponent(out Health h))
        h.TakeDamage(10);
}
```

### Rule 5: GetComponentsInChildren/InParent는 극도로 비쌈 - 캐싱 필수

```
GetComponent: ~300ns
GetComponentInChildren (자식 100개): ~30,000ns!
```

자식/부모 트리 전체를 재귀 검색. 반드시 캐싱하고, 자식 구조가 바뀌면 재캐싱.

### Rule 6: 매니저는 싱글톤 또는 ScriptableObject 참조

```csharp
// 싱글톤
GameManager.Instance.AddScore(10);

// ScriptableObject 참조
[SerializeField] private GameData gameData;
gameData.score.Add(10);
```

---

## Tag와 Layer

### Tag (분류용, 문자열)
- **목적**: 게임 로직에서 카테고리 식별
- **사용**: `CompareTag("Player")` (== 비교는 GC 발생)
- **개수**: 무제한
- **예시**: "Enemy", "Player", "Respawn"

### Layer (시스템용, 비트마스크)
- **목적**: 물리, 렌더링, 카메라 시스템 필터링
- **개수**: 32개
- **성능**: 비트 연산으로 매우 빠름

### Layer 주요 활용
1. **Physics Layer Matrix** — Enemy끼리 충돌 안 하게 등
2. **Raycast Layer Mask** — 특정 Layer만 검사
3. **Camera Culling Mask** — 카메라별로 다른 오브젝트 그리기
4. **Light Culling Mask** — 조명 범위 제한

### 비교

| | Tag | Layer |
|---|---|---|
| 목적 | 게임 로직 분류 | 시스템 최적화 |
| 개수 | 무제한 | 32개 |
| 성능 | CompareTag로 사용 | 비트 연산, 매우 빠름 |
| 비교 | 문자열 | 비트마스크 |

---

## Prefab Variant (특별 설명)

### Prefab Variant란?

**기반 Prefab의 자식 버전**. 일반 Prefab을 상속하는 개념.

비유: iPhone (기본) → iPhone Pro/Mini/SE (Variant)

### 왜 필요한가?

```
적 3종류가 필요:
  - Goblin, Orc, Troll

방법 1: 별도 프리팹 3개
  → 공통 기능 수정 시 3번 작업

방법 2: Prefab Variant
  Enemy.prefab (기본)
  ├── Goblin.prefab (Variant)
  ├── Orc.prefab (Variant)
  └── Troll.prefab (Variant)
  → 공통 수정 1번으로 전부 반영!
```

### 만드는 법

Project 창에서 Enemy.prefab 우클릭 → Create > Prefab Variant

### Variant에서 할 수 있는 것

1. **값 오버라이드** (덮어쓰기)
   - Enemy의 HP: 100 → Goblin에서 50으로 변경 (파란색 표시)

2. **새 Component 추가**
   - Goblin에 Bow 추가 (Enemy에는 없는 것)

3. **자식 GameObject 추가**
   - Orc에 Armor, LargeAxe 자식 추가

4. **Component 제거** (제한적)
   - 기본 Prefab의 Component는 보통 제거 불가
   - Disable로 처리

### 실제 예시

```
기본 Enemy.prefab:
  Transform, MeshRenderer, Collider, Rigidbody
  HealthSystem (HP: 100)
  EnemyAI (Speed: 5)

Goblin (Variant):
  ├─ 상속: Transform, Collider, Rigidbody, ...
  ├─ 오버라이드: HP 50, Speed 8
  └─ 추가: BowAttack + Bow 자식

Orc (Variant):
  ├─ 상속: ...
  ├─ 오버라이드: HP 200, Speed 3
  └─ 추가: TwoHandedWeapon + GreatAxe

Boss (Variant):
  ├─ 상속: ...
  ├─ 오버라이드: HP 5000
  └─ 추가: PhaseSystem + BossSpecialAttack
```

### Variant vs Nested Prefab

- **Variant** = Prefab의 상속 (같은 종류의 변형)
- **Nested** = Prefab 안에 다른 Prefab 포함 (구성 관계)
- 예: Tank.prefab 안에 Turret.prefab (Nested)
- 예: Goblin은 Enemy의 Variant

---

## 오버로드(Overload)가 뭐야

### 정의

**"같은 이름으로 다른 동작을 정의하는 것"**

두 종류:
1. **메서드 오버로드** — 같은 이름, 다른 파라미터
2. **연산자 오버로드** — +, -, ==, != 같은 연산자 재정의

### 예시

```csharp
// 메서드 오버로드
public int Add(int a, int b) { ... }
public float Add(float a, float b) { ... }
public int Add(int a, int b, int c) { ... }

// 연산자 오버로드
public class Vector2D
{
    public static Vector2D operator +(Vector2D a, Vector2D b) { ... }
    public static bool operator ==(Vector2D a, Vector2D b) { ... }
}

// Unity의 == 오버로드
public class Object
{
    public static bool operator ==(Object x, Object y)
    {
        return CompareBaseObjects(x, y);  // 네이티브 생존 체크 포함
    }
}
```

### 비유

"열다"라는 말이 문맥에 따라 다름:
- 문을 열다, 마음을 열다, 가게를 열다
- 같은 이름, 다른 동작

C#의 `==`도 클래스마다 다르게 정의 가능.

---

## C#의 `==` vs `is` 차이

### 핵심 차이

| | == | is |
|---|---|---|
| 오버로드 | 가능 | 불가 (언어 내장) |
| 목적 | 값/참조 비교 | 타입 체크, 순수 null |
| 동작 | 클래스마다 다를 수 있음 | 항상 동일 |
| 성능 | ~5~10ns (오버로드 시) | ~1ns |

### 용도

```csharp
// == : 값/참조 비교 (오버로드 가능)
if (a == b)  // 클래스가 정의한 비교

// is : 타입 체크 또는 순수 null 체크
if (obj is string)          // 타입 체크
if (obj is string s)        // 타입 체크 + 캐스트 (C# 7+)
if (obj is null)            // 순수 null
if (obj is not null)        // C# 9+
```

### 예시로 차이 보기

```csharp
public class Money
{
    public int amount;
    public static bool operator ==(Money a, Money b)
    {
        // 값 비교로 재정의
        return a.amount == b.amount;
    }
}

Money wallet1 = new Money { amount = 1000 };
Money wallet2 = new Money { amount = 1000 };

wallet1 == wallet2   // true (값 비교, 오버로드)
wallet1 is Money     // true (타입)
ReferenceEquals(wallet1, wallet2)  // false (다른 객체)

Money empty = null;
empty == null        // true (오버로드가 처리)
empty is null        // true (언어 내장, 항상 순수 null)
```

---

## Unity와 C#의 Null 체크 종합

### 이해 요약

```
Destroy(obj) → C++ 객체 파괴, C# 주소는 GC 돌기 전까지 유지
→ Unity는 == 연산자를 오버로드해서 Fake Null 감지
→ is null은 C# 순수 null 체크라 Fake Null 놓침

규칙:
  Unity 객체 (GameObject, Component, SO 등) → == null 사용
  일반 C# 객체 → is null 사용 (권장)
```

### 위험한 상황

```csharp
GameObject obj = ...;
Destroy(obj);

if (obj == null) { }       // ✅ true (파괴 감지)
if (obj is null) { }       // ❌ false (놓침!)
if (obj?.transform) { }    // ❌ ?.도 순수 null 체크라 위험
obj ?? fallback;           // ❌ ??도 순수 null 체크
```

### 올바른 사용

```csharp
// Unity 객체
if (unityObj == null) return;
if (unityObj != null) { }

// 일반 C# 객체
if (obj is null) return;
if (obj is not null) { }
```

---

## 면접 답변 예시

```
Q: "Unity의 GameObject와 Component 시스템을 설명해주세요"

A: "GameObject는 씬에 배치되는 컨테이너이고, Component는 기능 단위입니다.
    GameObject 자체는 아무 기능도 없고, Transform만이 유일한 필수 Component입니다.

    이 구조는 컴포지션 기반 설계를 가능하게 합니다. 상속이었다면
    '날면서 폭발하는 적' 같은 조합이 다중 상속 문제로 어려웠겠지만,
    Component 조합으로 자유롭게 만들 수 있습니다.

    GetComponent는 P/Invoke 오버헤드가 크고 선형 검색이라
    매 프레임 호출하면 안 됩니다. Awake에서 캐싱하거나
    SerializeField로 Inspector에서 연결하거나
    TryGetComponent로 null 체크와 가져오기를 동시에 합니다.

    Tag는 문자열 기반 게임 로직 분류용이고,
    Layer는 32개 비트마스크로 물리 충돌, Raycast, 카메라 Culling에
    쓰이는 시스템 최적화 도구입니다.
    Tag는 CompareTag로 GC를 피하고,
    Layer는 Physics Layer Matrix로 불필요한 충돌 체크를 제거합니다."
```

---

## 면접 레벨별 기대

| 레벨 | 기대 |
|---|---|
| 주니어 | GameObject/Component, Tag/Layer 기본, GetComponent 성능 |
| 미들 | 컴포지션 원칙, 캐싱 패턴, CompareTag, LayerMask |
| 시니어 | TryGetComponent, Prefab Variant, Unity Null 체크 특수성 |
| 리드 | 대규모 프로젝트 Component 아키텍처, 의존성 설계 |

---

## 오답 노트

1. **GameObject는 빈 컨테이너** — Transform만 필수, 나머지는 Component 조립.

2. **컴포지션 > 상속** — Unity가 이 철학으로 설계됨. 다중 상속 문제 해결.

3. **GetComponent의 진짜 비용은 P/Invoke** — 선형 검색 비용보다 매니지드↔네이티브 전환이 큼.

4. **Fake Null 문제** — Destroy 후 C# 주소는 살아있음. Unity가 == 오버로드로 해결.

5. **Unity 객체는 == null, 일반 C#은 is null** — 이유를 이해하고 구분해서 사용.

6. **`??`와 `?.`도 순수 null 체크** — Unity 객체에 쓰면 Fake Null 놓침.

7. **Tag vs Layer는 용도가 다름** — Tag는 로직 분류, Layer는 시스템 최적화.

8. **Prefab Variant는 Prefab의 상속** — 공통 + 차이점 분리로 유지보수 편의.

9. **오버로드는 같은 이름에 다른 동작** — 연산자/메서드 모두 재정의 가능.

10. **is는 타입 체크 + 순수 null, ==는 오버로드 가능한 비교** — 용도가 다름.
