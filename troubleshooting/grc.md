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

## 2026-10-18: [Resolved] 프롬프트 문자열 들여쓰기 공백 유입, 초기 시나리오 중복 헤더 오염 및 구조화 디코딩 스키마 강제

### 1. 현상 (Symptom)
- C# `@""` 축어적 리터럴 및 문자열 결합(`+`) 사용 시 소스코드 들여쓰기 공백이 LLM 프롬프트에 그대로 유입되어 토큰 낭비 및 어텐션 오염 발생.
- `ChatViewModel`에서 `InjectInitialScenario` 호출 시 `[초기 상황]\n` 헤더를 중복 부착하여 `<initial_scenario>\n[초기 상황]\n...` 이중 헤더 오염 발생.

### 2. 원인 (Root Cause)
- `@""` 리터럴은 좌측 들여쓰기 공백을 스트리핑하지 않으며, 초기 시나리오 주입 경로에 멱등성 가드 및 디코딩 레벨 스키마(`responseSchema`)가 누락됨.

### 3. 해결책 및 불변식 (Resolution & Invariants)
- **C# 11 원시 문자열 리터럴(`"""` / `$$"""`) 전면 일원화**: 컴파일 타임 자동 들여쓰기 스트리핑으로 공백 토큰 유입 원천 차단.
- **초기 시나리오 멱등성 가드 및 `responseSchema` 등록**: `InjectInitialScenario`에서 순수 시나리오 원문만 `Trim()` 후 주입하고, `LorebookService`·`ReplySuggestionService`·`SessionArchitectService`에 `responseSchema`를 등록하여 디코딩 문법 무결성 보장.
- **핵심 부정 제약 (Never use X)**: **절대 다중 줄 프롬프트 문자열에 `@""` 축어적 리터럴을 사용하지 말 것.** 반드시 C# 11 `"""` / `$$"""` 원시 문자열 리터럴만 사용할 것.

---

## 2026-10-19: [Resolved] AI 답변 추천(ReplySuggestionService)의 대화 슬라이싱 하드코딩 제거 및 미압축 단기 기억 전수 연동

### 1. 현상 (Symptom)
- `ChatViewModel`에서 답변 추천 생성 시 `ChatHistory.TakeLast(9)` 고정 슬라이싱으로 인해 중기 기억(`Plot`, 18개 메시지 임계치) 압축 대기 중인 6~8턴 전 미압축 대화가 누락되는 과도기 기억 사각지대(Transitional Gap) 발생 및 `<current_snapshot>` 이중 중첩 XML 충돌 발생.

### 2. 원인 (Root Cause)
- 중기 기억 압축 임계치(18개 메시지)와 UI 컬렉션 고정 슬라이싱(`TakeLast(9)`) 간 라이프사이클 불일치 및 래퍼 태그명 중복.

### 3. 해결책 및 불변식 (Resolution & Invariants)
- **미압축 단기 기억 전수 연동 (`IMemoryManagerService.GetCurrentShortTermMemory`)**: UI 하드코딩 슬라이싱(`TakeLast`)을 제거하고 `_memoryService.GetCurrentShortTermMemory()`를 전달하여 `Plot`(압축 완료) + `_shortTermMemory`(미압축 최근 대화 전수) 무손실 결합 확립.
- **태그 계층 충돌 해소 (`ReplySuggestionService.cs`)**: 외곽 래퍼를 `<session_context>`로 변경하여 내부 `<current_snapshot>`과의 중첩 제거.
- **핵심 부정 제약 (Never use X)**: **절대 단기 기억 대화를 UI 컬렉션(`ChatHistory`)에서 임의 개수(`TakeLast`)로 잘라내지 말 것.** 반드시 `_memoryService.GetCurrentShortTermMemory()`와 동기화할 것.

---

## 2026-10-24: [Resolved] Session Architect 추론 토큰 잠식 방어, Turn 0 시딩 및 반플레이어 억지 저항 가드레일 제거

### 1. 현상 (Symptom)
- `gemini-4-argon` (`ThinkingLevel.high`) 호출 시 `maxOutputTokens: 8192` 제한으로 인해 내부 추론 토큰이 출력 예산을 잠식하여 장문 JSON 절단 위험 발생 및 단계별 LLM 자가 검수 재호출로 인한 자기강화 환각·지연 누적.
- 세션 생성 시 `ChapterContext`(`Chars`, `Places`, `Items`)가 빈 컬렉션(`new()`)으로 방치되어 Turn 1 엔티티 기반 인출이 작동하지 않음.
- 구버전 프롬프트의 적대적 가드레일("인물의 가치관에 어긋날 시 거절·경계 반응")이 고성능 추론 모델에서 역량 격차를 무시한 반플레이어 억지 실패 및 시간 경과 정체(마이크로 스톨링) 유발.

### 2. 원인 (Root Cause)
- Thinking 토큰과 가시 출력 토큰이 합산 차감되는 예산 구조 미반영, Turn 0 초기 상태 시딩 누락, 초기 LLM의 예스맨 방어용 적대적 가드레일이 고성능 모델에서 과보정됨.

### 3. 해결책 및 불변식 (Resolution & Invariants)
- **`ArchitectMaxOutputTokens = 32768` 상향 및 100% C# 결정론적 검수 관문 (`ReviewStepContentAsync`)**: 전 단계 출력 상한을 `32768`로 일원화하고, LLM 자가 감사 호출을 폐기하여 C# 정규식·파서 기반 무결성 관문(결함 시에만 핀포인트 재생성)으로 전환.
- **Turn 0 `ChapterContext` 시딩 및 역량 기반 인과율 정립**: 오프닝 기반 `InitialChars`·`InitialPlaces`·`InitialItems`를 Turn 0에 즉시 시딩하고, 코어 GM 헌법을 당사자 간 실제 역량·상황 격차에 부합하는 현실적 인과율 판정으로 일원화.
- **핵심 부정 제약 (Never use X)**: **절대 NPC에게 무조건적 적대·억지 저항을 지시하는 인위적 실패 가드레일을 주입하거나, 단계마다 LLM 감사관을 재호출하지 말 것.**

---

## 2026-10-24: [Resolved] Session Architect 동적 GM 수칙의 PC 주체성 침해(God-modding 유도) 및 지시문 상충 해소

### 1. 현상 (Symptom)
- `SessionArchitectService`가 생성한 동적 GM 수칙(`genreDirectives`) 및 세계관 톤에 유저 캐릭터(PC)의 성격, 감정, 살기 표출, 전투 행동을 규정하는 명령문이 포함되어 코어 2번 수칙(PC 대사·행동·내면 대리 서술 금지)과 상충(Instruction Collision)하며 런타임 God-modding 유발.

### 2. 원인 (Root Cause)
- 생성기 프롬프트에서 GM 통제 영역(배경 환경·NPC 태도)과 유저 통제 영역(PC 행동·감정)의 주어 경계가 분리되지 않음.

### 3. 해결책 및 불변식 (Resolution & Invariants)
- **GM 연출 대상과 PC 통제 영역의 물리적 분리**: `surfaceTone`, `coreWeight`, `worldviewOutline` 및 동적 GM 수칙(4·5번)의 주어를 오직 **배경 환경의 감각 묘사, NPC의 대사 템포·태도, 행동에 따른 세계/세력의 반응 판정**으로만 한정.
- **핵심 부정 제약 (Never use X)**: **절대 시스템 프롬프트의 GM 연출 지침(`genreDirectives`, `worldviewOutline`)에 유저 캐릭터(PC)의 성격, 태도, 감정, 살기 표출, 무력 사용을 기정사실화하거나 주어로 삼는 명령문을 포함하지 말 것.**

---

## 2026-10-25: [Resolved] Session Architect 런타임 정제·스탯 바인딩 불일치 및 다국어 토크나이저 글리치 문자 혼입 해결

### 1. 현상 (Symptom)
- LLM이 `worldviewOutline` 번호 섹션과 하이픈 불릿을 줄바꿈 없이 단일 줄로 평탄화하거나, 다국어 토크나이저 글리치로 한국어 산문 중간에 데바나가리 문자(`सिर` 등)를 방출.
- 인물명이 풀네임(`아리아 폰 블랑셰`)이고 스탯명이 첫 어절 약칭(`아리아 호감도`)이거나 접미사형(`엘레나 발렌타인 신뢰도`)일 때 `<entity_relationship>` 바인딩에 실패하고 `<current_state>`의 `주요 수치`에 중복 잔류.
- 상태창 갱신 시 `"120/100"`, `"-15/100"` 등 범위 초과 게이지가 무검증 반영되고, `initialChars.value`에 `"초기 태도(...)"` 래퍼 및 장문 복장 묘사가 중복 포함됨.

### 2. 원인 (Root Cause)
- `Sanitize*` 정제기에 줄바꿈 복원 및 비한국어 글리치 유니코드 블록 제거 정규식이 부재했으며, `MemoryManagerService.cs`가 `CustomStats` 단일 키 완전 일치만 검사하고 게이지 클램핑을 수행하지 않음.

### 3. 해결책 및 불변식 (Resolution & Invariants)
- **결정론적 줄바꿈 복원 및 다국어 글리치 스크립트 제거 (`SessionArchitectService.cs`)**:
  - `SanitizeWorldview`에서 번호 섹션 앞(`\n\n`) 및 하이픈 불릿 앞(`\n- `) 줄바꿈을 정규식으로 강제 복원하고, `SanitizeInitialCharValue`로 `초기 태도(...)` 래퍼를 제거.
  - `StripForeignGlitchScripts`를 통해 데바나가리·아랍·태국·키릴·히브리 등 비정상 유니코드 블록(`[\u0400-\u06FF\u0900-\u0E7F]+`)을 모든 `Sanitize*` 파이프라인에서 일괄 제거.
- **2단계 스탯 Prefix 매칭 및 게이지 클램핑 (`MemoryManagerService.cs`)**:
  - Exact Match 실패 시 풀네임(`charName + " "`) 및 고유 첫 어절(`firstToken + " "`) 접두사 매칭으로 스탯을 `<entity_relationship>`에 결합하고, 매칭된 키(`matchedStatKeys`)는 `remainingCustomStats`에서 제외하여 중복 노출을 차단.
  - `ClampGaugeValue`로 범위형 게이지를 `0~max`로 즉시 클램핑하고 `statusUpdateGuide`는 순수 `<custom_rules>` XML 태그로만 주입.

---

## 2026-10-25: [Resolved] 스토리 시드 엔진(`<story_seeds>`) 오탐 소모 방어, 2단계 유대 키 분리 및 세션 직렬화 동시성 격리

### 1. 현상 (Symptom)
- 일반 명사 키워드 등장이나 일상 동료 대사만으로 미등장 인물/유대 심화 시드가 연쇄 오탐 소모(False-Positive Consumption)되며, 호감도 30+ 유대 에피소드와 60+ 내면 비밀 해금이 단일 `BOND:{Name}` 키를 공유하여 30+ 소모 후 60+ 시드가 영구 잠김.
- `ExportSession`이 디스크 중복 방지를 위해 `Preset with { Lorebooks = null }`로 직렬화함에 따라 복원 시 `_cachedLorebooks`가 비어 있게 되고, 백그라운드 JSON 직렬화 중 컬렉션 변경 예외 위험 존재.

### 2. 원인 (Root Cause)
- 시드 소모 판정에 일반 명사 키워드까지 무분별하게 포함되었고 단계별 시드 키가 분리되지 않았으며, 세션 스냅샷 내보내기/복원 시 프리셋 로어북 재결합 및 깊은 복사 격리가 누락됨.

### 3. 해결책 및 불변식 (Resolution & Invariants)
- **고유명사/핵심어 이원화 소모 판정 및 `BOND:` / `BOND_SECRET:` 키 분리 (`MemoryManagerService.cs`)**:
  - Pool A/B는 고유명사 매칭으로만 소모 판정하고, Pool C(유대 심화)는 동료 이름(`PrimaryKeywords`)과 유대/내면 본문 핵심 명사(`SecondaryKeywords`)가 **동시에** 등장할 때만 소모.
  - 호감도 30~59 구간(`BOND:{Name}`)과 60+ 구간(`BOND_SECRET:{Name}`)의 소모 키를 분리.
- **`ChapterContext.Clone()` 스냅샷 격리 및 `_cachedLorebooks` 복원 (`ChatViewModel.cs`)**:
  - 세션 복원 시 `savedSession with { Preset = sessionPreset }`으로 `_cachedLorebooks`를 즉시 초기화하고, `ExportSession`/`RestoreSession` 시 `ChapterContext.Clone()` 깊은 복사로 백그라운드 직렬화 중 컬렉션 변경 예외를 원천 차단.

---

## 2026-10-26: [Resolved] EntityNameResolver 및 NarrativeContract SSOT 도입 (상태창 인물 덮어쓰기 차단, 비인물 첫 어절 오염 제거, Turn 1 오프닝 스캔)

### 1. 현상 (Symptom)
- 비인물 복합 명칭(`"지하 유적 제단"`, `"그림자 사냥개 길드"`)의 첫 어절(`"지하"`, `"그림자"`)이 단독 키워드 및 `HOOK:` 시드 트리거로 자동 추출되어 일반 단어 등장만으로 미방문 장소 시드가 오탐 소모됨.
- 상태창 갱신 시 `entry.Keywords` 부분 문자열 포함(`cleanKey.Contains(kw)`) 검사로 인해, `"성녀"` 키워드를 가진 주연(`"세실리아 본 로젠버그"`)의 상태창 슬롯을 단역(`"성녀 호위대장"`)이 덮어쓰는 상태 오염 발생.
- Turn 1에 직전 `model` 응답(`InitialScenarioAckText`)만 스캔하여 `<initial_scenario>` 오프닝 본문 속 시작 장소·소품·세력 로어북이 1차 스캔에서 누락됨.

### 2. 원인 (Root Cause)
- 엔티티 별칭 파생, 동일 인물 판정, 시드 소모 경계 매칭, 서사 문장부호 계약이 여러 서비스에 서로 다른 휴리스틱으로 파편화되어 있었음.

### 3. 해결책 및 불변식 (Resolution & Invariants)
- **`EntityNameResolver` (`GRC/Helpers/EntityNameResolver.cs`) & `NarrativeContract` (`GRC/Services/Cognitive/NarrativeContract.cs`) SSOT 확립**:
  - `DeriveAliases`: 복합 명칭 첫 어절 자동 추출은 오직 `인물` 카테고리에서만 수행.
  - `ResolveCharacterKey`: 일반 `Keywords`(직함·소속) 부분 문자열 매칭을 전면 배제하고, 전체 엔트리명 일치 또는 고유 이름 첫 어절(`GetCharacterGivenName`) 완전 일치 시에만 매핑.
  - `DeriveHookPrimaryKeywords` & `ContainsEntityMention`: Pool B(`HOOK:`) 시드는 전체 엔트리명 등장 또는 2어절 이상 핵심 구분자 + 서사 훅 본문 핵심어 동시 등장 시에만 한국어 조사 경계 정규식으로 소모 판정.
  - `SanitizeScenario`: 로어북 고유명사를 감싼 강조용 낫표(`「...」`)를 결정론적으로 스트리핑(`「」`는 속마음/독백 전용).
- **Turn 1 `<initial_scenario>` 자동 스캔 연동 (`MemoryManagerService.cs`)**: `lastModelMessage.Text == NarrativeContract.InitialScenarioAckText`인 Turn 1에는 직전 `<initial_scenario>` 원문을 `loreScanQuery`에 결합하여 오프닝 장소·소품 로어북 즉각 인출 보장.
- **핵심 부정 제약 (Never use X)**:
  - **절대 비인물(`장소`, `아이템`, `세계관`) 복합 명칭의 첫 어절을 단독 키워드나 시드 소모 트리거로 추출하지 말 것.**
  - **절대 상태창 인물 Key 정규화(`ResolveCharacterKey`)에 일반 직함/소속 `Keywords` 부분 문자열 포함 검사(`.Contains`)를 사용하지 말 것.**

---

## 2026-10-26: [Resolved] `<world_lore>` 미해금 스포일러 조기 노출 및 유대 심화 완료형 문구에 의한 인과 건너뛰기(보상 선지급 환각) 차단

### 1. 현상 (Symptom)
- 친밀도 미달(30 미만) 상태에서도 인출된 인물 로어북의 심화 내면 동기와 유대 에피소드, 감정 개방 대사가 `<world_lore>`에 그대로 노출되어 초기 대사에 후반부 본심이 조기 반영됨.
- 인물 유대 심화 설정이 과거 완료형 결과 요약문으로 작성될 경우, 친밀도 30 달성 시 `<internal_chronicle_facts>`에 선행 과업 수행 이력이 없음에도 서사 AI가 중간 의뢰 과정을 건너뛰고 최종 보상을 즉시 선지급함.

### 2. 원인 (Root Cause)
- `<world_lore>` 주입 시 인물별 현재 관계도 수치(`CustomStats`)에 따른 단계별 노출 제어가 부재했으며, `<story_seeds>` 안내문에 미발생 과거 사건 날조 금지 가드 및 `[발단 계기]`와 `[완수 후 변화]`의 구조적 분리가 결여됨.

### 3. 해결책 및 불변식 (Resolution & Invariants)
- **호감도/신뢰도 3단계 스포일러 게이팅 (`MaskLorebooksForPrompt`, `RevealStage 1 | 2 | 3`)**: 인물별 최대 관계도(`GetCharacterMaxAffinity`) 및 시드 소모 이력에 따라 **Stage 1(30 미만: 표면 페르소나·일상 대사만 공개)**, **Stage 2(30~59: 유대 발단·완수 변화 추가 공개)**, **Stage 3(60 이상: 심층 내면·감정 대사 전면 공개)**로 차등 렌더링(`CurrentActiveLorebooks`는 원본 유지).
- **`<story_seeds>` 인과율 방어 및 발단-완수 분리**: `<story_seeds>` 헤더에 `<internal_chronicle_facts>`에 없는 과거 사건 날조 및 중간 과업 생략(보상 선지급) 절대 금지를 명시하고, 인물 로어북의 유대 발단(`BondTrigger`)과 완수 후 변화(`BondPayoff`)를 분리하여 반드시 '발단(의뢰·제안·단서 발견)'부터 개막하도록 강제.

---

## 2026-10-27: [Resolved] 에이전트 생성 세션 자산 결정론적 검증 게이트(`SessionAssetValidator`) 도입 및 PowerShell 5.1 CP949 인코딩 함정 방어

### 1. 현상 (Symptom)
- 외부 에이전트 워크플로우에서 생성된 세션 자산(`Preset.json`, `Session.json`, `FullHistory.json`)을 눈대중(Blind Audit)으로 판정하거나, 일회성 PowerShell 스크립트로 검수할 때 `NarrativeContract` 정규식이 복제되어 SSOT가 파손되고 Windows PowerShell 5.1에서 BOM 없는 UTF-8 `.ps1` 내 한글 리터럴이 CP949로 오독되어 파싱 에러 발생.

### 2. 원인 (Root Cause)
- 외부 에이전트 세션 생성 파이프라인에 앱의 실제 C# 모델·정규식·정제기 고정점을 그대로 실행하는 결정론적 테스트 러너 게이트가 없었음.

### 3. 해결책 및 불변식 (Resolution & Invariants)
- **관계도 키워드 SSOT 승격 및 C# 검증 게이트 확립 ([NarrativeContract.cs](../../GRC/GRC/Services/Cognitive/NarrativeContract.cs), [SessionAssetValidator.cs](../../GRC/tests/GRC.Tests/SessionAssetValidator.cs), [SessionAuditTests.cs](../../GRC/tests/GRC.Tests/SessionAuditTests.cs))**:
  - 관계도 판별 키워드(`호감`, `신뢰`, `유대`, `친밀`, `동조`, `애정`)를 `NarrativeContract.AffinityStatTerms` SSOT로 통합하고, `CoreSystemDirectivesHeader`(코어 규칙 1~3)는 `F04_SYSPROMPT` 바이트 일치 검증 대상이므로 임의 수정을 금지.
  - 앱 저장 세션 호환을 위해 `F08_LORE_META`는 `Embedding == null || Embedding.Length == 768`만 허용하며, `SessionAuditFactAttribute`를 통해 `GRC_AUDIT_SESSION_DIR` 미지정 시 일반 테스트에서 위장 통과 대신 명시적 `Skip` 처리.
- **핵심 부정 제약 (Never use X)**: **절대 세션 자산 검수를 위해 일회성 PowerShell 정규식 스크립트(`.ps1`)를 작성하지 말 것.** 반드시 `GRC_AUDIT_SESSION_DIR` 환경변수를 지정하고 `dotnet test --filter "Category=SessionAudit"` C# 검증 게이트를 실행할 것.

---

## 2026-10-27: [Resolved] 로어북 문자열 내장 DSL(정규식 구분자) 및 `LegacyLoreParser` 완전 폐기, 서브컬처 시드 엔진 Option B(상태 전이 + Status LLM 화이트리스트 소모) 전환

### 1. 현상 (Symptom)
- `LorebookEntry.Content` 단일 평문 문자열 내부에 다중 구분자(`/`, `->`, `""`) DSL을 내장하거나 `LegacyLoreParser` 폴백을 유지할 경우 불필요한 정규식 부채가 잔존함.
- 시작 거점(`Priority: 0` 장소)이나 초기 소모품이 초기 상태창(`Places`/`Items`)에만 있고 `ConsumedSeeds`에 기록되지 않아, 거점을 떠나거나 아이템을 소모한 직후 첫 막간(`isSceneResolved == true`)에 이미 방문한 시작 장소·아이템이 미방문 시드로 재주입됨(Pool B Head-of-Line Blocking).
- 단어 출현(`BondHookStopwords` 어절 매칭)으로 `BOND:`/`HOOK:` 시드 소모를 추측함에 따라 일상 대화에서 오탐 소모가 발생하거나, 호감도가 60 이상으로 점프했을 때 `CompanionSeedKeyToMark`가 `BOND:`와 `BOND_SECRET:`을 동시에 소모 처리하여 1단계 유대 에피소드가 증발함.
- 또한 `AddModelResponse`(타이핑 완료 시점)가 `_lastOfferedSeeds`를 즉시 비워버려 비동기로 도착하는 `UpdateContextStatus`와 경합 조건(Race Condition)이 발생함.

### 2. 원인 (Root Cause)
- 초기/현재 상태창(`Chars`, `Places`, `Items`)과 `ConsumedSeeds` 간의 결정론적 동기화가 부재했고, 시드 개시 판정을 상태창 LLM(`BuildStatusRequest`) 대신 어휘 휴리스틱에 의존함.

### 3. 해결책 및 불변식 (Resolution & Invariants)
- **강타입 구조화 스키마 단일화 및 Zero-Legacy 정리 ([LoreDetails.cs](../../GRC/GRC/Models/LoreDetails.cs), [LorebookEntry.cs](../../GRC/GRC/Models/LorebookEntry.cs), [LoreRenderer.cs](../../GRC/GRC/Services/Cognitive/LoreRenderer.cs))**:
  - `LegacyLoreParser.cs`, `LorebookEntry.LegacyContent`, `NarrativeContract` 내 평문 불릿 정규식 6종을 전면 삭제하고, `CharacterLoreDetails`(11필드)와 `WorldLoreDetails`(4필드)만을 유일한 SSOT로 확립.
- **상태 전이 자동 동기화 및 Status LLM 화이트리스트 시드 소모 ([MemoryManagerService.cs](../../GRC/GRC/Services/MemoryManagerService.cs), [StatusPayload.cs](../../GRC/GRC/Models/StatusPayload.cs))**:
  - `SyncStateBasedSeedConsumptionLocked`: `Chars`, `Places`, `Items`에 존재하는 엔티티의 `CHAR:{Name}` 및 `HOOK:{Name}`을 즉시 `ConsumedSeeds`에 기록하여 방문 거점·소비 아이템 재주입 원천 차단.
  - `_pendingStatusSeeds` & `StatusPayload.TriggeredSeeds`: `AddModelResponse`에서 비워지지 않는 `_pendingStatusSeeds` 버퍼를 유지하여 `BuildStatusRequest`에 `<offered_story_seeds>`를 주입하고, `UpdateContextStatus`에서 화이트리스트 검증을 통과한 시드만 소모 처리(UI 단독 스탯 수정 시 `IsSceneResolved` 초기화 방어 포함).
  - `Pool A` 공간 맥락 정렬(동일 `Priority` 내 현재 `Places` 연계 인물 우선), `Pool C` `BOND:` -> `BOND_SECRET:` 순차 해금 분리, `Retrieve` 이전 선행 `MaskLorebooksForPrompt` 적용, `RestoreSession` 시 `TotalTurnCount == 0` 기준 Turn 1 `ThinkingLevel.high` 앵커 보존, `EventLedgerExtractor` 정서적/관계적 이정표 추출 규칙 추가.
- **핵심 부정 제약 (Never use X)**:
  - **절대 로어북 파싱이나 시드 소모 판정에 평문 불릿 정규식이나 본문 단어 출현 추측(`BondHookStopwords`)을 사용하지 말 것.**
  - **절대 `AddModelResponse`에서 상태창 API(`UpdateContextStatus`)가 참조할 `_pendingStatusSeeds` 버퍼를 조기 초기화하지 말 것.**

---

> [!NOTE]
> **[정제 완료 기준선]** 2026-10-27 이전 상위 항목은 정제 완료됨. 신규 인시던트는 이 아래에 추가됩니다.



---

## 2026-10-28: [Resolved] VirtualizingStackPanel 픽셀 스크롤 내부 보정으로 인한 오토스크롤 오탐 비활성화 및 도중 멈춤 결함 해결

### 1. 현상 (Symptom)
- LLM 모델이 긴 답변을 실시간 스트리밍으로 출력할 때, 오토스크롤이 바닥을 잘 따라가다가 답변 중간에 갑자기 멈추는(Freeze) 현상 발생.
- 텍스트가 계속 아래로 출력되어도 스크롤이 더 이상 내려가지 않다가, 답변이 완전히 끝나 메시지 교체 시점이 되면 맨 아래로 다시 스크롤됨.
- 사용자가 마우스나 키보드를 전혀 건드리지 않은 상태에서도 재현됨.

### 2. 원인 (Root Cause)
1. **VirtualizingStackPanel 자체 오프셋 보정의 오탐**:
   - `ItemsControl`의 `VirtualizingStackPanel`(`ScrollUnit="Pixel"`)이 긴 스트리밍 텍스트 렌더링 중 레이아웃 패스에서 스스로 뷰포트 오프셋을 정확히 1행(`LineHeight=24px`)만큼 위로 보정(`dV = -24.00, dExt = 0.00, dVp = 0.00`).
   - 기존 `AutoScrollHelper.cs`는 `ExtentHeightChange == 0 && ViewportHeightChange == 0 && VerticalChange < 0` 조건을 "순수 사용자 조작(`isPureUserScroll`)"으로 간주하도록 작성되어 있었음.
   - 프레임워크 자체의 1행 재배치(`dV = -24.00`)를 사용자의 상향 스크롤 의도로 오인하여 `SetIsAutoScrollActive(sv, false)`를 호출함으로써 오토스크롤이 꺼짐.
2. **동결 상태 고착**:
   - `IsAutoScrollActive`가 `false`가 되면 이후 스트리밍 텍스트가 늘어나도(`dExt > 0`) 스크롤을 시도하지 않음.
   - 스트리밍 종료 시점에 `StreamingPanel`이 숨겨지고 `FinalText`가 반영되면서 컨텐츠 높이 급증(`dV = 64.00`)과 함께 바닥 판정(`isAtBottomNow`)이 만족되어 `IsAutoScrollActive`가 뒤늦게 `true`로 복구되었던 것.

### 3. 해결책 및 불변식 (Resolution & Invariants)
- **실제 물리적 사용자 입력 기반 상태 전이 확립 ([AutoScrollHelper.cs](../../GRC/GRC/Helpers/AutoScrollHelper.cs))**:
  - `e.VerticalChange < 0` 부호만으로 사용자 의도를 추정하는 휴리스틱을 전면 폐기.
  - 실제 사용자 입력(`PreviewMouseWheel`, 스크롤바 Thumb/Track 마우스 클릭, 방향키/PgUp/PgDn/Home/End)이 최근 500ms 이내에 발생했거나 스크롤바 Thumb을 드래그 중인 경우에만 `isUserDriven = true`로 판정하여 오토스크롤을 비활성화.
  - 사용자 입력이 수반되지 않은 프레임워크 자체 상향 오프셋 보정(`dV < 0`)은 무시하고, 오토스크롤 활성 상태라면 즉시 최하단으로 재정렬(`QueueScrollToEnd`).
- **디스패처 우선순위 동기화**:
  - `ScrollToEnd` 호출을 60FPS UI 렌더 타이머와 동일한 `DispatcherPriority.Render`로 정렬하여 스트리밍 텍스트와의 렌더링 지연(Lag) 제거.
- **핵심 부정 제약 (Never use X)**: **절대 `ExtentHeightChange == 0 && VerticalChange < 0`이라는 기하학적 조건만으로 사용자 상향 스크롤 여부를 단정하지 말 것.** 가상화 패널(`VirtualizingStackPanel`)의 픽셀 스크롤 환경에서는 레이아웃 재배치로 인한 내부 오프셋 조정이 빈번하므로, 반드시 실제 입력 디바이스 이벤트(마우스 캡처/휠/키 입력) 타임스탬프와 결합하여 검증할 것.
