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


