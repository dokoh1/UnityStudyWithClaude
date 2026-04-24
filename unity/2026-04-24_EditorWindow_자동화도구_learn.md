# Unity 학습 기록 - 2026-04-24 (Q19)

## 세션 정보
- 분야: Unity
- 주제: Custom Editor Window / 자동화 도구 → AssetPostprocessor → 빌드 자동화 → 리드의 도구 설계
- 모드: learn
- 날짜: 2026-04-24

---

## Editor Window란?

Custom Inspector = 기존 Inspector 커스터마이징 (특정 컴포넌트 종속)
Editor Window = 완전히 새로운 독립 창 (Inspector처럼 탭으로 도킹 가능)

실무에서 만드는 예: 아이템 에디터, 레벨 에디터, 로컬라이제이션 도구, 에셋 검사, 빌드 도구, CSV 임포터

---

## 레벨별 에디터 확장 개념

### 주니어: MenuItem과 기본 자동화

**MenuItem** = 에디터 메뉴에 내 기능 추가

```
알아야 하는 것:
├── MenuItem 위치 규칙 ("Tools/" 아래 팀 도구)
├── Selection API (선택한 오브젝트에 대해 작업)
├── Undo (Ctrl+Z 되돌리기 지원)
├── EditorUtility (SetDirty, DisplayDialog, ProgressBar)
└── 단축키 연결 (%, #, &)
```

실전: 적 HP 리셋, (Clone) 제거, Missing Script 찾기 등 반복 작업 자동화.

### 미들: EditorWindow와 데이터 주도 도구

**EditorWindow** = 프로젝트 전용 독립 도구 창

```
알아야 하는 것:
├── AssetDatabase (에셋 검색/로드/생성/삭제)
├── SerializedObject in EditorWindow (Undo/Redo 지원)
├── GUI 레이아웃 (OnGUI = 매 프레임, Immediate Mode)
├── EditorWindow 생명주기 (OnEnable, OnGUI, OnSelectionChange)
└── 설계 원칙: 실수 방지, 시각적 피드백, 검색/필터링
```

만들 줄 알아야 하는 것: 데이터 뷰어/에디터 (SO 테이블), 씬 도우미, 프리팹 관리 도구

### 시니어: AssetPostprocessor와 파이프라인 자동화

**AssetPostprocessor** = 에셋 임포트 시 자동 처리

```
개념:
  EditorWindow: 수동 실행 (사람이 클릭)
  AssetPostprocessor: 자동 실행 (임포트 시)
  
이벤트 체계:
  파일이 Assets에 들어옴
    → OnPreprocess____() ← 임포트 전, 설정 변경 가능
    → Unity가 임포트
    → OnPostprocess____() ← 임포트 후, 결과 확인

알아야 하는 것:
├── 폴더 기반 규칙 ("UI/" → Sprite + Mipmap OFF)
├── 에셋 검증 (4096 초과 텍스처 경고)
├── 빌드 전후 자동화 (IPreprocessBuild/IPostprocessBuild)
└── 빌드 파이프라인 (BuildPipeline.BuildPlayer)
```

빌드 자동화 단계:
1. 빌드 전 검증 (에셋 규칙, 디버그 코드)
2. 빌드 실행 (플랫폼별)
3. 빌드 후 처리 (CDN 업로드, 알림, 리포트)
4. CI/CD 연동 (GitHub Actions, Jenkins)

### 리드/TA: 팀 워크플로우 설계

리드는 "도구를 만드는 것"이 아니라 **"워크플로우를 설계"**

```
도구 설계 원칙:
├── 누구를 위한 도구? (프로그래머/디자이너/아티스트)
├── 만들 가치가 있는가? (ROI: 제작 시간 vs 절약 시간)
├── 유지보수할 수 있는가? (만든 사람 퇴사하면?)
└── 기존 도구로 안 되는가? (Odin, 에셋스토어)
```

#### 전체 도구 체계

```
에셋 파이프라인: AssetPostprocessor(자동), 에셋 검증
데이터 파이프라인: CSV 임포터, 데이터 에디터
빌드 파이프라인: 빌드 전 검증, 원클릭 빌드, CDN 업로드
디버그 파이프라인: 성능 모니터, 게임 상태 뷰어, 치트 콘솔
```

#### 도구 제작 우선순위
- Phase 1 (초기): AssetPostprocessor + 폴더/명명 규칙
- Phase 2 (프로토 후): CSV 임포터 + 기본 빌드 도구
- Phase 3 (본격 개발): 데이터 에디터 + 에셋 검증
- Phase 4 (알파/베타): 디버그 도구 + CI/CD

#### TA의 도구
셰이더 프리뷰, LOD 생성, 텍스처 채널 패킹, 아틀라스 생성, VFX 프리뷰

---

## 자동화의 4단계

```
Level 1 (수동/주니어):    MenuItem 버튼 클릭
Level 2 (반자동/미들):    EditorWindow 도구
Level 3 (자동/시니어):    AssetPostprocessor, IPreprocessBuild
Level 4 (강제/리드):      CI/CD에서 규칙 위반 시 빌드 실패
```

리드의 판단: "이 작업은 어떤 Level이어야 하는가?"
- 텍스처 압축: Level 3 (자동, 실수하면 치명적)
- 아이템 데이터: Level 2 (반자동, 디자이너 편의)
- 디버그 치트: Level 1 (수동, 가끔만)
- 빌드 전 검증: Level 4 (강제, 배포 품질)

---

## 실전 도구 예시

### 아이템 에디터 (EditorWindow)
좌측 테이블(목록/검색/필터) + 우측 상세(SerializedObject 편집). AssetDatabase.FindAssets로 SO 검색, 생성/삭제/수정, Undo 지원.

### CSV → ScriptableObject 임포터
구글 시트에서 CSV 다운로드 → 드래그&드롭 → 자동 변환. 기존 에셋이면 업데이트, 없으면 새로 생성.

### AssetPostprocessor
폴더 기반 자동 설정: UI/ → Sprite+Mipmap OFF, BGM/ → Streaming, SFX/ → Mono+DecompressOnLoad

---

## 면접 레벨별 기대

| 레벨 | 기대 |
|---|---|
| 주니어 | MenuItem으로 간단한 자동화 |
| 미들 | EditorWindow (데이터 뷰어, 검색/필터, AssetDatabase) |
| 시니어 | AssetPostprocessor, 빌드 자동화, 파이프라인 설계 |
| 리드/TA | 팀 도구 체계 설계, ROI 판단, CI/CD, 워크플로우 자동화 |

---

## 오답 노트

1. **EditorWindow vs Custom Inspector** — Inspector는 컴포넌트 종속, Window는 독립적. 여러 에셋을 동시에 다루려면 Window.

2. **AssetPostprocessor = 자동화의 핵심** — 사람에게 의존하지 않는 품질 관리. 아티스트가 설정을 몰라도 자동 적용.

3. **도구 제작 ROI** — 3일 투자로 200시간 절약 = 만들 가치. 2주 투자로 2시간 절약 = 만들 가치 없음.

4. **자동화는 단계적** — 수동(MenuItem) → 반자동(Window) → 자동(Postprocessor) → 강제(CI/CD). 모든 것을 Level 4로 할 필요 없음.

5. **리드는 도구가 아니라 워크플로우를 설계** — "어떤 도구가 필요한가"보다 "팀이 실수할 수 없는 구조"를 만드는 것.
