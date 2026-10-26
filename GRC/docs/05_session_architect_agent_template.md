---
description: GRC 세션 아키텍트 2.0 및 스토리 시드 엔진에 정합된 안티그래비티 멀티 에이전트 오케스트레이션 템플릿 및 자동화 명세.
related:
  - ../README.md
  - ./00_project_overview.md
  - ./01_grc_architecture.md
  - ./02_user_quickstart_guide.md
  - ./03_grc_user_guide.md
  - ./04_implementation_roadmap.md
  - ../../ai_agent_refs/AI_Agent_Architecture_Paradigms_Guidelines.md
  - ../../ai_agent_refs/AI_Prompt_Engineering_Guidelines.md
  - ../../troubleshooting/grc.md
---
# GRC Session Architect 2.0 Agentic Orchestration Template

본 문서는 GRC(GenAI Roleplay Chat)의 **세션 아키텍트 2.0(Session Architect 2.0)** 4단계 인과 파이프라인과 **Cognitive Core 3.0 스토리 시드(Story Seeds) 엔진** 기술을 안티그래비티(Antigravity) 멀티 에이전트 협동 환경에서 구동하여, 배포용 고품질 세션 팩(`Preset.json`, `Session.json`, `FullHistory.json`)을 자율 제작할 때 준수해야 하는 **공식 오케스트레이션 템플릿 및 실행 절차서(SOP)**를 정의합니다.

> [!IMPORTANT]
> **기술 종속성 및 런타임 파서 계약 계승**
> 본 템플릿은 GRC 전용 데이터 규격([CharacterPreset.cs](../../../GRC/GRC/Models/CharacterPreset.cs), [ChatSession.cs](../../../GRC/GRC/Models/ChatSession.cs), [ChapterContext.cs](../../../GRC/GRC/Models/ChapterContext.cs), [SessionArchitectModels.cs](../../../GRC/GRC/Models/SessionArchitectModels.cs)), [SessionArchitectService.cs](../../../GRC/GRC/Services/SessionArchitectService.cs)의 결정론적 정제기(Sanitizers), [EntityNameResolver.cs](../../../GRC/GRC/Helpers/EntityNameResolver.cs)의 엔티티 명칭·별칭·경계 판정 엔진, [NarrativeContract.cs](../../../GRC/GRC/Services/Cognitive/NarrativeContract.cs)의 서사 문장부호·정규식 SSOT 계약, 그리고 [MemoryManagerService.cs](../../../GRC/GRC/Services/MemoryManagerService.cs)의 3대 스토리 시드 파서(`Pool A/B/C`)에 바이트 단위로 종속됩니다. 에이전트 팀 토폴로지와 계쇄 거버넌스는 [AI_Agent_Architecture_Paradigms_Guidelines.md](../../ai_agent_refs/AI_Agent_Architecture_Paradigms_Guidelines.md) 및 [AI_Prompt_Engineering_Guidelines.md](../../ai_agent_refs/AI_Prompt_Engineering_Guidelines.md)를 단일 진실 공급원(SSOT)으로 준용합니다.

---

<agent_topology>

## 1. 멀티 에이전트 조직 및 역할 분담 (Agent Team Topology)

안티그래비티 환경에서 세션 아키텍트를 구동할 때는 단일 앱 런타임([SessionArchitectService.cs](../../../GRC/GRC/Services/SessionArchitectService.cs))의 **4단계 통합 상태 기계(청사진·세계관 통합 -> 2-Wave 인과 로어북 -> 오프닝 시나리오 선행 -> 상태창·GM 수칙 역추출)**를 그대로 계승하면서, 컨텍스트 오염 차단과 교차 검증 능력을 극대화하기 위해 다음과 같은 4대 전문 워커 및 이중 감사 게이트로 조직합니다.

```mermaid
flowchart TD
    UserPrompt["컨셉 투입: 유저 서사 키워드 / 캠페인 요구사항"]
    
    subgraph ORCHESTRATOR_TIER ["(1) 오케스트레이터 계층 (Stage 1)"]
        MainAgent["Main Architect Agent (청사진 및 4대 섹션 세계관 통합 설계)"]
    end
    
    subgraph WORKER_TIER ["(2) 전문 워커 계층 (Stages 2 to 4)"]
        subgraph CAUSAL_LORE ["Stage 2: 2-Wave 인과 앙상블 로어북"]
            W_Wave1["Wave 1 Stage Scribe: 비인물 무대 인프라 (장소/세력/아이템)"]
            W_Wave2["Wave 2 Ensemble Scribe: 등장인물 전원 통합 앙상블 캐스팅"]
            W_Wave1 -->|"확정된 무대 로어북 주입"| W_Wave2
        end
        
        W_Scenario["Stage 3 Scenario Director: 오프닝 산문 집필 (대사 신규 창작 및 본심 은닉)"]
        W_Status["Stage 4 Status and GM Engineer: Turn 0 상태 역추출 및 GM 4, 5번 수칙 컴파일"]
    end
    
    subgraph AUDIT_TIER ["(3) 독립 감사 계층 (Dual-Layer Gate)"]
        Auditor["Narrative and Schema Auditor (Read-Only 결정론 + 의미론 감사)"]
        AuditGate{"Gate: 검수 합격?"}
    end
    
    subgraph EXPORT_TIER ["(4) 물리적 자산 영구화 (Asset Export)"]
        Export["GRC 세션 폴더 물리 생성 (Preset / Session / FullHistory, Embedding: null 시딩)"]
    end
    
    UserPrompt --> MainAgent
    MainAgent -->|"청사진 + 마스터 세계관 확정"| W_Wave1
    W_Wave2 -->|"그물망 로어북 완성"| W_Scenario
    W_Scenario -->|"오프닝 시나리오 확정"| W_Status
    W_Status -->|"세션 후보 페이로드"| Auditor
    Auditor --> AuditGate
    AuditGate -- "FAIL (핀포인트 수정 대상 지목)" --> MainAgent
    AuditGate -- "PASS (무결점 승인)" --> Export
```

### 서브에이전트별 불변 역할 규격

| 역할 명칭 | 대응 파이프라인 단계 | 도구 권한 | 핵심 책임 및 산출물 |
|---|---|---|---|
| **Main Architect** | Stage 1 (Orchestrator) | Read / Write / Command | 유저 컨셉 분석, `AgentPlan` 및 `<master_setting>`용 4대 번호 섹션 개조식 세계관 동시 수립, 외과수술식 보정 조율, 최종 물리 파일 및 임베딩 생성 |
| **Wave 1 Stage Scribe** | Stage 2 - Wave 1 | Read-Only | 비인물(`장소`, `아이템`, 세부 `세계관`/조직) 무대 인프라 전체를 단일 추론 공간에서 저작 (3줄 불릿 규격 및 `* 서사 훅` 필수) |
| **Wave 2 Ensemble Scribe** | Stage 2 - Wave 2 | Read-Only | 확정된 Wave 1 무대 로어북 전문을 바탕으로 모든 `인물` 전원을 단일 추론 공간에서 앙상블 캐스팅 (5줄 불릿 규격, 말투/결핍 차별화, 양방향 관계망 직조) |
| **Scenario Director** | Stage 3 (Chained) | Read-Only | 확정된 세계관과 로어북 전문 기반 3~4문단 순수 문학 산문 오프닝 집필 (로어북 예시 대사는 어조 참고용으로만 활용한 신규 창작, 내면 본심 은닉, PC 행동/대사 불간섭) |
| **Status & GM Engineer** | Stage 4 (Chained) | Read-Only | 오프닝 현장 기반 Turn 0 상태(`initialChars`, `initialPlaces`, `initialItems`) 역추출, 신규 인물 동적 추적 포함 `StatusUpdateGuide` 설계, 비수치 GM 수칙 4·5번 컴파일 |
| **Narrative Auditor** | Evaluator Gate | Read-Only (판정 전용) | C# 결정론적 정규식 계약(Story Seeds `Pool A/B/C` 파싱 호환성) 및 의미론적 정합성(양방향 관계망, Zero-Numeric, PC 불간섭) 전수 감사 |

</agent_topology>

---

<data_contracts>

## 2. 물리적 데이터 규격 및 JSON 스키마 계약 (Data Contracts)

안티그래비티 에이전트가 최종 생성하는 파일은 GRC 데스크톱 클라이언트의 `Sessions/` 폴더 내에 즉시 로드되어 첫 턴부터 **0초 지연 스트리밍**과 **스토리 시드(`ExtractAvailableStorySeeds`)**가 완벽히 작동해야 합니다.

### 1) 파일 시스템 배치 경로 ([ChatDataHelper.cs](../../../GRC/GRC/Helpers/ChatDataHelper.cs))
폴더명은 `GenerateSessionFileName` 규격(`{SafeSessionTitle}_{yyyyMMdd}_{6자리HexGuid}`)을 준수합니다.
```text
GRC/bin/Debug/net8.0-windows/Sessions/{SanitizedSessionTitle}_{yyyyMMdd}_{6CharHex}/
├── Preset.json          (CharacterPreset 스키마: 로어북 Embedding은 null로 시딩 -> GRC 세션 진입 시 자동 백필)
├── Session.json         (ChatSession 스키마: <initial_scenario> 래핑 이력 및 Turn 0 ChapterContext 포함)
└── FullHistory.json     (UI 말풍선 렌더링용 초기 로그: 반드시 [초기 상황] user 메시지 1개 포함)
```

### 2) Preset.json 데이터 규격 ([CharacterPreset.cs](../../../GRC/GRC/Models/CharacterPreset.cs))
```json
{
  "Name": "세션 고유 제목",
  "Worldview": "1. 배경 및 세계의 핵심 법칙\n- 항목명: 명사구 중심 개조식 내용\n\n2. 세력 구도 및 사회적 현실\n- 항목명: 명사구 중심 개조식 내용\n\n3. 주인공(PC)의 초기 입지 및 직면 환경\n- 항목명: PC의 객관적 위장 신분, 물리적 제약, 당면 과제 (성격/감정 단정 금지)\n\n4. 서사 톤앤매너 및 감각적 디테일\n- 항목명: 일상 호흡(surfaceTone)과 위기 무게감(coreWeight)의 오감 환경 및 NPC 압박감 묘사 기준",
  "SystemPrompt": "<system_directive>\nRole: 3인칭 전지적 게임 마스터\n</system_directive>\n\n<rules>\n1. 추상적 감정 라벨링, 시스템 수치 언급, 문단 끝의 교훈적 요약을 배제하고, 오감(시·청·후각 등)에 기반한 물리적 행동, 시선, 어조, 환경 변화로만 객관적 서술.\n2. 유저 캐릭터(PC)의 대사, 행동, 내면의 생각·감정을 임의로 대신 서술하지 말고, PC의 선택이나 대응이 필요한 시점에서 서술을 멈추고 턴을 넘길 것.\n3. 작위적인 주인공 보정(무조건적 호의·기적적 성공)과 억지 실패 보정을 모두 배제하고, 세계관 법칙 및 당사자 간의 실제 역량·무력·상황 격차와 인물 가치관에 부합하는 현실적 인과율로 판정할 것.\n4. 평소에는 주변 환경의 감각 묘사와 NPC들의 일상 대화 템포를 중심으로 서술하되, 위기나 금기 접촉 시 공기의 냉각과 적대 세력의 압박감으로 환경 분위기를 급반전시킬 것.\n5. 세계관과 로어북의 고유 법칙·제약에 기반하여 유저 행동의 개연성, 대가(Risk), 세력 및 환경의 반작용을 공정하게 판정할 것.\n</rules>",
  "Temperature": null,
  "MaxOutputTokens": 8192,
  "Lorebooks": [
    {
      "Name": "[인물 정식 명칭]",
      "Keywords": ["[인물 정식 명칭]", "[이름 첫 어절]", "[소속 결합 직함]", "[고유 별칭]"],
      "Content": "분류(인물): [신분/소속 및 사회적 역할·정체성 한 줄 정의]\n  * 외양/특징: [출신·직업·성격에서 도출한 고유 시각 특징(머리색·눈동자색·체형·복식)과 습관적 버릇 명사구 나열]\n  * 표면/내면: 표면([대외적 페르소나 및 공적 태도]) / 내면([친밀·위기 시 드러나는 독자적 욕망, 결핍, 비밀 또는 심층 감정])\n  * PC 관계: 초기 접점([Turn 0 또는 본편 첫 대면 계기와 초기 태도]) / 유대 심화(발단 계기: 신뢰·호감 30 이상 달성 시 [NPC가 먼저 제안·요청·공유하는 구체적 사건/의뢰의 시작점] -> 완수 후 변화: [해당 과업을 실제로 해결한 뒤에야 개방되는 관계 전환 및 조력 보상])\n  * 타 인물/세력 관계: [실제로 직접 얽혀 있는 핵심 인물·세력 2~3개와의 구체적 역학 및 갈등]\n  * 말투/대사: [어조 특징 요약]. \"[Turn 0 오프닝과 무관하게 반복되는 일상 상황의 평상시 기본 어조 대사 샘플]\" / \"[현재 관계 단계에서 감정 고조 또는 신뢰 형성 시 어조 변화 대사 샘플]\"",
      "Category": "인물",
      "Priority": 0,
      "Embedding": null
    },
    {
      "Name": "[장소 정식 명칭]",
      "Keywords": ["[장소 정식 명칭]", "[후행 고유 핵심 명사]", "[2어절 이상 고유 복합 별칭]"],
      "Content": "분류(장소): [위치 및 핵심 기능 한 줄 요약]\n  * 감각/외관: [시·청·후각적 환경 디테일 및 공간 구조]\n  * 규칙/원리: [해당 공간 고유의 작동 법칙, 출입 규율 및 제약/대가]\n  * 서사 훅: [직접 연관된 핵심 인물·세력 고유명사 1~3개] 및 [접촉 시 점화될 구체적 사건·단서]",
      "Category": "장소",
      "Priority": 1,
      "Embedding": null
    }
  ],
  "CustomStats": {
    "체력": "100/100",
    "[장르 특화 핵심 지표]": "50/100",
    "[Turn 0 등장 인물명] 신뢰도": "25/100"
  },
  "StatusUpdateGuide": "- 체력: [부상·과부하 조건] 시 -10~-30 감소, [치료·휴식 조건] 시 +15~+40 회복 (0~100 범위 유지)\n- [장르 특화 핵심 지표]: [상승 조건] 시 +3~+10 상승, [하락 조건] 시 -5~-20 하락 (0~100 범위 유지)\n- [Turn 0 등장 인물명] 신뢰도: [인물 가치관에 부합하는 행동] 시 +3~+10 상승, [가치관에 반하는 행동] 시 -5~-20 하락 (0~100 범위 유지)\n- 신규 인물 관계도: 본편 진행 중 아직 상태창에 없는 로어북 주요 인물과 대면·상호작용 시 해당 인물의 호감도 또는 신뢰도(초기치 20~30/100)를 상태창에 신규 추가하여 추적"
}
```

### 3) Session.json 데이터 규격 ([ChatSession.cs](../../../GRC/GRC/Models/ChatSession.cs) & [ChapterContext.cs](../../../GRC/GRC/Models/ChapterContext.cs))
> [!WARNING]
> **런타임 파서 및 상태창 오염 방어 필수 계약**
> 1. `History[0].Text`는 반드시 `<initial_scenario>\n...\n</initial_scenario>` 태그로 감싸야 합니다. [MemoryManagerService.cs](../../../GRC/GRC/Services/MemoryManagerService.cs#L1212)의 `BuildStatusRequest`는 `<initial_scenario>` 포함 여부를 검사하여 장문의 오프닝이 매 턴 상태창 갱신 컨텍스트(`<recent_context>`)에 섞여 들어가는 현상을 차단합니다.
> 2. `History[1].Text`는 이스케이프된 안쪽 따옴표 없이 순수 문자열 `"주어진 세계관과 초기 상황의 흐름을 받아들여, 첫 발걸음을 뗄 준비를 마쳤습니다."`로 고정합니다.
> 3. `CurrentContext`에는 존재하지 않는 `Chapter` 필드 대신, 반드시 `Plot`(`"[제 1장]\n* {ScenarioOutline}"`)과 `ActiveStorySeeds`(`[]`)를 포함해야 하며, `LongTermSummary`의 초기값은 `"아직 요약된 줄거리가 없습니다."`로 설정합니다.

```json
{
  "Preset": { /* 상기 Preset.json 객체와 동일 */ },
  "History": [
    {
      "Role": "user",
      "Text": "<initial_scenario>\n[3~4문단 순수 문학 산문으로 작성된 오프닝 시나리오 전문]\n</initial_scenario>",
      "Timestamp": "2026-10-26T09:00:00+09:00"
    },
    {
      "Role": "model",
      "Text": "주어진 세계관과 초기 상황의 흐름을 받아들여, 첫 발걸음을 뗄 준비를 마쳤습니다.",
      "Timestamp": "2026-10-26T09:00:01+09:00"
    }
  ],
  "CurrentContext": {
    "Plot": "[제 1장]\n* [ScenarioOutline: 오프닝 장소, 대면 인물, 갈등 점화 및 주인공의 첫 선택 상황 요약]",
    "CustomStats": {
      "체력": "100/100",
      "[장르 특화 핵심 지표]": "50/100",
      "[Turn 0 등장 인물명] 신뢰도": "25/100"
    },
    "Chars": {
      "[Turn 0 등장 인물명]": "[핵심 감정·태도 키워드], [즉각적 행동 요약]"
    },
    "Items": [
      "[주인공 휴대 소지품 1]",
      "[주인공 휴대 소지품 2]"
    ],
    "Places": [
      "[오프닝 시작 장소명]"
    ],
    "EventLedgers": [],
    "ConsumedSeeds": [],
    "IsSceneResolved": false,
    "LastSeedTriggeredTurn": 0,
    "ActiveStorySeeds": []
  },
  "LongTermSummary": "아직 요약된 줄거리가 없습니다.",
  "ChapterCount": 1,
  "TotalTurnCount": 0,
  "MediumTermUpdateCount": 0
}
```

### 4) FullHistory.json 데이터 규격 ([FullHistoryLogger.cs](../../../GRC/GRC/Helpers/FullHistoryLogger.cs))
GRC 클라이언트는 세션을 열 때 `FullHistory.json`을 역직렬화하여 채팅 화면의 말풍선 목록을 렌더링합니다. **빈 배열(`[]`)로 두면 세션 진입 시 오프닝 시나리오가 화면에 표시되지 않으므로**, 반드시 아래와 같이 `[초기 상황]` 접두어가 붙은 `user` 메시지 1개를 기록해야 합니다([SessionArchitectService.cs](../../../GRC/GRC/Services/SessionArchitectService.cs#L1715-L1716) 준용).
```json
[
  {
    "Role": "user",
    "Text": "[초기 상황]\n[3~4문단 순수 문학 산문으로 작성된 오프닝 시나리오 전문]",
    "Timestamp": "2026-10-26T09:00:00+09:00"
  }
]
```

</data_contracts>

---

<story_seeds_contract>

## 3. Cognitive Core 3.0 스토리 시드(`Pool A/B/C`) 정규식 파싱 계약

런타임 중 장면이 일단락(`IsSceneResolved == true`)되거나 유저가 `[빠른 진행]`(`fast_forward`)을 실행하면, [MemoryManagerService.cs](../../../GRC/GRC/Services/MemoryManagerService.cs#L228-L406)의 `ExtractAvailableStorySeeds`가 로어북 `Content`를 정규식으로 파싱하여 다음 에피소드 떡밥(`<story_seeds>`)을 자동 주입합니다. 에이전트가 저작하는 모든 로어북은 아래 **정규식 추출 계약**을 100% 만족해야 합니다.

| 시드 풀(Pool) | 발동 대상 및 해금 조건 | 런타임 정규식 추출 대상 필드 (로어북 `Content` 내부) | 슬래시(`/`) 2단계 분리 및 필수 작성 규격 |
|---|---|---|---|
| **Pool A: 미등장 인물 첫 만남** (`CHAR:{Name}`) | `Category == "인물"` 중 아직 `Chars`에 등재되지 않고 미소모된 최우선(`Priority` 오름차순) 인물 1명 | 1) `분류(인물): ...`<br>2) `  * 외양/특징: ...`<br>3) `  * 말투/대사: ...`<br>4) `  * PC 관계: ...` | - **1턴 딜레이 방지 프리뷰**: 첫 조우 턴에는 아직 유저 입력에 해당 인물명이 없어 로어북이 미인출되므로, 엔진이 `분류`, `외양/특징`, `말투/대사`의 첫 번째 큰따옴표 대사(`"..." / "..."` 중 앞쪽)를 추출해 시드에 직접 동봉함.<br>- `* PC 관계:` 값에서 `/ 유대 심화` 앞부분(**초기 접점**)을 추출하여 첫 대면 계기로 제시.<br>- **[시점 불변 원칙]**: Turn 0 오프닝 현장에 없는 인물의 `초기 접점`은 반드시 **Turn 0 이후 본편에서 주인공과 처음 조우(과거 구면인 경우 본편 내 첫 재회·접촉)하여 사건을 점화하는 장면 계기**여야 함. Turn 0 이전에 이미 종료된 과거 배경사를 `초기 접점`에 적으면 이미 지나간 과거 사건이 미래 시드로 주입되고 정작 본편 1차 갈등이 `유대 심화`(호감도 30 해금)에 묶여 데드락이 발생함. |
| **Pool B: 미방문 거점·사건 훅** (`HOOK:{Name}`) | `Category != "인물"` 중 아직 `Places`나 `Items`에 등록되지 않고 미소모된 최우선 항목 1개 | `  * 서사 훅: ...` (`NarrativeHookFieldRegex`) | - 해당 장소·아이템·세력과 접촉했을 때 발생할 구체적인 사건·위기·비밀 단서와 직접 연관된 핵심 인물·세력 고유명사(1~3개)를 1줄로 명시. |
| **Pool C (1단계): 동료 유대 심화** (`BOND:{Name}`) | 이미 만난 인물(`Chars` 등록) 중 `CustomStats` 내 `{인물명} 호감도/신뢰도/유대/친밀` 수치가 **30 이상** 달성 시 | `  * PC 관계: 초기 접점(...) / 유대 심화(...)` (`PcRelationFieldRegex`) | - 반드시 ` / ` 또는 `/ 유대 심화` 구분자를 포함해야 함.<br>- 뒷부분(**유대 심화**)에 주인공과 함께 겪을 중반부 핵심 에피소드 및 관계 전환 조건(장르에 따라 **기밀 공유·이해관계 결탁·전우애·사상적 경쟁·정서적 유대/연심** 등)을 구체적 명사 키워드와 함께 서술. |
| **Pool C (2단계): 내면·비밀 해금** (`BOND_SECRET:{Name}`) | 이미 만난 인물 중 호감도/신뢰도 수치가 **60 이상** 달성 시 (`BOND`와 동시 또는 순차 해금) | `  * 표면/내면: 표면(...) / 내면(...)` (`InnerThoughtFieldRegex`) | - 반드시 `표면(...) / 내면(...)` 2단계 구분자를 포함해야 함.<br>- 뒷부분(**내면**)에 깊은 신뢰·유대 또는 극한의 위기 관계에서만 드러나는 **독자적 신념, 트라우마, 은폐된 진실, 진짜 목적 또는 본심**을 서술. |

> [!CAUTION]
> **인물 로어북 필드명 오기 주의**
> 인물(`Category == "인물"`) 로어북에서 `* 외양/특징:` 대신 비인물용 헤더인 `* 감각/외관:`을 사용하면 `AppearanceFieldRegex`(`^\s*[\*\-]\s*(?:외양/특징|외양|외모)\s*:\s*(.+)$`) 매칭이 실패하여 미등장 인물 첫 조우 시 외양 정보가 누락됩니다. 인물 5줄 불릿(`분류(인물)` -> `* 외양/특징` -> `* 표면/내면` -> `* PC 관계` -> `* 타 인물/세력 관계` -> `* 말투/대사`)과 비인물 3줄 불릿(`분류(카테고리)` -> `* 감각/외관` -> `* 규칙/원리` -> `* 서사 훅`)을 엄격히 구분하십시오.

</story_seeds_contract>

---

<orchestration_sop>

## 4. 단계별 안티그래비티 오케스트레이션 실행 절차 (Execution SOP)

### Stage 1: 컨셉 분석, 방대한 마스터 세계관 및 전 장르 범용 청사진 수립 (Planning & Deep Worldbuilding)
* **주체**: Main Architect Agent
* **입력**: 유저의 한 줄 아이디어 또는 캠페인 요구사항 (판타지, SF, 사이버펑크, 현대 느와르, 미스터리, 시대극, 포스트 아포칼립스, 서브컬처 등 모든 장르 대응)
* **절차**:
  1. **전 장르 범용 아키타입 매핑 (Genre-Agnostic Universal Adaptation)**:
     * 유저 컨셉의 장르적 본질에서 세계 법칙·관계 역학·추적 스탯을 직접 도출하여 아래와 같이 맞춤형으로 설계합니다:

| 장르 도메인 | 거시 세계관(1·2번 섹션) 핵심 축 | 인물 `표면/내면` 및 `PC 관계` 초점 | 권장 추적 스탯 (`CustomStats`) |
|---|---|---|---|
| **하드 SF / 사이버펑크 / 스페이스 오페라** | 기술 패러다임(항성간 항법·신경 임플란트·AI 통제), 메가코프·식민지 지정학, 자원·에너지 희소성과 신체 개조 대가 | 공적 직무·기업 입장(표면) vs 은폐된 아젠다·사상적 회의·생존 목적(내면) / 전술적 공조·기밀 거래·파트너십(`신뢰도`) | `신체 부하`, `신경 동기화율`, `크레딧/보급`, `{인물명} 신뢰도` |
| **현대 / 느와르 / 미스터리 / 정치·첩보** | 법제도·사법 사각지대, 카르텔·정보기관·언론·재벌 간 권력 역학, 수사/정보전의 현실적 절차와 사회적 압력 | 사회적 가면·공식 알리바이(표면) vs 약점·과거 범죄·숨겨진 동기·내부고발 의지(내면) / 정보 교환·공범 의식·수사 공조(`신뢰도`/`친밀`) | `스트레스/정신력`, `수사 노출도/위장도`, `{인물명} 신뢰도` |
| **정통·다크 판타지 / 무협 / 시대극** | 마법·내공·신성력의 원리와 등가교환 대가, 제국·종단·문파·길드 간 역사적 분쟁과 계급 모순 | 가문·문파의 명분과 직책(표면) vs 숙원·복수심·금기 연구·고독(내면) / 생사를 나눈 전우애·주종/동맹 서약·연심(`신뢰도`/`호감도`) | `체력`, `마력/내공/침식도`, `{인물명} 신뢰도/호감도` |
| **서브컬처 / 로맨스 / 아카데미·일상** | 일상 공간의 교류 규율과 이면의 갈등 구조, 관계 위계 및 성장 과제 | 대외적 태도·성격(표면) vs 정서적 결핍·반전 면모·연심(내면) / 정서적 교감·주도권 밀당·반려 서약(`호감도`/`신뢰도`) | `체력/컨디션`, `관계/주도권 지표`, `{인물명} 호감도/신뢰도` |

  2. `AgentPlan` 스키마를 준수하여 `sessionTitle`, 일상 호흡(`surfaceTone`), 핵심 갈등 무게감(`coreWeight`), PC 초기 객관적 입지(`pcInitialPosition`)를 도출합니다.
     * **PC 불간섭 원칙**: `surfaceTone`과 `coreWeight`는 세계 환경과 NPC들의 일상/위기 연출 기준으로만 작성하고, PC의 성격·감정은 유저의 선택 영역으로 남겨 둡니다.
  3. **방대하고 깊이 있는 4대 섹션 마스터 세계관(`Worldview`) 작성 (거시적 독립성 보장)**:
     * 번호 섹션 헤더(`1. 배경 및 세계의 핵심 법칙`, `2. 세력 구도 및 사회적 현실`, `3. 주인공(PC)의 초기 입지 및 직면 환경`, `4. 서사 톤앤매너 및 감각적 디테일`)와 각 하위 하이픈 불릿(`- 항목명: 내용`, **섹션당 4~6개의 고밀도 항목**)으로 작성합니다.
     * **[거시 세계관의 독립성]**: 1·2번 섹션은 **주인공이 없더라도 스스로 굴러가는 거대한 세계의 역사적 기원, 물리/기술/마법/법률 인프라, 지정학적 파벌 대립, 경제·계급적 모순**을 입체적으로 구축하는 데 할당합니다. 주인공 개인의 특수 체질과 특정 인물과의 관계는 3번 섹션과 로어북이 전담합니다.
     * 서술어 어미(`~이다`, `~한다`)를 배제한 **명사구 중심 개조식(Telegraphic Compression)**으로 압축하여 토큰당 정보 밀도를 극대화합니다.
  4. **고용량 그물망 로어북 기획(`lorebookPlan`, 12~20개 이상 권장)**:
     * **런타임 토큰 예산과의 조화**: GRC의 `HybridLorebookRetriever`는 전체 로어북이 10개든 25개든 매 턴 현재 맥락에 매칭된 항목만 최대 3,000토큰 예산 내에서 동적 인출합니다. 따라서 안티그래비티 에이전트는 **주요 인물(5~8명), 구체적 단일 장소(3~6곳), 핵심 단일 아이템/무구/단서(2~4개), 고유 세부 조직/사건/기술(2~4개) 등 총 12~20개 내외의 풍부한 엔트리 풀**을 설계하여 장기 캠페인의 서사 수명과 세계관 깊이를 극대화하십시오.
     * **역할 분리 및 단일 개체 원칙**: 거시적 세계 법칙은 `Worldview`(`<master_setting>`)가 상시 전담하므로, 로어북은 세부 개체(인물·장소·아이템·고유 조직/사건/기술)만 다루며 각 `Name`은 단 하나의 고유 개체명으로 지정합니다(복수 개체는 각각 별도 항목으로 분리). 같은 개체에 딸린 부속 구성요소(동봉품, 병설 공간 등)는 `Name`에 병기하는 대신 대표 고유 명사 1개로 `Name`을 짓고, 본문과 `Keywords`에서 서술합니다.

### Stage 2: 2-Wave 인과 앙상블 로어북 저작 (Causal 2-Wave Ensemble Lorebook Generation)
* **주체**: Wave 1 Stage Scribe & Wave 2 Ensemble Scribe
* **원칙 ([SessionArchitectService.GenerateLorebooksCausallyAsync](../../../GRC/GRC/Services/SessionArchitectService.cs) 준용)**:
  1. **정보 직교성(Information Orthogonality) 및 80/20 고유 밀도 법칙 (어항 효과 & 특성 고착 원천 차단)**:
     * **[단일 소유 기술 원칙]**: 각 소품·신체 특징·사건 문구는 그것을 직접 소유하거나 관장하는 1개 로어북에만 상세 기술하고, `Worldview`·`SystemPrompt`·다른 로어북에서는 고유명사로만 참조합니다. (근거: 동일 문구가 여러 로어북에 중복 기재되면 런타임에 3~4개 로어북이 동시 인출될 때 서사 AI가 매 문단 같은 버릇과 소품을 반복 묘사하는 **특성 과잉 고착(Trait Fixation)**과 **어항 속 세계관(Fishbowl Effect)**이 발생합니다.)
     * **[80/20 고유 정보 할당]**: 각 로어북 항목 텍스트의 **80% 이상은 다른 어디에도 적혀 있지 않은 그 개체만의 고유한 디테일**(장소의 역사적 유래와 공간 구조, 조직의 내부 파벌과 작동 교리, 아이템의 세부 메커니즘, 인물의 고유한 가치관·과거사·독립적 과업)에 할당하고, 나머지 **20% 이내에서만 실제로 직접 얽혀 있는 핵심 대상(2~3개)과의 관계**를 연결하십시오.
  2. **Wave 1 — 비인물 무대 인프라 통합 저작 (`Category != "인물"`)**:
     * 장소, 아이템, 세부 조직/법칙 항목 전체를 먼저 저작하여 지리적·제도적·기술적 설정을 확정합니다.
     * **비인물 3줄 불릿 구조**:
       ```text
       분류(장소|아이템|세계관): 핵심 정의 한 줄 요약
         * 감각/외관: 시·청·후각적 핵심 환경/외형 디테일 및 고유 공간/구조적 특징 (명사구 중심)
         * 규칙/원리: 고유 법칙, 규율, 작동 메커니즘, 역사적 유래 및 제약/대가
         * 서사 훅: 직접 연관된 핵심 인물·세력 고유명사(1~3개) 및 상호작용 시 발생 가능한 구체적 사건·단서·이면
       ```
  3. **Wave 2 — 등장인물 전원 통합 앙상블 캐스팅 (`Category == "인물"`)**:
     * 확정된 Wave 1 무대 로어북 전문(`<established_world_lore>`)과 전체 청사진 명단(`<full_blueprint_roster>`)을 참조하여 **모든 등장인물 전원**을 동시 조형합니다.
     * **인과적 캐릭터 진정성 및 독립적 주체성(Independent Agency) 4대 원칙**:
       1. **[독립된 삶의 목적]**: 모든 등장인물에게 **주인공과 무관하게 추구하는 자신만의 삶의 목표, 직업적·정치적 과업, 사상적 신념, 혹은 해결해야 할 개인적 문제**를 부여합니다.
       2. **[미대면 인물의 조건부 유대 설계]**: Turn 0 이전에 주인공과 깊은 인연이 없는 **미대면 인물**(감사관, 경쟁자, 타 세력 인물, 낯선 정보상 등)의 `내면`에는 그 인물 본연의 고독·야망·비밀·가치관만 기술하고, 주인공과의 깊은 신뢰나 정서적 유대는 첫 대면(`초기 접점`) 이후 공동의 사건(`유대 심화`)을 통과하며 형성되는 조건부 가능성으로 설계합니다.
       3. **[앙상블 관계 스펙트럼 및 시각 정체성 분산]**: 인물별 관계의 결을 장르와 역할에 맞춰 **이해관계에 따른 전략적 제휴, 사상적 라이벌리, 냉철한 검증과 시험, 생사를 나눈 전우애, 지적 호기심, 점진적 연심** 등으로 서로 다르게 배정합니다. 각 인물의 외양(머리색·눈동자색·체형·복식·습관적 제스처)은 그 인물의 출신·직업·소속 문화권·성격에서 인과적으로 도출하여, 세션 내 인물들이 외양만으로도 한눈에 구별되도록 설계합니다(유저 컨셉에 외모 지정이 있으면 해당 지정을 우선).
       4. **[유대 심화 인과 단계 분리(`발단 계기 -> 완수 후 변화`)]**: `* PC 관계`의 `유대 심화`는 **`유대 심화(발단 계기: 신뢰·호감 30 이상 달성 시 [NPC가 먼저 꺼내는 구체적 의뢰·거래 제안·비밀 공유·사건 개막] -> 완수 후 변화: [해당 과업을 실제로 해결한 뒤에야 전환되는 관계·보상])`** 형태로 작성합니다. `발단 계기`에는 현재 시점의 미해결 시작점만, `완수 후 변화`에는 과업 해결 이후의 상태만 기술합니다. (근거: 과업 완료를 전제한 단일 요약문은 런타임 호감도 30 도달 시 서사 AI가 중간 과정을 생략하고 최종 보상부터 지급하거나 미발생 사건을 환각하는 원인이 됩니다.) `* 말투/대사`의 첫 번째 대사는 Turn 0 오프닝 장면과 무관하게 반복되는 일상 상황(식사·업무·이동 등)에서의 평상시 기본 어조로 작성합니다. (근거: 첫 번째 대사는 3단계 마스킹 대상이 아니어서 관계 단계와 무관하게 매 턴 `<world_lore>`로 주입되므로, 특정 사건 직후의 일시적 감정 상태를 담으면 관계가 진전된 뒤에도 그 인물의 말투가 해당 상태에 고착됩니다.) 두 번째 대사는 현재 관계 단계에서 감정이 고조되거나 신뢰가 싹틀 때의 어조 변화만 보여주도록 작성합니다.
     * **인물 5줄 불릿 구조**:
       ```text
       분류(인물): 신분/소속, 사회적 역할 및 핵심 정체성 한 줄 정의
         * 외양/특징: 출신·직업·성격에서 도출한 핵심 시각적 인장, 복장, 습관적 행동/버릇 (명사구 나열)
         * 표면/내면: 표면(대외적 페르소나·공적 태도·역할) / 내면(친밀·위기·이해관계 충돌 시 드러나는 독자적 욕망, 신념, 결핍, 비밀 또는 심층 감정 — 미대면 인물은 그 인물 본연의 동기로만 구성)
         * PC 관계: 초기 접점(Turn 0 현장에 없는 인물은 Turn 0 이후 본편에서의 첫 조우·재회 및 1차 사건 점화 계기와 초기 태도로 작성) / 유대 심화(발단 계기: 신뢰·호감 30 이상 달성 시 NPC가 먼저 꺼내는 구체적 의뢰·거래 제안·비밀 공유·사건 개막 -> 완수 후 변화: 해당 과업을 실제로 해결한 뒤에야 전환되는 기밀 공유·동맹 결탁·전우애·경쟁 심화·정서적 유대)
         * 타 인물/세력 관계: 실제로 직접 얽혀 있는 핵심 인물·세력·장소(2~3개)만 선별한 구체적 역학·이해관계·갈등 (고유명사 직접 명시)
         * 말투/대사: 어조 특징 요약. "Turn 0 오프닝과 무관하게 반복되는 일상 상황의 평상시 기본 어조 대사 샘플" / "현재 관계 단계에서 감정 고조 또는 신뢰 형성 시 어조 변화 대사 샘플"
       ```
     * **서브컬처 메타 은어의 세계관 내 번역(In-Universe Translation)**: 유저 컨셉에 서브컬처 캐릭터 유형 은어·메타 용어가 포함된 경우, 로어북 본문과 `StatusUpdateGuide`에는 해당 개념을 세계관에 어울리는 구체적인 행동·심리·관계 주도권 묘사로 번역하여 작성합니다(문체 오염 방지).
  4. **우선순위(`Priority`) 및 키워드(`Keywords`) 정규화 ([EntityNameResolver.DeriveAliases](../../../GRC/GRC/Helpers/EntityNameResolver.cs) 오탐 방지 수칙)**:
     * `Priority`는 오름차순 스케일(`0`=최우선 핵심 주연/거점, `1`=주요 조연/장소, `2`=일반, `3`=배경)로 부여합니다.
     * `Keywords`는 ① 엔트리 정식 `Name`, ② **`인물` 카테고리에 한하여** 고유 인명 첫 어절(`"{이름} {성}"` -> `"{이름}"`), ③ 한국어 조사가 제거된 2어절 이상의 복합 고유명사·직함·별칭을 합쳐 최대 8개로 구성합니다.
     * **[비인물 키워드 고유성 원칙]**: 비인물(`장소`, `아이템`, `세계관`) 키워드는 정식 명칭, 명칭 내 후행 고유 핵심 명사, 또는 2어절 이상의 고유 복합명사로만 등록합니다. (근거: 복합 명칭의 선행 수식어(방위·재질·색채·형용사)나 일상 보통명사를 단독 키워드로 등록하면 `HybridLorebookRetriever` 정규식 스캔과 `Pool B` 시드 소모가 매 턴 오탐(False Positive)을 일으킵니다.)

### Stage 3: 오프닝 시나리오 선행 집필 (Scenario-First Prose Authoring)
* **주체**: Scenario Director Subagent
* **절차**:
  1. 확정된 `Worldview`와 전체 로어북 전문을 바탕으로 Turn 0 첫 직면 장면을 **3~4문단 분량의 순수 문학 산문**으로 집필합니다.
  2. **문장 부호 및 순수 산문 규격 (`FormatRule` 100% 일치)**:
     * 외부 발화 대사는 큰따옴표(`"..."`), **인물의 내면 독백(속마음)은 낫표(`「...」`)**, 행동과 배경 지문은 기호 없는 순수 평문으로만 작성합니다.
     * **[낫표 독백 전용 원칙]**: 낫표(`「...」`)는 인물의 속마음 독백에만 사용하고, 지문 내 장소명·아이템명·세력명 등 고유명사는 기호 없는 평문으로 표기합니다. (근거: Turn 0 본문에서 낫표가 고유명사 강조에 쓰이면 런타임 서사 LLM이 이를 모방하여 속마음 기호와 고유명사 표기가 뒤섞이는 문체 오염이 발생합니다.)
  3. **3대 오프닝 연출 수칙**:
     * **[대사 신규 창작]**: 로어북 `* 말투/대사` 예시 문장은 어조 참고용으로만 활용하고, 말투 특징을 유지한 채 오프닝 상황에 맞는 새로운 대사를 창작합니다.
     * **[본심 은닉]**: 유대가 깊어진 뒤에야 드러나는 2단계 `내면`의 본심은 첫 만남에서 은닉하고, 표면 페르소나와 미세한 행동 단서로만 암시합니다.
     * **[PC 주체성 보존]**: 주인공(PC)의 대사·의사결정·감정은 유저의 몫으로 남기고, NPC(1~3명 이내로 밀도 있게 집중)가 말을 건네거나 사건이 발생하여 주인공이 첫 행동을 선택해야 하는 직면 순간에서 서술을 멈춥니다.

### Stage 4: Turn 0 상태창 역추출 및 하이브리드 GM 시스템 프롬프트 컴파일 (Status Induction & Prompt Compilation)
* **주체**: Status & GM Engineer Subagent
* **절차**:
  1. **초기 스탯(`CustomStats`) 및 상태창 갱신 가이드(`StatusUpdateGuide`) 설계**:
     * 주인공 핵심 상태 수치(범위형 `"100/100"`, 단일 정수 `"100"`, 상태형 `"평온"`)와 오프닝 현장에 직접 등장한 인물의 초기 관계도(`"{인물명} 호감도"` 또는 `"{인물명} 신뢰도"`, 예: `"25/100"`)를 설정합니다.
     * `StatusUpdateGuide`는 순수 하이픈 개조식(`- 스탯명: 상승 조건(+폭) / 하락 조건(-폭) / 유효 범위`)으로 작성하며, **미등장 로어북 인물과 본편에서 신규 조우 시 해당 인물의 호감도/신뢰도(초기치 20~30/100)를 상태창에 동적 추가하여 추적하는 규칙**을 반드시 마지막 줄에 포함합니다(Pool C 시드 해금 필수 조건).
  2. **Turn 0 현장 컨텍스트(`initialChars`, `initialPlaces`, `initialItems`) 정밀 역추출**:
     * `initialChars`: 오프닝 시나리오에 직접 등장해 대사·행동을 주고받은 NPC만 등록합니다. **Key는 반드시 로어북 인물 정식 `Name`과 100% 일치**시켜야 하며(`NormalizeInitialCharKeys`), **Value는 오프닝 마지막 순간의 `"핵심 감정·태도 키워드, 즉각적 행동 요약"`만 3~6어절(25자 이내)**로 간결하게 작성합니다(복장은 로어북, 장소는 `initialPlaces`가 전담).
     * `initialPlaces`: 오프닝 장면이 실제로 벌어지고 있는 구체적인 시작 장소명 배열(1~2개).
     * `initialItems`: 오프닝 마지막 순간 주인공이 몸에 직접 휴대·착용 중이거나 손이 닿는 현장 소지품(1~4개)만 등록합니다.
  3. **하이브리드 시스템 프롬프트(`SystemPrompt`) 컴파일 (`CompileHybridSystemPrompt`)**:
     * 불변 상단 헌법(`CoreSystemDirectivesHeader`, 규칙 1~3번) 하단에 세션 특화 비수치 GM 수칙 4번·5번(`genreDirectives`)을 결합하고 `</rules>`로 닫습니다:
       * `4. {서사 분위기 및 환경 연출 수칙}`: 일상 호흡(`surfaceTone`)과 위기 긴장감(`coreWeight`)의 온도차를 **배경 환경의 감각 묘사(공기, 빛, 소리, 냄새)와 NPC들의 대사 템포·태도·압박감**으로만 연출하도록 지시(지시문 주어는 환경과 NPC로 한정).
       * `5. {세계관 인과율 및 위험 판정 수칙}`: 세계관과 로어북의 고유 법칙·제약에 기반하여 유저 행동의 개연성, 대가(Risk), 세력/환경의 반작용을 공정하게 판정하도록 지시(수치는 `StatusUpdateGuide`가 전담 — Zero-Numeric).

### Stage 5: 독립 감사관 이중 게이트 실사 (Dual-Layer Deterministic & Semantic Gate)
* **주체**: Narrative & Schema Auditor Subagent (Read-Only)
* **검수 기준 (Auditor Rubric)**:
  * [ ] **결정론적 구조 검수 ([ReviewStepContentAsync](../../../GRC/GRC/Services/SessionArchitectService.cs#L1498-L1623) 동등성)**:
    * 기획된 모든 로어북 항목이 누락 없이 생성되었는가?
    * 모든 `인물` 로어북이 5줄 필수 불릿(`* 외양/특징`, `* 표면/내면`, `* PC 관계`, `* 타 인물/세력 관계`, `* 말투/대사`)을 갖추었으며, `* 표면/내면`과 `* PC 관계`에 `/` 구분자가 존재하고, `* 말투/대사`에 큰따옴표 샘플 대사 2개(`"..." / "..."`)가 포함되었는가?
    * 모든 비인물 로어북이 3줄 필수 불릿(`* 감각/외관`, `* 규칙/원리`, `* 서사 훅`)을 갖추었는가?
    * 텍스트 내 비정상 다국어 글리치 문자(`[\u0400-\u06FF\u0900-\u0E7F]+`), 마크다운 볼드/별표(`**`, `*`), `#` 헤더, 대괄호 의사 태그(`[표면]` 등)가 완전히 제거되었는가?
    * `Session.json`의 `History[0].Text`에 `<initial_scenario>` 래퍼가 있고, `CurrentContext.Plot` 및 `FullHistory.json`의 `[초기 상황]` 엔트리가 정확히 기록되었는가?
  * [ ] **의미론적 심층 검수 (Multi-Agent Deep Audit)**:
    * **거시 세계관의 독립성 및 스케일**: `Worldview`의 1·2번 섹션이 세계 자체의 역사적 기원·지정학·기술/마법/사회 시스템·구조적 모순을 깊이 있게 구축하고 있는가?
    * **정보 직교성(Orthogonality) 및 단일 소유 기술**: 각 소품·신체 특징·사건 문구가 그것을 소유·관장하는 1개 로어북에만 상세 기술되고 나머지에서는 고유명사로만 참조되는가? 각 로어북이 해당 개체 고유의 독자적 디테일에 80% 이상을 할애하고 직접 얽힌 2~3개 대상만 연결했는가?
    * **인과적 캐릭터 진정성 및 미대면 인물 조건부 유대**: 모든 등장인물이 주인공과 무관한 자신만의 독립된 삶의 목적(야망, 신념, 과업)을 지니고 있는가? Turn 0 미대면 인물의 `내면`이 그 인물 본연의 동기로만 구성되고, 주인공과의 유대가 첫 대면 이후 사건을 거쳐 형성되는 조건부 가능성으로 설계되었는가?
    * **미등장 인물 `초기 접점` 시점 정합성**: Turn 0(`initialChars`)에 등장하지 않는 인물들의 `* PC 관계: 초기 접점(...)`이 Turn 0 이후 본편에서 실제로 발생할 1차 대면/사건 점화 계기로 작성되었는가?
    * **키워드 고유성**: 비인물/장소/아이템 로어북 `Keywords`가 정식 명칭, 후행 고유 핵심 명사, 2어절 이상 고유 복합명사로만 구성되었는가?
    * **낫표 용도 및 메타 은어 번역**: 오프닝 시나리오 지문에서 낫표(`「...」`)가 속마음 독백에만 쓰였는가? 유저 컨셉의 서브컬처 메타 용어가 로어북 및 `StatusUpdateGuide`에서 세계관 내 행동·심리 묘사로 번역되었는가?
    * **장르 적합성 및 앙상블 분산**: 장르(SF, 현대, 미스터리, 판타지 등)에 맞는 어휘와 관계 역학이 적용되었으며, 인물별 대사 어조·내면 동기·관계 양상·외양(머리색·눈동자색·체형·복식·제스처)이 서로 구별되는가?
    * **오프닝 대사 창작 및 본심 은닉**: 오프닝 시나리오의 대사가 로어북 샘플과 다른 새 문장으로 창작되었으며, 첫 만남에서 2단계 내면이 행동 단서로만 암시되었는가?
    * **기본 어조 샘플 및 대표 명칭**: 모든 인물의 `* 말투/대사` 첫 번째 대사가 Turn 0 오프닝 장면과 무관한 평상시 기본 어조로 작성되었는가? 모든 로어북 `Name`이 부속 구성요소를 병기하지 않은 대표 고유 명사 1개로 지어졌는가?
    * **PC 불간섭 및 Zero-Numeric**: `Worldview`, `SystemPrompt`(4·5번 수칙), `Scenario`가 환경과 NPC만을 서술 주체로 삼고, 시스템 수치는 `StatusUpdateGuide`에만 격리되었는가?
* **외과수술식 핀포인트 보정 (`[수정 대상: 항목명]`)**:
  * 특정 로어북 항목에 결함이 발견된 경우 `[수정 대상: 항목1, 항목2]` 형태로 결함 항목만 핀포인트 재생성하여 병합합니다(전체 배열 재작성 시 압축 손실(Lazy Compression) 발생, [ReviseLorebooksSurgicallyAsync](../../../GRC/GRC/Services/SessionArchitectService.cs#L1064-L1203) 준용).

### Stage 6: 물리 파일 영구화 및 런타임 임베딩 자동 백필 위임 (Physical Export & Runtime Auto-Backfill)
* **주체**: Main Architect Agent (Write / File Tool 실행)
* **절차**:
  1. **로어북 `Embedding: null` 시딩 및 더미 벡터 절대 금지**:
     * 에이전트가 `Preset.json`을 생성할 때 각 로어북 엔트리의 `"Embedding"` 필드는 반드시 **`null`**로 기록합니다.
     * [ChatViewModel.InitializeWithSession](../../../GRC/GRC/ViewModels/ChatViewModel.cs#L231-L297)은 세션을 로드할 때 `Embedding == null || Embedding.Length != 768`인 로어북을 감지하면 백그라운드(`Task.Run`)에서 자동으로 [LorebookService.EnsureEmbeddingsAsync](../../../GRC/GRC/Services/LorebookService.cs#L133-L194)를 호출하여 `"{Name} ({Keywords}): {Content}"` 포맷 기반 `gemini-embedding-001` (`SEMANTIC_SIMILARITY`, 768차원) 실측 벡터를 발급·병합하고 `Preset.json`과 `Session.json`에 영구 저장합니다.
     * **주의**: 만약 에이전트가 임의의 가짜 숫자 768개를 채워 넣으면 `EnsureEmbeddingsAsync`가 이미 유효 임베딩이 있는 것으로 오판(`entry.Embedding.Length == 768` 스킵 가드)하여 시맨틱 검색이 영구 손상되므로, 더미 배열 생성을 엄격히 금지합니다.
  2. **물리 파일 기록**:
     * `Sessions/{SanitizedSessionTitle}_{yyyyMMdd}_{6CharHex}/` 디렉터리에 `Preset.json`, `Session.json`, `FullHistory.json` 3개 파일을 UTF-8(`WriteIndented: true`, `UnsafeRelaxedJsonEscaping`)로 저장합니다.

</orchestration_sop>

---

<prompt_blueprints>

## 5. 프롬프트 청사진 및 핵심 불변 지침 (Prompt Blueprints & Invariants)

### 1) 시스템 헌법 접두사 ([CoreSystemDirectivesHeader](../../../GRC/GRC/Services/SessionArchitectService.cs#L139))
에이전트가 생성하는 모든 GM 지시문(`SystemPrompt`) 상단에는 반드시 아래 불변 헌법이 100% 바이트 단위로 보존되어야 하며, 4번·5번에 세션 특화 비수치 수칙이 결합됩니다.

```xml
<system_directive>
Role: 3인칭 전지적 게임 마스터
</system_directive>

<rules>
1. 추상적 감정 라벨링, 시스템 수치 언급, 문단 끝의 교훈적 요약을 배제하고, 오감(시·청·후각 등)에 기반한 물리적 행동, 시선, 어조, 환경 변화로만 객관적 서술.
2. 유저 캐릭터(PC)의 대사, 행동, 내면의 생각·감정을 임의로 대신 서술하지 말고, PC의 선택이나 대응이 필요한 시점에서 서술을 멈추고 턴을 넘길 것.
3. 작위적인 주인공 보정(무조건적 호의·기적적 성공)과 억지 실패 보정을 모두 배제하고, 세계관 법칙 및 당사자 간의 실제 역량·무력·상황 격차와 인물 가치관에 부합하는 현실적 인과율로 판정할 것.
4. {세션_특화_배경환경_감각묘사_및_NPC_템포_연출수칙_PC불간섭}
5. {세션_특화_세계관법칙_인과관계_및_위험대가_판정수칙_ZeroNumeric}
</rules>
```

### 2) 9대 불변 제약 (Invariant Rules)
[AI_Prompt_Engineering_Guidelines.md](../../ai_agent_refs/AI_Prompt_Engineering_Guidelines.md) 원칙 10(Affirmative Framing)에 따라 각 제약은 수행해야 할 대체 행동으로 기술합니다.
1. **Always anchor Turn 0 with prose**: 0번 턴 `model` 응답은 표준 3인칭 산문 앵커(`"주어진 세계관과 초기 상황의 흐름을 받아들여, 첫 발걸음을 뗄 준비를 마쳤습니다."`)로 고정합니다.
2. **Always seed `<initial_scenario>` and `FullHistory.json`**: `Session.json`의 `History[0].Text`는 `<initial_scenario>` 태그로 감싸고, `FullHistory.json`에는 `[초기 상황]` `user` 메시지 1개를 기록합니다.
3. **Keep `SystemPrompt` static and non-numeric**: `SystemPrompt`에는 불변 헌법과 비수치 GM 수칙만 두고, 수치 증감 규칙은 `StatusUpdateGuide`, 인물 프로필은 로어북에 격리합니다.
4. **Leave PC personality to the user**: `surfaceTone`, `coreWeight`, `Worldview` 3·4번 섹션, `SystemPrompt` 4번 수칙, `Scenario`는 환경과 NPC만을 서술 주체로 삼습니다.
5. **One entity per lorebook, macro rules in `<master_setting>`**: 거시 세계관은 `Worldview`가 전담하고, 각 로어북은 단일 고유 개체만 다루며 `Name`은 대표 고유 명사 1개로 짓습니다(부속 구성요소는 본문과 `Keywords`에서 서술).
6. **Single-owner descriptions (Orthogonality)**: 각 소품·신체 특징·사건 문구는 소유 개체 1개 로어북에만 상세 기술하고, `* 타 인물/세력 관계`에는 직접 얽힌 2~3개 대상만 기재합니다.
7. **Conditional bonds for unacquainted NPCs**: Turn 0 미대면 인물의 `내면`은 그 인물 본연의 동기로만 구성하고, 주인공과의 유대는 첫 대면 이후 사건을 통해 형성되도록 설계합니다(Independent Agency).
8. **Fresh dialogue and concealed inner truth in the opening**: 오프닝 대사는 로어북 샘플과 다른 새 문장으로 창작하고, 2단계 `내면`은 행동 단서로만 암시합니다.
9. **Surgical edits touch only flagged entities**: 부분 수정 시 결함이 지목된 엔트리만 재생성하고, 나머지 엔트리는 원문 그대로 보존합니다.

</prompt_blueprints>

---

<verification_criteria>

## 6. 최종 산출물 검증 기준 (Ground Truth Verification)

안티그래비티 에이전트가 빌드한 세션 자산이 실제 GRC 세션 아키텍트 2.0과 동등 이상으로 작동하는지 확인하는 필수 QA 기준입니다.

1. **스키마 및 역직렬화 무결성**:
   * C# .NET 클라이언트의 `SessionService.LoadSessionAsync`, `PresetStorageService.LoadPresetAsync`, `FullHistoryLogger.LoadFullHistoryAsync`를 통해 단 하나의 예외 없이 메모리에 로드되고, 세션 진입 즉시 오프닝 시나리오 말풍선과 상태창(`Chars`, `Places`, `Items`, `CustomStats`)이 화면에 렌더링되어야 합니다.
2. **스토리 시드(`Pool A/B/C`) 파싱 호환성**:
   * `MemoryManagerService.ExtractAvailableStorySeeds` 실행 시 미등장 인물의 `CHAR:{Name}` 시드에 `(분류: ... | 외양: ... | 말투: "...") -> 초기 접점(...)`이 누락 없이 조합되고, 미방문 장소/아이템의 `HOOK:{Name}` 시드 및 호감도 30/60 달성 시의 `BOND:{Name}` / `BOND_SECRET:{Name}` 시드가 정규식으로 정확히 추출되어야 합니다.
3. **임베딩 시딩 및 자동 백필 호환성**:
   * 에이전트가 최초 생성한 `Preset.json`의 모든 로어북 `Embedding` 필드는 `null`로 깨끗하게 시딩되어 있어야 하며(임의의 가짜 768개 부동소수점 배열 주입 절대 금지), GRC에서 해당 세션을 열었을 때 `ChatViewModel.InitializeWithSession`의 백그라운드 백필을 통해 실제 `gemini-embedding-001` (`float[768]`) 벡터가 자동 채워지고 디스크에 저장되어야 합니다.

</verification_criteria>
