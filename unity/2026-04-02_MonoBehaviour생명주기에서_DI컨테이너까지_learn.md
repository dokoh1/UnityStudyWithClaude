# Unity 학습 기록 - 2026-04-02

## 세션 정보
- 분야: Unity
- 주제: MonoBehaviour 생명주기 → 초기화 아키텍처 → DI 컨테이너
- 모드: learn
- 날짜: 2026-04-02

---

## 이야기의 시작: Awake()와 Start()

모든 것은 하나의 질문에서 시작됐다.

```csharp
public class PlayerHealth : MonoBehaviour
{
    private GameManager gm;

    void Awake()
    {
        gm = FindObjectOfType<GameManager>();
        gm.RegisterPlayer(this);
    }
}
```

> "이 코드에서 어떤 문제가 발생할 수 있을까?"

**답: Awake()에서 다른 오브젝트를 참조하면 안 된다. Start()로 옮겨야 한다.**

이유는 간단하다. Awake()끼리의 실행 순서는 보장되지 않기 때문이다. `GameManager.Awake()`가 아직 실행되지 않은 상태에서 `PlayerHealth.Awake()`가 먼저 실행될 수 있다. 그러면 GameManager 내부의 리스트가 초기화되지 않은 상태에서 `RegisterPlayer()`를 호출하게 되고, NullReferenceException이 발생한다.

그래서 원칙이 생겼다:
> **Awake() = 자기 자신 초기화, Start() = 다른 오브젝트 참조**

---

## 첫 번째 궁금증: "에디터에서 GameManager를 붙인 채 실행하면 NullReference 안 뜨지 않아?"

좋은 질문이었다. 맞다 — `FindObjectOfType<GameManager>()`는 씬에 오브젝트가 존재하기만 하면 찾아온다. Awake() 실행 여부와 무관하게. 그래서 `gm` 자체는 null이 아니다.

**진짜 문제는 그 다음 줄이다:**

```csharp
gm.RegisterPlayer(this);
```

GameManager.Awake()에서 내부 리스트를 초기화한다면:

```csharp
public class GameManager : MonoBehaviour
{
    private List<PlayerHealth> players;

    void Awake()
    {
        players = new List<PlayerHealth>();  // 아직 실행 안 됐을 수 있음
    }

    public void RegisterPlayer(PlayerHealth p)
    {
        players.Add(p);  // players가 null → NullReferenceException!
    }
}
```

- `PlayerHealth.Awake()`가 먼저 실행되면 → `players`가 아직 null → 터짐
- `GameManager.Awake()`가 먼저 실행되면 → 정상 작동

**NullReference가 뜰 수도 있고 안 뜰 수도 있다.** 이게 더 위험하다. 재현이 어려운 버그가 되니까.

---

## 면접에서 이 주제를 얼마나 깊이 파는가

### Level 1 (주니어): 기본 질문
> "Awake()에서 다른 오브젝트를 참조하면 왜 문제인가?"

### Level 2 (꼬리질문): 해결 방법
> "Awake()끼리의 실행 순서를 보장하는 방법은?"
> → Script Execution Order, `[DefaultExecutionOrder]` 어트리뷰트

### Level 3 (미들): 설계 판단
> "Script Execution Order를 10개 이상의 스크립트에 지정하고 있다면, 이 설계에 어떤 문제가 있는가?"

### Level 4 (시니어): Unity 내부 동작
> "MonoBehaviour의 Awake/Start 말고, 초기화 순서를 완전히 제어하려면?"

### Level 5 (리드): 아키텍처 설계
> "MonoBehaviour 생명주기에 의존하지 않는 초기화 아키텍처를 설계한다면?"

---

## 두 번째 궁금증: "Start()끼리도 꼬일 수 있잖아. 어떻게 해결해?"

Awake/Start 분리 패턴으로 대부분 해결되지만, Start끼리도 순서가 보장되지 않는다.

### 해결 방법 1: 이벤트 기반 (가장 권장)

순서에 의존하지 않고, "준비되면 알려줘" 패턴:

```csharp
public class GameManager : MonoBehaviour
{
    public static event Action OnAllPlayersRegistered;
    
    public void RegisterPlayer(PlayerHealth p)
    {
        players.Add(p);
        if (players.Count >= expectedPlayers)
            OnAllPlayersRegistered?.Invoke();
    }
}

public class UIManager : MonoBehaviour
{
    void OnEnable()  { GameManager.OnAllPlayersRegistered += InitUI; }
    void OnDisable() { GameManager.OnAllPlayersRegistered -= InitUI; }
}
```

### 해결 방법 2: 코루틴으로 대기

```csharp
public class UIManager : MonoBehaviour
{
    IEnumerator Start()  // Start는 코루틴으로 만들 수 있다!
    {
        yield return new WaitUntil(() => GameManager.Instance.GetPlayerCount() > 0);
        InitUI();
    }
}
```

### 해결 방법 3: Bootstrap 패턴

```csharp
public class Bootstrap : MonoBehaviour
{
    void Start()
    {
        GameManager.Instance.Init();
        PlayerManager.Instance.Init();
        UIManager.Instance.Init();
    }
}
```

하나의 진입점에서 순서를 명시적으로 제어한다.

---

## Level 3 깊이 파기: Script Execution Order의 한계

10개 이상의 스크립트에 순서를 지정하면:

1. **암묵적 의존성** — 코드만 봐서는 "왜 -300이어야 하는지" 알 수 없음
2. **취약한 구조** — 새 매니저 추가 시 "이건 몇 번이어야 하지?" 고민
3. **순환 의존성을 감추게 됨** — 순서를 아무리 조정해도 해결 불가능한 근본 설계 문제

> 면접에서 기대하는 답변: "Script Execution Order는 2-3개 핵심 매니저에만 제한적으로 사용하고, 나머지는 이벤트 기반 또는 명시적 초기화 패턴으로 해결해야 합니다."

---

## Level 4 깊이 파기: RuntimeInitializeOnLoadMethod

MonoBehaviour 없이, 씬 로드 이전에 코드를 실행할 수 있는 방법이다.

### 왜 필요한가?

- MonoBehaviour의 Awake/Start는 씬에 오브젝트가 있어야 실행됨
- "어떤 씬에서 Play를 눌러도 동작하는" 개발 환경을 만들 때 필수

### 실행 흐름

```
Play 버튼 클릭
│
▼
① SubsystemRegistration ← 가장 먼저, static 변수 리셋
│
▼
② BeforeSceneLoad ← 씬 로드 전, 핵심 시스템 생성
│  (GameManager, AudioManager 등을 DontDestroyOnLoad로 생성)
│
▼
③ 씬 로드 ← 씬의 오브젝트들이 생성됨
│  Player.Awake() → GameManager는 이미 있음! 안전!
│
▼
④ AfterSceneLoad ← 씬 로드 완료 후 후처리
│
▼
게임 루프 시작
```

### 핵심: Domain Reload 비활성화 시

Unity에서 Enter Play Mode Settings > Domain Reload을 끄면 에디터 재생이 빨라지지만, static 변수가 초기화되지 않는다. `SubsystemRegistration`에서 static을 리셋하는 것이 안전한 패턴:

```csharp
[RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.SubsystemRegistration)]
static void ResetStatics()
{
    Instance = null;
}
```

### "어떤 씬에서든 Play 가능"한 구조

```
❌ MonoBehaviour만 쓸 때:
  레벨3 씬에서 Play → GameManager가 MainMenu 씬에만 있음 → NullReferenceException

✅ RuntimeInitializeOnLoadMethod:
  레벨3 씬에서 Play → Bootstrap이 씬 로드 전에 GameManager 자동 생성 → 정상 동작
```

---

## Level 5 깊이 파기: Composition Root 패턴

### MonoBehaviour 기반 설계의 근본적 한계

- 초기화 순서를 엔진이 제어 → 개발자가 완전히 통제 불가
- 의존성이 암묵적 → `GameManager.Instance`로 숨겨짐
- 테스트 어려움 → MonoBehaviour는 `new`로 생성 불가

### Composition Root란?

앱의 진입점 한 곳에서 모든 객체를 생성하고 의존성을 연결하는 패턴이다.

```csharp
public class GameRoot : MonoBehaviour  // 유일한 MonoBehaviour
{
    void Awake()
    {
        var config = new GameConfig();
        var playerMgr = new PlayerManager(config);
        var enemyMgr = new EnemyManager(config, playerMgr);
        var game = new GameService(playerMgr, enemyMgr);
        game.Initialize();
    }
}
```

장점:
- 의존성이 생성자에 명시적으로 드러남
- 순환 의존성이 있으면 컴파일 타임에 발견됨
- 순수 C# 클래스이므로 유닛 테스트 가능

---

## DI 컨테이너의 등장: Zenject과 VContainer

Composition Root가 커지면 (매니저 20개 이상) 수동 조립이 힘들어진다. DI 컨테이너가 이를 자동화한다.

### 리플렉션 vs 코드 생성

DI 컨테이너의 내부 동작 방식에는 두 가지가 있다.

**리플렉션 (Zenject):**
프로그램이 실행 중에 자기 자신의 구조를 들여다보는 것. 요리사가 레시피를 모르고, 매번 레시피 책을 꺼내서 읽는 것과 같다.

```csharp
Type type = typeof(PlayerManager);
ConstructorInfo ctor = type.GetConstructors()[0];     // 생성자 정보 조회
ParameterInfo[] params = ctor.GetParameters();        // 파라미터 분석
object[] args = ResolveAll(params);                   // 의존성 찾기
object instance = ctor.Invoke(args);                  // 생성
```

**코드 생성 (VContainer):**
컴파일할 때 필요한 코드를 미리 자동으로 만들어두는 것. 요리사가 레시피를 외우고 있는 것과 같다.

```csharp
// 컴파일 타임에 자동 생성된 코드
static PlayerManager Create(IObjectResolver resolver)
{
    var p0 = resolver.Resolve<GameConfig>();
    var p1 = resolver.Resolve<Logger>();
    return new PlayerManager(p0, p1);  // 직접 new!
}
```

**성능 차이가 나는 이유 3가지:**

1. **타입 정보 조회 비용** — 리플렉션은 매번 메타데이터 테이블을 검색하고 새 객체를 힙에 할당
2. **간접 호출 비용** — `ctor.Invoke()`는 파라미터 검증, 언박싱, 보안 검사를 거침. `new()`는 1 instruction
3. **GC 압박** — 리플렉션 1회에 ConstructorInfo, ParameterInfo[], object[] 등 여러 힙 할당 발생 → GC 발동 → 프레임 스파이크

```
벤치마크 (10,000회 Resolve):
Zenject:    ~3.2ms,  GC Alloc ~4.8KB
VContainer: ~0.4ms,  GC Alloc ~0KB
```

---

## Zenject 상세

2015년경부터 개발된 Unity DI의 원조격. 기능이 풍부하지만 복잡하다.

핵심 구조:
- **MonoInstaller** — 바인딩을 등록하는 곳
- **SceneContext** — 씬별 컨테이너
- **ProjectContext** — 앱 전체 글로벌 컨테이너
- **[Inject]** — 필드, 메서드, 생성자에 붙여서 주입받음

```csharp
// Zenject는 필드 주입이 가능
public class PlayerView : MonoBehaviour
{
    [Inject] private PlayerManager pm;     // 편하지만 의존성이 숨겨짐
    [Inject] private UIManager ui;
}
```

---

## VContainer 상세

2020년 출시. 성능에 극도로 집중한 경량 DI 컨테이너.

### 핵심: VContainer는 MonoBehaviour를 없애는 게 아니다

```
흔한 오해: "VContainer 쓰면 MonoBehaviour 안 써도 된다" → ❌
실제: "MonoBehaviour를 꼭 필요한 곳에만 쓰고, 나머지는 순수 C#으로 빼는 것" → ✅
```

MonoBehaviour가 반드시 필요한 것: Transform, Collider, 충돌 콜백, 코루틴
MonoBehaviour가 필요 없는 것: 데이터 관리, 게임 규칙, 네트워크 통신, 상태 관리

### VContainer는 코드를 자동으로 분리해주지 않는다

분리는 개발자가 직접 해야 한다. VContainer는 이미 분리된 클래스들의 의존성을 연결해주는 배관공이다.

```
분리 전:
┌──────────────────────────────┐
│       PlayerManager          │
│      (MonoBehaviour)         │
│  이동 + 점수 + HP + 인벤토리  │
└──────────────────────────────┘

분리 후 (개발자가 직접):
┌─────────────────┐     ┌─────────────────┐
│   PlayerView    │     │ PlayerService   │
│ (MonoBehaviour) │────→│ (순수 C# 클래스) │
│  이동, 애니메이션│ 참조 │  점수, HP, 로직  │
└─────────────────┘     └─────────────────┘
```

### LifetimeScope — VContainer의 심장

```csharp
public class GameLifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        builder.Register<PlayerService>(Lifetime.Singleton);         // 순수 C#
        builder.RegisterComponentInHierarchy<PlayerView>();           // MonoBehaviour
    }
}
```

LifetimeScope은 계층 구조를 가진다:

```
RootLifetimeScope (앱 전체, DontDestroyOnLoad)
├── SceneLifetimeScope (씬 단위, 씬 전환 시 파괴)
│   └── ChildScope (팝업 등, 닫으면 파괴)
```

자식은 부모의 등록을 사용할 수 있고, 부모는 자식을 모른다. 씬 전환 시 자동 정리된다.

### MonoBehaviour를 VContainer에 등록하는 3가지 방법

1. **씬에서 찾기**: `builder.RegisterComponentInHierarchy<GameManager>();`
2. **프리팹에서 생성**: `builder.RegisterComponentInNewPrefab(prefab, Lifetime.Singleton);`
3. **직접 참조 연결**: `builder.RegisterComponent(gameManager);`

### EntryPoint 시스템 — MonoBehaviour 없는 게임 루프

```csharp
public class GameLoop : IStartable, ITickable, IDisposable
{
    public void Start() { }      // Start() 대체
    public void Tick() { }       // Update() 대체
    public void Dispose() { }    // OnDestroy() 대체
}
```

VContainer가 내부적으로 하나의 MonoBehaviour(TickRunner)를 가지고, 등록된 ITickable들을 순회한다. C++ → C# 경계를 1회만 넘기므로 MonoBehaviour 100개의 Update()보다 훨씬 빠르다.

### 실행 순서 — LifetimeScope의 등록 순서는 상관없다

**실행 순서를 결정하는 건 등록 순서가 아니라 생성자의 의존성이다.**

```csharp
// 이렇게 등록해도
builder.RegisterEntryPoint<FirestoreService>();   // 3번째로 실행됨
builder.RegisterEntryPoint<AuthService>();         // 2번째로 실행됨
builder.RegisterEntryPoint<FirebaseService>();     // 1번째로 실행됨

// 이렇게 등록해도
builder.RegisterEntryPoint<FirebaseService>();     // 1번째로 실행됨
builder.RegisterEntryPoint<AuthService>();         // 2번째로 실행됨
builder.RegisterEntryPoint<FirestoreService>();    // 3번째로 실행됨

// 결과는 동일!
```

VContainer가 생성자를 분석해서 의존성 그래프를 그리고, 자동으로 순서를 결정한다:

```
FirebaseService (의존 없음) → AuthService (Firebase 필요) → FirestoreService (둘 다 필요)
```

**개발자가 할 일: 생성자에 의존성 선언 + LifetimeScope에 등록. 끝!**

---

## IAsyncStartable — 비동기 초기화의 혁신

### 기존 방식의 문제

```csharp
// MonoBehaviour + UniTask
async void Start()
{
    await ConnectToServer();  // 2초 걸림
}
// → async void Start()는 "기다려주지 않음"
// → Unity는 다음 Start()를 바로 실행
// → 씬 전환 시 async가 계속 돌아서 MissingReferenceException
```

### IAsyncStartable이 해결하는 것

```csharp
public class NetworkService : IAsyncStartable
{
    public async UniTask StartAsync(CancellationToken ct)
    {
        await ConnectToServer(ct);
    }
}

public class LobbyService : IAsyncStartable
{
    private readonly NetworkService network;

    public LobbyService(NetworkService network)
    {
        this.network = network;
    }

    public async UniTask StartAsync(CancellationToken ct)
    {
        // NetworkService 완료 후 자동 실행!
    }
}
```

- 의존성 그래프에서 순서가 자동으로 나옴
- while문 대기 불필요
- 씬 전환 시 CancellationToken 자동 취소 → 메모리 누수 방지

---

## 실전 적용: Fusion2 + Firebase + VContainer

```csharp
public class AppLifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        builder.RegisterEntryPoint<FirebaseService>();
        builder.RegisterEntryPoint<AuthService>();
        builder.RegisterEntryPoint<FirestoreService>();
        builder.RegisterEntryPoint<FusionNetworkService>();
        builder.RegisterEntryPoint<OnlineGameService>();
    }
}
```

VContainer가 자동으로 결정한 순서:

```
① FirebaseService + FusionNetworkService (서로 의존 없으면 병렬!)
│
▼
② AuthService (Firebase 필요)
│
▼
③ FirestoreService (Firebase + Auth 필요)
│
▼
④ OnlineGameService (Auth + Fusion + Firestore 모두 필요)
│
▼
게임 루프 시작
```

개발자는 의존성만 선언하면 된다. 순서, 병렬/직렬, 대기, 취소는 VContainer가 전부 처리한다.

---

## 전체 여정 정리

```
Awake/Start 순서 문제
  → "Awake는 자기 초기화, Start는 남 참조"
    → "Start끼리도 꼬이면?"
      → 이벤트 기반, 코루틴, Bootstrap 패턴
        → "Script Execution Order 남용하면?"
          → 암묵적 의존성 문제
            → RuntimeInitializeOnLoadMethod로 씬 독립적 초기화
              → Composition Root로 명시적 의존성 관리
                → DI 컨테이너 (Zenject/VContainer)로 자동화
                  → IAsyncStartable로 비동기 초기화까지 해결
                    → Fusion, Firebase 같은 실무 서비스에 적용
```

| 단계 | 해결하는 문제 | 면접 레벨 |
|------|-------------|----------|
| Awake/Start 분리 | 기본 순서 문제 | 주니어 |
| Script Execution Order | 명시적 순서 보장 | 주니어~미들 |
| 이벤트 기반 설계 | Start끼리의 순서 문제 | 미들 |
| RuntimeInitializeOnLoadMethod | 씬 독립적 초기화 | 시니어 |
| Composition Root | 명시적 의존성 관리 | 시니어~리드 |
| DI 컨테이너 (VContainer) | 대규모 프로젝트 자동화 | 리드 |

---

## 오답 노트

해당 세션에서 틀린 문제는 없었으나, 학습 중 혼동했던 개념:

1. **VContainer가 코드를 자동 분리해주는가?** → 아니다. 분리는 개발자가 직접 하고, VContainer는 분리된 것들을 연결만 해준다.
2. **LifetimeScope의 등록 순서가 실행 순서인가?** → 아니다. 생성자의 의존성이 실행 순서를 결정한다.
3. **IAsyncStartable과 MonoBehaviour + UniTask의 차이는?** → 순서 자동 결정, CancellationToken 자동 관리, 씬 전환 시 안전한 취소가 핵심 차이다.
