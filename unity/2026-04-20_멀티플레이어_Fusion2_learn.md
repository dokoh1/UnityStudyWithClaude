# Unity 학습 기록 - 2026-04-16~20 (Q17)

## 세션 정보
- 분야: Unity
- 주제: 멀티플레이어 → 네트워크 구조 → Fusion 2 vs NGO → 예측/롤백 → 실전 코드 → 면접 레벨별
- 모드: learn
- 날짜: 2026-04-16 ~ 2026-04-20

---

## 멀티플레이어가 어려운 이유

- 인터넷 지연 (서울↔도쿄 ~30ms, 서울↔미국 ~150ms)
- 패킷 유실, 순서 뒤바뀜
- 두 컴퓨터의 시간 불일치
- 치팅 방지

---

## 네트워크 구조 (토폴로지)

### 클라이언트-서버
서버가 "진짜 게임 상태" 관리. 치팅 방지, 일관된 상태. 서버 비용 필요.
→ 대부분의 온라인 게임 (LOL, 오버워치, 발로란트)

### 호스트 모드 (Host = 서버 + 클라이언트)
플레이어 한 명이 서버 역할. 서버 비용 없음. 호스트 이탈 시 문제.
→ 소규모 co-op, 인디 (Among Us, 다크소울)
→ **Fusion 2의 Host Mode**

### Shared Mode (Fusion 2 전용)
각 클라이언트가 자기 오브젝트 권한. 서버는 중계만. 구현 가장 간단. 치팅 취약.
→ 캐주얼, 소셜, co-op
→ **Fusion 2의 Shared Mode**

---

## Unity 멀티플레이어 솔루션 비교

| | NGO (Unity 공식) | Fusion 2 | PUN 2 |
|---|---|---|---|
| 틱 시스템 | 프레임 기반 | 고정 틱 (결정론적) | 가변 |
| 예측+롤백 | 수동 구현 | **내장!** | 없음 |
| Shared Mode | 미지원 | 지원 | Cloud |
| Host Migration | 미지원 | 지원 | 미지원 |
| 동기화 | NetworkVariable | [Networked] | PhotonView |
| RPC | [ServerRpc]/[ClientRpc] | [Rpc(...)] | [PunRPC] |

**Fusion 2가 인기인 이유**: 틱 기반 시뮬레이션, 예측+롤백 내장, Shared Mode, Host Migration.

---

## Fusion 2 핵심 개념

### NetworkObject + NetworkBehaviour

```csharp
public class PlayerController : NetworkBehaviour
{
    [Networked] public int HP { get; set; }  // 자동 동기화!

    public override void FixedUpdateNetwork()  // 틱마다 호출
    {
        if (GetInput(out NetworkInputData input))
        {
            // 입력 처리 + 이동
        }
    }
}
```

- **[Networked]**: 값 바뀌면 모든 클라이언트에 자동 전달
- **FixedUpdateNetwork**: 네트워크 틱마다 호출 (Update 대신)
- **Render()**: 매 프레임 호출 (시각적 보간용)

### 권한 (Authority)

- **HasInputAuthority**: 내가 이 오브젝트에 입력 가능 (내 캐릭터)
- **HasStateAuthority**: 이 오브젝트의 상태를 내가 확정 (Host/Server)
- **Runner.IsServer**: 서버인가?

### 오브젝트 관리

```csharp
Runner.Spawn(prefab, position, rotation, inputAuthority);  // 네트워크 생성
Runner.Despawn(obj);  // 네트워크 제거
// ⚠️ Instantiate/Destroy 사용 금지!
```

### RPC

```csharp
[Rpc(RpcSources.All, RpcTargets.StateAuthority)]  // 모두→서버
public void RPC_TakeDamage(int damage) { HP -= damage; }

[Rpc(RpcSources.StateAuthority, RpcTargets.All)]  // 서버→모두
public void RPC_ShowEffect(Vector3 pos) { /* 이펙트 */ }
```

### [Networked] vs RPC
- **[Networked]**: 지속적 상태 (HP, Position, Score)
- **RPC**: 순간적 이벤트 (이펙트 재생, 데미지 처리)

---

## 모드 선택 기준

| 모드 | 적합한 게임 |
|---|---|
| Shared | 캐주얼, co-op, 프로토타입, 치팅 무관 |
| Host | 중규모, PvE, 서버 비용 절감, 적당한 치팅 방지 |
| Server | 경쟁 PvP, 치팅 방지 필수, e스포츠급 |

---

## 지연 해결 기법

### 1. 클라이언트 예측 (Client Prediction)
서버 응답 기다리지 않고 미리 실행. 유저 입장에서 지연 안 느껴짐.
→ Fusion 2: FixedUpdateNetwork에서 자동 예측

### 2. 롤백 (Rollback)
예측이 틀리면 서버 상태로 되돌리기. [Networked] 변수가 자동 롤백 지원.

### 3. 보간 (Interpolation)
상대방 움직임을 부드럽게. Render()에서 틱 사이를 Lerp.

### 4. Lag Compensation
"내 화면에서 맞았으면 맞은 것". 서버가 과거 상태를 재현해서 판정.
→ Fusion 2: `Runner.LagCompensation.Raycast()`

---

## NGO vs Fusion 2 핵심 차이

```
Fusion 2:
  → "같은 코드가 서버와 클라이언트에서 실행"
  → 예측: 입력 결과를 미리 계산
  → 롤백: 서버와 다르면 자동 되돌리기

NGO:
  → "서버 코드와 클라이언트 코드가 분리"
  → ServerRpc로 "서버야 이거 해줘" 요청
  → 서버 응답까지 기다림 (예측 없음 기본)

빠른 액션 → Fusion 2 유리
턴제/느린 게임 → NGO도 충분
```

---

## 면접 레벨별 기대

### 주니어: "기본 개념을 아는가"

알아야 하는 것:
- 클라이언트-서버 구조가 뭔지
- 동기화가 뭔지
- 지연(Latency)이 뭔지
- Fusion 2/NGO를 써봤는가

질문 예시:
- "멀티플레이어에서 동기화란?" → 여러 플레이어의 상태를 네트워크로 일치시키는 것
- "왜 멀티가 어려운가?" → 인터넷 지연 때문에 모든 화면이 동시에 같은 상태 보여주기 어려움

### 미들: "실제로 써봤고 핵심 개념 이해"

알아야 하는 것:
- Shared/Host/Server 모드 차이와 선택 근거
- [Networked] vs RPC 구분
- 권한(Authority) 개념 (Input/State)
- NetworkObject, NetworkTransform, Spawn/Despawn
- 기본 동기화 구현 경험

질문 예시:
- "Shared vs Host 차이?" → Shared는 클라이언트 권한(간단, 치팅 취약), Host는 중앙 관리(치팅 방지)
- "[Networked]와 RPC 언제 쓰나?" → [Networked]는 지속 상태, RPC는 순간 이벤트
- "FixedUpdateNetwork와 Update 차이?" → 틱 기반 vs 프레임 기반

### 시니어: "왜 그렇게 동작하는지, 문제 해결 가능"

알아야 하는 것:
- 예측(Prediction)과 롤백(Rollback)의 원리와 Re-simulation
- Lag Compensation (지연 보상) — 과거 상태에서 판정
- 대역폭 최적화 — Interest Management(AOI), Delta Compression, 틱 레이트 조정
- 서버 아키텍처 — Dedicated Server, 매치메이킹, 리전 분산
- 보안/치팅 방지 — 서버 권위, 입력 검증, 이상 행동 탐지, Shared Mode 취약점

질문 예시:
- "예측이 틀리면?" → 서버 상태로 롤백 후 Re-simulation. Fusion 2는 [Networked]가 자동 롤백
- "FPS 총 판정?" → Lag Compensation으로 과거 상태에서 Raycast
- "대역폭 최적화?" → AOI(주변만 동기화), Delta(변경분만), 중요도별 틱 분리
- "Shared Mode 보안?" → 클라이언트가 자기 데이터 조작 가능, 경쟁 게임 부적합

### 리드: "전체 아키텍처 설계, 팀 관리"

알아야 하는 것:
- 네트워크 토폴로지 선택과 비용/성능/보안 근거
- 서버 인프라 설계 (클라우드, 로드 밸런서, 매치메이킹, DB, 모니터링)
- 네트워크 코드 레이어 분리 (Transport/Network/Logic/Presentation/Input)
- 팀 가이드라인 (코딩 규칙, 테스트 방법, 성능 기준)
- 라이브 운영 (모니터링, 장애 대응, 확장)

질문 예시:
- "100명 배틀로얄 아키텍처?" → 전용 서버 + AOI + 리전 매치메이킹 + 치팅 탐지
- "네트워크 코드 팀 관리?" → 가이드라인 문서화, PR마다 2클라이언트 테스트, 지연 시뮬레이션

---

## 레벨별 한 줄 요약

```
주니어: "클라이언트-서버, 동기화가 뭔지 안다. 써봤으면 플러스"
미들:   "모드 차이, [Networked]/RPC 구분, 권한 이해, 기본 구현"
시니어: "예측/롤백/Lag Compensation 원리, 대역폭 최적화, 보안"
리드:   "토폴로지 선택, 서버 인프라 설계, 팀 가이드라인, 라이브 운영"
```

---

## 면접 답변 예시

```
Q: "멀티플레이어 개발 경험이 있으신가요?"

A: "Photon Fusion 2로 멀티플레이어를 구현한 경험이 있습니다.

    Fusion 2는 틱 기반 시뮬레이션으로
    클라이언트 예측과 롤백이 프레임워크에 내장되어
    NGO보다 빠른 액션 게임에 적합합니다.

    [Networked]로 변수를 자동 동기화하고
    FixedUpdateNetwork에서 틱 기반으로 로직을 처리하며
    Render에서 보간으로 부드러운 화면을 만듭니다.

    Shared Mode로 프로토타입을 빠르게 만들고
    필요에 따라 Host/Server Mode로 전환합니다.

    멀티플레이어의 핵심 과제는 지연 처리인데,
    클라이언트 예측으로 반응성을 확보하고
    서버 권위적 방식으로 치팅을 방지하며
    보간으로 상대방 움직임을 부드럽게 합니다."
```

---

## 오답 노트

1. **Instantiate/Destroy 대신 Spawn/Despawn** — 네트워크 오브젝트는 반드시 Runner를 통해 생성/제거.

2. **[Networked]는 상태, RPC는 이벤트** — HP(지속)는 [Networked], 이펙트(순간)는 RPC.

3. **Shared Mode는 치팅에 취약** — 클라이언트가 State Authority라 데이터 조작 가능. 경쟁 게임에 부적합.

4. **예측+롤백이 Fusion 2의 핵심 차별점** — NGO는 수동 구현. Fusion은 [Networked]가 자동 롤백.

5. **FixedUpdateNetwork ≠ Update** — 네트워크 틱 기반. 같은 코드가 서버(확정)와 클라이언트(예측)에서 실행.

6. **Render()는 보간용** — 틱 사이의 시각적 부드러움. 로직은 FixedUpdateNetwork에서만.

7. **Lag Compensation** — "내 화면에서 맞았으면 맞은 것". 서버가 과거 상태를 재현해서 판정.

8. **대역폭은 AOI로 줄인다** — 100명 매치에서 주변 20명만 상세 동기화하면 트래픽 80% 감소.
