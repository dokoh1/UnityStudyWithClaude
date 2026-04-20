# Unity 학습 기록 - 2026-04-13 (Q7)

## 세션 정보
- 분야: Unity
- 주제: Draw Call → 배칭 기법들 → GPU Instancing → MaterialPropertyBlock → URP 자동 적용
- 모드: learn
- 날짜: 2026-04-13

---

## 이야기의 시작: "Draw Call은 멀수록 품질 낮추는 거 아니야?"

> "Draw Call 이란 멀수록 렌더링을 안하거나 그래픽 품질을 낮춰서 최적화 하는거 아닌가?"

이건 **LOD(Level of Detail)나 Culling**의 개념이다. Draw Call은 완전히 다른 이야기.

---

## Draw Call이 뭔가

**Draw Call = CPU가 GPU에게 "이것 좀 그려줘"라고 보내는 명령 하나**

```
CPU가 GPU에게 말함:
"어떤 메시를, 어떤 셰이더로,
 어떤 텍스처를 써서, 어느 위치에 그려줘"

이 대화 1번 = Draw Call 1번
```

**비유**: 사장님(CPU)이 주방(GPU)에 주문을 전달하는 것. 주문 100개 = 왔다갔다 100번 = 힘듦. 주문을 묶으면 = 1번에 해결 = Draw Call 최적화.

### 왜 성능 문제인가?

- Draw Call 1번: 약 0.1~0.5ms
- 60fps 예산: 16.67ms
- Draw Call 1000개 = 100~500ms → 완전히 렉
- **CPU가 명령 보내느라 바빠서 GPU는 놀고 있음 = CPU 병목**

### Draw Call의 기본 단위

```
같은 메시 + 같은 셰이더 + 같은 머티리얼 + 같은 텍스처 → 1 Draw Call
다르면 각각 Draw Call
```

---

## 최적화 기법들

### 1. Static Batching (정적 배칭)

**"움직이지 않는 오브젝트들을 하나로 합치는 것"**

- 적용: Inspector에서 "Static" 체크 + 같은 머티리얼 + 움직이지 않음
- 원리: 씬 시작 시 메시들을 하나의 큰 메시로 합침 → 1 Draw Call
- 장점: 가장 효과적, CPU 부하 없음
- 단점: 메모리 증가 (큰 메시 저장), 움직이는 오브젝트 불가
- **주의**: 1000개를 Static으로 만들면 메시가 1000배 → 메모리 터짐

### 2. Dynamic Batching (동적 배칭)

**"움직이는 작은 오브젝트들을 매 프레임 합치는 것"**

- 적용: 정점 300개 이하 + 같은 머티리얼
- 장점: 움직이는 것도 가능
- 단점: CPU 비용, 정점 제한, 최신 Unity에서 효과 제한적
- **URP에서는 기본 비활성화** (비효율적이라)

### 3. SRP Batcher (가장 효과적)

**URP/HDRP 전용, 가장 중요한 배칭**

- 기존 배칭: 같은 머티리얼이어야 함
- SRP Batcher: **같은 셰이더면 됨 (머티리얼 달라도 OK)**
- 원리: Draw Call 수는 비슷해도, Draw Call당 비용을 극도로 낮춤
- 머티리얼 속성을 GPU 버퍼에 미리 올려두고 포인터만 변경
- **URP 사용 시 자동 활성화** — 아무것도 안 해도 됨!
- 제약: SRP 호환 셰이더 필요, MaterialPropertyBlock과 충돌

### 4. GPU Instancing

**"같은 메시를 여러 번 그릴 때 특화"**

- 적용: 머티리얼에서 "Enable GPU Instancing" 체크 + 셰이더 지원 필수
- 원리: 메시 1개 + 위치 배열 → 1 Draw Call로 N개 렌더링
- Static Batching과 비교: 메모리 효율적, 움직일 수 있음
- 적합: 풀, 나무, 총알, 적 떼

```csharp
Graphics.DrawMeshInstanced(mesh, 0, material, matrices); // 1023개까지
```

### 5. Texture Atlas

**"여러 텍스처를 하나의 큰 텍스처로 합치기"**

- 텍스처가 다르면 머티리얼도 다름 = 배칭 안 됨
- 하나로 합치면 같은 머티리얼 사용 가능
- UI: Unity의 Sprite Atlas (자동)
- 3D: 직접 제작 또는 툴 사용

### 6. Occlusion Culling

**"다른 오브젝트에 가려진 것을 안 그림"**

- Frustum Culling (자동): 카메라 밖 안 그림
- Occlusion Culling: 카메라 안이라도 가려진 것 안 그림
- Window > Rendering > Occlusion Culling에서 Bake 필요
- 복잡한 실내/도시에서 효과적

### LOD (질문에서 언급한 것)

**"멀수록 그래픽 품질 낮춤" = LOD**

- Draw Call 최적화가 아니라 "렌더링 비용" 최적화
- 같은 오브젝트의 여러 버전 (고/중/저 해상도)을 거리에 따라 전환
- Draw Call 최적화와는 다른 차원의 문제 — 같이 쓰면 효과적

---

## URP를 쓰면 자동인가? — 첫 번째 의문

### URP 자동 적용

- ✅ **SRP Batcher** — 호환 셰이더 쓰면 자동
- ✅ **Frustum Culling** — 모든 프로젝트 기본

### 수동 설정 필요

- ❌ Static Batching — 각 오브젝트 Inspector에서 체크
- ❌ GPU Instancing — 각 머티리얼에서 체크
- ❌ Occlusion Culling — Bake 필요
- ❌ LOD — LOD Group 컴포넌트 추가
- ❌ Sprite Atlas — 수동 생성

**URP 전환만으로 SRP Batcher 덕분에 10~50% 성능 향상 가능하지만, 나머지는 여전히 직접 설정해야 함.**

---

## MaterialPropertyBlock이 뭐야? — 두 번째 의문

### 문제 상황

총알 100개의 색깔을 각각 다르게 하고 싶을 때:
- 머티리얼 100개 → 메모리 낭비
- `renderer.material.color = ...` → Unity가 자동 복제 → 배칭 깨짐

### MaterialPropertyBlock의 역할

**"머티리얼을 복제하지 않고 속성만 오브젝트별로 오버라이드"**

```csharp
private MaterialPropertyBlock propBlock;

void SetColor(Color color)
{
    renderer.GetPropertyBlock(propBlock);
    propBlock.SetColor("_BaseColor", color);
    renderer.SetPropertyBlock(propBlock);
    // material.color 대신 이걸 씀!
}
```

- 머티리얼 1개 + 오브젝트별 속성 오버라이드
- 메모리 효율적
- (기존에는) 배칭 유지

### SRP Batcher와의 충돌

**URP에서는 MaterialPropertyBlock이 SRP Batcher를 깨뜨림!**

- SRP Batcher는 머티리얼 속성을 GPU 버퍼에 미리 올려두는 방식
- MaterialPropertyBlock은 이 버퍼를 쓰지 않고 별도 경로 사용
- **URP 환경: GPU Instancing으로 대체 권장**
- **Built-in 환경: MaterialPropertyBlock 적극 사용**

---

## GPU Instancing은 그냥 다 켜면 안 되나? — 세 번째 의문

### 단점 5가지

**1. 셰이더 지원 필요**
- URP/Lit, URP/Unlit, Standard는 지원
- 오래된 커스텀 셰이더, Shader Graph 일부는 미지원
- 미지원 셰이더에서 체크해도 효과 없음

**2. 특정 상황에서만 효과적**
- 같은 메시 10개 이상일 때만 이득
- 1~2개뿐이면 오버헤드만 발생
- URP는 SRP Batcher가 이미 처리하는 경우 많음

**3. Light Mapping 제약**
- Light Map UV 때문에 제한됨
- 정적 환경은 Static Batching이 나을 수도

**4. MaterialPropertyBlock 사용 제한**
- 같이 쓰려면 셰이더에 특별한 코드 필요
- `multi_compile_instancing`, `UNITY_INSTANCING_BUFFER` 등

**5. 인스턴스당 데이터 크기 제한**
- DrawMeshInstanced: 최대 1023개
- 그 이상은 DrawMeshInstancedIndirect (더 복잡)

### 결론

**"효과 있을 때만 켜라"**
- 같은 메시 10개+ → 켠다
- 풀, 나무, 돌, 적 무리, 총알 → 켠다
- 오브젝트 1~2개 → 끈다
- 실무: 대량 복제용 머티리얼만 켬, 나머지는 기본 끔

---

## 추가 최적화 기법

### Shader Variant 관리
셰이더 설정에 따라 내부 Variant가 여러 개 생성됨. 안 쓰는 Variant 제거.

### Mesh Combining
코드로 여러 메시를 하나로 합치기. Static Batching의 수동 버전.

### Canvas 분리 (UI)
자주 바뀌는 UI와 정적 UI를 별도 Canvas로 분리. Canvas 재빌드 비용 감소.

### GPU Skinning
SkinnedMeshRenderer는 기본 배칭 안 됨. GPU Skinning 활성화.

### Culling Group API
Bounding Sphere로 수천 개 오브젝트의 가시성을 직접 제어.

### Render Layer + Camera Culling Mask
카메라별로 그릴 오브젝트를 Layer로 분리.

---

## Draw Call 목표 수치

| 플랫폼 | 권장 |
|---|---|
| 모바일 (저사양) | 100 이하 |
| 모바일 (일반) | 100~300 |
| PC (중간) | 1000~2000 |
| PC/콘솔 (고사양) | 2000~5000 |
| VR | 150 이하 |

---

## 실무 체크리스트

```
프로젝트 시작:
□ URP 사용 결정
□ SRP Batcher 활성화 확인
□ Dynamic Batching 비활성화 (URP 자동)
□ 셰이더 표준화

에셋 제작:
□ 대량 복제 머티리얼: GPU Instancing 체크
□ UI: Sprite Atlas
□ 텍스처 재활용: Texture Atlas
□ 셰이더 수 최소화

씬 구성:
□ 정적 오브젝트: Static 체크
□ 복잡한 실내: Occlusion Culling Bake
□ 먼 오브젝트: LOD 설정
□ 카메라별 Culling Mask

최적화:
□ Frame Debugger 분석
□ Stats 창 모니터링
□ 실제 기기 테스트
```

---

## 면접 레벨별 기대

| 레벨 | 기대 |
|---|---|
| 주니어 | Draw Call 정의, CPU 병목 이해 |
| 미들 | 배칭 기법 4종 차이 (Static, Dynamic, SRP, Instancing) |
| 시니어 | 상황별 기법 선택, Frame Debugger 분석 |
| 리드 | 파이프라인 선택, 셰이더 전략, 아트팀 협업 기준 |

---

## 전체 여정 정리

```
"Draw Call이 멀수록 품질 낮추는 거 아니야?" → LOD와 혼동
  → Draw Call = CPU가 GPU에게 보내는 명령
    → 많으면 CPU 병목
      → 최적화 기법 6가지
        → Static/Dynamic Batching
        → SRP Batcher (URP 자동)
        → GPU Instancing
        → Texture Atlas
        → Occlusion Culling
          → "URP 쓰면 자동이야?"
            → SRP Batcher만 자동, 나머지는 수동
              → "MaterialPropertyBlock은?"
                → 속성만 오버라이드, 하지만 SRP Batcher와 충돌
                  → "GPU Instancing 다 켜면?"
                    → 셰이더 지원, 효과 상황 한정, 단점 존재
```

---

## 오답 노트

이 세션에서 새로 정리된 인사이트:

1. **Draw Call ≠ LOD/Culling** — Draw Call은 CPU→GPU 통신 횟수 문제, LOD는 GPU 렌더링 비용 문제. 다른 차원의 최적화.

2. **SRP Batcher는 "배칭"이라는 이름이지만 다른 접근** — Draw Call 수를 줄이는 게 아니라, Draw Call당 비용을 극도로 낮춤. 셰이더 단위로 머티리얼 속성을 GPU 버퍼에 미리 올려둠.

3. **URP = SRP Batcher 자동, 나머지는 수동** — URP 전환만으로 얻는 것은 SRP Batcher뿐. GPU Instancing, Static, Occlusion은 여전히 직접 설정.

4. **MaterialPropertyBlock과 SRP Batcher는 상극** — Built-in에서는 좋지만 URP에서는 SRP Batcher를 깨뜨림. URP에서는 GPU Instancing으로 대체하거나 머티리얼 분리.

5. **GPU Instancing은 항상 좋은 게 아니다** — 같은 메시 10개 이상일 때만 효과적. 셰이더 지원, Light Map, MaterialPropertyBlock 호환 등 제약이 많음. "대량 복제 에셋의 머티리얼만" 켜는 게 실무 패턴.

6. **"같은 머티리얼이면 배칭"의 시대는 지났다** — 최신 Unity(URP)에서는 SRP Batcher가 "같은 셰이더" 기준으로 동작. 머티리얼 수보다 셰이더 수를 관리하는 게 중요.
