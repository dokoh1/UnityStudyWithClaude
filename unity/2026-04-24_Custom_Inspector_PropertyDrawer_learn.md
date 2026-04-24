# Unity 학습 기록 - 2026-04-20~24 (Q18)

## 세션 정보
- 분야: Unity
- 주제: Custom Inspector / Property Drawer — 에디터 확장 기초
- 모드: learn
- 날짜: 2026-04-20 ~ 2026-04-24
- 비고: Odin Inspector 사용 경험 있음

---

## 에디터 확장이란?

Unity 에디터 자체를 커스터마이징하는 것. Inspector를 프로젝트에 맞게 바꾸거나, 자동화 도구를 만들거나, 에디터 전용 윈도우를 만드는 것.

---

## 기본 규칙

### Editor 폴더
에디터 스크립트는 반드시 `Editor` 폴더 안에. 빌드에 포함 안 됨.

```
Assets/Scripts/Editor/EnemyEditor.cs  ← 에디터 전용
Assets/Scripts/Game/Enemy.cs          ← 런타임 코드
```

### 핵심 네임스페이스
```csharp
using UnityEditor;  // Editor 폴더에서만 사용 가능
```

---

## Custom Inspector

### 기본 패턴
```csharp
[CustomEditor(typeof(Enemy))]
public class EnemyEditor : Editor
{
    public override void OnInspectorGUI()
    {
        DrawDefaultInspector();  // 기본 Inspector + 아래에 추가
        
        Enemy enemy = (Enemy)target;
        if (GUILayout.Button("HP 리셋"))
        {
            enemy.hp = enemy.maxHp;
            EditorUtility.SetDirty(target);
        }
    }
}
```

### 완전 커스텀 (SerializedProperty 사용)
```csharp
[CustomEditor(typeof(Enemy))]
public class EnemyEditor : Editor
{
    SerializedProperty hpProp, maxHpProp;
    bool showStats = true;
    
    void OnEnable()
    {
        hpProp = serializedObject.FindProperty("hp");
        maxHpProp = serializedObject.FindProperty("maxHp");
    }
    
    public override void OnInspectorGUI()
    {
        serializedObject.Update();  // ① 시작
        
        // Foldout, Slider, ProgressBar, Button 등 자유 배치
        showStats = EditorGUILayout.Foldout(showStats, "스탯");
        if (showStats)
        {
            EditorGUILayout.IntSlider(hpProp, 0, maxHpProp.intValue, "HP");
            // HP바 시각화 등
        }
        
        serializedObject.ApplyModifiedProperties();  // ④ 적용
    }
}
```

### SerializedProperty를 쓰는 이유
- Undo/Redo 자동 지원
- 멀티 오브젝트 편집 자동
- Prefab Override 파란색 표시

---

## Property Drawer

특정 타입 또는 Attribute의 Inspector 표시 방식을 커스텀.

### ReadOnly Attribute
```csharp
// 런타임 (Editor 폴더 밖)
public class ReadOnlyAttribute : PropertyAttribute { }

// Editor 폴더
[CustomPropertyDrawer(typeof(ReadOnlyAttribute))]
public class ReadOnlyDrawer : PropertyDrawer
{
    public override void OnGUI(Rect pos, SerializedProperty prop, GUIContent label)
    {
        EditorGUI.BeginDisabledGroup(true);
        EditorGUI.PropertyField(pos, prop, label);
        EditorGUI.EndDisabledGroup();
    }
}
```

### ShowIf Attribute (조건부 표시)
```csharp
public class ShowIfAttribute : PropertyAttribute
{
    public string conditionField;
    public ShowIfAttribute(string field) { conditionField = field; }
}
// isBoss가 true일 때만 phaseCount 표시
```

---

## 자주 쓰는 API

```csharp
// 표시
EditorGUILayout.LabelField("제목", EditorStyles.boldLabel);
EditorGUILayout.HelpBox("경고", MessageType.Warning);
EditorGUILayout.Space(10);

// 입력
EditorGUILayout.PropertyField(prop);
EditorGUILayout.IntSlider(prop, 0, 100);

// 레이아웃
EditorGUILayout.BeginHorizontal(); / EndHorizontal();
foldout = EditorGUILayout.Foldout(foldout, "그룹");
EditorGUI.indentLevel++;

// 버튼
if (GUILayout.Button("실행")) { }
```

---

## Scene View 시각화

```csharp
void OnDrawGizmosSelected()
{
    Gizmos.color = Color.green;
    Gizmos.DrawWireSphere(transform.position, radius);
    Handles.Label(transform.position + Vector3.up * 2, "Spawn Point");
}
```

---

## Odin Inspector와의 관계

Odin Inspector = 에디터 확장을 코드 없이 Attribute만으로 해결하는 유료 에셋.

```
기본 Unity:
  Custom Inspector 코드 작성 필요 (50~200줄)
  Property Drawer 코드 작성 필요

Odin Inspector:
  [ShowIf("isBoss")] int phaseCount;   ← 코드 1줄!
  [Button] void ResetHP() { }          ← 버튼 1줄!
  [FoldoutGroup("Stats")] int hp;      ← 그룹 1줄!
  [ReadOnly] string info;              ← 읽기전용 1줄!
  [ProgressBar(0, 100)] int hp;        ← HP바 1줄!
  
→ Odin은 "Custom Inspector를 안 짜도 되게" 해주는 도구
→ 하지만 원리를 알아야 Odin 없이도 할 수 있음
→ Odin이 없는 프로젝트도 있으니 기본기 필요
```

---

## 면접 레벨별 기대

| 레벨 | 기대 |
|---|---|
| 주니어 | [Header], [Range], [Tooltip] 기본 Attribute |
| 미들 | Custom Inspector 기본 (DrawDefaultInspector + 버튼) |
| 시니어 | SerializedProperty, Property Drawer, 복잡한 UI |
| 리드/TA | 팀 도구 설계, AssetPostprocessor, EditorWindow |

---

## 오답 노트

1. **Editor 폴더 필수** — 에디터 스크립트가 폴더 밖에 있으면 빌드 시 에러.
2. **SerializedProperty > 직접 참조** — Undo/Redo, 멀티 편집, Prefab Override 지원.
3. **Update/ApplyModifiedProperties 필수 짝** — 안 하면 값이 저장 안 됨.
4. **Odin은 편의 도구** — 원리를 알면 없이도 가능. 있으면 생산성 극대화.
