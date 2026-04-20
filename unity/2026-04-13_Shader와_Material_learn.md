# Unity 학습 기록 - 2026-04-13 (Q9)

## 세션 정보
- 분야: Unity
- 주제: Shader와 Material → ShaderLab/HLSL/Shader Graph → 실무 활용
- 모드: learn
- 날짜: 2026-04-13

---

## 렌더링이 실제로 어떻게 일어나는가

GPU가 오브젝트를 그리는 5단계:

1. **메시 준비** — 정점 정보 (위치, 법선, UV, 색상)
2. **Vertex Shader** — 각 정점을 3D→카메라→화면 좌표로 변환
3. **Rasterization** — GPU가 자동으로 삼각형을 픽셀로 변환
4. **Fragment/Pixel Shader** — 각 픽셀의 최종 색상 계산
5. **화면 출력** — 프레임 버퍼에 기록

---

## Shader가 뭔가

**Shader = GPU에서 실행되는 프로그램**

```
C# 코드 → CPU에서 실행
Shader 코드 → GPU에서 실행

CPU: 코어 8~16개, 순서대로 복잡한 처리
GPU: 코어 수천 개, 대량 병렬 처리
→ 1920x1080 = 200만 픽셀을 병렬 처리하기 위해 GPU 사용
```

### Shader 종류

- **Vertex Shader** — 정점마다 1번, 최종 위치 결정
- **Fragment/Pixel Shader** — 픽셀마다 1번, 최종 색상 결정 (가장 많이 실행)
- **Compute Shader** — 렌더링과 무관한 GPU 계산 (파티클, 이미지 처리)
- **Geometry Shader** — 정점 입력 → 새 기하학 생성 (최신 GPU에서 거의 안 씀)
- **Tessellation Shader** — 메시를 런타임에 쪼갬 (지형 디테일)

---

## Material이 뭔가

**Material = Shader + Shader에 넣을 값들**

```
Shader = 레시피 (요리법)
Material = 이 레시피에 구체적인 재료를 넣은 것

Shader 1개 → Material A, B, C 여러 개가 공유 가능
Material마다 텍스처, 색상, 파라미터가 다름
```

### Shader와 Material의 관계

```
Shader (코드, 공유)
  ├── Material A (값 A) → 오브젝트 1, 2
  ├── Material B (값 B) → 오브젝트 3
  └── Material C (값 C) → 오브젝트 4, 5, 6
```

### Shader 프로퍼티

Shader가 정의한 "입력받을 값"을 Material Inspector에서 조정:
```
Texture: [sword.png]     ← _MainTex
Color: [■ 파란색]        ← _Color
Smoothness: 0.5         ← _Smoothness
Metallic: 0.2           ← _Metallic
```

---

## Shader 작성 방법 3가지

### 방법 1: ShaderLab + HLSL (코드)

```
ShaderLab = Unity만의 셰이더 래퍼 언어
HLSL = 실제 셰이더 코드 (DirectX 표준)

Shader "Custom/MyShader"
{
    Properties { ... }          ← Material 조정값
    SubShader
    {
        Pass
        {
            HLSLPROGRAM
            // Vertex Shader
            // Fragment Shader
            ENDHLSL
        }
    }
}
```

URP/HDRP의 공식 셰이더는 전부 HLSL. 2019+에서 CG에서 HLSL로 전환.

### 방법 2: Shader Graph (노드)

```
코드 대신 노드를 연결해서 셰이더 제작:

[Texture 2D]──┐
              ├→[Multiply]──→[Surface]
[Color]──────┘
```

- 장점: 코드 몰라도 가능, 실시간 프리뷰, 아티스트 친화적
- 단점: 복잡한 로직 어려움, 최적화 제한적
- URP/HDRP 전용 (Built-in 지원 안 함)

### 방법 3: Compute Shader

- 렌더링과 무관한 GPU 계산
- 파티클 시뮬레이션, 이미지 처리, GPGPU
- HLSL로 작성

---

## 3가지 방법 비교

|  | ShaderLab/HLSL | Shader Graph | Compute Shader |
|---|---|---|---|
| 방식 | 코드 | 노드 | 코드 |
| 학습 곡선 | 높음 | 중간 | 높음 |
| 성능 제어 | 최대 | 제한적 | 최대 |
| 복잡한 로직 | 용이 | 어려움 | 최대 |
| 아티스트 | 어려움 | 쉬움 | 불가 |
| 파이프라인 | 모두 | URP/HDRP | 모두 |

### 언제 뭘 쓰는가

- **ShaderLab/HLSL**: 모바일 극한 최적화, Built-in, 복잡한 로직
- **Shader Graph**: URP/HDRP 프로젝트, 아티스트 사용, 일반 효과
- **Compute Shader**: 파티클, 이미지 처리, GPU 범용 계산

---

## Surface Shader (레거시)

Built-in 전용 편의 기능. 조명 계산을 Unity가 자동. URP/HDRP에는 없고 Shader Graph로 대체. 2026년 기준 사실상 레거시.

---

## 렌더 파이프라인별 호환성

|  | Built-in | URP | HDRP |
|---|---|---|---|
| ShaderLab/HLSL | ✅ | URP 헤더 | HDRP 헤더 |
| Shader Graph | ❌ | ✅ | ✅ |
| Surface Shader | ✅ | ❌ | ❌ |

**호환 안 되면 보라색(Magenta)으로 표시** — 이게 "핑크 머티리얼" 에러의 정체.

---

## 실무에서 Shader를 작성/수정하는 이유

### 1. 시각적 효과
- 체력 낮을 때 화면 붉게 (Vignette)
- 피격 시 흰색 깜빡 (Hit Flash)
- 얼음/디졸브 효과
- 홀로그램 UI, 아웃라인
- 물, 잔디 바람

### 2. 최적화
- 모바일에서 PBR 대신 Lambert
- Alpha Blend → Alpha Cut
- 다중 텍스처 → RGBA 채널 통합
- 계산을 Vertex Shader로 이동

### 3. 아트 스타일
- 셀 셰이딩, 픽셀 아트, 수채화
- Genshin Impact, Arcane 같은 커스텀 스타일

### 4. 포스트 프로세싱
- Color Grading, Bloom
- 커스텀 오버레이 (크리티컬 연출)

---

## 레벨별 실무 현실

```
주니어: 기존 셰이더 수정 (색상, 텍스처 교체)
        Shader Graph로 간단한 효과

미들: Shader Graph로 다양한 효과
      HLSL 기초 이해
      기존 셰이더 커스터마이징

시니어: HLSL로 복잡한 셰이더 작성
        최적화 셰이더 제작
        커스텀 렌더링 Pass

리드/TA: 프로젝트 전체 셰이더 시스템 설계
        아티스트 협업 도구 제작
        렌더 파이프라인 커스터마이징
```

---

## 머티리얼 인스턴스의 함정

```csharp
// ❌ 위험
GetComponent<Renderer>().material.color = Color.red;
// → .material 호출 시 Unity가 자동 복제!
// → 오브젝트마다 머티리얼 인스턴스 생성 → 메모리 낭비, 배칭 깨짐

// ✅ 해결 1: 공유 머티리얼 수정
GetComponent<Renderer>().sharedMaterial.color = Color.red;

// ✅ 해결 2: MaterialPropertyBlock
var block = new MaterialPropertyBlock();
block.SetColor("_BaseColor", Color.red);
renderer.SetPropertyBlock(block);
// Built-in에서 좋음, URP는 SRP Batcher와 충돌 주의

// ✅ 해결 3: GPU Instancing
```

---

## 면접 답변 예시

```
Q: "Shader와 Material 시스템을 설명해주세요"

A: "Shader는 GPU에서 실행되는 프로그램으로,
    정점의 위치를 결정하는 Vertex Shader와
    픽셀의 색상을 결정하는 Fragment Shader가 기본입니다.

    Material은 Shader에 구체적인 값(텍스처, 색상, 파라미터)을
    넣은 인스턴스로, 하나의 Shader를 여러 Material이
    공유할 수 있습니다. Shader는 레시피, Material은
    재료를 넣은 완성된 요리에 비유할 수 있습니다.

    Unity에서 Shader 작성은 세 가지 방법이 있습니다.
    ShaderLab과 HLSL은 코드 기반으로 세밀한 제어가 가능하지만
    학습 곡선이 높습니다. Shader Graph는 URP/HDRP에서
    노드 기반으로 시각적으로 작성할 수 있어서
    아티스트도 사용 가능하고 프로토타이핑에 좋습니다.
    Compute Shader는 렌더링과 무관한 GPU 계산에 사용됩니다.

    주니어 레벨에서는 기존 셰이더를 수정하거나
    Shader Graph로 간단한 효과를 만드는 정도지만,
    시니어 이상은 HLSL로 최적화된 셰이더를 직접 작성하거나
    커스텀 렌더링 Pass를 구현합니다."
```

---

## 면접 레벨별 기대

| 레벨 | 기대 |
|---|---|
| 주니어 | Shader/Material 차이, Shader Graph/HLSL 차이 |
| 미들 | Vertex/Fragment 차이, 기본 최적화 원칙 |
| 시니어 | HLSL 작성, 커스텀 Pass, Compute Shader 활용 |
| 리드/TA | 전체 셰이더 시스템 설계, 파이프라인 커스터마이징 |

---

## 오답 노트

1. **Shader는 GPU 프로그램** — C#이 CPU에서 돌듯, Shader는 GPU에서 돈다. GPU는 코어가 수천 개라 대량 병렬 처리에 유리.

2. **Material은 Shader의 인스턴스** — Shader(레시피) + 값 = Material. Shader 1개를 여러 Material이 공유.

3. **Fragment Shader가 Vertex보다 훨씬 많이 실행** — 1920x1080 = 200만 픽셀을 매 프레임. 그래서 Fragment 최적화가 중요.

4. **renderer.material의 함정** — 호출 시 자동 복제. sharedMaterial이나 MaterialPropertyBlock 사용.

5. **Surface Shader는 Built-in 전용, 레거시** — URP/HDRP에는 없고 Shader Graph로 대체됨.

6. **호환 안 되면 보라색** — "핑크 머티리얼" 에러 = 파이프라인 미호환. Built-in 셰이더를 URP에서 쓰면 발생.

7. **HLSL이 표준** — 2019+ 이후 URP/HDRP의 표준. 과거엔 CG를 썼음.

8. **주니어에서는 Shader를 직접 작성 거의 안 함** — 기존 셰이더 수정, Material 값 조정이 대부분. Shader Graph로 간단한 효과 정도.
