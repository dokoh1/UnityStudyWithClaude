# Unity Study With Claude

Unity 게임 엔진 기술 면접 준비를 위한 학습 기록입니다.
Claude Code와 함께 Q&A 형식으로 깊이 있게 학습한 내용을 정리했습니다.

## 학습 목록 (Q1~Q17)

| # | 주제 | 핵심 키워드 |
|---|------|------------|
| Q1 | MonoBehaviour 생명주기 - DI 컨테이너 | Awake/Start, Script Execution Order, VContainer, Zenject |
| Q2 | 카메라 시스템과 Cinemachine | LateUpdate, Lerp vs SmoothDamp, Cinemachine |
| Q3 | Object Pooling과 메모리의 본질 | GC, 힙/스택, C#/C++ 메모리 비교 |
| Q4 | 비동기와 멀티스레드의 모든 것 | Coroutine, UniTask, Job System, Burst, LLVM |
| Q5 | ScriptableObject 완전정복 | SO 이벤트, 변수 공유, 전략 패턴 |
| Q6 | Addressable 완전정복 | AssetBundle, CDN, 디스크 vs 메모리 |
| Q7 | Draw Call 최적화 | Batching, SRP Batcher, GPU Instancing |
| Q8 | 렌더 파이프라인 비교 | Built-in vs URP vs HDRP |
| Q9 | Shader와 Material 시스템 | Vertex/Fragment Shader, Shader Graph |
| Q10 | 물리 엔진 완전정복 | Rigidbody/Collider, Kinematic, PhysX |
| Q11 | GameObject와 Component 시스템 | 컴포지션, GetComponent, Tag/Layer, Null 체크 |
| Q12 | MonoBehaviour 생명주기 종합 | Awake-OnEnable-Start, FixedUpdate/Update/LateUpdate |
| Q13 | UI 시스템 (UGUI / UI Toolkit) | Canvas, Anchor, Mask, 3D Text, UI 최적화 |
| Q14 | 디자인 패턴 완전정복 | Singleton, Observer, State, Command, Strategy |
| Q15 | Profiler 성능 진단 | CPU/GPU/Memory Profiler, Frame Debugger |
| Q16 | DOTS/ECS 기초 | Entity/Component/System, Archetype/Chunk |
| Q17 | 멀티플레이어 (Fusion 2) | Fusion 2 vs NGO, 예측/롤백, Lag Compensation |

## 면접 레벨별 핵심

- **주니어**: 생명주기, GameObject/Component, Rigidbody/Collider, Coroutine, Object Pooling, UGUI
- **미들**: 카메라, ScriptableObject, Draw Call 최적화, GetComponent 캐싱, UniTask, UI 최적화
- **시니어**: DI(VContainer), Job System+Burst, Addressable, 디자인 패턴, Profiler 심화, 예측/롤백
- **리드**: 아키텍처 설계, 성능 예산, CI/CD, 팀 가이드라인, 서버 인프라, DOTS 도입 판단
