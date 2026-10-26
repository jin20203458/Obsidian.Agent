---
description: GRC(GenAI Roleplay Chat) C# WPF 데스크톱 클라이언트 트러블슈팅 런북.
related:
  - ../README.md
  - ../GRC/README.md
---
# GRC Troubleshooting

본 문서는 GRC(GenAI Roleplay Chat) 데스크톱 클라이언트(C#/.NET 8.0 WPF, CommunityToolkit.Mvvm, Google Gemini API, Vertex AI TTS, System.Threading.Channels 스트리밍 및 O(1) 파일 로거) 개발 및 운영 중 발생하는 시스템 예외, 동시성 버그, 라이브러리 충돌 및 트러블슈팅 내역을 관리하는 중앙 기술 런북입니다.

---

## 사전 주의사항 및 알려진 기술적 제약 (Known Constraints)

### 1. WPF UI 스레드 (STA) 비차단 불변식
* **현상**: 대규모 LLM 토큰 스트리밍 또는 디스크 쓰기 작업 시 UI 스레드 블로킹으로 인한 창 멈춤(Freezing) 발생 가능성.
* **대응책**: 모든 API 호출 및 I/O는 `System.Threading.Channels` 기반 백그라운드 큐로 오프로드하고, UI 렌더링은 디스패처 배치 업데이트를 통해서만 수행.

---

## 2026-10-14: [Resolved] Vertex AI 최신 주력 모델의 특정 리전 엔드포인트 404 오류 및 글로벌 엔드포인트 단일화

### 1. 현상 (Symptom)
* Google Cloud Vertex AI 연동 시 로컬 리전(예: `asia-northeast3`, `us-central1`)을 지정하여 최신 주력 모델(`gemini-3.8-flash`, `gemini-4-argon`)을 호출할 경우 `404 Not Found` 또는 리전 미지원 예외 발생.

### 2. 원인 (Root Cause)
* Google Cloud Vertex AI 정책상 최신 주력 모델(Gemini 3.x/4.x)은 전역 엔드포인트(`locations/global`)에서만 우선 서비스되며 로컬 리전에는 배포되지 않음 (단, Vertex AI TTS 음성 모델은 `us-central1` 등 특정 리전에서만 지원).

### 3. 해결책 (Resolution)
* **LLM 엔드포인트 전역 고정 및 부정 제약**: LLM 추론 시 개별 로컬 리전을 지정하지 말고 `locations/global` 단일 엔드포인트로 고정.
* **TTS 리전 분리**: `GeminiTtsService`는 Google Cloud TTS 지원 리전인 `us-central1`로 분리 고정.

---

## 2026-10-14: [Resolved] LlmJsonParser의 사전 잡담 괄호 오인식, 문자열형 숫자/Enum 역직렬화 실패 및 스트리밍 절단 복구

### 1. 현상 (Symptom)
* LLM이 JSON 페이로드 앞뒤에 설명용 템플릿 괄호(`Format {template}...`)를 출력하거나 따옴표 내부에 괄호가 포함될 때 단순 `rawText.IndexOf('{')` 방식이 오작동하여 역직렬화 실패(`null` 반환).
* 큰따옴표로 감싸진 숫자(`"count": "42"`)나 DTO의 `enum` 문자열 역직렬화 실패 및 토큰 제한으로 스트리밍 도중 절단(Truncated)된 JSON 파싱 불가.

### 2. 원인 (Root Cause)
* 괄호 계수기가 문자열 리터럴/이스케이프(`\"`) 상태를 추적하지 않고 단순 `IndexOf('{')`로 개시 지점을 탐색했으며, `JsonSerializerOptions` 관용 플래그 부재 및 미완성 괄호 복구 엔진이 결여됨.

### 3. 해결책 (Resolution)
* **`LenientOptions` 정적 캐싱**: `AllowReadingFromString`, `AllowTrailingCommas`, `ReadCommentHandling = Skip`, `JsonStringEnumConverter`를 사전 구성하여 정적 캐싱하고 struct/record 역직렬화 지원.
* **Index 0 기준 문자열 리터럴 추적 및 다중 후보 순차 검증**: 마크다운 ```` ```json ```` 블록을 우선 파싱하고, 따옴표/이스케이프를 추적하며 후보군 스키마를 순차 검증.
* **`RepairTruncatedJson` 엔진 탑재 및 체크포인트 롤백**: 절단된 응답 감지 시 열린 따옴표를 닫고 dangling 키/콜론/쉼표를 정제한 후 스택에 보존된 여는 괄호의 짝(`]`, `}`)을 LIFO 역순으로 닫아 구조적 유효성 복원. 미완성 토큰 발생 시 `SafeRollbackLength` 기반 체크포인트 롤백 적용.
* **핵심 부정 제약 (Never use X)**: **절대 LLM 스트리밍 응답 파싱 시 단순 정규식이나 `rawText.IndexOf('{')`, `IndexOf("```")`에 의존하지 말 것.** 반드시 `LlmJsonParser` 전용 엔진을 사용할 것.

---

## 2026-10-14: [Resolved] 백그라운드 스레드 및 단위 테스트 환경 내 WPF STA 스레드 위반 방지를 위한 AudioService / DialogService Dispatcher 사전 캡처

### 1. 현상 (Symptom)
* `AudioService`의 `MediaPlayer` 또는 `DialogService`의 윈도우 생성이 비동기 백그라운드 작업(`Task.Run`)에서 호출될 때 `System.InvalidOperationException` (STA 스레드 선점 위반) 발생 또는 헤드리스 단위 테스트(xUnit) 환경에서 UI 디스패처 부재로 데드락 발생.

### 2. 원인 (Root Cause)
* WPF 컨트롤 및 `MediaPlayer`는 생성된 STA 스레드 친화성을 지니며, 런타임에 임의 스레드에서 `Application.Current.Dispatcher`를 직접 참조할 경우 null 참조 또는 스레드 액세스 위반 발생.

### 3. 해결책 (Resolution)
* **생성자 내 Dispatcher 사전 캡처**:
  `_dispatcher = Application.Current?.Dispatcher ?? Dispatcher.CurrentDispatcher;`
* **`RunOnUIThread` 스레드 스위칭 캡슐화**:
  `_dispatcher.CheckAccess()`를 검사하여 UI 스레드인 경우 즉시 동기 실행하고, 백그라운드 스레드인 경우에만 `_dispatcher.Invoke(action)` 또는 `_dispatcher.InvokeAsync(action)`로 전환.

---

## 2026-10-14: [Resolved] xUnit 테스트 러너 환경에서 WPF Application 인스턴스 중복 생성(AppDomain Singleton Trap) 해결

### 1. 현상 (Symptom)
* WPF 리소스 딕셔너리나 뷰를 검증하는 단위 테스트를 xUnit 병렬 러너로 실행할 때 `System.InvalidOperationException: 동일한 AppDomain에서 둘 이상의 Application 개체를 만들 수 없습니다.` 오류 발생.

### 2. 원인 (Root Cause)
* WPF `System.Windows.Application`은 단일 `AppDomain` 내에서 단 하나의 인스턴스만 허용하는 싱글톤 불변식을 지님.

### 3. 해결책 (Resolution)
* **`EnsureApplication` 정적 가드 패턴 도입**:
  ```csharp
  private static void EnsureApplication()
  {
      if (Application.Current != null) return;
      try
      {
          _ = new Application();
      }
      catch (InvalidOperationException)
      {
          // 타 테스트 스레드에 의한 경쟁 인스턴스 생성 예외 안전 흡수
      }
  }
  ```

---

## 2026-10-14: [Resolved] Application.LoadComponent 사용 시 절대 Pack URI 지정으로 인한 ArgumentException 및 상대 URI 표준화

### 1. 현상 (Symptom)
* C# 코드-비하인드 또는 단위 테스트에서 `Application.LoadComponent(new Uri("pack://application:,,,/...", UriKind.Absolute))` 호출 시 `System.ArgumentException: 절대 URI를 사용할 수 없습니다.` 발생.

### 2. 원인 (Root Cause)
* WPF `Application.LoadComponent` API는 BAML 리소스 로딩 규격상 반드시 어셈블리 상대 URI(`UriKind.Relative`)를 요구함 (XAML 마크업 내 `ResourceDictionary.Source`와 상이).

### 3. 해결책 (Resolution)
* **상대 URI 표준화 및 부정 제약**: C# 코드에서 `Application.LoadComponent` 호출 시 절대 Pack URI(`pack://...`)를 절대 전달하지 말고 상대 경로(`UriKind.Relative`)로 강제할 것:
  ```csharp
  var uri = new Uri("/GRC;component/Themes/PrimitiveTokens.xaml", UriKind.Relative);
  var dict = (ResourceDictionary)Application.LoadComponent(uri);
  ```
* **PackUriHelper 스킴 사전 등록**: 헤드리스 테스트 환경에서는 `_ = PackUriHelper.UriSchemePack;`를 사전 호출하여 pack 스킴을 파서에 등록.

---

## 2026-10-14: [Resolved] LLM 스트리밍 및 가상화 스크롤 시 상단 스크롤 하이재킹 및 1틱 탈출 불가 트랩 해결

### 1. 현상 (Symptom)
* LLM 텍스트 실시간 스트리밍 또는 접기/삭제로 스크롤 높이(`ExtentHeight`)가 줄어들 때(`extentHeightChange < 0`), 과거 대화를 읽고 있는 사용자의 뷰포트가 최하단으로 강제 스크롤되는 현상 발생.
* AI 스트리밍 도중 마우스 휠을 위로 회전해도 화면이 올라가지 못하고 즉시 최하단으로 강제 롤백되는 1틱 탈출 불가 트랩 발생.

### 2. 원인 (Root Cause)
1. **정적 기하학 왜곡**: 하단 판정식에서 높이 변화량에 `Math.Abs`를 차감할 경우 컨텐츠 축소 시 기준 높이가 왜곡되어 유저가 최하단에 있지 않음에도 바닥인 것으로 오판단 (`ScrollViewer.Tag` 속성 불일치 포함).
2. **무상태 판정 한계 (1틱 트랩)**: 24px 허용오차 내에서 16ms마다 토큰이 유입되면 `extentHeightChange`로 인해 휠 1틱(15px) 조작 시 허용오차를 탈출하지 못해 순수 수학식이 계속 `true`를 반환.
3. **가상화 패널 점프 및 디스패처 지연**: `VirtualizingStackPanel` 아이템 실체화로 높이가 수백 px 급증하고, 직전 프레임의 `ScrollToEnd()` 비동기 태스크가 휠 조작 직후 뒤늦게 실행되어 화면을 강제 텔레포트시킴.

### 3. 해결책 (Resolution)
* **순수 수학적 위치 판정 함수 교정 (`AutoScrollHelper.cs`)**:
  ```csharp
  public static bool CalculateIsAtBottom(double verticalOffset, double scrollableHeight, double extentHeightChange, double tolerance = 24.0)
  {
      if (scrollableHeight <= 0) return true;
      double referenceHeight = extentHeightChange > 0 ? scrollableHeight - extentHeightChange : scrollableHeight;
      return verticalOffset >= (referenceHeight - tolerance);
  }
  ```
* **사용자 의도 기반 상태 머신 (`IsAutoScrollActive` Attached DP)**:
  - `PreviewMouseWheel`: 휠 상향 회전(`e.Delta > 0`) 즉시 `SetIsAutoScrollActive(sv, false)`.
  - `ScrollChanged`: 순수 사용자 조작(`ExtentHeightChange == 0 && ViewportHeightChange == 0`)의 상향 이동(`VerticalChange < 0`) 감지 시 비활성화.
* **디스패처 지연 실행 가드 (Lazy Guard)**: `ScrollToEnd()` 비동기 실행 시점에 `if (GetIsAutoScrollActive(sv))`를 재확인하여 큐 대기 중 발생한 휠 조작 보호.
* **재활성화 조건**: 능동적 하향 조작 후 바닥 도달, 스크롤바 부재, 또는 신규 메시지 전송 시에만 `true`로 복구.
* **핵심 부정 제약 (Never use X)**: **절대 무상태 수학식(`CalculateIsAtBottom`)만 단독으로 사용하지 말 것.** 16ms 단위 토큰 스트리밍 환경에서는 휠 1틱(15px) 조작 시 24px 허용오차를 즉시 벗어날 수 없어 바닥으로 재강제 스크롤되는 1틱 감금 트랩이 발생하므로, 반드시 `IsAutoScrollActive` Attached DP 상태 머신 및 디스패처 지연 가드를 결합하여 구성할 것.

---

## 2026-10-15: [Resolved] TensorPrimitives.CosineSimilarity의 영벡터(Zero-Vector) 연산 시 NaN 발생 및 방어 가드

### 1. 현상 (Symptom)
* `.NET 8.0 TensorPrimitives.CosineSimilarity` 연산 시 영벡터(Zero-Vector) 입력에 대해 `float.NaN`이 반환되어 상위 유사도 랭킹 및 필터링 비교식이 침묵 파손되는 결함.

### 2. 원인 (Root Cause)
* 코사인 유사도 연산 중 영벡터의 크기(L2 Norm = 0)로 인한 0 나누기(Division by Zero) 발생.

### 3. 해결책 및 불변식 (Resolution & Invariants)
* **`VectorMathHelper` 삼항 연산 가드**: `float.IsNaN(similarity) ? 0.0f : similarity` 클램핑 적용.
* **핵심 부정 제약 (Never use X)**: **절대 `TensorPrimitives.CosineSimilarity` 반환값을 `float.IsNaN` 검증 없이 직접 비교식에 사용하지 말 것.**

---

## 2026-10-15: [Resolved] Gemini API Part 빈 문자열 입력 시 HTTP 400 Bad Request 방어 및 Auto-Play 안전 Fallback

### 1. 현상 (Symptom)
* `[Auto-Play]` 실행이나 빈 텍스트 전송 시 Gemini API가 `HTTP 400 Bad Request (contents.parts.text must not be empty)`를 반환하며 스트리밍 세션 크래시 발생.

### 2. 원인 (Root Cause)
* Gemini API 규격상 `contents[].parts[].text`는 null 및 빈 문자열을 불허하고 1자 이상의 유효 텍스트를 요구함.

### 3. 해결책 및 불변식 (Resolution & Invariants)
* **다단계 안전 Fallback (`CognitivePromptAssembler`)**: `string.IsNullOrWhiteSpace` 감지 시 `PlotCatalyst` 지침을 우선 채택하고, 부재 시 `"(서사를 계속 진행하십시오.)"` 대체 텍스트를 강제 주입.
* **핵심 부정 제약 (Never use X)**: **절대 검증되지 않은 빈 유저 입력을 검사 없이 `new Part(text)`에 직접 전달하지 말 것.**

---

## 2026-10-17: [Resolved] 구버전 세션 로어북 백그라운드 임베딩 백필 중 세션 전환/수정 시 데이터 손실 및 파일 오염 경합 조건 해결

### 1. 현상 (Symptom)
- 구버전 세션 로드 시 백그라운드 비동기 임베딩 생성 도중(1~2초), 사용자가 로어북을 신규 등록하거나 다른 세션으로 전환할 경우:
  - 신규 등록 로어북이 백그라운드 완료 시점에 구버전 목록으로 덮어써져 영구 유실됨.
  - 다른 세션을 열었을 때 이전 세션의 로어북 데이터가 현재 세션 파일에 덮어써져 저장되는 파일 오염 발생.

### 2. 원인 (Root Cause)
- `ChatViewModel.cs`의 `Task.Run` 내부에서 시작 시점의 스냅샷 없이 인스턴스 멤버(`CurrentPreset`, `_currentFileName`)를 직접 참조하고, 완료 후 `CurrentPreset with { Lorebooks = updatedLores }`로 무조건 덮어씀.

### 3. 해결책 및 불변식 (Resolution & Invariants)
- **세션 수명주기 앵커링**: `_isSessionActive` 플래그 및 `string targetFileName = _currentFileName;` 스냅샷을 캡처하고, `GoBack()` 호출 시 `_isSessionActive = false;`로 전환. 비동기 완료 후 `!_isSessionActive || _currentFileName != targetFileName`일 경우 즉시 작업 폐기.
- **비파괴적 선택적 임베딩 병합 (Selective Merge)**: 전체 리스트를 덮어쓰지 않고, 완료된 768차원 임베딩을 `(Name, Content)` 키 딕셔너리로 구축하여 임베딩이 결여된 항목에만 선택적으로 주입. 백그라운드 실행 중 유저가 수정한 로어북 무손실 보존.
- **WPF STA 스레드 안전 디스패칭**: `Application.Current?.Dispatcher` 가드를 통해 UI 스레드 위반 없이 안전하게 상태를 반영하고 세션 파일과 동기화.

---

## 2026-10-17: [Resolved] dotnet test 백그라운드 실행 시 솔루션 단위 평가로 인한 MSBuild/VSTest I/O 파이프 블로킹 해결 및 테스트 프로젝트 직접 타겟팅 규격 확립

### 1. 현상 (Symptom)
- 백그라운드 태스크로 `dotnet test GRC.sln --no-build` 실행 시 테스트 실행 자체는 완료되었으나 프로세스가 종료되지 않고 셸이 무한 대기(RUNNING/Deadlock)에 머무는 결함 발생.

### 2. 원인 (Root Cause)
- `dotnet test` CLI 규격상 `-nodeReuse:false`가 VSTest(`vstest.console.exe`) 및 솔루션 그래프를 평가하는 MSBuild 백그라운드 작업자 노드에 완벽히 상속되지 않음.
- 자식 작업자 노드가 백그라운드 셸의 표준 출력(stdout) 스트림 핸들을 점유한 채 반환하지 않아 프로세스 종료 신호(EOF)가 셸에 전달되지 않음.

### 3. 해결책 및 불변식 (Resolution & Invariants)
- **테스트 프로젝트 직접 타겟팅 규격 확립**:
  솔루션(`.sln`) 대신 테스트 프로젝트 파일(`.csproj`)을 명시적으로 직접 지정하고 일반 상세도(`-v n`)를 적용하여 불필요한 솔루션 중계 노드 생성을 차단:
  ```powershell
  dotnet test tests/GRC.Tests/GRC.Tests.csproj --filter "Category!=Integration" --no-build -v n
  ```
- **핵심 부정 제약 (Never use X)**: **백그라운드 터미널에서 `dotnet test` 실행 시 절대 솔루션 파일(`.sln`)을 타겟으로 지정하지 말 것.** 반드시 해당 테스트 프로젝트 파일(`tests/GRC.Tests/GRC.Tests.csproj`)을 직접 지정하여 VSTest/MSBuild I/O 파이프 데드락을 방지할 것.

---

## 2026-10-17: [Resolved] 메타 지시문(MetaDirective) 합성에 따른 768차원 임베딩 쿼리 벡터 오염 해결 및 EventLedgerExtractor 정적 SystemInstruction KV Cache 분리

### 1. 현상 (Symptom)
- 시스템 이벤트나 챕터 전환, 자동 이어쓰기 시 `[시스템 개입: ...]` 메타 텍스트가 `userMessage.Text`에 직접 병합되어 `GetEmbeddingAsync`로 전달됨에 따라, 순수 유저 의도 검색이어야 할 768차원 쿼리 임베딩 벡터가 메타 단어로 희석(Vector Dilution)되어 로어북/과거 기억 RAG 검색 정확도가 저하됨.
- 4턴 주기 백그라운드 사건 팩트 추출기(`EventLedgerExtractor`)가 `SystemInstruction: null`로 호출되어 약 400토큰의 정적 규칙과 스키마가 매번 캐시 미스(Cache Miss)를 일으켜 API 비용 및 지연 누적.

### 2. 원인 (Root Cause)
1. `MemoryManagerService.cs`에서 `userMessage.Text`와 `metaDirective`를 단순 문자열 결합하여 임베딩 및 프롬프트 조립에 공용 사용.
2. `EventLedgerExtractor.cs`가 단일 `user` 메시지에 모든 지침과 이력을 연결하여 `SystemInstruction` 캐싱 인프라 미활용.

### 3. 해결책 및 불변식 (Resolution & Invariants)
1. **메타 지시문 독립 데이터 파이프라인 및 순수 유저 임베딩 복원**:
   - `CognitiveContext`에 `MetaDirective` 프로퍼티를 신설하여 메타 지시문을 독립 전달.
   - `MemoryManagerService.cs`에서 순수 `userMessage.Text`로만 768차원 임베딩 및 `loreScanQuery`를 생성하여 쿼리 벡터 희석 원천 차단.
   - `CognitivePromptAssembler.cs`에서 메타 지시문이 존재할 때만 `<current_action>` 앞쪽에 `<meta_directive>` 독립 상위 태그로 조건부 조립.
2. **EventLedgerExtractor SystemInstruction 분리 및 샘플링 최적화**:
   - 무인격 기능 명세, 5대 인과 규칙 및 TypeScript 인터페이스를 `SystemInstruction`으로 물리 분리하여 4턴 주기 KV Cache 100% 적중 달성.
   - `GenerationConfig.Temperature: null`로 교정하여 구글 Gemini 기본 최적화 샘플링 엔진에 위임.
3. **핵심 부정 제약 (Never use X)**: **절대 시스템 메타 지시문을 유저 발화 원문에 병합하여 임베딩을 생성하지 말 것(Vector Dilution 방지).**

---

## 2026-10-17: [Resolved] 단위 테스트 환경에서 종료된 임시 STA 스레드의 Application.Current Dispatcher Invoke 호출로 인한 무한 대기(Deadlock) 해결

### 1. 현상 (Symptom)
- `dotnet test` 전체 테스트 스위트 실행 시 특정 테스트 통과 후 `DialogServiceTests`에서 프로세스가 멈추고 무한 대기(Hang/Deadlock)에 빠지는 결함 발생 (단독 실행 시에는 정상 통과).

### 2. 원인 (Root Cause)
- 앞서 실행된 XAML 리소스 검증 테스트가 임시 STA 스레드에서 `new Application()`을 호출하고 종료됨에 따라, 전역 `Application.Current.Dispatcher`가 이미 종료된 스레드(`Thread.IsAlive == false`)에 바인딩된 상태로 잔존함.
- 뒤이어 xUnit 작업자 스레드에서 `DialogService.RunOnUIThread`가 호출되었을 때, `Application.Current != null` 및 `CheckAccess() == false`로 인해 `dispatcher.Invoke(action)`를 호출했으나 대상 스레드가 사망하여 메시지 펌프가 돌지 못해 영원히 블로킹됨.

### 3. 해결책 및 불변식 (Resolution & Invariants)
- **스레드 생존성 및 종료 상태 가드 보강 (`IDialogService.cs`)**:
  - `dispatcher == null || dispatcher.CheckAccess() || !dispatcher.Thread.IsAlive || dispatcher.HasShutdownStarted || dispatcher.HasShutdownFinished` 조건을 추가하여, UI 스레드가 사망했거나 셧다운된 테스트/헤드리스 환경에서는 백그라운드 스레드에서 즉시 인라인 실행하도록 방어.
  - `RunOnUIThread`에 500ms 비동기 타임아웃(`dispatcher.InvokeAsync(action).Task.Wait(500)`) 및 `try-catch` 가드를 적용하여 실패 시에도 즉시 인라인 폴백 보장.
- **핵심 부정 제약 (Never use X)**: **절대 `Application.Current.Dispatcher` 참조 시 스레드 생존성(`Thread.IsAlive`)과 셧다운 여부 검증 없이 `dispatcher.Invoke`를 직접 호출하지 말 것.** (상단 2026-10-14 Dispatcher 캡처 엔트리 참조)

---

## 2026-10-17: [Resolved] Gemini Thinking 모델 MaxOutputTokens 축소 시 JSON 절단 방어 및 TTS 프롬프트 순수 구어체 자연어 보존

### 1. 현상 및 잠재적 장애 모드 (Symptom & Failure Modes)
- **Thinking 모델의 MaxOutputTokens 축소 시 JSON 파손**: Gemini Thinking 아키텍처에서 `maxOutputTokens`는 생각 토큰(Thought Tokens)과 가시적 응답 토큰(Visible Output Tokens)의 통합 총합이므로, 이를 1,024~2,048로 강제 축소할 경우 생각 토큰에 예산이 잠식되어 JSON 페이로드가 중간에 잘려 파싱 에러(`LlmJsonParser` 예외) 발생.
- **TTS 음성 변조 및 태그 음독 결함**: 음성 합성 모델(`gemini-3.1-flash-tts-preview`) 및 사전 번역기에 XML 태그나 불릿포인트를 주입하면 음성의 운율(Prosody) 왜곡, 기계음 발생, 태그 음독 결함 발생.

### 2. 원인 (Root Cause)
1. 백그라운드 구조화 출력 작업에 Thinking 예산과 가시적 토큰의 통합 메커니즘을 고려하지 않고 `maxOutputTokens`를 과도하게 제한함.
2. TTS 사전 번역 및 합성 프롬프트에 마크다운/XML 구조화 태그를 주입하여 오디오 생성 모델의 자연어 운율 인코딩을 왜곡함.

### 3. 해결책 및 불변식 (Resolution & Invariants)
1. **Thinking 토큰 상한 보존 및 저비용 추론 가드**:
   - 지연 시간 및 연산 비용 제어는 오직 `ThinkingConfig(ThinkingLevel.low)`로만 통제하며, `MaxOutputTokens`는 8,192의 넉넉한 상한을 유지하여 생각 토큰 소모로 인한 JSON 페이로드 잘림을 원천 방어.
2. **TTS 프롬프트 100% 순수 자연어 구어체 보존**:
   - 사전 번역 프롬프트와 음성 합성 프롬프트(`dynamicPrompt`)는 일체의 태그/기호 없이 **100% 순수 자연어 구어체 지시문**으로만 유지하여 운율 왜곡 및 태그 음독 원천 차단.
3. **핵심 부정 제약 (Never use X)**: **절대 Thinking이 활성화된 모델의 `MaxOutputTokens`를 1,024~2,048 수준으로 임의 축소하지 말 것(JSON 절단 트랩 방어).**

---

## 2026-10-18: [Resolved] text-embedding-004(AI Studio EOL) 및 002(Vertex 레거시) 대응 차세대 통합 gemini-embedding-001(서울, 768 MRL) 전환 및 RAG 임계치(0.75f, 0.78f) 재보정

### 1. 현상 및 배경 (Symptom & Background)
- Google AI Studio의 구형 `text-embedding-004` 서비스 종료(HTTP 404) 및 Vertex AI `text-multilingual-embedding-002` 레거시 지정.
- `gemini-embedding-001` 전환 시 한국어/다국어 표현력 향상으로 코사인 유사도 점수 분포가 전반적으로 +0.07~+0.09 상향 이동(Scale Shift)하여 기존 임계치(0.68f, 0.71f) 유지 시 오탐(False-Positive) 급증 위험 발생.
- 로어북 엔트리의 키워드가 비어 있을 때 캐릭터 이름(`Name`) 직접 언급 시 0ms 즉각 인출이 누락되는 위험 존재.

### 2. 해결책 및 아키텍처 SSOT 규격 (Resolution & Invariants)
1. **차세대 통합 임베딩 모델 일원화 (`GeminiApiService.cs`)**:
   - Vertex AI 및 AI Studio 양쪽 모두 `gemini-embedding-001` 단일 모델명 및 `outputDimensionality: 768` (MRL)로 통일.
   - 서울(`asia-northeast3`) 리전 직결을 통해 레이턴시 단축 및 768차원 float 벡터 연산 효율 보존.
2. **RAG 임계치 전면 재보정**:
   - `GatedRAGRetriever.cs` (사건 불릿 렛저 RAG): **`0.78f`**
   - `HybridLorebookRetriever.cs` (로어북 RAG): **`0.75f`**
3. **하이브리드 로어북 이름(Name) 1차 스파스 매칭 보강**:
   - `entry.Keywords` 누락 시에도 `entry.Name`에 대한 단어 경계(`\b`) 매칭을 기본 수행하여 0ms 즉각 인출 보장 및 `DistinctBy` 중복 제거.
4. **아키텍처 불변식**:
   - **임베딩 모델은 반드시 `gemini-embedding-001` (서울 asia-northeast3 리전) 및 `outputDimensionality: 768`을 사용할 것.**
   - **로어북 시맨틱 임계치는 `0.75f`, 사건 렛저 RAG 임계치는 `0.78f`를 유지할 것.** (상단 2026-10-15 리전 분리 규격 참조)

---

## 2026-10-18: [Resolved] 동적 상태창 갱신 파이프라인의 Nullable DTO 빈 배열 소거 및 비동기 체이닝 확립

### 1. 현상 (Symptom)
- 인벤토리 마지막 아이템 소비 시 LLM이 `"items": []`를 반환해도 `StatusPayload` 기본값(`new()`)과 `Count > 0` 조건으로 인해 빈 배열 덮어쓰기가 무시되어 아이템이 영구 잔존하는 결함 발생.
- `ChatWorkflowService` 내 `_ = Task.Run(...)`이 완료를 보장하지 않아 유저가 연속 턴 입력 시 이전 턴 상태창 반영 전에 다음 턴 프롬프트가 조립되는 비동기 타이밍 누수 발생.

### 2. 원인 (Root Cause)
1. `StatusPayload` DTO 프로퍼티가 `new()`로 강제 초기화되어 부분 갱신과 전체 소거(빈 배열)를 구분할 수 없었음.
2. `onStatusUpdated` 콜백이 동기 `Action`으로 선언되어 UI 디스패치 및 디스크 저장을 순차 비동기 체이닝하지 못함.

### 3. 해결책 및 불변식 (Resolution & Invariants)
1. **Nullable StatusPayload 컬렉션 전환 (`StatusPayload.cs`, `MemoryManagerService.cs`)**:
   - `CustomStats`, `Chars`, `Items`, `Places`를 nullable(`? = null`)로 전환하고, `payload.Items != null` 조건으로 빈 리스트(`[]`) 수신 시 인벤토리를 정상 소거.
2. **완전 비동기 체이닝 및 세이프가드 확립 (`IChatWorkflowService.cs`, `ChatWorkflowService.cs`)**:
   - `onStatusUpdated`를 `Func<StatusPayload, Task>?`로 변경하고, `statusTask` 내부에서 `await onStatusUpdated(payload)`를 통해 네트워크 통신~UI 디스패치~디스크 영구 저장까지 단일 비동기 파이프라인으로 안전 대기.
3. **핵심 부정 제약 (Never use X)**: **동적 상태창 DTO의 컬렉션 프로퍼티를 `new()`로 사전 초기화하지 말 것.** 미제공(null, 부분 갱신)과 빈 컬렉션(`[]`, 전체 소거)의 의미론적 차이를 보존할 것.

---

## 2026-10-18: [Resolved] ChatViewModel 백그라운드 상태창 콜백의 세션 이탈 오염 방어 및 백그라운드 I/O 스레드 분리

### 1. 현상 (Symptom)
- 상태창 비동기 API가 실행되는 도중 유저가 세션을 나가거나(`GoBack`) 다른 세션 파일로 전환했을 때 세션 활성화 가드가 없어 이전 세션의 상태창 결과가 새 세션 메모리와 디스크에 덮어써지는 파일/상태 오염 발생.
- `onStatusUpdated` 콜백 내부 전체가 `Application.Current.Dispatcher.InvokeAsync`로 묶여 있어, 백그라운드에서 돌아야 할 `SaveSessionAsync` 파일 I/O까지 UI 스레드에서 경합하고 헤드리스 환경에서 데이터 저장이 스킵됨.

### 2. 원인 (Root Cause)
1. `ChatViewModel.SendMessageAsync` 내 `onStatusUpdated` 콜백에 세션 활성 플래그 및 파일명 일치 가드가 누락됨.
2. UI 갱신과 순수 데이터/디스크 저장(`SaveSessionAsync`)의 실행 스레드 경계가 분리되지 않고 통째로 UI Dispatcher에 바인딩됨.

### 3. 해결책 및 불변식 (Resolution & Invariants)
1. **세션 식별자 스냅샷 및 활성 가드 보강 (`ChatViewModel.cs`)**:
   - `SendMessageAsync` 진입 시 `string targetFileName = _currentFileName;`을 캡처하고, `onStatusUpdated` 진입 즉시 `if (!_isSessionActive || _currentFileName != targetFileName) return;` 방어로 세션 이탈/전환 시 즉각 폐기.
2. **디스패처 의존성 제거 및 백그라운드 I/O 분리**:
   - `onStatusUpdated` 외부의 Dispatcher 래핑을 걷어내고, 스레드 안전한 `UpdateContextStatus`와 비동기 디스크 I/O `SaveSessionAsync`는 백그라운드 태스크에서 즉시 실행.
   - UI 컬렉션 조작만 `UpdateStatusUI` 내부의 `_dialogService.RunOnUIThread`에 위임하여 완벽한 스레드 분리 달성.
3. **핵심 부정 제약 (Never use X)**: **절대 비동기 디스크 I/O(`SaveSessionAsync`)를 UI Dispatcher 스레드 안으로 감싸지 말 것.** 비동기 콜백 처리 시 반드시 세션 식별자 스냅샷(`targetFileName`)과 활성 상태(`_isSessionActive`)를 선행 검증할 것. (상단 2026-10-17 세션 전환 경합 조건 참조)

---

> [!NOTE]
> **[정제 완료 기준선]** 2026-10-18 이전 상위 항목은 정제 완료됨. 신규 인시던트는 이 아래에 추가됩니다.

---

## 2026-10-18: [Resolved] GRC 전역 프롬프트 아키텍처 및 문자열 리터럴 전면 표준화 (PICCO, C# 11 Raw String, 2계층 계약, 토큰 바운딩)

### 1. 현상 (Symptom)
- 구세대 C# 관용구(`@""`, `"..." +`)의 혼용으로 인해 C# 소스코드 들여쓰기 공백이 LLM 프롬프트에 그대로 유입되어 불필요한 토큰 낭비 및 어텐션 오염 발생.
- 기능적 변환 태스크(`ReplySuggestionService`, `GeminiTtsService`)에 불필요한 2인칭 인격화("You are...", "전문가...")와 대괄호 비정형 마크다운(`[상황: ...] 출력:`)이 잔존하여 출력 일관성 저하.
- `InjectInitialScenario` 호출 경로에서 `ChatViewModel`이 `[초기 상황]\n` 헤더를 붙여 전달하여 메모리 내부에 `<initial_scenario>\n[초기 상황]\n...`의 중복 헤더 결함 발생.
- 단순 변환/추천 태스크(선택지 3개 추출, 번역 등)에 `MaxOutputTokens: 8192`가 과잉 할당되어 비정상 루프 및 통신 지연 리스크 잔존.

### 2. 원인 (Root Cause)
- 프롬프트 엔지니어링 표준 수립 이전 구현된 레거시 프롬프트들이 C# 버전 상승과 기능 추가 과정에서 세대별 파편화(Legacy Drift)를 겪음.

### 3. 해결책 및 불변식 (Resolution & Invariants)
1. **C# 11 원시 문자열 리터럴(`"""` / `$$"""`) 100% 전면 일원화**:
   - 코드 들여쓰기 공백 컴파일 타임 자동 스트리핑(Stripping)으로 공백 토큰 누출을 원천 차단.
2. **PICCO 구조화 및 무인격 기능 명세 확립**:
   - 결정론적 태스크는 `Domain: ...`으로 무인격화하고, 비정형 마크다운 예시를 `<input>`, `<output>` 표준 XML 태그로 개편.
3. **2계층 스키마 계약 및 토큰 상한 현실화**:
   - `LorebookService`, `ReplySuggestionService`, `SessionArchitectService`에 `responseSchema`를 등록하여 디코딩 레벨에서 100% 문법 무결성 보장.
   - 단발성 추천(1024), 번역(1024), 로어북 추출(2048)로 `MaxOutputTokens`를 바운딩하여 불필요한 지연과 비용 차단.
4. **초기 시나리오 주입 멱등성 보장**:
   - `ChatViewModel`에서 순수 시나리오 텍스트를 전달하고 `InjectInitialScenario`에서 `Trim()` 및 `IsNullOrWhiteSpace` 가드를 적용하여 중복 헤더 오염 차단.
5. **핵심 부정 제약 (Never use X)**: **절대 프롬프트 문자열에 `@""` 축어적 리터럴을 사용하지 말 것.** C# 11 `"""` 및 `$$"""` 원시 문자열 리터럴만을 사용할 것.

---

## 2026-10-19: [Resolved] AI 답변 추천(ReplySuggestionService)의 대화 슬라이싱 하드코딩 제거 및 미압축 단기 기억 전수 연동

### 1. 현상 (Symptom)
- `ChatViewModel`에서 AI 답변 추천 생성 시 `ChatHistory.TakeLast(9)`로 최근 4~5턴만 고정 슬라이싱하여 전달함에 따라, 4턴 단위 중기 기억(`Plot`) 압축 대기 중이던 6~8턴 전의 미압축 대화가 `Plot`과 슬라이싱 목록 양쪽에서 모두 누락되는 과도기 기억 사각지대(Transitional Gap) 발생.
- `ReplySuggestionService` 프롬프트 본문에서 `<current_snapshot>` 태그 내부에 `ChapterContext.ToPromptString()`의 `<current_snapshot>`이 또다시 중첩되는 XML 태그 충돌 발생.

### 2. 원인 (Root Cause)
- 중기 기억 압축 임계치(18개 메시지)와 답변 추천의 고정 슬라이싱(`TakeLast(9)`) 간의 라이프사이클 불일치.

### 3. 해결책 및 불변식 (Resolution & Invariants)
1. **단기 기억 전수 스냅샷 인터페이스 확장 (`IMemoryManagerService.cs`, `MemoryManagerService.cs`)**:
   - `IReadOnlyList<ChatMessage> GetCurrentShortTermMemory()`를 제공하여 스레드 안전하게 현재 `_shortTermMemory` 전수를 캡처.
2. **UI 하드코딩 슬라이싱 제거 (`ChatViewModel.cs`)**:
   - `ChatHistory.TakeLast(9)` 및 `TakeLast(10)`을 전면 제거하고, `_memoryService.GetCurrentShortTermMemory()`를 전달하여 `Plot`(이미 압축된 과거) + `_shortTermMemory`(아직 압축되지 않은 최근 대화 전부)의 무손실 톱니바퀴 결합 확립.
3. **태그 계층 충돌 해소 (`ReplySuggestionService.cs`)**:
   - 중복 `<current_snapshot>` 태그를 `<session_context>`로 변경하여 의미론적 XML 트리 무결성 확보.
4. **핵심 부정 제약 (Never use X)**: **절대 단기 기억 대화를 UI 컬렉션(`ChatHistory`)에서 임의의 숫자(`TakeLast`)로 잘라내지 말 것.** 반드시 `_memoryService.GetCurrentShortTermMemory()`를 통해 메모리 라이프사이클과 직접 동기화할 것.

---

## 2026-10-23: [Resolved] SessionArchitect 2.0 전면 개편 (2축 서사 톤 설계, 로어북 마이크로 배치 병렬 생성, C# 슬롯 정제기 및 Turn 0 시딩)

### 1. 현상 (Symptom)
- 기존 `SessionArchitectService`가 전 단계를 `ModelTier.Flash` 및 짧은 요약 제약(`1~2줄 요약`, `4문단 이내`)으로 실행하여 세계관과 캐릭터 설정의 깊이가 얕고 평면적으로 생성됨.
- 3단계(`LorebookGen`)에서 모든 로어북 항목을 단일 JSON 배열 호출 1회로 일괄 생성함에 따라 후반부 항목이 1~2문장으로 축약되는 출력 압축(Lazy Compression) 현상 발생.
- 5단계(`ScenarioGen`)에서 로어북 본문(`Content`)을 누락하고 이름(`Name`)만 전달하여 오프닝 대사와 로어북 말투가 불일치하며, 세션 생성 시 `ChapterContext`의 `Chars`, `Places`, `Items`가 빈 컬렉션(`new()`)으로 방치되어 Turn 1에 `BiTemporalEntityProjector`와 `HybridLorebookRetriever`의 인물 기반 인출이 작동하지 않음.
- 6단계(`PromptGen`)에서 메인 서사용 `SystemPrompt` 하단에 스탯/상태창 업데이트 가이드를 포함하도록 지시하여 `AI_Prompt_Engineering_Guidelines.md` 원칙 2(Zero-Numeric)를 위반하고, 자동 모드(`IsAutoMode`) 실행 중에도 채팅창에 세계관/로어북 심층 비밀 전문을 노출하여 플레이 시작 전 스포일러를 유발함.

### 2. 원인 (Root Cause)
- 세션당 1회성으로 실행되는 초기 설계 파이프라인에 하위 모델(`Flash`)과 단일 패스(Monolithic Single-Pass) 생성 방식을 적용하고, 런타임 주입 슬롯(`<master_setting>`, `<world_lore>`, `<system>`)의 포맷 제약 및 Cognitive Core 3.0의 Turn 0 초기 상태 요구사항을 반영하지 못함.

### 3. 해결책 (Resolution)
- **전 단계 `ModelTier.Pro` (`ThinkingLevel.high`) 격상, 2축 톤(`SurfaceTone` / `CoreWeight`) 설계 및 아키텍트 내부 프롬프트 표준화**: 일상 대화 호흡과 배경/갈등 무게감을 분리 설계하고, `SessionArchitectService` 내부 하드코딩 프롬프트 전수를 무인격 기능 명세(`Domain` & `Task` + `Output Language: Korean`), 2계층 스키마 계약(1계층 TypeScript `interface` + 2계층 `ResponseSchema`), 긍정 프레이밍(Affirmative Framing), `<final_instruction>` 종결 구조로 표준화.
- **로어북 마이크로 배치 병렬 합성 및 구조적 1줄 명사구 압축 (`GenerateLorebooksInParallelAsync`)**: `LorebookPlan`을 최대 3개 단위의 마이크로 배치로 분할하여 `Task.WhenAll` + `ModelTier.Pro`로 병렬 생성함으로써 출력 압축을 원천 차단하고, 추론 예산을 낭비하는 명시적 글자 수(`N자 이내`) 숫자 강제 대신 **구조적 슬롯 바운딩(지정된 하위 불릿 × 각 불릿당 1줄 명사구 개조식) 및 Few-Shot 길이 앵커링**을 적용하여 토큰 소모량을 약 65% 절감함과 동시에 우선순위 오름차순 규격(`0~3`, `0`이 최우선) 및 인물 대표 대사 샘플(`"..."` 2개)을 강제.
- **C# 결정론적 슬롯 정제기 및 코드 하이브리드 `SystemPrompt` 컴파일러**: `SanitizeWorldview`, `SanitizeLorebookContent`, `SanitizeStatusGuide`, `SanitizeScenario`를 통해 불필요한 XML 태그, 마크다운 헤더(`#`), 볼드(`**`) 오염을 물리적으로 제거하고, `CompileHybridSystemPrompt`에서 스탯 지침을 완전 배제(Zero-Numeric)한 채 코어 5대 행동 헌법과 장르 연출 디렉팅을 결합.
- **Turn 0 `ChapterContext` 시딩 및 자동 모드 스포일러 프리 UX (`SessionArchitectViewModel.cs`)**: `StatusDesignPayload`를 통해 초기 등장인물(`InitialChars`, `[초기: ...]` 포맷), 시작 장소(`InitialPlaces`), 소지품(`InitialItems`)을 Turn 0에 즉시 시딩하고, `IsAutoMode`(기본값 `true`) 실행 시 채팅창에는 스포일러 없는 단계별 완료 요약만 표시하도록 개선 (`dotnet test GRC.sln` 단위 테스트 131/131 및 Pro 통합 테스트 통과).

---

## 2026-10-24: [Resolved] Session Architect 2.0 4단계 통합 파이프라인, Gemini 4 Argon 최적화 및 코어 3대 GM 수칙 개편

### 1. 현상 (Symptom)
- **Flash 시절 6단계 파이프라인 및 레거시 병목 잔존**: 과거 `Flash` 모델의 출력 한계를 보완하기 위해 만든 6단계 분할 구조(1단계 기획과 2단계 세계관 중복, 5단계 상태창과 6단계 2줄 GM 수칙 분리), `maxOutputTokens: 8192` 하드코딩, `batchSize = 3` 인위적 로어북 조각 분할, 단계별 LLM 감사관 재호출이 `Gemini 4 Argon` (`ThinkingLevel.high`)의 거시적 통합 추론과 앙상블 캐스팅을 저해하고 불필요한 API 왕복(11~13회)을 유발.
- **반플레이어 편향(Anti-Player Bias) 및 마이크로 스톨링**: 구버전 프롬프트 수칙 4번("작위적 성공 배제. 인물의 가치관에 어긋날 시 거절·경계 반응을 보일 것")으로 인해 일반인 NPC가 살인마 빌런 PC를 물리적으로 제압해버리는 등 세계관 무력/역량 격차를 무시한 억지 실패 유발. 또한 수칙 3번("시간 임의 요약 금지. 단일 사건 직후 즉시 멈출 것")으로 인해 유저가 명시적으로 시간 경과를 요구해도 대화가 정체되는 마이크로 스톨링 발생.

### 2. 원인 (Root Cause)
- **현상 유지 편향(Status-Quo Bias) 및 레거시 제약**: `gemini-4-argon` (`ThinkingLevel.high`)은 내부 추론 토큰을 `max_output_tokens` 예산에서 차감하므로 `32,768` 이상의 헤드룸이 필요함에도 `8192`로 묶여 있었고, 4~6명의 등장인물을 3명씩 쪼개어 병렬 호출함에 따라 전체 인물의 말투 대비와 다각 관계망을 한 번에 조망하지 못함. 또한 동일 모델에게 자가 검수를 맡기는 LLM 재호출이 자기강화 환각과 지연을 유발.
- **구형 가드레일의 역효과**: 초기 LLM의 무조건적 예스맨 현상을 막기 위해 삽입했던 적대적 가드레일이 고성능 추론 모델에서 '유저 행동 무조건 방해 및 억지 실패'로 과보정됨.

### 3. 해결책 및 불변식 (Resolution & Invariants)
1. **4단계 통합 파이프라인 개편 (`SessionArchitectService.cs`, `SessionArchitectViewModel.cs`, `SessionArchitectWindow.xaml`)**:
   - **1단계 (청사진 및 마스터 세계관 통합 — 1회 호출)**: `AgentPlan.WorldviewOutline`에 `<master_setting>`용 4대 번호 섹션 개조식 레퍼런스를 직접 작성하도록 통합하여 거시 세계관과 8~15개 미시 로어북 기획(`connectedEntities`)을 단일 추론 공간에서 공동 설계.
   - **2단계 (2-Wave 통합 앙상블 로어북 — 2회 호출)**: `batchSize = 3` 분할을 폐기하고, Wave 1에서 비인물 무대 인프라 전체를 1회 호출로 확정한 뒤, Wave 2에서 확정된 무대 전문(`<established_world_lore>`) 위에서 모든 등장인물 전원을 1회 통합 앙상블 캐스팅 호출로 동시 생성.
   - **3단계 (오프닝 시나리오 선행 집필 — 1회 호출)**: JSON 이스케이프 및 개조식 문체 전이(Style Bleed) 방지를 위해 독립된 `text/plain` 산문 모드로 집필.
   - **4단계 (상태창·Turn 0 시딩 및 시스템 지시문 통합 — 1회 호출 + C# 컴파일러)**: 오프닝 본문 기반 초기 스탯(`stats`), 수치 갱신 가이드(`guide`), Turn 0 컨텍스트(`initialChars`/`initialPlaces`/`initialItems`, `NormalizeInitialCharKeys` 키 정규화), 그리고 세션 특화 비수치 GM 수칙 4·5번(`genreDirectives`)을 단일 `StatusDesignPayload` JSON으로 추출 후 `CompileHybridSystemPrompt`로 즉시 결합 (총 API 호출 11~13회 $\to$ 5회로 단축).
2. **`ArchitectMaxOutputTokens = 32768` 상수 일원화 및 100% C# 결정론적 검수 관문 (`ReviewStepContentAsync`)**:
   - 아키텍트 전 단계의 `maxOutputTokens`를 `32768`로 상향하여 `ThinkingLevel.high` 추론 깊이 확보.
   - LLM 자가 감사 호출 및 `ReviewResponseSchema`를 전면 삭제하고, C# 파서·정규식 기반의 결정론적 무결성 관문(API 호출 0회)으로 전환하여 결함 적발 시에만 `ReviseLorebooksSurgicallyAsync` 핀포인트 재생성 수행.
3. **불변 코어 GM 헌법 3대 수칙 통합 및 역량 기반 현실적 인과율 정립 (`CoreSystemDirectivesHeader`)**:
   - 1번(오감 중심 물리적 객관 서술), 2번(PC 대사·행동 대신 서술 금지 및 자연스러운 턴 양도), 3번(작위적 주인공 보정과 억지 실패 보정을 모두 배제하고 당사자 간 실제 역량·무력·상황 격차와 인물 가치관에 부합하는 현실적 인과율 판정)으로 통합하고 세션 특화 수칙을 4·5번 슬롯에 매핑.
4. **핵심 부정 제약 (Never use X)**:
   - **절대 NPC에게 무조건적인 적대나 억지 저항을 지시하는 인위적 실패 가드레일을 주입하지 말 것.**
   - **절대 아키텍트 단계마다 LLM 감사관을 재호출하거나 등장인물 로어북을 인위적 마이크로 배치(`batchSize = 3`)로 쪼개어 앙상블 조망을 끊지 말 것.**

---

## 2026-10-24: [Resolved] Session Architect 동적 GM 수칙의 PC 주체성 침해(God-modding 유도) 및 지시문 상충(Instruction Collision) 해소

### 1. 현상 (Symptom)
- `SessionArchitectService`가 생성한 4번 GM 수칙(`genreDirectives`)에 유저 캐릭터(PC)의 성격, 태도, 살기 표출 및 전투 행동을 지시하는 문장이 포함됨 (예: *"은퇴 아저씨의 느긋한 유머를 유지하되, 위기 시 전설적인 재의 검성다운 서늘한 살기와 비릿한 철의 감각 묘사로 급반전시킬 것"*).
- 이는 코어 2번 수칙(*"유저 캐릭터(PC)의 대사, 행동, 내면의 생각·감정을 임의로 대신 서술하지 말고, PC의 선택이나 대응이 필요한 시점에서 서술을 멈추고 턴을 넘길 것"*)과 정면으로 충돌하여, 런타임에 GM 모델이 적대 세력 등장 시 유저의 입력 없이도 멋대로 *"카일의 눈빛이 차갑게 가라앉으며 전설적인 검성의 살기가 뿜어져 나왔다"*라고 서술하는 PC 조작(God-modding) 버그를 유발.

### 2. 원인 (Root Cause)
- **GM 통제 영역(환경·NPC)과 유저 통제 영역(PC)의 경계 미비**:
  - 1단계(`GeneratePlanAsync`)에서 유저 컨셉의 주인공 특성을 세션 전체의 톤앤매너(`surfaceTone`, `coreWeight`, `worldviewOutline` 4번 섹션)로 변환할 때, PC의 성격과 감정을 환경 분위기와 분리하라는 제약이 결여되어 PC의 태도("은퇴 아저씨의 유머", "퇴역 영웅의 고뇌", "검성의 살기")가 세계관 서사 톤으로 오염됨.
  - 4단계(`StatusGen` 및 `GenerateGenreDirectivesAsync`)에서 4번 수칙의 연출 대상을 명시하지 않아, 모델이 예시 템플릿의 문장 구조를 모방하며 PC를 주어로 삼는 지시문을 생성함.

### 3. 해결책 및 불변식 (Resolution & Invariants)
- **1단계 `GeneratePlanAsync` 프롬프트 PC 비간섭 경계 강화**:
  - `surfaceTone`과 `coreWeight`는 **주변 환경의 분위기와 NPC들의 일상 대화 템포, 위기 시 적대 세력·환경의 압박감 등 오직 '세계와 NPC들의 연출 기준'으로 한정**하며, PC의 개인적 성격·유머·감정 상태를 규정하는 것을 엄격히 금지.
  - `worldviewOutline` 3번 섹션(`주인공의 초기 입지 및 직면 환경`)은 PC의 객관적 위장 신분, 물리적 제약/패널티, 시작 당면 과제만 객관적으로 서술하도록 제한하고, 4번 섹션(`서사 톤앤매너 및 감각적 디테일`)은 오감 환경 묘사 및 적대 NPC의 압박감 묘사 기준만 명시하도록 통제.
- **4단계 `StatusGen` 및 `GenerateGenreDirectivesAsync` 주체 분리 강제**:
  - 4번 수칙(서사 분위기 연출)의 주어를 **반드시 배경 환경의 감각 묘사(공기, 빛, 소리, 냄새)와 NPC들의 대사 템포·태도·압박감만으로 한정**하고, PC의 감정·태도·유머·살기 표출·전투 행동을 대신 규정하거나 지시문 주어로 삼는 것을 엄격히 금지.
  - 5번 수칙(세계관 인과율 및 위험 판정)은 세계관/로어북의 제약에 기반하여 유저 행동에 따른 위험과 세력/환경의 반작용만을 공정하게 판정하도록 정립.
- **핵심 부정 제약 (Never use X)**:
  - **절대 시스템 프롬프트의 GM 연출 지침(`genreDirectives`, `worldviewOutline`)에 유저 캐릭터(PC)의 성격, 태도, 감정, 살기 표출, 무력 사용 여부를 기정사실화하는 명령문을 포함하지 말 것.**

---

---

## 2026-10-25: [Resolved] Session Architect 2.0 라이브 연동 5대 런타임 및 조립 결함 교정 (Round 1)

### 1. 현상 (Symptom)
1. **`<master_setting>`(`worldviewOutline`) 평탄화**: LLM이 JSON 문자열 내에서 4대 번호 섹션과 하위 하이픈 불릿을 줄바꿈(`\n`) 없이 공백으로 이어붙여 1,100자 단일 줄로 평탄화하여 서사 GM 가독성 저하.
2. **`BiTemporalEntityProjector` 인물-수치 바인딩 실패**: `<entity_relationship>`에 인물 호감도/신뢰도가 결합되지 않고 `<current_state>`의 `주요 수치`에 고립 잔류 (예: `Chars["엘레나 발렌타인"]` vs `CustomStats["엘레나 발렌타인 신뢰도"]` 키 불일치).
3. **`<master_setting>` vs `lorebookPlan` 토큰 이중 주입**: 거시 세계관 법칙이 상시 주입되는 `<master_setting>`에 이미 존재함에도, `Priority 0~1`의 `세계관` 로어북으로 중복 생성되어 매 턴 약 750토큰 이상 이중 낭비.
4. **비단일 개체 복합 명칭 및 비휴대 무구의 인벤토리 등록**: 로어북에 `"봉인된 흑철 대검과 은빛 알껍질"`처럼 2개 개체가 묶이고, 오프닝 씬에서 지하실에 봉인되어 꺼낼 수 없는 무구가 Turn 0 `initialItems`에 포함됨.
5. **Turn 0 인물 상태 태도 장문 중복**: `initialChars`의 `value`에 40자 이상의 장문 서사 설명이 작성되어 `<world_lore>`의 `* PC 관계` 프로필과 중복 토큰 낭비.

### 2. 원인 (Root Cause)
- 프롬프트 지시문의 `"1줄 개조식"` 표현을 모델이 단일 줄 평탄화로 오인했으며 C# 정제기에 줄바꿈 복원 정규식이 부재함.
- `MemoryManagerService.cs`가 `CustomStats.TryGetValue(charName)` 단일 일치만 수행하고 `CustomStats`, `Items`, `Places`를 `CognitiveContext`로 온전히 전달하지 못함.
- 프롬프트에서 상시 제공되는 `<master_setting>`과 조건부 인출되는 `lorebookPlan`의 명확한 역할 분리 및 단일 개체 원칙이 결여됨.

### 3. 해결책 및 불변식 (Resolution & Invariants)
1. **`SanitizeWorldview` 결정론적 줄바꿈 복원 엔진 (`SessionArchitectService.cs`)**:
   - `(?<!^)(?<!\r?\n)\s+(?=[1-9]\.\s+[가-힣a-zA-Z])` $\to$ `\n\n` (번호 섹션 헤더 앞 줄바꿈 강제)
   - `(?<!^)(?<!\r?\n)\s+-\s+(?=[가-힣a-zA-Z0-9()·/_\s]{2,35}:\s)` $\to$ `\n- ` (하이픈 불릿 앞 줄바꿈 강제)
2. **인물-수치 2단계 프리픽스 매칭 및 중복 제거 (`MemoryManagerService.cs`)**:
   - 1단계 Exact Match 실패 시 2단계 Prefix Match(`charName` 또는 `firstWord` 접두사 검사)를 통해 `"엘레나 발렌타인 신뢰도"`에서 접미사 `"신뢰도"`를 추출하여 `(신뢰도: 60/100)`으로 엔티티 슬롯에 직결.
   - 바인딩된 스탯 키를 `matchedStatKeys`로 추적하여 `remainingCustomStats`에서 제외함으로써 `<current_state>`의 `주요 수치` 내 중복 노출을 100% 제거.
   - `remainingCustomStats`, `itemsSnapshot`, `placesSnapshot`을 `new CognitiveContext(...)`로 무손실 전달.
3. **거시 법칙 vs 미시 개체 역할 분리 및 1개체 1항목 원칙 (`GeneratePlanAsync`)**:
   - `worldviewOutline`(`<master_setting>`)이 세계 거시 법칙을 상시 전담하므로 `lorebookPlan`에서 중복 '세계관' 작성을 금지하고 조건부 인출 대상(인물, 단일 장소, 단일 무구, 고유 세부 사건)만을 전담하도록 분리.
   - 1항목 1개체 원칙을 강제하여 `"A와 B"` 식의 복합 명칭 엄격 금지.
4. **현장 휴대 소지품 한정 및 감정 키워드 압축 (`StatusGen`)**:
   - `initialItems`는 오프닝 현장에서 몸에 직접 휴대/착용 중인 물품으로만 한정(지하실/창고 등 격리 보관물 제외).
   - `initialChars.value`의 태도 기술을 2~4어절의 핵심 감정 키워드로 압축(`초기 태도(경계와 호기심)`)하여 마이크로 토큰 낭비 제거.
5. **수치 클램핑 및 가이드 XML 순수화 (`MemoryManagerService.cs`)**:
   - `UpdateContextStatus`에서 `"120/100"`, `"-15/100"` 등 범위형 게이지를 `0~max`로 즉시 클램핑(`ClampGaugeValue`).
   - `BuildStatusRequest`의 `statusUpdateGuide`를 순수 XML 태그(`<custom_rules>...</custom_rules>`)로 주입하고 구형 `[CUSTOM_RULES]` 의사 태그 완전 제거.

---

## 2026-10-25: [Resolved] Session Architect 2.0 라이브 테스트 5대 런타임·정제 결함 정밀 교정 (Round 2)

### 1. 현상 (Symptom)
1. **풀네임 인물-약칭 스탯 매칭 누락**: 인물명은 풀네임(`아리아 폰 블랑셰`, `시엘 클로버`)이고 스탯명은 첫 어절 약칭(`아리아 호감도`, `시엘 호감도`)인 경우, 접두사 불일치로 `<entity_relationship>`에 스탯이 바인딩되지 못하고 `<current_state>`의 `주요 수치`에 중복 잔류.
2. **로어북 샘플 대사 오프닝 복붙 및 본심 조기 소진**: `ScenarioGen`의 `"말투/대사를 로어북 원문과 100% 일치"` 지침으로 인해, 모델이 로어북에 작성된 예시 대사(심지어 후반부 본심 대사)를 오프닝 시나리오에 토씨 하나 바꾸지 않고 복사해 넣고 4명의 인물을 과밀 등장시킴.
3. **Turn 0 인물 상태 태도 장문화 및 인원 누락**: `StatusGen`의 `초기 태도(...)` 래퍼 및 복장·장소 요구로 인해 30단어 이상의 장문이 출력되고, `주요 NPC(1~3명)` 제한으로 오프닝에 등장한 4번째 인물(칼리아)이 누락됨.
4. **다중 히로인 스탯 비대칭 및 집착 클리셰 수렴**: 6인 히로인 중 3명만 Turn 0 호감도 스탯이 생성되어 비대칭이 발생하고, Wave 2 인물 일괄 생성 시 다수 인물의 내면 결핍이 '독점욕/집착'으로 획일화됨.
5. **다국어 글리치 토큰 혼입**: Gemini 모델이 다국어 토크나이저 글리치로 인해 한국어 산문 중간에 데바나가리 문자(`सिर`) 등을 방출하였으나 정제기에 외국 문자 제거 로직이 부재함.

### 2. 원인 (Root Cause)
- `MemoryManagerService.cs`가 전체 이름 접두사(`charName + " "`)만 검사하고 고유 첫 어절(`firstToken + " "`) 검사를 지원하지 않음.
- `ScenarioGen` 지시문이 어조 유지가 아닌 예시 대사 문자열 자체의 100% 일치를 강제하여 발생한 모델의 직독직해(Literal Translation).
- `StatusGen` 지시문 및 예시가 런타임 상태창 포맷과 달리 장문 위치/복장 서술을 유도했고 인원수 상한을 둠.
- `Sanitize*` 파이프라인에 비한국어/비라틴/비CJK 비정상 유니코드 블록 필터링 부재.

### 3. 해결책 및 불변식 (Resolution & Invariants)
1. **인물명 첫 어절 토큰 접두사 매칭 확장 (`MemoryManagerService.cs`)**:
   - 전체 이름 접두사(`"아리아 폰 블랑셰 "`)와 고유 첫 어절 접두사(`"아리아 "`)를 모두 후보에 포함하여, `"아리아 호감도"`를 `- 인물(아리아 폰 블랑셰): ... (호감도: 25/100)`으로 자동 결합 및 `<current_state>` 중복 제거.
2. **로어북 대사 복붙 금지 및 등장인물 밀도 제어 (`SessionArchitectService.cs`)**:
   - `ScenarioGen` 및 `ScenarioReview`에 `Output Language: Korean` 추가.
   - 로어북 샘플 대사 문장의 단순 복사·인용을 엄격히 금지하고 상황에 맞는 신규 대사 창작 및 본심 조기 소진 차단 지침 명시.
   - 오프닝 현장 직접 대면 인원을 1~3명으로 집중.
3. **`initialChars` 전수 등록 및 `value` 15~25자 경량화 (`SessionArchitectService.cs`)**:
   - 오프닝에 등장한 모든 NPC를 누락 없이 등록하고, `"초기 태도(...)"` 래퍼 및 복장·장소 중복 서술을 배제한 채 `3~6어절(25자 이내)`의 감정·즉각 행동으로 프롬프트 및 예시 개정.
   - C# `SanitizeInitialCharValue` 결정론적 정제기를 통해 잔여 `초기 태도(...)` 래퍼를 정규식으로 자동 제거.
4. **다중 히로인 동적 호감도 확장 및 내면 결 다양화 (`SessionArchitectService.cs`)**:
   - `guide`에 미등장 주요 인물과 본편 조우 시 `{인물명} 호감도` 스탯을 동적 추가한다는 룰을 명시하여 스탯 비대칭 해소.
   - Wave 2 인물 생성 프롬프트에 '독점욕·집착' 단일 클리셰로 획일화되지 않도록 관계 발전 양상(라이벌, 헌신, 지적 호기심, 비호 등) 차별화 지침 추가.
5. **다국어 글리치 스크립트 정제기 (`StripForeignGlitchScripts`)**:
   - 데바나가리, 아랍, 태국, 키릴, 히브리 등 한국어 RP 텍스트에 사용되지 않는 유니코드 블록(`[\u0400-\u06FF\u0900-\u0E7F]+`)을 일괄 제거하는 헬퍼를 추가하여 모든 `Sanitize*` 파이프라인에 공통 적용.

---

## 13. 서브컬처 에피소드 & 시드 엔진 (`<story_seeds>`) 스키마 정합성 및 동시성/오탐 방어

### 증상 및 잠재적 결함 (Root Cause)
1. **세션 아키텍트 스키마와 시드 파서 정규식 간 불일치**:
   - `SessionArchitectService`가 생성하는 인물 로어북은 첫 줄에 `분류(인물): ...` (선행 `*` 없음) 및 `* 표면/내면: 표면(...) / 내면(...)` 포맷을 사용하나, `MemoryManagerService`의 초기 정규식이 `^\*\s*분류\s*:`와 `^\*\s*내면\s*:`만 매칭하여 AI 생성 세션에서 분류 및 60+ 내면 단서가 누락될 수 있었음.
2. **일반 명사 키워드 및 일상 동료 대사에 의한 시드 오탐 소모 (False-Positive Consumption)**:
   - `ConsumeMatchedSeedsLocked`에서 로어북의 모든 `Keywords`(예: `"수습생"`, `"마탑"`)를 소모 트리거로 포함할 경우, 한 수습생 캐릭터가 등장했을 때 동일 키워드를 공유하는 다른 미등장 캐릭터 시드까지 연쇄 소모되거나, 동행 중인 동료가 일상 대사를 한 것만으로 `[동료 유대 심화]` 시드가 즉시 소모되는 결함이 있었음.
3. **동료 유대 심화 2단계(호감도 30+ vs 60+) 단일 키 잠금 문제**:
   - 호감도 30+ 구간의 유대 심화 에피소드와 60+ 구간의 내면 비밀 해금이 동일한 `BOND:{Name}` 키를 공유할 경우, 30+에서 유대 에피소드를 한 번 소모하면 이후 호감도가 60+에 도달해도 내면 비밀 시드가 영구 차단됨.
4. **세션 저장/복원 시 `Lorebooks = null` 슬림화 및 백그라운드 상태창 태스크 경합**:
   - `ExportSession`이 디스크 중복 방지를 위해 `Preset with { Lorebooks = null }`로 직렬화하므로, 세션 로드 시 `savedSession`을 그대로 `RestoreSession`에 넘기면 첫 턴 전까지 `_cachedLorebooks`가 비어 있게 됨. 또한 `ChatDelay == 0` 환경에서 `AddModelResponse`와 `onDownloadComplete`/`BuildStatusRequest` 간 메시지 타이밍 경합이 발생할 수 있었음.

### 해결책 (Resolution)
1. **양방향 호환 정규식 및 내면 파서 (`MemoryManagerService.cs`)**:
   - `CategoryFieldRegex`가 `^\*\s*분류(?:\([^)]*\))?\s*:`와 `^분류(?:\([^)]*\))?\s*:`를 모두 지원하고, `InnerThoughtFieldRegex` + `ExtractInnerThoughtText`가 `* 표면/내면: 표면(...) / 내면(...)`에서 `내면(...)` 구간만 정확히 분리 추출하도록 개선.
2. **고유명사/유대 핵심어 이원화 매칭 (`OfferedSeedCandidate`)**:
   - Pool A/B(인물 조우·장소 훅)는 `ExtractProperNameKeywords(entry.Name)` 고유명사 매칭으로만 소모 판정하고, Pool C(동료 유대 심화)는 동료 이름(`PrimaryKeywords`)과 유대/내면 본문에서 추출한 핵심 명사(`SecondaryKeywords`, `ExtractBondContentKeywords`)가 **동시에** 등장할 때만 소모되도록 강화.
3. **`BOND:{Name}` (30+) 및 `BOND_SECRET:{Name}` (60+) 키 분리**:
   - 호감도 30~59 구간(`BOND:`)과 60+ 내면 비밀 해금 구간(`BOND_SECRET:`)의 소모 키를 분리하여 단계별 서사 보상이 모두 정상 작동하도록 보장.
4. **`ChapterContext.Clone()` 스냅샷 격리 및 `AwaitPendingStatusTaskAsync` 동기화 (`ChatViewModel.cs`)**:
   - 세션 로드 및 아키텍트 편집 복원 시 `savedSession with { Preset = sessionPreset }`을 전달하여 `_cachedLorebooks`를 즉시 초기화하고, `ExportSession`/`RestoreSession`에서 `ChapterContext.Clone()` 깊은 복사로 백그라운드 JSON 직렬화 중 컬렉션 변경 예외를 원천 차단.

---

## 2026-10-26: [Resolved] EntityNameResolver 및 NarrativeContract SSOT 도입 (상태창 인물 덮어쓰기 차단, 비인물 첫 어절 오염 제거, Turn 1 오프닝 로어북 스캔)

### 1. 현상 (Symptom)
1. **비인물 로어북 첫 어절 일반 수식어 자동 주입 및 Pool B(`HOOK:`) 시드 오탐 소모**:
   - `SessionArchitectService.NormalizeKeywords`가 카테고리를 구분하지 않고 모든 복합 명칭의 첫 어절(`nameParts[0]`)을 키워드에 무조건 추가하여, `"지하 유적 제단"`에서 `"지하"`, `"그림자 사냥개 길드"`에서 `"그림자"`가 단독 키워드로 등록됨.
   - `MemoryManagerService.ExtractProperNameKeywords` 역시 첫 어절을 `PrimaryKeywords`에 추가하여, 서사 응답에 `"지하 창고"`나 `"그림자가 드리웠다"`만 등장해도 미방문 장소/세력 시드(`HOOK:`)가 오탐 소모(`ConsumedSeeds`)됨.
2. **런타임 상태창 갱신(`UpdateContextStatus`) 시 일반 직함 키워드 충돌로 인한 기존 인물 상태 덮어쓰기**:
   - `NormalizeInitialCharKeys`가 `entry.Keywords` 전체에 대해 부분 문자열 포함(`cleanKey.Contains(kw)`)을 검사함에 따라, `"성녀"` 키워드를 가진 `"세실리아 본 로젠버그"`가 있는 세션에서 단역 `"성녀 호위대장"`이 등장하면 `"세실리아 본 로젠버그"`의 상태창 슬롯을 덮어쓰는 상태 오염 버그 존재.
3. **Turn 1 오프닝 장면의 장소·소품·세력 로어북 인출 공백**:
   - `InjectInitialScenario`가 Turn 0를 `[User: <initial_scenario>...]` + `[Model: "상황을 파악했습니다. 당신의 행동을 기다립니다."]`로 구성하고 `BuildNarrativeRequestAsync`는 직전 `model` 응답만 스캔하여, Turn 1에 오프닝 시나리오 본문 속 시작 장소/소품/배경 세력 로어북이 1차 키워드 스캔에서 누락됨.
4. **오프닝 시나리오 고유명사 낫표(`「...」`) 오염 및 서브컬처 메타 은어("갭 모에") 혼입**:
   - `ScenarioGen` 지시문의 `"독백/속마음 및 강조용은 낫표(「...」)"` 문구가 런타임 `CognitivePromptAssembler.FormatRule`과 상충하여 고유명사에 `「...」`가 남용되고, `GeneratePlanAsync`의 `"갭 모에"` 지시어가 다크 판타지 마스터 세계관에 노출됨.

### 2. 원인 (Root Cause)
- 엔티티 명칭·별칭 파생, 동일 엔티티 판정, 시드 소모 경계 매칭, 서사 문장부호 계약이 `SessionArchitectService`, `MemoryManagerService`, `CognitivePromptAssembler` 3개 클래스에 각각 다른 휴리스틱으로 파편화되어 있었음.

### 3. 해결책 및 불변식 (Resolution & Invariants)
1. **`EntityNameResolver` (`GRC/Helpers/EntityNameResolver.cs`) SSOT 도입**:
   - `DeriveAliases(entryName, rawKeywords, category)`: 복합 명칭의 첫 어절 자동 추출은 오직 `인물` 카테고리에서만 수행하며, 비인물(`장소`, `아이템`, `세계관`)에서는 첫 어절 수식어가 단독 키워드로 등록되는 것을 원천 차단.
   - `ResolveCharacterKey(rawKey, characterLorebooks)`: 일반 `Keywords`(직함·소속) 부분 문자열 매칭을 전면 배제하고, 오직 전체 엔트리명 일치 또는 고유 이름 첫 어절(`GetCharacterGivenName`) 완전 일치 시에만 정규화하여 단역 NPC에 의한 주연 상태 덮어쓰기 차단.
   - `DeriveHookPrimaryKeywords` 및 `ContainsEntityMention`: Pool B(`HOOK:`) 시드는 전체 엔트리명 등장(Tier 1) 또는 2어절 이상 핵심 구분자 + `* 서사 훅` 본문 핵심어 동시 등장(Tier 2) 시에만 소모되도록 이원화하고, 한국어 조사 경계 정규식을 적용하여 합성어 부분 일치 오탐 차단.
2. **`NarrativeContract` (`GRC/Services/Cognitive/NarrativeContract.cs`) SSOT 도입 및 `SanitizeScenario` 정제**:
   - 런타임과 아키텍트의 문장부호 규칙(`""` 대사, `「」` 속마음/독백 전용, 고유명사/강조용 `「」` 금지)과 로어북 필드 정규식을 단일 정적 계약으로 통합하고, `SanitizeScenario(raw, lorebooks)`에서 로어북 고유명사를 감싼 강조용 `「...」`를 결정론적으로 스트리핑.
   - `GeneratePlanAsync` 및 `GetLorebookBatchSystemInstruction`에서 `"갭 모에"` 등 메타 은어를 `"반전 면모"` 및 세계관 내 서술어로 치환하고, 오프닝 등장 예정 인물의 `* PC 관계: 초기 접점(...)`이 오프닝 직면 상황과 일치하도록 제약 명시.
3. **Turn 1 `<initial_scenario>` 자동 스캔 연동 (`MemoryManagerService.cs`)**:
   - `lastModelMessage.Text == NarrativeContract.InitialScenarioAckText`인 Turn 1에는 직전 `<initial_scenario>` 원문을 `loreScanQuery`에 결합하여 오프닝 현장의 장소·소품·세력 로어북이 첫 턴부터 즉시 인출되도록 보장.
4. **핵심 부정 제약 (Never use X)**:
   - **절대 비인물(`장소`, `아이템`, `세계관`) 복합 명칭의 첫 어절을 단독 키워드나 시드 소모 트리거로 자동 추출하지 말 것.**
   - **절대 상태창 인물 Key 정규화(`ResolveCharacterKey`)에 일반 직함/소속 `Keywords` 부분 문자열 포함 검사(`.Contains`)를 사용하지 말 것.**

---

## 2026-10-26: [Resolved] `<world_lore>` 미해금 스포일러 조기 노출 및 `유대 심화` 완료형 문구에 의한 인과 건너뛰기(보상 선지급 환각) 차단

### 1. 현상 (Symptom)
1. **초기 단계(호감도/신뢰도 30 미만) `<world_lore>` 스포일러 누수**:
   - `HybridLorebookRetriever`가 인출한 인물 로어북 원문이 마스킹 없이 `<world_lore>`에 그대로 노출됨에 따라, 아직 친밀도가 낮은 상태에서도 `* 표면/내면`의 `/ 내면(...)`, `* PC 관계`의 `/ 유대 심화(...)`, 그리고 **`* 말투/대사`의 두 번째 심화 대사(`/ "..."`)**에 적힌 후반부 기밀/연심 대사가 조기 반영됨.
2. **완료형 `유대 심화` 문구로 인한 선행 퀘스트 건너뛰기 및 보상 선지급 환각**:
   - 인물 로어북의 `유대 심화`가 `"대미궁 중층부에서 오빠의 부러진 인식표를 회수해 오자 깊은 신뢰를 보내며 재판소 비밀 검문 일정을 빼돌려 주는 조력자로 거듭남"`처럼 **과거 완료형 결과 요약문**으로 작성된 경우, 신뢰도 30 달성 후 유저가 검문 일정을 물으면(`<internal_chronicle_facts>`에 인식표 회수 사실이 없음에도) 서사 AI가 인식표 수색 의뢰 과정을 통째로 건너뛰고 최종 보상인 비밀 검문 일정을 즉시 유출함.
   - 또한 `"~회수하는 사건을 계기로 ~조력자로 전환"`처럼 발단과 결과를 뭉뚱그린 문구를 주입할 경우, NPC가 플레이어에게 수색 의뢰(발단)조차 제안하지 못한 채 키워드 일치만으로 `BOND:{Name}` 시드가 1턴 만에 영구 소모(`ConsumedSeeds`)되어 에피소드가 증발함.

### 2. 원인 (Root Cause)
- `<world_lore>` 주입 파이프라인에 인물의 현재 관계도 수치(`CustomStats`)에 따른 단계별 필드 마스킹이 부재했고, `* 말투/대사`의 두 번째 예시 대사에 완료 시점 스포일러 대사가 포함됨.
- `<story_seeds>` 안내문에 `<internal_chronicle_facts>`에 없는 과거 사건 날조 및 중간 과업 생략 금지 가드가 없었으며, 생성기 프롬프트가 `유대 심화`의 **[발단 계기(NPC의 의뢰/제안)]**와 **[완수 후 변화(과업 해결 후 보상)]**를 분리하도록 강제하지 않음.

### 3. 해결책 및 불변식 (Resolution & Invariants)
1. **`MaskLorebooksForPrompt` 호감도/신뢰도 단계별 3중 마스킹 (`MemoryManagerService.cs`)**:
   - `<world_lore>`용 `RetrievedLores` 생성 시 인물별 최대 관계도(`GetCharacterMaxAffinity`) 및 시드 소모 이력을 확인하여:
     - **30 미만 (`!unlockBond`)**: `/ 유대 심화(...)`, `/ 내면(...)`, `* 말투/대사`의 두 번째 대사(`/ "..."`) 3곳 모두 마스킹.
     - **30~59 구간 (`unlockBond && !unlockSecret`)**: `/ 유대 심화(...)`만 공개하고 `/ 내면(...)` 및 두 번째 대사는 60 달성 시까지 마스킹 유지.
     - **60 이상 (`unlockSecret`)**: 전 구간 공개 (`CurrentActiveLorebooks` 프로퍼티는 UI/참조 무결성을 위해 원본 유지).
2. **`<story_seeds>` 인과율 방어 헤더 및 `[발단 계기 -> 완수 후 변화]` 작성 규격 (`CognitivePromptAssembler.cs`, `SessionArchitectService.cs`)**:
   - `<story_seeds>` 안내문에 **`<internal_chronicle_facts>`에 기록되지 않은 과거 사건 날조 및 중간 과업 생략(완수 보상 선지급) 절대 금지, 반드시 사건의 '발단(의뢰 제안·거래 제시·고민 토로·단서 발견)' 단계부터 개막할 것**을 명시.
   - `SessionArchitectService` 및 에이전트 템플릿의 인물 로어북 규격에서 `유대 심화(발단 계기: ... -> 완수 후 변화: ...)` 구조화를 강제하고 두 번째 예시 대사에 미발생 퀘스트 완수 스포일러 기재를 금지.

---

## 2026-10-26: [Resolved] 세션 생성기 프롬프트의 예시 오염(Few-Shot Anchoring) 및 구체어 열거형 부정 지시문 정리

### 1. 현상 (Symptom)
- 에이전트 템플릿( 5_session_architect_agent_template.md)으로 생성한 세션에서 메인 히로인 외양이 반복적으로 같은 조합(은백색·백금발 장발, 푸른 눈동자, 백은·백색 제복, 옷자락·장갑 끝을 만지는 버릇)으로 수렴함.

### 2. 원인 (Root Cause)
- 템플릿 Preset.json 스키마 예시에 구체적 인물(이름, 외양, 버릇, 장소, 스탯)이 하드코딩되어 Few-Shot 앵커로 작동함.
- 같은 템플릿과 SessionArchitectService.cs 생성기 프롬프트 곳곳에 과거 세션·과거 결함에서 가져온 구체어를 나열한 부정 지시문(금지 키워드 예시, 금지 클리셰 목록, 과거 세션 소품명, 메타 은어 목록, 학원물 genreDirectives 예시 등)이 남아 흰 곰 효과(원칙 10)로 해당 어휘의 어텐션을 오히려 활성화함.

### 3. 해결책 및 불변식 (Resolution & Invariants)
1. 템플릿 Preset.json/Session.json 예시는 전부 대괄호 구조 플레이스홀더로 교체(형식만 제시, 내용 앵커 제거).
2. 구체어 열거형 부정 지시문을 긍정형 대체 행동으로 전환: 키워드 고유성 원칙, 단일 소유 기술 원칙, 낫표 독백 전용 원칙, 대사 신규 창작, 본심 은닉, 조건부 유대 설계, 외양의 출신·직업·문화권 인과 도출. 템플릿의 Never Rules 9개도 Invariant Rules(Always 형식)로 재작성.
3. 생성기 GetLorebookBatchSystemInstruction() 인물 예시의 유대 심화를 발단 계기 -> 완수 후 변화 규격으로 동기화하고, Wave 2 <final_instruction>에 외양 인과 도출·인물 간 구별 지시를 긍정형으로 추가.
4. **불변식**: 생성기 프롬프트와 템플릿의 예시는 구조 플레이스홀더만 사용하고, 금지 대상을 구체 어휘로 열거하지 말 것. 부정형은 원칙 10의 예외 범주(PC 불간섭 안전선, 형식 고정, 임베딩 더미 벡터 금지 같은 기술 가드)로 한정.

### 4. 후속 보정 (2026-10-26, 등불 골목 세션 실측 기반)
- **첫 번째 샘플 대사 고착**: 생성 결과에서 인물 `* 말투/대사` 첫 번째 대사가 Turn 0 위기 직후의 일시적 감정 상태로 작성됨. 첫 번째 대사는 3단계 마스킹 대상이 아니어서 매 턴 `<world_lore>`로 주입되므로, 관계가 진전된 뒤에도 말투가 해당 상태에 고착될 위험이 있음. 생성기 규격과 템플릿에서 첫 번째 대사를 "Turn 0 오프닝과 무관하게 반복되는 일상 상황의 평상시 기본 어조"로 재정의.
- **대표 명칭 규칙**: 동봉품·병설 공간을 `Name`에 병기한 복합 명칭이 생성됨(런타임 기능 결함은 없음, `IsSamePlaceOrItem`이 `Keywords`로 대조). `Name`은 대표 고유 명사 1개로 짓고 부속 구성요소는 본문과 `Keywords`에서 서술하도록 Plan 단계 규칙에 추가.
- **유지 결정**: 메인 인물 외양에 남은 단일 색채 수식어 잔상은 사전학습 분포 편향으로 판정. 특정 색을 지시문에 명시하면 흰 곰 효과가 재발하므로 템플릿에서 추가 억제하지 않음(외모 고정이 필요하면 유저 컨셉에 직접 명시).
