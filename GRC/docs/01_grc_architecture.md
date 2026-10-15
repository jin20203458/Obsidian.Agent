---
description: GRC C#/.NET WPF 클라이언트 상세 아키텍처 명세 및 UI/비동기 패턴 구성도.
related:
  - ../README.md
---
# GRC Architecture & Deep Dive

본 문서는 GRC(GenAI Roleplay Chat) 클라이언트의 핵심 엔진 구조와, C#/.NET 8.0 환경에서 달성한 동시성 제어 및 메모리 관리 기법을 상세히 기술합니다.

---

<memory_pipeline>

## 1. 인지 코어 3.0 메모리 파이프라인 (Cognitive Core 3.0 Memory Pipeline)

대규모 롤플레잉 서사를 진행할 때, 모든 대화를 프롬프트에 무제한 누적하면 컨텍스트 윈도우 초과, 프롬프트 캐시 미스, 비용 폭발 및 서사 왜곡(Lost in the middle)을 초과합니다. GRC v3.0은 이를 해결하기 위해 **불변 접두사 동결(Prefix-Frozen SystemInstruction)**, **사건 렛저(Event Ledger) 팩트 추출**, **게이티드 하이브리드 RAG**, **이중 계층 복선 추적(Clue Tracker)**을 유기적으로 결합한 Cognitive Core 3.0 아키텍처를 탑재했습니다.

`[IMPLEMENTED]` 메모리 파이프라인 및 인지 모듈 구현체:
- 파이프라인 총괄: [MemoryManagerService.cs](../../../GRC/GRC/Services/MemoryManagerService.cs)
- 인지 프롬프트 조립기: [CognitivePromptAssembler.cs](../../../GRC/GRC/Services/CognitivePromptAssembler.cs)
- 사건 렛저 추출기: [EventLedgerExtractor.cs](../../../GRC/GRC/Services/EventLedgerExtractor.cs)
- 하이브리드 로어북 인출기: [HybridLorebookRetriever.cs](../../../GRC/GRC/Services/HybridLorebookRetriever.cs)
- 살리언스 게이티드 RAG: [GatedRAGRetriever.cs](../../../GRC/GRC/Services/GatedRAGRetriever.cs)
- 이중 계층 복선 추적기: [ClueTracker.cs](../../../GRC/GRC/Services/ClueTracker.cs)
- 시간 인식 관계 투영기: [BiTemporalEntityProjector.cs](../../../GRC/GRC/Helpers/BiTemporalEntityProjector.cs)
- SIMD 벡터 연산 헬퍼: [VectorMathHelper.cs](../../../GRC/GRC/Helpers/VectorMathHelper.cs)

```mermaid
flowchart TB
    subgraph COGNITIVE_CORE ["Cognitive Core 3.0 Pipeline"]
        SYS["불변 시스템 지시문 (Prefix-Frozen SystemInstruction)<br>- 페르소나 및 세계관 불변 동결 (Prompt Cache Hit Rate 극대화)"]
        
        subgraph TIERS ["3-Tier Memory & Reasoning Pipeline"]
            M1["단기 기억 (Raw History): 최근 18턴"]
            M2["중기 기억 (Event Ledger): 불변 팩트 불릿 + 768차원 임베딩 누적"]
            M3["장기 기억 (Chronicle): Gemini Pro 살리언스 증발 1,500자 연대기"]
            
            M1 -->|"18턴 도달 시 8턴 절삭 후 백그라운드 추출"| M2
            M2 -->|"10회 누적(80턴) 시 챕터 마감 및 큐 이관"| M3
        end
        
        subgraph AUXILIARY ["Selective Auxiliary Cognitive Engines"]
            HL["하이브리드 로어북 (HybridLorebookRetriever)<br>- 1차 Regex 완전 일치 + 2차 text-embedding-004 코사인 유사도"]
            GR["살리언스 게이티드 RAG (GatedRAGRetriever)<br>- 살리언스 3점 이상 시 과거 챕터 렛저 SIMD 핀포인트 인출"]
            CT["이중 계층 떡밥 추적기 (ClueTracker)<br>- 활성/휴면 단서 추적, 최대 20턴 TTL 관리"]
            BP["시간 인식 관계 투영기 (BiTemporalEntityProjector)<br>- 초기 상태 vs 현재 상태 감정 낙차 1줄 투영"]
        end
        
        ASSEMBLER["인지 프롬프트 조립기 (CognitivePromptAssembler)<br>- 토큰 예산 패킹 & User 최신 턴 직전 동적 슬롯 주입"]
        
        SYS --> ASSEMBLER
        TIERS --> ASSEMBLER
        AUXILIARY --> ASSEMBLER
        ASSEMBLER --> LLM["Gemini 3.8 Flash 서사 스트리밍 (0초 응답)"]
    end
```

### 1) 불변 접두사 동결 (Prefix-Frozen SystemInstruction)
*   세계관, 시스템 페르소나, 불변 지시문을 `SystemInstruction`에 완전히 격리하여 고정합니다.
*   동적으로 변하는 로어북, 사건 렛저, 단서, 챕터 줄거리는 최신 유저 메시지 직전의 **동적 컨텍스트 슬롯(Dynamic Context Slot)**에 주입함으로써, LLM 엔진의 프롬프트 캐시 적중률을 극대화하고 문체 오염을 방지합니다.

### 2) 단기 기억 (Raw History)
*   가장 최근에 주고받은 날것의 대화 18턴을 손실 없이 보존합니다.
*   **비동기 압축 트리거**: 18턴에 도달하면 가장 오래된 8턴을 절삭하여 백그라운드 스레드(`Task.Run`)에서 사건 팩트 추출을 시작합니다. 낙관적 락(`_shortTermVersion`)을 통해 동시성 정합성을 보장합니다.

### 3) 중기 기억 (Event Ledger 팩트 누적)
*   절삭된 8턴은 **Gemini 3.5 Flash-Lite** 모델을 통해 문학적 허구나 감상적 수사를 배제한 **3~5개의 불변 팩트 불릿(Fact Bullets)**으로 정제 추출됩니다.
*   추출된 각 불릿은 구글 다국어 임베딩(`text-multilingual-embedding-002`, 서울 리전 `asia-northeast3` 직결) API를 통해 768차원 시맨틱 벡터가 결합되어 현재 챕터의 `EventLedgers`에 누적됩니다.
*   소설 재작성(덮어쓰기)으로 인한 팩트 유실 및 문체 왜곡을 원천 차단합니다. 10회 누적(약 80턴) 시 챕터를 전환하고 이전 챕터를 큐로 밀어냅니다.

### 4) 장기 기억 (Chronicle 살리언스 증발)
*   챕터 큐가 가득 차면(Max 3), 가장 오래된 챕터가 큐에서 빠져나와 **Gemini 4.0 Pro** 모델에 전달됩니다.
*   **살리언스 증발(Salience Evaporation)** 기법을 적용하여 핵심 인과율과 세계관의 변화(Canon Events)만을 선별해 1,500자 이내의 대과거 연대기로 영구 보존합니다.

### 5) 보조 인지 엔진 (Selective Auxiliary Cognitive Engines)
*   **하이브리드 로어북 (`HybridLorebookRetriever`)**: 정규식 1차 키워드 스캔과 시맨틱 임베딩 2차 유사도 검색(골디락스 임계치 0.70f, 3턴 잔존 Sticky)을 결합하여 오탈자나 간접적 언급에도 설정집을 누락 없이 인출합니다. (대화 우클릭 수동 기억 추출은 단일 책임 원칙(SRP)에 따라 [LorebookService.cs](../../../GRC/GRC/Services/LorebookService.cs)로 완전 분리 격리됨)
*   **살리언스 게이티드 RAG (`GatedRAGRetriever`)**: 유저 입력의 복잡도/복선 살리언스 점수가 3점 미만일 때는 0ms로 바이패스하고, 3점 이상일 때만 과거 완료된 챕터들의 사건 렛저를 SIMD(`TensorPrimitives`, 임계치 0.70f)로 즉시 인출합니다.
*   **이중 계층 단서 추적기 (`ClueTracker`)**: 상태창 모델이 발견한 서사 떡밥을 Active/Archived 계층으로 추적하며, 3단계 하이브리드 게이팅(키워드 -> 3턴 Sticky -> 0.70f 시맨틱)을 통해 플레이어가 직접 상호작용한 대상 단서만 핀포인트로 인출합니다.
*   **시간 인식 관계망 (`BiTemporalEntityProjector`)**: 인물 관계를 단순 호감도 숫자가 아닌 `[초기: 불신과 경계] -> [현재: 깊은 유대와 의지]` 형태로 시간 궤적을 1줄 투영하여 극적 감정선의 깊이를 부여합니다.

</memory_pipeline>

---

<storage_optimization>

## 2. O(1) 무부하 파일 쓰기 (Zero-Deserialize JSON Append)

대화 이력을 디스크에 영구 저장(`FullHistory.json`)할 때 발생하는 오버헤드를 극단적으로 최적화했습니다.

`[IMPLEMENTED]` 파일 조작 알고리즘은 [FullHistoryLogger.cs](../../../GRC/GRC/Helpers/FullHistoryLogger.cs)에 구현되어 있습니다.

*   **문제점**: 일반적으로 JSON 배열에 요소를 추가할 때 전체 파일을 메모리로 읽어 역직렬화(Deserialize)한 뒤, 리스트에 Add하고 다시 직렬화하여 덮어씁니다. 서사가 길어질수록 이 과정에서 발생하는 GC(가비지 컬렉터) 스파이크와 I/O 지연이 UI를 멈추게 합니다.
*   **해결책 (`FullHistoryLogger.cs`)**:
    역직렬화를 완전히 포기하고 바이너리 스트림 조작으로 대체했습니다.
    1.  `FileStream`으로 파일을 엽니다.
    2.  끝에서부터 역방향으로 탐색(Seek)하여 배열을 닫는 괄호 `]`를 찾습니다.
    3.  새로운 메시지 텍스트를 바이트 단위로 직접 주입(Append)하고 다시 `]`를 덮어씌웁니다.
    4.  결과적으로 파일의 크기가 10MB이든 1GB이든 **O(1)의 일정한 시간 복잡도**로 파일 쓰기가 완료됩니다.

</storage_optimization>

---

<concurrency_streaming>

## 3. 비차단 동시성 제어 및 60FPS 스트리밍

네트워크 통신과 UI 렌더링 간의 병목을 완벽히 격리하여 '0초 지연(Zero-Delay)' 타이핑 경험을 제공합니다.

`[IMPLEMENTED]` 렌더링 틱 제어는 [StatefulStreamingHelper.cs](../../../GRC/GRC/Helpers/StatefulStreamingHelper.cs)에, 통신 채널은 [ChatWorkflowService.cs](../../../GRC/GRC/Services/ChatWorkflowService.cs)에 분리 구현되어 있습니다.

### 생산자-소비자 스트리밍 격리 (`System.Threading.Channels`)
*   `ChatWorkflowService` 내부에서 Gemini API로부터 문자 스트림을 받아오는 스레드(생산자)와, WPF 대화창에 문자를 그리는 스레드(소비자)를 `Channel<char>`로 분리했습니다. 네트워크 지연(Lag)이 발생해도 생산자 큐만 대기할 뿐 UI 렌더러는 절대 굳지 않습니다.

### 60FPS 배치 렌더링 (`StatefulStreamingHelper.cs`)
*   문자가 들어올 때마다 WPF의 `Dispatcher`를 호출하면 윈도우 메시지 루프가 포화되어 렌더링 스레드가 마비됩니다.
*   이를 막기 위해 유입되는 문자를 `ConcurrentQueue<char>`에 담아둡니다.
*   `DispatcherTimer`를 `DispatcherPriority.Render` 우선순위로 16ms(60FPS)마다 한 번씩 틱(Tick)을 발생시켜, 큐에 쌓인 문자를 일괄 Drain한 뒤 **틱당 단 한 번만 UI에 반영(FlushBuffer)**합니다.

</concurrency_streaming>

---

<session_architect>

## 4. AI Session Architect (자율 저작 에이전트)

유저가 기획서를 작성할 필요 없이, 키워드 한 줄만으로 TRPG 세션을 빌드해 주는 6단계 에이전트입니다.

`[IMPLEMENTED]` 6단계 스테이트 머신은 [SessionArchitectService.cs](../../../GRC/GRC/Services/SessionArchitectService.cs)에 구현되어 있습니다.

*   **상태 기계(State Machine)**: 기획(Planning) → 세계관 → 로어북 → 스탯 → 초기 시나리오 → GM 지시문으로 이어지는 단계별 컨텍스트 상속 모델을 구축했습니다.
*   **AI 감사관 자가 검수(Self-Review)**:
    자동 모드를 켜면, 매 단계가 끝날 때마다 또 다른 내부 AI(Auditor)가 생성된 JSON 규격과 개연성을 검사합니다.
    오류가 발견되면 `{ "pass": false, "issues": "문제 사유" }`를 반환하며, 이 사유를 바탕으로 `ReviseContentAsync`를 호출해 인간의 개입 없이 코드를 스스로 수정하고 다음 단계로 전진합니다.

</session_architect>

---

<multimodal_sync>

## 5. 멀티모달 감정 연기 TTS 동기화

텍스트가 렌더링되는 동안 발생하는 빈 시간을 활용하여 오디오를 미리 생성(Prefetch)하고 싱크를 맞춥니다.

`[IMPLEMENTED]` 보이스 프리페치 및 재생은 [GeminiTtsService.cs](../../../GRC/GRC/Services/GeminiTtsService.cs)와 연동됩니다.

1.  **실시간 대사 파싱**: 스트리밍 루프가 따옴표(`"..."`)를 만나면 이를 감지하여 대화 버퍼를 열고, 닫는 따옴표가 등장하면 즉시 백그라운드에서 `GeminiTtsService`로 오디오 번역 및 음성 합성(WAV)을 요청합니다.
2.  **재생 동기화 대기 (Sync Hold)**: UI 소비자가 문자를 그리다가 닫는 따옴표에 도달하면, 프리페치 큐에서 방금 생성 완료된 음성을 꺼내 재생합니다. 이때 `TaskCompletionSource`를 사용하여 음성 재생이 완전히 끝날 때까지 다음 지문 텍스트의 출력을 멈추어(Hold) 소리와 자막의 완벽한 템포를 맞춥니다.

</multimodal_sync>

---

<design_tokens>

## 6. 3계층 디자인 토큰 아키텍처 (3-Tier Design Token Architecture)

확장성과 테마 일관성을 확보하고 렌더링 성능을 극대화하기 위해 UI 자원을 3개 계층으로 분리하여 관리합니다.

`[IMPLEMENTED]` 토큰 및 스타일 정의는 [PrimitiveTokens.xaml](../../../GRC/GRC/Themes/PrimitiveTokens.xaml), [DarkTheme.xaml](../../../GRC/GRC/Themes/DarkTheme.xaml), [ModernStyles.xaml](../../../GRC/GRC/Themes/ModernStyles.xaml)에 구현되어 있습니다.

```mermaid
flowchart TD
    T1["Tier 1: Primitive Tokens (PrimitiveTokens.xaml)<br>- 불변 Color (Zinc 스케일 #09090B ~ #FAFAFA)<br>- 치수 CornerRadius, 폰트 패밀리"]
    T2["Tier 2: Semantic Theme Brushes (DarkTheme.xaml)<br>- BrushBackground, BrushSurface, BrushText<br>- po:Freeze='True' GPU 메모리 동결"]
    T3["Tier 3: Component Styles (ModernStyles.xaml)<br>- BaseButtonStyle, ActionButton 계열<br>- ControlTemplate 및 VisualStateManager"]
    
    T1 --> T2 --> T3
```

1. **Tier 1 (Primitive Tokens)**: 특정 테마에 종속되지 않는 절대적 색상(Zinc 팔레트 10단계), 상태 팔레트(Emerald, Amber, Rose, Cyan), UI 치수(`CornerRadiusCard`, `CornerRadiusBubble`) 및 타이포그래피 상수를 정의합니다.
2. **Tier 2 (Semantic Theme Brushes)**: 역할 기반(Background, Surface, Border, Text Primary/Muted)으로 시맨틱 SolidColorBrush를 구성합니다. 모든 브러시에는 `po:Freeze="True"` 속성을 선언하여 WPF 렌더링 파이프라인에서 불변(Frozen) 상태로 캐싱되도록 함으로써 멀티스레드 접근 안정성을 보장하고 드로우콜 오버헤드를 최소화합니다.
3. **Tier 3 (Component Styles & Controls)**: 시맨틱 브러시를 조합하여 암시적 버튼 스타일, 스크롤바, 리스트뷰 아이템 템플릿 등을 구성합니다.

</design_tokens>

---

<mvvm_dialog_service>

## 7. IDialogService 기반 순수 MVVM 분리 (Pure MVVM Decoupling)

뷰모델(ViewModel)이 구체적인 XAML 뷰(`Window`)를 직접 생성하거나 참조하는 강결합을 해제하여 테스트 가능성과 확장성을 확보했습니다.

`[IMPLEMENTED]` 다이얼로그 서비스 규격은 [IDialogService.cs](../../../GRC/GRC/Services/IDialogService.cs)에 정의되어 있습니다.

```mermaid
sequenceDiagram
    participant VM as SessionListViewModel
    participant DS as DialogService (IDialogService)
    participant UI as WPF STA Dispatcher
    participant W as NewSessionSetupWindow

    VM->>DS: ShowNewSessionSetupDialog()
    DS->>UI: RunOnUIThread()
    UI->>W: new NewSessionSetupWindow()
    DS->>W: SetOwner(MainWindow)
    W-->>DS: ShowDialog() 결과 반환
    DS-->>VM: NewSessionSetupResult (Confirmed, Preset, Title)
```

* **뷰모델 순수성 보장**: `SessionListViewModel` 및 `ChatViewModel`은 `Views.*` 네임스페이스를 일체 참조하지 않으며, 모든 팝업 및 창 전환을 `IDialogService` 추상화 인터페이스를 통해 비동기로 위임합니다.
* **STA 스레드 격리 및 소유자 바인딩**: 백그라운드 스레드에서 다이얼로그 메서드가 호출되더라도 `RunOnUIThread` 캡슐화를 통해 안전하게 UI 스레드로 전환되며, `SetOwner(MainWindow)` 가드를 통해 창의 최소화/복원 시 팝업이 뒤로 숨겨지는 Z-Order 버그를 방지합니다.

</mvvm_dialog_service>

---

<domain_service_extraction>

## 8. 도메인 서비스 추출 및 ChatViewModel 경량화

단일 책임 원칙(SRP)을 준수하고 AI 에이전트의 유지보수 컨텍스트를 최적화하기 위해, 거대해지기 쉬운 `ChatViewModel`의 부가 책임을 독립 도메인 서비스로 격리했습니다.

* **음성 합성 및 번역 캡슐화 ([GeminiTtsService.cs](../../../GRC/GRC/Services/GeminiTtsService.cs))**: 대사 스트리밍 도중 발생하는 일본어 번역 전처리, Google Cloud Vertex AI 음성 합성 호출, 취소 토큰(`CancellationToken`) 생명주기를 서비스 내부로 은닉했습니다.
* **오디오 재생 스레드 안전성 ([AudioService.cs](../../../GRC/GRC/Services/AudioService.cs))**: `MediaPlayer` 인스턴스의 STA 스레드 친화성을 보장하기 위해 생성자에서 UI `Dispatcher`를 캡처하고, 모든 재생 및 정지 명령을 디스패처로 보호했습니다.
* **타이핑 사운드 수명주기 관리 ([ChatWorkflowService.cs](../../../GRC/GRC/Services/ChatWorkflowService.cs))**: 텍스트 타이핑 효과음의 시작과 정지를 스트리밍 루프 및 `try-finally` 블록 내부에 결합하여 예외 발생 시에도 소리가 무한 반복되는 누수를 방지했습니다.
* **스탯 UI 파싱 팩토리 ([StatusPayload.cs](../../../GRC/GRC/Models/StatusPayload.cs))**: 뷰모델에 흩어져 있던 비정형 스탯 문자열 파싱 로직을 `CustomStatUI.Create` 도메인 팩토리 메서드로 이전하여 `CultureInfo.InvariantCulture`, finite 검증 등 방어적 파싱을 일원화했습니다.

</domain_service_extraction>

---

<smart_autoscroll>

## 9. 스마트 오토스크롤 (SmartAutoScroll) 무결성 알고리즘

LLM 토큰 단위 스트리밍 출력과 사용자의 과거 대화 탐색(Upward Scrolling) 간의 충돌을 수학적 판정식으로 해결했습니다.

`[IMPLEMENTED]` 스크롤 판정 로직은 [AutoScrollHelper.cs](../../../GRC/GRC/Helpers/AutoScrollHelper.cs)에 구현되어 있습니다.

* **ScrollViewer.Tag 오염 제거 및 사용자 의도 상태 머신 (`IsAutoScrollActive`)**: 과거 레거시에서 스크롤 상태 플래그를 컨트롤의 `Tag` 속성에 의존하던 방식을 완전히 폐기하고, ScrollViewer별 `IsAutoScrollActive` Attached DependencyProperty 기반의 유한 상태 머신으로 전환했습니다.
* **사용자 상향 스크롤 즉각 해제 (Zero-Latency Disengage)**:
  - 마우스 휠 상향 회전(`PreviewMouseWheel e.Delta > 0`) 또는 순수 뷰포트 상향 이동(`isPureUserScroll && e.VerticalChange < 0`) 감지 즉시 `IsAutoScrollActive = false`로 전이하여 추가 토큰 유입이나 가상화 높이 변동에 의한 하이재킹을 원천 차단합니다.
  - 비동기 디스패처 큐잉(`DispatcherPriority.Loaded`) 실행 시점에도 `if (GetIsAutoScrollActive(sv))` 지연 재검증(Lazy Guard)을 수행하여 큐 대기 시간 중 발생한 휠 조작을 보호합니다.
* **1틱 상향 스크롤 탈출 트랩 및 창 리사이징 오인 방어**:
  - 스트리밍 토큰 유입(`ExtentHeightChange > 0, VerticalChange == 0`) 시 24px 이내 상향 스크롤 유저가 바닥으로 되돌아가지 않도록 능동적 하향 조작(`e.VerticalChange > 0`) 또는 스크롤바 부재(`sv.ScrollableHeight <= 0`) 시에만 재활성화합니다.
  - 창 최대화/세로 확장(`ViewportHeightChange > 0`)으로 인한 클램핑 역방향 오프셋 변동은 사용자 상향 스크롤로 오인하지 않도록 `isPureUserScroll` 가드로 격리합니다.
* **컨텐츠 팽창/축소 조건부 판정식**:
  $$\text{referenceHeight} = \begin{cases} \text{scrollableHeight} - \Delta\text{extentHeight}, & \text{if } \Delta\text{extentHeight} > 0 \\ \text{scrollableHeight}, & \text{otherwise} \end{cases}$$
  컨텐츠가 아래로 스트리밍 팽창할 때(`extentHeightChange > 0`)는 직전 프레임의 바닥 높이를 역산하여 사용자가 바닥에 있었는지를 정확히 판정(24px 허용오차 적용)하고, 접기/삭제 등 컨텐츠가 축소될 때(`extentHeightChange <= 0`)는 현재 가용 높이와 직접 비교함으로써 사용자의 스크롤 뷰포트를 강제로 하단으로 끌고 내려가는 스크롤 하이재킹(Scroll Hijacking)을 원천 차단했습니다.

</smart_autoscroll>

