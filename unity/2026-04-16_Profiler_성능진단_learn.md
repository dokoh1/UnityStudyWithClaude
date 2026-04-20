# Unity 학습 기록 - 2026-04-16 (Q15)

## 세션 정보
- 분야: Unity
- 주제: Profiler 사용법 → CPU/GPU/Memory 진단 → Frame Debugger → 수동 마킹 → 메모리 누수 → 성능 예산 시스템
- 모드: learn
- 날짜: 2026-04-16

---

## Profiler란?

게임이 느린 이유를 찾아주는 진단 도구. "감으로 최적화"하면 안 아픈 곳을 치료하는 꼴. 데이터 기반 접근 필수.

열기: `Window > Analysis > Profiler` (Ctrl+7)

---

## CPU Profiler — 가장 많이 사용

### 그래프 읽기
- 60fps 기준선: 16.67ms
- 색상: 초록(스크립트), 파랑(렌더링), 주황(물리), 보라(UI)
- 기준선 넘는 프레임 = 프레임 드랍

### Hierarchy 뷰 (핵심!)
| 열 | 의미 |
|---|---|
| Total% | 이 함수 + 하위 함수 전체 시간 |
| Self% | 이 함수 자체만의 시간 |
| Calls | 이 프레임에서 호출 횟수 |
| GC Alloc | 힙 할당량 |

**읽는 법**: Self%가 높으면 이 함수 자체가 느림. Total은 높고 Self는 낮으면 하위가 느림. Calls가 비정상적이면 과다 호출.

### Timeline 뷰
시간 순서대로 스레드별 시각화. CPU가 GPU를 기다리는지, 병렬 처리 상태 파악.

### GC Alloc 열 활성화
Hierarchy 뷰 > 상단 열 우클릭 > GC Alloc 체크 → GC Alloc 기준 정렬 → 할당량 높은 함수가 최적화 대상.

---

## GPU Profiler

### Frame Debugger (핵심 도구)
`Window > Analysis > Frame Debugger`

- Enable 클릭 → Draw Call 목록 표시
- 각 Draw Call 클릭 → 그 시점까지 그려진 화면 확인
- **배칭 깨지는 이유 표시**: "Objects have different materials" 등
- 불필요한 Draw Call 발견 가능

### Stats 창
Game View 상단 Stats 버튼:
- Batches: Draw Call 수 (모바일 300 이하)
- Tris: 삼각형 수
- SetPass Calls: 셰이더 변경 수

### Overdraw 모드
Scene View > 셰이딩 모드 > Overdraw. 빨간색 = 심각한 겹침.

---

## Memory Profiler

### Simple 뷰
카테고리별 메모리: Textures(보통 가장 큼), Meshes, Audio, GC Alloc(관리형 힙)

### Detailed 뷰
"Take Sample" → 개별 오브젝트 목록. 비정상적으로 큰 텍스처, 중복 에셋 찾기.

### Memory Profiler 패키지 (별도 설치)
- Tree Map: 큰 에셋 시각화
- 스냅샷 비교 (Diff): 메모리 누수 탐지
- References: 참조 체인 추적

---

## 문제별 진단 플로차트

### "게임이 느려요"
1. Profiler 열기
2. CPU 그래프에서 기준선(16.67ms) 넘는 프레임 클릭
3. 색상으로 1차 진단 (스크립트/렌더링/물리/UI)
4. 해당 항목 상세 분석

### 스크립트가 크면
Self% 정렬 → GC Alloc 확인 → Calls 확인.
흔한 원인: Update에서 GetComponent/Find, string 결합, LINQ, new List.

### 렌더링이 크면
Stats에서 Batches 확인 → Frame Debugger로 배칭 분석 → Overdraw 모드.
흔한 원인: Sprite Atlas 미사용, 머티리얼 과다, 투명 오브젝트, 그림자 과함.

### 물리가 크면
Physics Profiler에서 Active Bodies, Contacts 확인.
흔한 원인: 물리 오브젝트 과다, MeshCollider, Layer Matrix 미설정.

### UI가 크면
Canvas.BuildBatch 확인.
흔한 원인: Canvas 미분리, 매 프레임 텍스트, Layout Group 중첩, Raycast Target 과다.

### GC 스파이크
스파이크 프레임에서 GC.Collect 확인 → GC Alloc 정렬.
흔한 원인: string 결합, new, LINQ, 박싱, WaitForSeconds 미캐싱, foreach Dictionary.

---

## Deep Profile vs 수동 마킹

### Deep Profile
모든 C# 함수 추적. 매우 느림 (5~10배). 정확한 함수별 시간.

### 수동 마킹 (권장)

```csharp
using UnityEngine.Profiling;

Profiler.BeginSample("Enemy.AI");
RunAI();
Profiler.EndSample();
```

오버헤드 거의 없음. 원하는 구간만 측정. 릴리즈에서 자동 제거.

### ProfileScope 패턴 (시니어)

```csharp
public struct ProfileScope : IDisposable
{
    public ProfileScope(string name) { Profiler.BeginSample(name); }
    public void Dispose() { Profiler.EndSample(); }
}

using (new ProfileScope("Enemy.AI")) { RunAI(); }
// Begin/End 짝 실수 방지, 예외 안전, GC 없음
```

---

## 시니어 심화: CustomSampler와 Recorder

### CustomSampler
BeginSample(string)보다 빠름. static으로 미리 생성.

### Recorder — 코드에서 성능 수치 읽기

```csharp
Recorder aiRecorder = Recorder.Get("EnemySystem.AI");
float aiMs = aiRecorder.elapsedNanoseconds / 1_000_000f;

if (aiMs > 5f) Debug.LogWarning("AI 예산 초과!");
```

- Profiler 안 열어도 실시간 수치 확인
- 인게임 디버그 UI에 표시
- 임계값 초과 시 자동 경고
- QA팀이 바로 발견 가능

### 자동 성능 리포트
매 분마다 평균 FPS, 최저 FPS, 스파이크 횟수 등 수집 → 로그/서버 전송.

---

## 시니어 심화: 메모리 누수 탐지

### 스냅샷 비교
1. 게임 시작 직후 스냅샷 A
2. 5분 플레이 (여러 씬 이동)
3. 메인 메뉴로 복귀
4. 스냅샷 B
5. A와 B Diff → 증가한 오브젝트 = 누수!

### 흔한 Unity 메모리 누수
- 이벤트 구독 해제 누락 (OnDisable 빠뜨림)
- 정적 변수에 씬 오브젝트 참조
- renderer.material 자동 복제 후 미해제
- Addressable Release 누락
- 코루틴 미정지
- Object Pool 반환 누락

### 코드로 메모리 추적
```csharp
long usedHeap = Profiler.GetMonoUsedSizeLong();
long totalMem = Profiler.GetTotalAllocatedMemoryLong();
// usedHeap 계속 증가 → 관리형 누수
// totalMem 계속 증가 → 네이티브 에셋 누수
```

---

## 리드 레벨: 성능 관리 시스템

### 성능 예산 (Performance Budget)

```csharp
[CreateAssetMenu]
public class PerformanceBudget : ScriptableObject
{
    public float aiMaxMs = 3f;
    public float physicsMaxMs = 4f;
    public float renderingMaxMs = 6f;
    public float uiMaxMs = 2f;
    // 총합 ~16ms (60fps)
}
```

각 팀원이 자기 시스템의 예산 책임. 초과 시 자동 경고.

### 인게임 성능 오버레이
개발 빌드에서 FPS, 메모리, GC 발생을 실시간 표시. QA팀이 즉시 이슈 발견.

### CI/CD 자동 성능 테스트

```csharp
[UnityTest]
public IEnumerator BattleScene_ShouldMaintain60FPS()
{
    // 10초간 측정
    Assert.Greater(avgFps, 55f);
    Assert.Less(spikeCount, 5);
}

[UnityTest]
public IEnumerator SceneTransition_ShouldNotLeakMemory()
{
    // 씬 왕복 후 메모리 비교
    Assert.Less(diffMB, 10);
}
```

PR마다 자동 실행. FPS 미달/누수 시 PR 블록.

### 모바일 서멀 스로틀링 대응
장시간 테스트로 열 관련 성능 저하 확인. 동적 품질 조절 (FPS 낮아지면 품질 낮춤).

### 코드 리뷰 성능 체크포인트
```
□ Update에 GetComponent?
□ Update에 new?
□ string + 매 프레임?
□ LINQ in Update?
□ OnEnable/OnDisable 짝?
□ Addressable Load/Release 짝?
□ renderer.material 사용?
```

---

## 팀 성능 규칙 문서 예시

```
프레임 예산: 16.67ms (AI:3, Physics:4, Render:6, UI:2, Audio:1)
GC 규칙: Update에서 GC Alloc = 0 필수
Draw Call: 모바일 200 이하, PC 1000 이하
메모리: 모바일 500MB 이하, 텍스처 단일 최대 2048
코드: GetComponent 캐싱, Find 금지, LINQ in Update 금지
의무: PR마다 Profiler 첨부, 주간 성능 리뷰, 출시 전 실기기 3종+
```

---

## 성능 목표 수치

| | 모바일 저사양 | 모바일 고사양 | PC |
|---|---|---|---|
| FPS | 30 | 60 | 60~144 |
| Draw Call | < 100 | < 300 | < 2000 |
| 삼각형 | < 100K | < 300K | < 1M |
| 텍스처 메모리 | < 100MB | < 300MB | < 1GB |
| 총 메모리 | < 500MB | < 1GB | < 2GB |
| GC/프레임 | 0 | 0 | < 1KB |

---

## 면접 레벨별 기대

| 레벨 | 기대 |
|---|---|
| 주니어 | Profiler 존재 인지, Stats 창, FPS 확인 |
| 미들 | CPU Hierarchy 분석, GC Alloc 추적, Frame Debugger |
| 시니어 | 수동 마킹(CustomSampler/Recorder), Memory Profiler 스냅샷 비교, 실기기 프로파일링 |
| 리드 | 성능 예산 시스템, CI/CD 자동 테스트, 팀 가이드라인, 서멀 대응 |

---

## 오답 노트

1. **Profiler 없이 최적화하지 마라** — 데이터 기반 접근 필수. "감"으로 하면 안 아픈 곳을 치료하는 꼴.

2. **에디터 프로파일링 ≠ 실제 성능** — 에디터 오버헤드, Mono vs IL2CPP 차이. 최종은 빌드 + 실기기.

3. **Deep Profile은 일상용이 아님** — 5~10배 느려짐. 수동 마킹(Profiler.BeginSample)으로 대체.

4. **GC Alloc 열을 반드시 활성화** — 기본 안 보임. 우클릭으로 추가. 매 프레임 할당 함수가 최적화 1순위.

5. **Frame Debugger가 "배칭 깨지는 이유"를 알려줌** — Draw Call 클릭 시 오른쪽에 표시.

6. **메모리 누수는 스냅샷 비교로** — 씬 왕복 후 메모리가 안 돌아오면 누수.

7. **Recorder API로 Profiler 없이 수치 읽기** — 인게임 디버그 UI, 자동 경고, 성능 리포트에 활용.

8. **모바일 서멀 스로틀링** — 5분 후 느려지면 열 문제. CPU/GPU 70% 이하 유지.

9. **성능 예산은 SO로 관리** — 시스템별 ms 분배, 초과 시 자동 경고, 팀원별 책임.

10. **CI/CD 성능 테스트** — PR마다 FPS/메모리 자동 검증. 기준 미달 시 블록.
