# Unity 학습 기록 - 2026-04-03

## 세션 정보
- 분야: Unity
- 주제: Object Pooling → Instantiate/Destroy가 느린 이유 → 힙 vs 스택 → C#/C++ 메모리 비교 → 풀 구현
- 모드: learn
- 날짜: 2026-04-03

---

## 이야기의 시작: Object Pooling이란?

> "Object Pooling이 무엇이고, 왜 필요한지, 어떻게 구현하는지 설명해주세요."

**답: 총알이나 이펙트 같은 오브젝트를 프리팹으로 매번 Instantiate하면 최적화가 망한다. 그래서 미리 만들어놓고 Destroy/Instantiate 없이 재사용하는 로직이다.**

개념과 목적은 정확했다. 하지만 여기서 근본적인 의문이 생겼다.

> "근데 궁금한 게, Destroy/Instantiate를 할 때마다 최적화가 왜 망하는 거지? 생성/삭제가 큰 문제인 건 알았지만, 왜 별로인지에 대해서 생각을 안 해봤네."

---

## Instantiate/Destroy가 느린 진짜 이유

### 이유 1: 힙 메모리 할당의 비용

`Instantiate(bulletPrefab)` 한 번 호출하면 내부에서 이런 일이 벌어진다:

1. 힙에서 빈 공간을 찾음
2. GameObject를 힙에 할당
3. 모든 Component를 **개별적으로** 힙에 할당 (Transform, Rigidbody, Collider, Renderer, 커스텀 스크립트 각각)
4. 프리팹의 데이터를 새 인스턴스에 복사
5. Unity 내부 시스템에 등록 (물리 엔진에 Collider 등록, 렌더링에 Renderer 등록, Scene Hierarchy에 등록)
6. Awake(), OnEnable() 호출

총알 하나 쏠 때마다 이 과정이 전부 실행된다. 초당 10발이면 초당 10번.

### 이유 2: Garbage Collection — 진짜 문제

Destroy()를 호출하면 Unity 내부 시스템에서 등록을 해제하고, 힙 메모리를 "사용 안 함"으로 표시한다. 하지만 **즉시 메모리를 반환하지 않는다.** "쓰레기(Garbage)"로 남아있다.

쓰레기가 쌓이면 결국 GC(Garbage Collector)가 발동한다. GC는 **전체 힙을 스캔**해서 "이거 쓰레기? 아닌 거?" 하나하나 확인하고, 쓰레기만 골라서 메모리를 반환한다. 이 과정 동안 **게임이 멈춘다 (Stop-The-World).**

```
일반 프레임: 16ms (60fps)
GC 발동 시:  16ms + GC 시간 (수 ms ~ 수십 ms)
→ 플레이어가 "끊김"을 느낌 = 프레임 스파이크
```

Instantiate/Destroy를 반복하면 쓰레기가 계속 쌓이고, GC가 수시로 발동하면서 게임이 끊긴다.

Object Pooling이면 시작 시 1번만 할당하고, 이후 SetActive(true/false)로만 토글한다. 힙 할당이 없으니 쓰레기도 없고, GC가 발동하지 않는다.

### 이유 3: Unity 내부 시스템 등록/해제

Instantiate는 물리 엔진(PhysX)에 Collider/Rigidbody 등록, 렌더링 시스템에 Renderer 등록, Batching 그룹 재계산, Shadow Caster 등록 등을 한다. Destroy는 이 모든 걸 역순으로 해제한다.

반면 `SetActive(true/false)`는 이미 등록된 상태에서 **활성/비활성 플래그만 토글**한다. 등록/해제를 새로 하지 않으니 훨씬 가볍다.

### 이유 4: 메모리 파편화

반복적인 할당/해제로 메모리에 빈틈이 생긴다. 작은 빈틈들이 여기저기 흩어져 있으면 큰 객체를 할당할 연속 공간이 부족해진다. GC가 Compaction(압축)을 해야 할 수 있고, 이건 더 큰 비용이다.

---

## 힙 할당 자체는 느리지 않다?

여기서 또 하나의 발견이 있었다.

> "그렇다면 결국 힙 할당은 그렇게 큰 이슈는 아니네. 오히려 그걸 삭제하면서 생기는 GC가 큰 문제라는 거네. 나는 힙 할당을 왠만해서 배제하고 스택을 쓰라고 교육받았거든."

맞다. 하지만 **이유를 정확히** 알아야 한다.

### C#의 힙 할당은 사실 꽤 빠르다

C#의 관리형 힙은 **순차 할당(Bump Allocator)** 방식이다. "다음 할당 위치"를 가리키는 포인터가 있고, new를 하면 그 포인터를 요청 크기만큼 앞으로 옮기는 게 전부다. Free List 검색 같은 건 없다.

### C++의 malloc은 더 복잡하다

C++에서 malloc을 호출하면:
1. Free List(빈 공간 목록)를 검색해야 함 — O(n)
2. 할당 전략에 따라 어떤 빈 공간을 쓸지 결정 (First Fit, Best Fit 등)
3. 선택한 블록을 분할하고 메타데이터 기록
4. 인접 빈 블록 병합 체크
5. 멀티스레드면 Lock 필요

**할당만 놓고 보면 C#이 C++의 malloc보다 빠를 수 있다.** 포인터 하나 옮기는 것 vs Free List 검색이니까.

### 그러면 왜 "힙 할당을 피하라"고 교육받았는가?

```
힙 할당이 느려서가 아니다.
힙 할당을 하면 나중에 반드시 GC가 와서 그 쓰레기를 치워야 하기 때문이다.

힙 할당 = 빚을 지는 것
GC = 빚쟁이가 찾아오는 것

빚을 1번 지면? → 괜찮음
빚을 매 프레임 지면? → 빚쟁이가 수시로 옴 → 게임 끊김
```

스택이 좋은 진짜 이유도 "빨라서"가 아니라 **"GC가 필요 없어서"**다. 함수가 끝나면 스택 포인터만 되돌리면 자동으로 사라진다. 쓰레기가 안 생긴다.

---

## 그래도 C++이 메모리 성능 최고인 이유

> "근데 메모리 관련 작업에서는 C++이 최고 성능이라고 들었어. Rust 빼고."

맞다. "할당 속도"와 "메모리 성능"은 다른 이야기다. C#의 힙 할당이 빠르다는 건 "new를 호출하는 그 순간의 속도"만 비교한 것이고, C++이 최고라는 건 메모리를 "전체적으로 어떻게 관리하느냐"의 이야기다.

### 1. 메모리를 직접 제어할 수 있다

C#은 "이만큼 줘" → 관리형 힙에서 알아서 할당. C++은 "이 주소에, 이만큼, 이 방식으로" 개발자가 모든 걸 제어한다. 커스텀 할당자(Pool Allocator, Arena Allocator)로 게임에 최적화된 메모리 관리가 가능하다.

### 2. 캐시 친화적 배치 — 실제 성능에서 가장 큰 차이

현대 CPU에서 L1 캐시는 ~1ns, 메인 RAM은 ~100ns. **100배 차이.** 연속된 메모리에 데이터가 모여있으면 캐시 히트, 흩어져 있으면 캐시 미스.

```
C++ — 연속 배열:
│ x y z │ x y z │ x y z │ x y z │ ...│
모든 데이터가 나란히 → 캐시 히트 → 빠름

C# — MonoBehaviour 객체들:
힙에 흩어져 있음, 순서도 제각각
→ 매번 캐시 미스 → 매번 RAM까지 가야 함 → 느림
```

적 1000마리의 위치를 업데이트할 때, C++의 연속 배열은 ~1μs면 될 것을 C#의 흩어진 객체는 ~100μs가 걸릴 수 있다.

### 3. GC가 없다

개발자가 free/delete를 호출하면 즉시 해제. 예측 가능한 시점에 예측 가능한 비용. 프레임 스파이크가 없다.

### 4. 오버헤드가 없다

C#의 모든 힙 객체는 16바이트 헤더(Sync Block + Type Handle)가 붙는다. C++은 순수 데이터만 저장 가능.

### C++의 대가

메모리 누수, 댕글링 포인터, 이중 해제, 버퍼 오버플로우 등 버그 위험이 크고 개발 속도가 느리다. 그래서 AAA 게임 엔진 내부는 C++, 게임 로직은 C# 같은 구조가 많다. Unity도 엔진 내부는 C++, 사용자 코드는 C#이다.

### Rust는?

C++의 성능 + 메모리 안전성. 소유권(Ownership) 시스템으로 컴파일 타임에 메모리 버그를 차단한다. GC 없이 직접 제어 가능하면서도 안전하다. 다만 학습 곡선이 가파르고 게임 엔진 생태계가 아직 작다.

### Unity의 시도: DOTS/ECS

Unity도 C#의 메모리 한계를 알고 있어서 DOTS(Data-Oriented Technology Stack)를 만들었다. 같은 종류의 데이터를 NativeArray에 연속 배치해서 C++처럼 캐시 친화적 메모리 레이아웃을 구현한다.

---

## 숨겨진 힙 할당 — new를 안 써도 발생하는 것들

교육받은 대로 `new`를 피하더라도, 몰래 힙 할당이 발생하는 경우가 많다.

```csharp
void Update()  // ❌ 매 프레임 힙 할당!
{
    string name = "Player" + score.ToString();     // 문자열 결합
    var alive = enemies.Where(e => e.IsAlive);     // LINQ
    object obj = 42;                               // 박싱
    Debug.LogFormat("{0}", score);                  // params 배열
}
```

해결 패턴:
- **캐싱** — GetComponent를 Awake에서 1번만
- **재사용** — StringBuilder, List를 필드로 선언 후 Clear & 재사용
- **NonAlloc API** — `Physics.OverlapSphereNonAlloc` 등
- **직접 순회** — LINQ 대신 for문

---

## Object Pooling 구현

### 기본 구현: Queue 기반

```csharp
public class ObjectPool : MonoBehaviour
{
    [SerializeField] private GameObject prefab;
    [SerializeField] private int initialSize = 20;
    private Queue<GameObject> pool = new Queue<GameObject>();

    void Awake()
    {
        for (int i = 0; i < initialSize; i++)
        {
            GameObject obj = Instantiate(prefab);
            obj.SetActive(false);
            pool.Enqueue(obj);
        }
    }

    public GameObject Get()
    {
        if (pool.Count > 0)
        {
            GameObject obj = pool.Dequeue();
            obj.SetActive(true);
            return obj;
        }
        return Instantiate(prefab);  // 비상용
    }

    public void Return(GameObject obj)
    {
        obj.SetActive(false);
        pool.Enqueue(obj);
    }
}
```

시작 시 미리 생성하고, Get()으로 꺼내고, Return()으로 반납한다. Instantiate/Destroy 없이 SetActive만 토글.

### 기본 구현의 문제점

1. 타입별로 풀을 만들어야 함
2. 풀이 비었을 때 Instantiate → GC 유발 가능
3. Return을 까먹으면 풀이 고갈됨
4. 반납 시 오브젝트 상태 초기화를 안 하면 이전 데이터가 남음

### 개선: 제네릭 + IPoolable 인터페이스

```csharp
public interface IPoolable
{
    void OnGetFromPool();      // 꺼낼 때 초기화
    void OnReturnToPool();     // 반납할 때 정리
}
```

IPoolable을 구현한 클래스는 풀에서 꺼낼 때 자동으로 초기화되고, 반납할 때 자동으로 정리된다. 총알이면 `rb.linearVelocity = Vector3.zero`, `trail.Clear()` 같은 것들.

### 자동 반납 패턴

Return을 까먹는 걸 방지하기 위해 AutoReturn 컴포넌트를 붙여서 일정 시간 후 자동 반납되게 한다.

### Unity 2021+ 내장 ObjectPool API

```csharp
using UnityEngine.Pool;

pool = new ObjectPool<Bullet>(
    createFunc: () => Instantiate(prefab),
    actionOnGet: bullet => bullet.gameObject.SetActive(true),
    actionOnRelease: bullet => bullet.gameObject.SetActive(false),
    actionOnDestroy: bullet => Destroy(bullet.gameObject),
    collectionCheck: true,     // 중복 반납 방지
    defaultCapacity: 20,
    maxSize: 100               // 최대 크기 (넘으면 Destroy)
);
```

직접 구현 대비 장점:
- 중복 반납 체크가 내장 (같은 오브젝트를 두 번 Release하면 에러)
- 최대 크기 제한으로 메모리 낭비 방지
- 생성/꺼내기/반납/파괴 콜백이 깔끔하게 분리

### 실무 아키텍처: 중앙 PoolManager

실무에서는 PoolManager 하나에서 모든 풀을 중앙 관리한다. Dictionary로 풀을 key(문자열)로 구분하고, RegisterPool로 등록, Get/Return으로 사용한다.

```
PoolManager
├── "Bullet" Pool      (Active 5, Inactive 25)
├── "HitEffect" Pool   (Active 2, Inactive 13)
├── "Enemy" Pool       (Active 8, Inactive 2)
└── "DamageText" Pool  (Active 3, Inactive 17)
```

---

## 전체 여정 정리

```
Object Pooling이 뭔지 → "미리 만들어서 재사용"
  → "근데 왜 Instantiate/Destroy가 느리지?"
    → 힙 할당 + GC + 시스템 등록/해제 + 파편화
      → "힙 할당 자체는 빠르지 않나?"
        → 맞다, C#의 Bump Allocator는 빠르다
        → 문제는 할당이 아니라 나중에 오는 GC
          → "근데 C++이 메모리 성능 최고 아니야?"
            → 맞다, 직접 제어 + 캐시 친화 + GC 없음
            → "할당 속도"와 "메모리 성능"은 다른 이야기
              → Object Pooling 구현
                → 기본 Queue → 제네릭 + IPoolable → Unity ObjectPool API
                  → 실무 PoolManager 아키텍처
```

| 깊이 | 내용 | 면접 레벨 |
|------|------|----------|
| Object Pooling 개념 | 미리 생성, SetActive로 재사용 | 주니어 |
| Instantiate가 느린 이유 | 힙 할당, GC, 시스템 등록 비용 | 주니어~미들 |
| 힙 vs 스택, GC 원리 | Bump Allocator, Stop-The-World | 미들 |
| C#/C++ 메모리 비교 | 캐시 친화성, 직접 제어 | 시니어 |
| 숨겨진 힙 할당 | LINQ, 문자열, 박싱, params | 시니어 |
| Unity ObjectPool API | 내장 API 활용 | 미들 |
| 풀 시스템 설계 | PoolManager, 프리워밍, 크기 전략 | 시니어~리드 |

---

## 오답 노트

틀린 문제는 없었으나, 이 세션에서 새로 정리된 인사이트:

1. **"힙 할당이 느려서 피하라"는 반만 맞다** — 할당 자체는 빠르다(Bump Allocator). 진짜 문제는 할당이 유발하는 GC다. "빚을 지는 건 쉽지만, 빚쟁이가 찾아오면 아프다."

2. **C#의 할당이 C++ malloc보다 빠를 수 있다** — 하지만 이건 "할당 순간의 속도"만 비교한 것. 전체 메모리 성능(직접 제어, 캐시 친화, GC 없음)에서는 C++이 압도적이다. "할당 속도"와 "메모리 성능"을 구분해야 한다.

3. **Object Pooling의 본질** — 단순히 "재사용해서 빠르게"가 아니라, "힙 할당을 0으로 만들어서 GC를 원천 차단하는 전략"이다. SetActive 토글은 시스템 등록/해제도 피하는 부수 효과.
