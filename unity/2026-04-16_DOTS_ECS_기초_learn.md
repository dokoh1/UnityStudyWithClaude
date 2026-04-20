# Unity 학습 기록 - 2026-04-16 (Q16)

## 세션 정보
- 분야: Unity
- 주제: DOTS/ECS → OOP vs DOD → Entity/Component/System → Archetype/Chunk → 실전 예시
- 모드: learn
- 날짜: 2026-04-16

---

## DOTS란?

**Data-Oriented Technology Stack** = 데이터 지향 기술 스택

세 기술의 묶음:
- **ECS** — 데이터를 연속 메모리에 배치하는 아키텍처
- **Job System** — 멀티코어 병렬 처리 (Q4에서 다룸)
- **Burst Compiler** — LLVM 기반 초고속 코드 생성 (Q4에서 다룸)

**별도 패키지 설치 필요** (기본 Unity에 포함되지 않음):
- `com.unity.entities`, `com.unity.entities.graphics`, `com.unity.burst`, `com.unity.collections`

---

## 왜 필요한가?

### MonoBehaviour의 한계

적 1000마리를 MonoBehaviour로 만들면:
- 각 GameObject의 데이터가 메모리에 **흩어져** 있음
- CPU가 데이터 읽을 때마다 **캐시 미스** (L1: ~1ns vs RAM: ~100ns)
- Update() 1000번 호출 (C++↔C# 경계 1000번)

### DOTS의 해결

- 같은 종류 데이터를 **연속 메모리**에 배치 → 캐시 히트
- System이 데이터를 **한꺼번에** 순회 → C++ 경계 최소화
- Job System으로 **멀티코어**, Burst로 **SIMD**
- 결과: 10000개 오브젝트를 **0.01ms**에 처리 (MonoBehaviour 대비 수백~수천 배)

---

## OOP vs DOD

### OOP (기존 MonoBehaviour)
```
"각 객체가 자기 데이터와 행동을 가진다"
Enemy 클래스 = hp + speed + Move() + Die()
→ 직관적이지만 데이터가 흩어짐
```

### DOD (DOTS)
```
"같은 종류의 데이터를 모아서 한꺼번에 처리"
HP 배열:   [100, 80, 50, ...]   ← 연속!
Speed 배열: [5.0, 3.0, 7.0, ...] ← 연속!
MoveSystem: 전체 배열을 순회하며 처리
→ 캐시 친화적, 극도로 빠름
```

비유: OOP = 각 사람이 자기 서랍에서 필요한 것 꺼냄 (여기저기 이동). DOD = 같은 종류를 한 서랍에 모아서 쭉 처리 (한 곳에서).

---

## ECS의 세 가지 요소

### Entity = "ID"
- GameObject가 아님! 그냥 **숫자(index + version)**
- 극도로 가벼움
- "이 Entity에 이런 Component가 있다"는 매핑의 식별자

### Component = "데이터만" (struct!)
```csharp
public struct HP : IComponentData { public int value; }
public struct Speed : IComponentData { public float value; }
// 로직 없음! struct! 연속 메모리 배치 가능!
```

### System = "로직만"
```csharp
[BurstCompile]
public partial struct MoveSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        foreach (var (transform, speed) in
            SystemAPI.Query<RefRW<LocalTransform>, RefRO<Speed>>())
        {
            transform.ValueRW.Position += direction * speed.ValueRO.value * dt;
        }
    }
}
// "Position + Speed를 가진 모든 Entity"를 한꺼번에 처리
```

### 관계
- Entity = "존재한다" (ID)
- Component = "이런 데이터를 가진다"
- System = "이런 데이터를 가진 것들을 이렇게 처리한다"
- System이 "어떤 Component 조합"을 쿼리 → 해당 Entity만 자동 필터링

---

## Archetype과 Chunk — 메모리의 마법

### Archetype
같은 Component 조합을 가진 Entity들의 그룹.
- Entity 0,1,5: [HP, Speed, Position] → Archetype A
- Entity 2,3: [HP, Position] → Archetype B

### Chunk (16KB)
각 Archetype의 데이터를 **16KB 청크** 단위로 저장.
- L1 캐시에 딱 맞는 크기
- 한 청크를 통째로 캐시에 올림
- Entity당 20B면 ~800개가 한 청크에!

```
Chunk 내부:
HP:       [100][80][50][100]...   ← 연속!
Speed:    [5.0][3.0][7.0][5.0]... ← 연속!
Position: [(1,0,0)][(5,0,3)]...   ← 연속!
```

---

## ECS는 GameObject가 아니다!

### 에디터 vs 런타임

**에디터에서**: 평소처럼 GameObject로 배치 + Authoring 컴포넌트 추가
**Play 누르면**: GameObject가 Entity로 **자동 변환** (Baking)

```
에디터:
  Enemy (GameObject) + EnemyAuthoring (MonoBehaviour)
    HP: 100, Speed: 5

Play →  Baking (자동 변환):
  Entity + [HP{100}, Speed{5}, LocalTransform, EnemyTag]
  
→ 디자이너는 평소처럼 씬에 배치!
→ 런타임에서만 Entity로 동작!
```

### Authoring + Baker 패턴

```csharp
// Authoring (에디터용)
public class EnemyAuthoring : MonoBehaviour
{
    public int hp = 100;      // Inspector에서 설정
    public float speed = 5f;
}

// Baker (변환기)
public class EnemyBaker : Baker<EnemyAuthoring>
{
    public override void Bake(EnemyAuthoring authoring)
    {
        Entity entity = GetEntity(TransformUsageFlags.Dynamic);
        AddComponent(entity, new HP { value = authoring.hp });
        AddComponent(entity, new Speed { value = authoring.speed });
    }
}
```

---

## 현업에서의 사용 현황 (2026년)

```
대부분의 프로젝트: MonoBehaviour (90%+)
DOTS 전체 적용: 매우 소수
하이브리드 (일부만 DOTS): 점점 증가

쓰이는 곳:
├── 대규모 RTS (유닛 수천 개)
├── 시뮬레이션 게임
├── 총알/파티클 대량 처리
├── 서버 사이드 시뮬레이션
└── 특정 시스템만 ECS

안 쓰는 곳:
├── 소규모 게임
├── UI 중심 게임
├── 프로토타입
├── 팀에 ECS 경험 없음
└── 에셋스토어 의존 높은 프로젝트
```

하이브리드가 실무 표준:
- 캐릭터/UI: MonoBehaviour (소수, 복잡)
- 적 대량/총알: ECS (수천~수만, 단순)

---

## 실전 예시: 총알 10000개 시스템

### 전체 구조

```
Assets/
├── Scripts/
│   ├── Components/
│   │   └── BulletComponents.cs      (데이터)
│   ├── Systems/
│   │   ├── BulletMoveSystem.cs      (이동)
│   │   ├── BulletLifetimeSystem.cs  (수명)
│   │   └── BulletSpawnSystem.cs     (스폰)
│   └── Authoring/
│       ├── BulletAuthoring.cs       (프리팹 설정)
│       └── BulletSpawnConfigAuthoring.cs (스포너)
├── Prefabs/
│   └── BulletPrefab.prefab
└── Scenes/
    └── GameScene.unity
```

### Component (데이터)
```csharp
public struct BulletMovement : IComponentData
{
    public float3 direction;
    public float speed;
}
public struct BulletLifetime : IComponentData
{
    public float remainingTime;
}
public struct BulletTag : IComponentData { }
```

### Authoring + Baker (에디터 설정)
```csharp
public class BulletAuthoring : MonoBehaviour
{
    public float speed = 20f;
    public float lifetime = 3f;
}
public class BulletBaker : Baker<BulletAuthoring>
{
    public override void Bake(BulletAuthoring authoring)
    {
        Entity entity = GetEntity(TransformUsageFlags.Dynamic);
        AddComponent(entity, new BulletMovement { direction = new float3(0,0,1), speed = authoring.speed });
        AddComponent(entity, new BulletLifetime { remainingTime = authoring.lifetime });
        AddComponent(entity, new BulletTag());
    }
}
```

### System (로직)
```csharp
[BurstCompile]
public partial struct BulletMoveSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        float dt = SystemAPI.Time.DeltaTime;
        foreach (var (transform, movement) in
            SystemAPI.Query<RefRW<LocalTransform>, RefRO<BulletMovement>>())
        {
            transform.ValueRW.Position += movement.ValueRO.direction * movement.ValueRO.speed * dt;
        }
    }
}
```

### 씬 설정
1. BulletPrefab (GameObject + MeshFilter + MeshRenderer + BulletAuthoring)
2. BulletSpawner (GameObject + BulletSpawnConfigAuthoring → prefab 연결)
3. Play → 자동 Baking → 매 프레임 100개 스폰 → 10000개 총알이 0.05ms로 동작!

### 성능 비교
- MonoBehaviour 6000개: ~8ms
- ECS + Burst 6000개: ~0.05ms
- **160배 빠름!**

---

## 전체 흐름

```
[에디터] 디자이너가 평소처럼 배치
    ↓
[Play] Baking: GameObject → Entity 자동 변환
    ↓
[런타임] System이 데이터를 한꺼번에 처리
    ↓
[결과] 수만 개 오브젝트가 0.1ms에 동작
```

---

## MonoBehaviour vs ECS 비교

| | MonoBehaviour | ECS (DOTS) |
|---|---|---|
| 사고방식 | 객체 지향 (OOP) | 데이터 지향 (DOD) |
| 데이터 위치 | 힙에 흩어짐 | 청크에 연속 |
| 로직 위치 | Component 안 | System (분리) |
| 생성 단위 | GameObject | Entity (ID) |
| 10000 오브젝트 | ~10ms | ~0.01ms |
| 학습 곡선 | 낮음 | 매우 높음 |
| 에셋스토어 | 풍부 | 거의 없음 |

---

## 면접 레벨별 기대

| 레벨 | 기대 |
|---|---|
| 주니어 | DOTS 존재 인지, MonoBehaviour와 다르다는 것 |
| 미들 | Entity/Component/System 역할, 왜 빠른지 (캐시) |
| 시니어 | Archetype/Chunk, Job+Burst 연결, 하이브리드 판단 |
| 리드 | 프로젝트에 DOTS 도입 판단, 팀 역량 고려, 아키텍처 설계 |

---

## 오답 노트

1. **DOTS는 별도 패키지** — 기본 Unity에 포함 안 됨. Package Manager에서 설치.

2. **ECS Entity ≠ GameObject** — Entity는 숫자(ID)일 뿐. 하지만 에디터에서는 평소처럼 GameObject로 배치하고 Play 시 자동 변환(Baking).

3. **Component는 struct, 로직 없음** — class가 아니라 struct라 연속 메모리 배치 가능. 로직은 System에.

4. **Archetype = 같은 Component 조합 그룹** — 16KB Chunk에 연속 저장 → L1 캐시에 딱 맞음.

5. **현업에서는 하이브리드가 표준** — 전체 ECS는 드묾. "대량 단순 반복"만 ECS, 나머지는 MonoBehaviour.

6. **DOTS의 진짜 위력은 ECS + Job + Burst 조합** — 연속 메모리(ECS) + 멀티코어(Job) + SIMD(Burst) = 수천 배 성능.

7. **학습 곡선이 매우 가파름** — 면접에서 "개념을 안다"만으로도 플러스. "써봤다"면 시니어급.
