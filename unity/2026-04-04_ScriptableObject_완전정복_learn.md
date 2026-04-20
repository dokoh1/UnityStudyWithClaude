# Unity 학습 기록 - 2026-04-04 (Q5)

## 세션 정보
- 분야: Unity
- 주제: ScriptableObject → 메모리 구조 → 이벤트 패턴 → 변수 관리 → 스킬 설계 → Addressable
- 모드: learn
- 날짜: 2026-04-04

---

## 이야기의 시작: ScriptableObject란?

> "ScriptableObject에 대해 자세히 설명해줘. MonoBehaviour와의 차이점, 실무 활용까지."

ScriptableObject는 "씬에 존재하지 않는 데이터 컨테이너"다. MonoBehaviour는 씬의 GameObject에 붙어야 하지만, ScriptableObject는 프로젝트(Assets 폴더)에 .asset 파일로 존재한다. Update 같은 생명주기도 없고, Transform(위치)도 없다.

핵심 가치: 여러 오브젝트가 같은 데이터를 참조(공유)할 수 있다. 데이터가 복사되지 않고 참조만 된다.

---

## 첫 번째 의문: "결국 각 Enemy마다 다른 HP를 가져야 하잖아. 메모리가 같은 거 아니야?"

### 변하지 않는 데이터 vs 변하는 데이터를 분리하는 것

적 1마리에 필요한 데이터를 두 종류로 나눌 수 있다:

- **변하지 않는 데이터** (설계 데이터): 이름 "고블린", 최대 HP 100, 공격력 15, 아이콘, 프리팹 — 모든 고블린이 동일
- **변하는 데이터** (런타임 상태): 현재 HP 73, 현재 위치, AI 상태 — 각 고블린마다 다름

ScriptableObject에는 변하지 않는 공유 데이터만 넣고, 변하는 데이터는 MonoBehaviour의 일반 필드로 각자 가진다.

```csharp
// ScriptableObject — 공유 데이터 (1개만 존재)
public class EnemyData : ScriptableObject
{
    public int maxHp;
    public int attack;
    public Sprite icon;
}

// MonoBehaviour — 개별 상태 (각자 가짐)
public class Enemy : MonoBehaviour
{
    [SerializeField] private EnemyData data;  // 참조 (8B)
    private int currentHp;  // 각자의 HP (4B)
}
```

### 메모리 비교 (고블린 100마리)

**SO 없이**: 공통 데이터 ~52B × 100 + 개별 데이터 ~8B × 100 = **6,000B**. 같은 값이 100번 복사됨.

**SO 사용**: 공통 데이터 52B × **1** + (참조 8B + 개별 데이터 8B) × 100 = **1,652B**. 약 **73% 절약**.

100마리면 작은 차이지만, 실제 게임에서 EnemyData에 스킬 데이터, 드롭 테이블, 대사 등 수백 바이트~수 KB가 들어가면 절약량이 수 MB 단위가 된다.

더 중요한 건 씬 파일 크기: MonoBehaviour에 데이터를 넣으면 씬 파일에 전부 직렬화되지만, ScriptableObject는 .asset 참조(GUID)만 기록되어 씬 파일이 작아지고 로드가 빨라진다.

---

## 실무 활용 1: 게임 데이터 관리

아이템, 캐릭터, 스킬, 퀘스트 등의 기획 데이터를 SO로 관리하면 디자이너가 코드 수정 없이 Inspector에서 밸런스를 조정할 수 있다. `[CreateAssetMenu]` 어트리뷰트로 에디터에서 우클릭으로 .asset 파일을 만들 수 있다.

## 실무 활용 2: 밸런스/설정 데이터

난이도 프리셋(Easy/Normal/Hard), 카메라 프리셋, 데미지 공식의 계수 등을 SO로 관리하면 에셋만 교체하여 밸런스를 조정할 수 있다. 계산 메서드도 SO 안에 넣을 수 있어서 `balanceConfig.CalculateDamage(attack, defense)` 같은 호출이 가능하다.

---

## 실무 활용 3: 이벤트 시스템 (SO 이벤트)

### 두 번째 의문: "GameEvent와 GameEventListener는 순환참조 아니야?"

순환참조가 아니라 **구독 패턴**이다.

- 순환참조: A가 B 없으면 동작 못 하고, B가 A 없으면 동작 못 하는 것
- 구독 패턴: 출판사(GameEvent)는 구독자가 0명이어도 Raise() 가능. 구독자(Listener)는 이벤트가 없어도 존재 가능. 서로 없어도 각자 동작한다.

OnDisable에서 해제하므로 GC 순환참조 문제도 없다.

### 이벤트 패턴이 여러 개인데, SO 이벤트는 언제?

| 패턴 | 적합한 상황 |
|---|---|
| C# event/Action | 같은 프리팹 안, 부모-자식 관계 (가장 빠름) |
| UnityEvent | 에디터에서 드래그&드롭 연결 (UI 버튼 등) |
| **SO 이벤트** | **씬을 넘나드는 전역 이벤트, 매니저 간 통신, 발행자와 구독자가 서로를 모르게** |
| EventBus/서드파티 | 대규모 복잡한 이벤트 시스템 |

SO 이벤트의 핵심 장점: 발행자(Player)가 구독자(UIManager, AudioManager)를 전혀 모른다. .asset 파일이라 씬 독립적이다. 새 매니저를 추가해도 Player 코드를 수정하지 않는다.

---

## 실무 활용 4: 변수 공유 패턴

### 네 번째 의문: "변수 패턴이 뭐야?"

ScriptableObject를 "공유 변수"로 사용하는 것.

```
직접 참조: class UIManager { Player player; }  → Player를 직접 알아야 함
변수 패턴: class UIManager { IntVariable playerHP; }  → HP "값"만 알면 됨
```

이벤트 패턴은 "뭔가 일어났다"는 1회성 신호이고, 변수 패턴은 값을 지속적으로 공유하는 것. 둘은 보완적이다.

### 세 번째 의문: "게임 끄면 초기화되나? [NonSerialized]가 뭐야?"

**에디터 Play → Stop**: SO의 변경된 값이 유지됨 (⚠️). **빌드된 게임**: 종료하면 원래 값으로 돌아감.

`[System.NonSerialized]`는 "이 변수를 .asset에 저장하지 마라"는 의미. 직렬화 안 됨 → Inspector에 안 보임 → Play → Stop하면 기본값으로 리셋. "메모리에만 존재하는 임시 변수"가 된다.

```csharp
[CreateAssetMenu]
public class IntVariable : ScriptableObject
{
    [SerializeField] private int initialValue = 100;  // 저장됨 (설계 데이터)
    [System.NonSerialized] public int runtimeValue;   // 저장 안 됨 (런타임 상태)

    void OnEnable() { runtimeValue = initialValue; }  // 항상 초기값으로 리셋
}
```

### 변수 종류별 관리 방법

| 종류 | 예시 | 관리 방법 |
|---|---|---|
| 설계 데이터 (불변) | 최대 HP, 아이템 가격 | SO의 직렬화된 필드 |
| 런타임 상태 (임시) | 현재 HP, AI 상태, 쿨타임 | [NonSerialized] 또는 MonoBehaviour 변수 |
| 세이브 데이터 (영구) | 레벨, 인벤토리, 진행도 | JSON, PlayerPrefs, 파일 시스템 |

---

## 실무 활용 5: 스킬 시스템

### 다섯 번째 의문: "SO로 못 만드는 스킬은? 안 쓰는 변수가 추가되면?"

#### SO로 못 만드는 복잡한 스킬

ScriptableObject는 "데이터"만 담는다. "적을 잡고 폭탄 설치하고 3초 후 연쇄 피해 + 쉴드" 같은 복잡한 로직은 데이터만으로 표현 불가.

**해결: 전략 패턴 — SO에 로직을 넣기**

```csharp
// 추상 클래스
public abstract class SkillEffect : ScriptableObject
{
    public abstract void Execute(Character caster, Character target);
}

// 구체 구현
[CreateAssetMenu] public class DamageEffect : SkillEffect { ... }
[CreateAssetMenu] public class HealEffect : SkillEffect { ... }
[CreateAssetMenu] public class PlaceBombEffect : SkillEffect { ... }

// 스킬 = 효과 배열의 조합!
[CreateAssetMenu]
public class SkillData : ScriptableObject
{
    public string skillName;
    public SkillEffect[] effects;  // 여러 효과를 순서대로 실행
}
```

에디터에서 "폭발 참수" = DamageEffect + PlaceBombEffect + ShieldEffect를 조합. 새 스킬 = 기존 Effect들 조합. 새 Effect가 필요할 때만 코드 추가.

#### 안 쓰는 변수 문제

하나의 SkillData에 모든 종류의 변수를 넣으면 힐 스킬에 damage, summonPrefab 등 안 쓰는 변수가 잔뜩 붙는다.

**해결 1: 상속** — SkillDataBase를 만들고 DamageSkillData, HealSkillData, SummonSkillData로 분리. 각 타입에 필요한 변수만 가진다.

**해결 2: 컴포지션** — 공통은 SkillData에, 특수 데이터/로직은 SkillEffect SO에. 가장 유연하지만 파일이 많아진다.

선택 기준: 스킬 종류가 적고 명확하면 상속, 효과를 자유롭게 조합해야 하면 컴포지션.

---

## 실무 활용 6: Addressable 연계

### 여섯 번째 의문: "Addressable이 뭐야? SO와 어떻게 연계해?"

#### 문제

아이템 500개의 SO를 전부 직접 참조하면, 게임 시작 시 500개의 아이콘, 이펙트 프리팹이 전부 메모리에 올라간다. 플레이어가 가진 건 10개뿐인데.

#### Addressable이란

"필요할 때만 로드하고, 안 쓰면 메모리에서 내리는 시스템"

- **직접 참조** (`public Sprite icon`) — SO가 로드되면 아이콘도 자동 로드. 안 써도 메모리에 있음.
- **Addressable** (`public AssetReferenceSprite iconRef`) — "주소"만 저장. LoadAssetAsync()를 호출해야 그때 로드. Release()로 해제.

비유: 직접 참조 = 책 500권을 전부 빌려온 것. Addressable = 카탈로그만 가져와서 읽고 싶은 책만 빌려오는 것.

#### 핵심 개념

- **Address**: 에셋에 붙이는 이름표 ("Items/Sword_Icon")
- **Label**: 에셋에 붙이는 태그 ("weapons", "level1")
- **Group**: 에셋을 묶는 빌드 단위
- **원격 에셋**: 서버(CDN)에서 다운로드 가능. 앱 업데이트 없이 콘텐츠 추가/밸런스 패치 가능!

#### SO + Addressable 연계

```csharp
// 기존: public Sprite icon;             → 항상 메모리에
// 변경: public AssetReferenceSprite iconRef; → 필요할 때만 로드

// 메모리 비교:
// 직접 참조: 500 아이콘 + 500 이펙트 = 수백 MB
// Addressable: 인벤토리 열 때 보유 10개만 로드 = 수 MB
```

밸런스 데이터 SO를 서버에 올려서, 게임 시작 시 최신 SO를 다운로드하면 앱 업데이트 없이 밸런스 패치도 가능하다.

#### 도입 시기

- **불필요**: 소규모, 에셋 적음, 콘텐츠 업데이트 없음
- **필요**: 모바일(메모리 제한), 에셋 많음(수백 MB+), 출시 후 콘텐츠 추가, DLC

---

## ScriptableObject 주의점 정리

1. **에디터에서 런타임 수정 시 원본이 바뀜** — [NonSerialized]로 런타임 값 분리하거나 Instantiate로 복사본 사용
2. **씬 오브젝트 참조 불가** — .asset은 프로젝트에, Transform은 씬에 있으므로 직접 참조 불가. 이벤트/변수 패턴으로 간접 참조
3. **Dictionary 직렬화 안 됨** — List로 대체하거나 커스텀 직렬화
4. **세이브 파일 대용 아님** — 빌드에서 SO 수정은 저장 안 됨. 세이브는 JSON/PlayerPrefs 사용

---

## 면접 레벨별 기대

| 레벨 | 기대 |
|---|---|
| 주니어 | SO가 뭔지, MonoBehaviour와 차이 (씬 독립, 데이터 공유) |
| 미들 | 게임 데이터 관리, 밸런스 설정, 이벤트 시스템 활용 |
| 시니어 | SO 이벤트/변수 아키텍처, 전략 패턴(스킬 시스템), 런타임 수정 주의점 |
| 리드 | 팀 데이터 파이프라인 설계, Custom Editor, Addressable 연계 |

---

## 전체 여정 정리

```
ScriptableObject 기본 개념
  → "메모리가 같은 거 아니야?"
    → 변하지 않는 데이터 vs 변하는 데이터 분리
    → 메모리 73% 절약 원리
      → 실무 활용: 데이터 관리, 밸런스, 이벤트, 변수 공유
        → "순환참조 아니야?" → 구독 패턴 설명
        → "NonSerialized가 뭐야?" → 직렬화와 런타임 변수 관리
        → "변수 패턴이 뭐야?" → SO를 공유 변수로 사용
          → "SO로 못 만드는 스킬은?" → 전략 패턴, 상속/컴포지션
            → "Addressable이 뭐야?" → 필요할 때만 로드, 원격 에셋
```

---

## 오답 노트

이 세션에서 새로 정리된 인사이트:

1. **SO의 메모리 절약 원리** — 데이터 복사가 아닌 참조 공유. 진짜 절약은 씬 파일 크기 감소와 에셋 참조 중복 제거에서 온다.

2. **SO 이벤트는 순환참조가 아니다** — 구독 패턴이며, OnDisable에서 해제하므로 메모리 문제도 없다. 발행자와 구독자가 서로를 모르는 "느슨한 결합"이 핵심.

3. **[NonSerialized]의 역할** — 직렬화를 막아서 .asset에 저장되지 않게 한다. 런타임에만 존재하는 임시 변수를 만드는 데 사용. 에디터 Play→Stop 시 값이 유지되는 문제를 해결한다.

4. **이벤트 패턴 vs 변수 패턴** — 이벤트는 "뭔가 일어났다"는 1회성 신호, 변수는 값의 지속적 공유. 둘은 보완적.

5. **SO로 스킬을 만들 때의 설계** — 단순 데이터만으로는 복잡한 스킬을 표현할 수 없다. 전략 패턴(SkillEffect 추상 SO)으로 로직도 데이터처럼 조합 가능하게 만든다. 안 쓰는 변수 문제는 상속 또는 컴포지션으로 해결.

6. **Addressable의 본질** — "필요할 때만 로드"하는 에셋 관리 시스템. SO와 연계하면 대규모 게임의 메모리를 극적으로 줄일 수 있고, 원격 에셋으로 앱 업데이트 없이 콘텐츠를 추가할 수 있다.
