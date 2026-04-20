# Unity Study With Claude

Unity 게임 엔진 기술 면접 준비를 위한 학습 기록입니다.
Claude Code와 함께 Q&A 형식으로 깊이 있게 학습한 내용을 정리했습니다.

## 학습 목록 (Q1~Q17) - 완료

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

---

## 심화 학습 목록 (Q18~Q32) - 예정

면접을 넘어 실무에서 차별화되는 Unity 지식.

### 에디터 확장

| # | 주제 | 핵심 키워드 |
|---|------|------------|
| Q18 | Custom Inspector / Property Drawer | CustomEditor, OnInspectorGUI, PropertyDrawer, SerializedProperty |
| Q19 | Custom Editor Window / 자동화 도구 | EditorWindow, MenuItem, AssetDatabase, BuildPipeline |

### 그래픽스 심화

| # | 주제 | 핵심 키워드 |
|---|------|------------|
| Q20 | Lighting과 Shadow 심화 | Realtime/Baked/Mixed, Light Probe, Reflection Probe, GI, Lightmap |
| Q21 | Post Processing과 VFX | Volume, Bloom, Color Grading, DoF, VFX Graph, Particle System |

### 애니메이션 심화

| # | 주제 | 핵심 키워드 |
|---|------|------------|
| Q22 | Animator Controller 심화 | Blend Tree, Animation Layer, Avatar Mask, State Machine Behaviour |
| Q23 | IK / Root Motion / Animation Rigging | Foot IK, Hand IK, Look At, Root Motion, 런타임 IK/FK |

### 씬 / 에셋 관리

| # | 주제 | 핵심 키워드 |
|---|------|------------|
| Q24 | 씬 관리와 로딩 시스템 | LoadSceneAsync, Additive, DontDestroyOnLoad, 로딩 화면, 멀티 씬 |
| Q25 | 에셋 임포트 설정과 자동화 | 텍스처 압축(ASTC/BC7), Mipmap, AssetPostprocessor, 오디오 설정 |

### 입력 / 오디오

| # | 주제 | 핵심 키워드 |
|---|------|------------|
| Q26 | New Input System | Action Map, Input Action, 게임패드/터치/키보드 통합, 로컬 멀티 |
| Q27 | 오디오 시스템과 Audio Mixer | AudioSource/Listener, 3D Sound, Mixer 채널, Snapshot, FMOD/Wwise |

### 수학 / 알고리즘

| # | 주제 | 핵심 키워드 |
|---|------|------------|
| Q28 | 게임 수학 (벡터, 쿼터니언, 보간) | Dot/Cross, Quaternion, Slerp, 짐벌 락, AnimationCurve |
| Q29 | Raycast / NavMesh / 경로 탐색 | Physics.Raycast, SphereCast, NonAlloc, NavMeshAgent, Off-Mesh Link |

### 보안 / 빌드 / 플랫폼

| # | 주제 | 핵심 키워드 |
|---|------|------------|
| Q30 | 보안과 에셋 보호 | IL2CPP 난독화, 메모리 해킹 방지, 서버 검증, PlayerPrefs 암호화 |
| Q31 | CI/CD와 빌드 자동화 + 플랫폼 대응 | BuildPipeline, GitHub Actions/GameCI, Git LFS, 모바일 GPU(TBDR) |

### 실전 Gotchas

| # | 주제 | 핵심 키워드 |
|---|------|------------|
| Q32 | Unity 알려진 함정 모음 (Gotchas) | Fake Null, Time.timeScale, Destroy 타이밍, Camera.main 캐싱, yield 박싱 |

---

## 학습 방식

- **learn 모드**: 개념 설명 - 질문 - 꼬리질문 - 심화 - 정리
- **스토리텔링**: 각 주제를 대화 흐름 그대로 기록
- **오답 노트**: 매 세션마다 새로 알게 된 인사이트 정리
- **면접 레벨별**: 주니어 / 미들 / 시니어 / 리드 기대 수준 구분

## 면접 레벨별 핵심 요약

### 주니어
- MonoBehaviour 생명주기 (Awake/Start/Update)
- GameObject/Component 개념
- Rigidbody/Collider 기본
- Coroutine 기본
- Object Pooling 개념
- UGUI 기본 (Canvas, Button, Image)

### 미들
- 카메라 추적 (LateUpdate, SmoothDamp)
- ScriptableObject 활용 (데이터, 이벤트)
- Draw Call 최적화 (배칭 기법 4종)
- GetComponent 캐싱, Tag/Layer 활용
- async/await vs Coroutine, UniTask
- UI 최적화 (Canvas 분리, Raycast Target)

### 시니어
- DI 컨테이너 (VContainer)
- Job System + Burst
- Addressable 에셋 관리
- 디자인 패턴 실전 적용 (EventBus, HFSM)
- Profiler 심화 (CustomSampler, Memory Profiler)
- 멀티플레이어 예측/롤백

### 리드
- 프로젝트 아키텍처 설계 (의존성 단방향)
- 성능 예산 시스템, CI/CD 성능 테스트
- 팀 가이드라인 (코딩 규칙, 코드 리뷰 체크포인트)
- 네트워크 토폴로지 선택, 서버 인프라 설계
- DOTS 도입 판단, 에셋 파이프라인 설계
