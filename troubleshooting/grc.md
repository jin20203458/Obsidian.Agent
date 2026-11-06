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

## 2026-10-14: [Resolved] Vertex AI 및 Gemini API 모델별 리전 엔드포인트 404 오류 방어 및 리전 라우팅 단일화

### 1. 현상 (Symptom)
* Google Cloud Vertex AI 연동 시 로컬 리전(`asia-northeast3`, `us-central1`)을 지정하여 최신 주력 추론 모델(`gemini-3.8-flash`, `gemini-4-argon`)을 호출하거나, 구형 `text-embedding-004` 엔드포인트를 호출할 경우 `404 Not Found` 예외 발생.

### 2. 원인 (Root Cause)
* Vertex AI 정책상 최신 주력 LLM은 전역 엔드포인트(`locations/global`)에서만 우선 서비스되며, TTS 음성 모델(`us-central1`) 및 통합 임베딩 모델(`gemini-embedding-001`, `asia-northeast3`)은 지원 리전과 출력 차원 규격이 상이함.

### 3. 해결책 및 불변식 (Resolution & Invariants)
* **역할별 엔드포인트 및 차원 고정 ([GeminiApiService.cs](../../GRC/GRC/Services/GeminiApiService.cs), [GeminiTtsService.cs](../../GRC/GRC/Services/GeminiTtsService.cs))**:
  * **LLM 추론**: 로컬 리전을 지정하지 말고 `locations/global` 단일 엔드포인트로 고정.
  * **TTS 음성 합성**: `GeminiTtsService`는 `us-central1` 리전으로 분리 고정.
  * **임베딩 생성**: Vertex AI 및 AI Studio 모두 `gemini-embedding-001` 단일 모델명, 서울 리전(`asia-northeast3`), `outputDimensionality: 768` (MRL)로 고정.
* **핵심 부정 제약 (Never use X)**: **절대 LLM 추론 호출에 로컬 리전을 하드코딩하거나 폐기된 `text-embedding-004`/`002` 모델명을 사용하지 말 것.**

---

## 2026-10-14: [Resolved] LlmJsonParser의 사전 잡담 괄호 오인식, 관용 역직렬화 및 스트리밍 절단 복구

### 1. 현상 (Symptom)
* LLM이 JSON 페이로드 앞뒤에 설명용 괄호(`Format {template}...`)를 출력하거나 따옴표 내부에 괄호가 포함될 때 단순 `rawText.IndexOf('{')` 방식이 오작동하여 역직렬화 실패(`null` 반환).
* 토큰 제한으로 스트리밍 도중 절단(Truncated)된 JSON 파싱 불가, 또는 경량 모델(`Flash-Lite`)이 `string` 필드에 숫자(`30`)/1원소 배열(`["장소"]`), `bool` 필드에 문자열(`"false"`), `Dictionary<string, string>`(`uiBadges`)에 평문 문자열(`"체력: 55/100, ..."`)을 반환할 때 역직렬화가 무음 실패함.

### 2. 원인 (Root Cause)
* 단순 `IndexOf('{')` 탐색이 문자열 리터럴/이스케이프(`\"`) 상태를 추적하지 못했으며, `System.Text.Json` 기본 컨버터가 LLM의 비표준 타입 변형 및 미완성 괄호 구조를 수용하지 못함.

### 3. 해결책 및 불변식 (Resolution & Invariants)
* **`LenientOptions` 및 관용 컨버터 정적 캐싱 ([LlmJsonParser.cs](../../GRC/GRC/Helpers/LlmJsonParser.cs), [ChapterContext.cs](../../GRC/GRC/Models/ChapterContext.cs))**:
  * `AllowReadingFromString`, `AllowTrailingCommas`, `ReadCommentHandling = Skip`, `JsonStringEnumConverter`와 함께 `LenientStringConverter`(`ReadAsPropertyName` 포함), `LenientBooleanConverter`, `LenientStringDictionaryConverter`(Key/Value `.Trim()`, 평문 `Key: Value` 및 배열 폴백)를 정적 등록.
  * `ChapterContext.ToStatusSnapshotString()`에서 `<previous_state>` 내 `uiBadges`, `characterConditionDesc`, `items`를 인라인 JSON 포맷으로 직렬화하여 LLM의 평문 모방 출력을 사전 예방.
* **리터럴 추적 및 `RepairTruncatedJson` 절단 복구**:
  * 마크다운 ```` ```json ```` 블록을 우선 파싱하고, 따옴표/이스케이프 상태를 추적하며 후보 스키마를 순차 검증.
  * 절단된 응답 감지 시 `SafeRollbackLength` 체크포인트 롤백 후 열린 따옴표와 여는 괄호의 짝(`]`, `}`)을 LIFO 역순으로 닫아 구조적 유효성 복원.
* **핵심 부정 제약 (Never use X)**: **절대 LLM 응답 파싱 시 단순 정규식이나 `rawText.IndexOf('{')`, 기본 `JsonSerializer.Deserialize`에 직접 의존하지 말 것.** 반드시 `LlmJsonParser` 전용 엔진을 사용할 것.

---

## 2026-10-14: [Resolved] 백그라운드 스레드 및 단위 테스트 환경 내 WPF STA 스레드 크래시·데드락 방어 (`Dispatcher` & `RunOnUIThread`)

### 1. 현상 (Symptom)
* 비동기 백그라운드 작업(`Task.Run`)에서 `MediaPlayer` 또는 `ObservableCollection`·윈도우를 조작할 때 `System.InvalidOperationException` / `NotSupportedException` (STA 스레드 위반) 발생.
* xUnit 전체 테스트 실행 시 앞선 테스트가 생성 후 종료한 임시 STA 스레드(`!Thread.IsAlive`)의 `Application.Current.Dispatcher`에 `Invoke`를 호출하여 무한 대기(Deadlock) 발생.
* 반대로 타임아웃(`500ms`) 초과 시 백그라운드 스레드에서 UI 액션을 직접 폴백 실행할 경우, 실제 WPF 앱 런타임에 `ObservableCollection.Clear()`에서 `NotSupportedException` 크래시가 발생하고 후속 `Session.json` 저장이 스킵됨.

### 2. 원인 (Root Cause)
* WPF 컨트롤과 바인딩 컬렉션은 STA 스레드 친화성을 지니며, 헤드리스 테스트 환경의 사망한 임시 스레드 디스패처와 실제 WPF 앱(`Application.Current is App`)의 실행 경계가 분리되지 않음.

### 3. 해결책 및 불변식 (Resolution & Invariants)
* **실제 WPF 앱과 단위 테스트 환경의 엄격한 분기 ([IDialogService.cs](../../GRC/GRC/Services/IDialogService.cs), [AudioService.cs](../../GRC/GRC/Services/AudioService.cs))**:
  * `Application.Current is not App || dispatcher == null || dispatcher.CheckAccess() || !dispatcher.Thread.IsAlive || dispatcher.HasShutdownStarted || dispatcher.HasShutdownFinished` 조건일 때(단위 테스트/헤드리스 또는 이미 UI 스레드인 경우)만 즉시 인라인 실행.
  * 실제 WPF 앱(`Application.Current is App`)의 백그라운드 스레드에서는 타임아웃 폴백 없이 반드시 `dispatcher.Invoke(action)`으로 STA 스레드 실행을 강제.
* **핵심 부정 제약 (Never use X)**:
  * **절대 `Application.Current.Dispatcher` 참조 시 스레드 생존성(`!dispatcher.Thread.IsAlive`)과 `Application.Current is not App` 검증 없이 `Invoke`를 직접 호출하지 말 것.**
  * **절대 실제 WPF 앱(`Application.Current is App`)에서 Dispatcher 타임아웃 후 백그라운드 스레드에서 UI 컬렉션 변경 액션을 직접 폴백 실행하지 말 것.**

---

## 2026-10-14: [Resolved] xUnit 테스트 러너 환경에서 WPF Application 인스턴스 중복 생성(AppDomain Singleton Trap) 해결

### 1. 현상 (Symptom)
* WPF 리소스 딕셔너리나 뷰를 검증하는 단위 테스트를 xUnit 병렬 러너로 실행할 때 `System.InvalidOperationException: 동일한 AppDomain에서 둘 이상의 Application 개체를 만들 수 없습니다.` 오류 발생.

### 2. 원인 (Root Cause)
* WPF `System.Windows.Application`은 단일 `AppDomain` 내에서 단 하나의 인스턴스만 허용하는 싱글톤 불변식을 지님.

### 3. 해결책 (Resolution)
* **`EnsureApplication` 정적 가드 패턴 도입**:
  ```csharp
  if (Application.Current != null) return;
  try { _ = new Application(); } catch (InvalidOperationException) { /* 병렬 스레드 경쟁 생성 안전 흡수 */ }
  ```

---

## 2026-10-14: [Resolved] Application.LoadComponent 사용 시 절대 Pack URI 지정으로 인한 ArgumentException 및 상대 URI 표준화

### 1. 현상 (Symptom)
* C# 코드-비하인드 또는 단위 테스트에서 `Application.LoadComponent(new Uri("pack://application:,,,/...", UriKind.Absolute))` 호출 시 `System.ArgumentException: 절대 URI를 사용할 수 없습니다.` 발생.

### 2. 원인 (Root Cause)
* WPF `Application.LoadComponent` API는 BAML 리소스 로딩 규격상 반드시 어셈블리 상대 URI(`UriKind.Relative`)를 요구함 (XAML 마크업 내 `ResourceDictionary.Source`와 상이).

### 3. 해결책 (Resolution)
* **상대 URI 표준화 및 `PackUriHelper` 스킴 사전 등록**:
  ```csharp
  _ = PackUriHelper.UriSchemePack; // 헤드리스 테스트 환경 pack 스킴 파서 등록
  var dict = (ResourceDictionary)Application.LoadComponent(new Uri("/GRC;component/Themes/PrimitiveTokens.xaml", UriKind.Relative));
  ```
* **핵심 부정 제약 (Never use X)**: **절대 C# `Application.LoadComponent` 호출 시 절대 Pack URI(`pack://...`)를 전달하지 말 것.**

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
* **다단계 안전 Fallback ([CognitivePromptAssembler.cs](../../GRC/GRC/Services/Cognitive/CognitivePromptAssembler.cs))**: `string.IsNullOrWhiteSpace` 감지 시 `PlotCatalyst` 지침을 우선 채택하고, 부재 시 `"(서사를 계속 진행하십시오.)"` 대체 텍스트를 강제 주입.
* **핵심 부정 제약 (Never use X)**: **절대 검증되지 않은 빈 유저 입력을 검사 없이 `new Part(text)`에 직접 전달하지 말 것.**

---

## 2026-10-17: [Resolved] ChatViewModel 백그라운드 비동기 작업(임베딩 백필·상태창 콜백) 중 세션 전환 오염 방어 및 디스크 I/O 스레드 분리

### 1. 현상 (Symptom)
* 백그라운드 비동기 작업(구버전 로어북 임베딩 백필 또는 상태창 갱신 API) 실행 도중 사용자가 로어북을 수정하거나 세션을 전환(`GoBack`/다른 세션 로드)할 경우, 이전 세션의 결과가 현재 세션 메모리와 디스크(`Session.json`)에 덮어써지는 파일 오염 및 신규 로어북 유실 발생.
* `onStatusUpdated` 콜백 전체를 `Dispatcher.InvokeAsync`로 감쌀 경우 백그라운드 디스크 I/O(`SaveSessionAsync`)까지 UI 스레드에서 경합하며 헤드리스 환경에서 저장이 누락됨.

### 2. 원인 (Root Cause)
* 비동기 작업 진입 시점의 세션 식별자 스냅샷 검증이 부재했으며, 메모리/디스크 I/O와 UI 컬렉션 갱신의 스레드 경계가 분리되지 않음.

### 3. 해결책 및 불변식 (Resolution & Invariants)
* **세션 수명주기 스냅샷 앵커링 및 비파괴 병합 ([ChatViewModel.cs](../../GRC/GRC/ViewModels/ChatViewModel.cs))**:
  * 작업 시작 시 `string targetFileName = _currentFileName;`을 캡처하고, 비동기 완료 후 `!_isSessionActive || _currentFileName != targetFileName`이면 즉시 폐기.
  * 로어북 임베딩 백필은 전체 리스트 덮어쓰기 대신 `(Name, Content)` 키 기반 선택적 병합(Selective Merge)으로 유저 수정본을 보존.
* **백그라운드 디스크 I/O와 UI 디스패치 분리**:
  * 스레드 안전한 `UpdateContextStatus`와 비동기 디스크 I/O `SaveSessionAsync`는 백그라운드 태스크에서 직접 실행하고, UI 컬렉션 조작만 `UpdateStatusUI` 내부의 `_dialogService.RunOnUIThread`에 위임.
* **핵심 부정 제약 (Never use X)**: **절대 비동기 디스크 I/O(`SaveSessionAsync`)를 UI Dispatcher 스레드 안으로 감싸거나, 세션 식별자 스냅샷(`targetFileName`) 검증 없이 백그라운드 콜백 결과를 반영하지 말 것.**

---

## 2026-10-17: [Resolved] dotnet test 백그라운드 실행 시 솔루션 단위 평가로 인한 MSBuild/VSTest I/O 파이프 블로킹 해결 및 테스트 프로젝트 직접 타겟팅 규격 확립

### 1. 현상 (Symptom)
* 백그라운드 태스크로 `dotnet test GRC.sln --no-build` 실행 시 테스트 실행 자체는 완료되었으나 프로세스가 종료되지 않고 셸이 무한 대기(RUNNING/Deadlock)에 머무는 결함 발생.

### 2. 원인 (Root Cause)
* `dotnet test` CLI 규격상 `-nodeReuse:false`가 솔루션 그래프를 평가하는 MSBuild/VSTest 자식 작업자 노드에 완벽히 상속되지 않아, 자식 노드가 백그라운드 셸의 표준 출력(stdout) 파이프 핸들을 점유한 채 종료 신호(EOF)를 블로킹함.

### 3. 해결책 및 불변식 (Resolution & Invariants)
* **테스트 프로젝트(`.csproj`) 직접 타겟팅 규격 확립**:
  ```powershell
  dotnet test tests/GRC.Tests/GRC.Tests.csproj --filter "Category!=Integration" --no-build -v n
  ```
* **핵심 부정 제약 (Never use X)**: **백그라운드 터미널에서 `dotnet test` 실행 시 절대 솔루션 파일(`.sln`)을 타겟으로 지정하지 말 것.** 반드시 `tests/GRC.Tests/GRC.Tests.csproj`를 직접 지정할 것.

---

## 2026-10-17: [Resolved] 메타 지시문(MetaDirective) 합성에 따른 768차원 임베딩 쿼리 벡터 오염(Vector Dilution) 해결

### 1. 현상 (Symptom)
* 시스템 이벤트, 챕터 전환, 자동 이어쓰기 시 `[시스템 개입: ...]` 메타 텍스트가 `userMessage.Text`에 직접 병합되어 `GetEmbeddingAsync`로 전달됨에 따라, 순수 유저 의도 검색이어야 할 768차원 쿼리 임베딩 벡터가 메타 단어로 희석(Vector Dilution)되어 RAG 검색 정확도가 저하됨.

### 2. 원인 (Root Cause)
* `MemoryManagerService.cs`에서 `userMessage.Text`와 `metaDirective`를 단순 문자열 결합하여 임베딩 생성과 프롬프트 조립에 공용 사용함.

### 3. 해결책 및 불변식 (Resolution & Invariants)
* **메타 지시문 독립 파이프라인 분리 ([MemoryManagerService.cs](../../GRC/GRC/Services/MemoryManagerService.cs), [CognitivePromptAssembler.cs](../../GRC/GRC/Services/Cognitive/CognitivePromptAssembler.cs))**:
  * `CognitiveContext.MetaDirective` 프로퍼티로 메타 지시문을 분리 전달하고, 768차원 임베딩 및 `loreScanQuery`는 순수 `userMessage.Text`로만 생성.
* **핵심 부정 제약 (Never use X)**: **절대 시스템 메타 지시문을 유저 발화 원문에 병합하여 임베딩을 생성하지 말 것(Vector Dilution 방지).**

---

## 2026-10-17: [Resolved] Gemini Thinking 모델 MaxOutputTokens 축소 시 JSON 절단 방어 및 TTS 프롬프트 순수 구어체 자연어 보존

### 1. 현상 (Symptom)
* Gemini Thinking 아키텍처에서 `maxOutputTokens`를 1,024~2,048(일반 구조화 작업) 또는 8,192(`ThinkingLevel.high` 장문 설계 작업)로 작게 제한할 경우, 내부 추론 토큰(Thought Tokens)이 출력 예산을 잠식하여 가시적 JSON 페이로드가 중간에 절단됨.
* 음성 합성 모델(`gemini-3.1-flash-tts-preview`) 및 사전 번역기에 XML 태그나 불릿포인트를 주입하면 운율(Prosody) 왜곡, 기계음 및 태그 음독 결함 발생.

### 2. 원인 (Root Cause)
* Thinking 토큰과 가시 출력 토큰이 합산 차감되는 예산 구조를 고려하지 않고 `maxOutputTokens`를 축소했으며, 오디오 합성 프롬프트에 마크다운/XML 구조화 기호를 주입함.

### 3. 해결책 및 불변식 (Resolution & Invariants)
* **Thinking 토큰 예산 상한 보존 ([SessionArchitectService.cs](../../GRC/GRC/Services/SessionArchitectService.cs))**:
  * 지연/비용 제어는 `ThinkingConfig`(`ThinkingLevel`)로만 통제하며, `MaxOutputTokens`는 일반 추론 최소 `8,192`, `SessionArchitectService` 장문 JSON 생성은 `ArchitectMaxOutputTokens = 32,768` 상한을 유지하고 단계별 검증은 LLM 재호출 대신 100% C# 정규식·파서(`ReviewStepContentAsync`)로 수행.
* **TTS 프롬프트 100% 순수 구어체 보존**: 사전 번역 및 음성 합성 프롬프트(`dynamicPrompt`)는 일체의 XML 태그·불릿 기호 없이 순수 자연어 구어체 지시문으로만 구성.
* **핵심 부정 제약 (Never use X)**: **절대 Thinking이 활성화된 모델의 `MaxOutputTokens`를 1,024~2,048 수준으로 임의 축소하지 말 것.**

---

## 2026-10-18: [Resolved] 동적 상태창 갱신 파이프라인의 Nullable DTO 빈 배열 소거 및 비동기 체이닝 확립

### 1. 현상 (Symptom)
* 인벤토리 마지막 아이템 소비 시 LLM이 `"items": []`를 반환해도 `StatusPayload` 기본값(`new()`)과 `Count > 0` 조건으로 인해 빈 배열 덮어쓰기가 무시되어 소모품이 영구 잔존함.
* `onStatusUpdated` 콜백이 동기 `Action`으로 선언되어 연속 턴 입력 시 이전 턴 상태창 반영 및 저장 완료 전에 다음 턴 프롬프트가 조립되는 타이밍 누수 발생.

### 2. 원인 (Root Cause)
* `StatusPayload` DTO 컬렉션이 `new()`로 강제 초기화되어 미제공(`null`, 부분 갱신)과 빈 배열(`[]`, 전체 소거)을 구분할 수 없었으며, 비동기 대기 체인이 단절됨.

### 3. 해결책 및 불변식 (Resolution & Invariants)
* **Nullable 컬렉션 전환 및 `Func<StatusPayload, Task>?` 비동기 체이닝 ([StatusPayload.cs](../../GRC/GRC/Models/StatusPayload.cs), [ChatWorkflowService.cs](../../GRC/GRC/Services/ChatWorkflowService.cs))**:
  * `CustomStats`, `Chars`, `Items`, `Places`를 nullable(`? = null`)로 선언하여 `payload.Items != null`일 때 빈 배열(`[]`) 소거를 정상 반영하고, `await onStatusUpdated(payload)`로 상태 갱신~디스크 저장까지 단일 파이프라인으로 대기.
* **핵심 부정 제약 (Never use X)**: **절대 동적 상태창 DTO(`StatusPayload`)의 컬렉션 프로퍼티를 `new()`로 사전 초기화하지 말 것.**

---

## 2026-10-25: [Resolved] Session Architect 다국어 토크나이저 글리치 문자 혼입 제거 및 2단계 스탯 Prefix 바인딩·게이지 클램핑

### 1. 현상 (Symptom)
* 다국어 토크나이저 글리치로 한국어 산문 중간에 데바나가리·아랍·태국·키릴 문자(`सिर` 등)가 간헐 방출됨.
* 인물명이 풀네임(`아리아 폰 블랑셰`)이고 스탯명이 첫 어절 약칭(`아리아 호감도`)일 때 `<entity_relationship>` 바인딩에 실패하고 `<current_state>`에 중복 노출되며, `"120/100"`, `"-15/100"` 등 범위 초과 게이지가 무검증 반영됨.

### 2. 원인 (Root Cause)
* 출력 정제기에 비한국어 글리치 유니코드 블록 제거 정규식이 없었으며, `MemoryManagerService`가 단일 키 완전 일치만 검사하고 게이지 클램핑을 수행하지 않음.

### 3. 해결책 및 불변식 (Resolution & Invariants)
* **비정상 유니코드 스크립트 일괄 스트리핑 ([SessionArchitectService.cs](../../GRC/GRC/Services/SessionArchitectService.cs))**: `StripForeignGlitchScripts`를 통해 데바나가리·아랍·태국·키릴·히브리 블록(`[\u0400-\u06FF\u0900-\u0E7F]+`)을 모든 `Sanitize*` 파이프라인에서 제거.
* **2단계 스탯 Prefix 매칭 및 게이지 클램핑 ([MemoryManagerService.cs](../../GRC/GRC/Services/MemoryManagerService.cs))**: Exact Match 실패 시 풀네임(`charName + " "`) 및 고유 첫 어절(`firstToken + " "`) 접두사로 `<entity_relationship>`에 바인딩 후 `remainingCustomStats`에서 제외하며, `ClampGaugeValue`로 범위형 게이지를 `0~max`로 클램핑.

---

## 2026-10-25: [Resolved] ExportSession/RestoreSession 시 _cachedLorebooks 복원 누락 해결 및 ChapterContext.Clone() 백그라운드 직렬화 동시성 격리

### 1. 현상 (Symptom)
* `ExportSession`이 디스크 중복 방지를 위해 `Preset with { Lorebooks = null }`로 직렬화함에 따라 세션 복원 시 `_cachedLorebooks`가 비어 있게 되고, 백그라운드 JSON 직렬화 도중 컬렉션 변경 시 `InvalidOperationException` 경합 위험이 발생함.

### 2. 원인 (Root Cause)
* 세션 스냅샷 내보내기/복원 시 프리셋 로어북 재결합 및 상태 컨테이너(`ChapterContext`)의 깊은 복사 격리가 누락됨.

### 3. 해결책 및 불변식 (Resolution & Invariants)
* **`ChapterContext.Clone()` 스냅샷 격리 및 로어북 재결합 ([ChatViewModel.cs](../../GRC/GRC/ViewModels/ChatViewModel.cs), [ChapterContext.cs](../../GRC/GRC/Models/ChapterContext.cs))**:
  * 세션 복원 시 `savedSession with { Preset = sessionPreset }`으로 `_cachedLorebooks`를 즉시 복원하고, `ExportSession`/`RestoreSession` 및 턴 시작 스냅샷 생성 시 반드시 `ChapterContext.Clone()` 단일 깊은 복사 경로를 사용하여 백그라운드 직렬화 중 컬렉션 변경 예외를 원천 차단.

---

## 2026-10-26: [Resolved] EntityNameResolver 및 NarrativeContract SSOT 도입 (상태창 인물 덮어쓰기 차단, 비인물 첫 어절 오탐 제거, Turn 1 오프닝 스캔)

### 1. 현상 (Symptom)
* 비인물 복합 명칭(`"지하 유적 제단"`, `"그림자 사냥개 길드"`)의 첫 어절(`"지하"`, `"그림자"`)이 단독 키워드로 자동 추출되어 일반 명사 등장만으로 오탐 인출됨.
* 상태창 갱신 시 `entry.Keywords` 부분 문자열 포함(`cleanKey.Contains(kw)`) 검사로 인해, `"성녀"` 키워드를 가진 주연(`"세실리아 본 로젠버그"`)의 상태창 슬롯을 단역(`"성녀 호위대장"`)이 덮어쓰는 상태 오염 발생.
* Turn 1에 직전 `model` 응답(`InitialScenarioAckText`)만 스캔하여 `<initial_scenario>` 오프닝 본문 속 시작 장소·소품·세력 로어북이 1차 스캔에서 누락됨.

### 2. 원인 (Root Cause)
* 엔티티 별칭 파생과 인물 키 정규화 규칙이 파편화되어 일반 직함/소속 키워드에 대해 `.Contains` 부분 매칭을 허용함.

### 3. 해결책 및 불변식 (Resolution & Invariants)
* **`EntityNameResolver` SSOT 및 Turn 1 오프닝 스캔 ([EntityNameResolver.cs](../../GRC/GRC/Helpers/EntityNameResolver.cs), [MemoryManagerService.cs](../../GRC/GRC/Services/MemoryManagerService.cs))**:
  * `DeriveAliases`: 복합 명칭 첫 어절 자동 추출은 오직 `인물` 카테고리에서만 수행.
  * `ResolveCharacterKey`: 일반 `Keywords`(직함·소속) 부분 문자열 매칭을 전면 배제하고, 전체 엔트리명 일치 또는 고유 이름 첫 어절(`GetCharacterGivenName`) 완전 일치 시에만 매핑.
  * Turn 1(`lastModelMessage.Text == NarrativeContract.InitialScenarioAckText`)에는 직전 `<initial_scenario>` 원문을 `loreScanQuery`에 결합하여 오프닝 로어북 즉각 인출 보장.
* **핵심 부정 제약 (Never use X)**:
  * **절대 비인물(`장소`, `아이템`, `세계관`) 복합 명칭의 첫 어절을 단독 키워드로 추출하지 말 것.**
  * **절대 상태창 인물 Key 정규화(`ResolveCharacterKey`)에 일반 직함/소속 `Keywords` 부분 문자열 포함 검사(`.Contains`)를 사용하지 말 것.**

---

## 2026-10-27: [Resolved] 에이전트 생성 세션 자산 결정론적 검증 게이트(SessionAssetValidator) 도입 및 PowerShell 5.1 CP949 인코딩 함정 방어

### 1. 현상 (Symptom)
* 외부 에이전트 워크플로우에서 생성된 세션 자산(`Preset.json`, `Session.json`, `FullHistory.json`)을 일회성 PowerShell 스크립트로 검수할 때 `NarrativeContract` 정규식이 복제되어 SSOT가 파손되고, Windows PowerShell 5.1에서 BOM 없는 UTF-8 `.ps1` 내 한글 리터럴이 CP949로 오독되어 파싱 에러가 발생함.

### 2. 원인 (Root Cause)
* 외부 에이전트 세션 생성 파이프라인에 앱의 실제 C# 모델·정규식·정제기 고정점을 그대로 실행하는 결정론적 테스트 러너 게이트가 없었음.

### 3. 해결책 및 불변식 (Resolution & Invariants)
* **C# 결정론적 검증 게이트 확립 ([NarrativeContract.cs](../../GRC/GRC/Services/Cognitive/NarrativeContract.cs), [SessionAssetValidator.cs](../../GRC/tests/GRC.Tests/SessionAssetValidator.cs), [SessionAuditTests.cs](../../GRC/tests/GRC.Tests/SessionAuditTests.cs))**:
  * 관계도 판별 키워드(`NarrativeContract.AffinityStatTerms`)와 `CoreSystemDirectivesHeader`(코어 규칙 1~3, `F04_SYSPROMPT` 바이트 일치 검증 대상)를 SSOT로 통합(구버전 세션은 `CompileHybridSystemPrompt` 재컴파일 시 신규 헤더로 자동 교체).
  * `SessionAuditFactAttribute`를 통해 `GRC_AUDIT_SESSION_DIR` 미지정 시 명시적 `Skip` 처리.
* **핵심 부정 제약 (Never use X)**: **절대 세션 자산 검수를 위해 일회성 PowerShell 정규식 스크립트(`.ps1`)를 작성하지 말 것.** 반드시 `GRC_AUDIT_SESSION_DIR` 환경변수를 지정하고 `dotnet test --filter "Category=SessionAudit"` C# 검증 게이트를 실행할 것.

---

## 2026-11-02: [Resolved] 서브컬처 시드 엔진 상태 전이·지문 조우 소모, 로어북 인출 우선순위 및 상태창 공백 중복 키 정화

### 1. 현상 (Symptom)
* **시드 조기 소모 및 경합**: `AddModelResponse`가 `_lastOfferedSeeds`를 즉시 비워 비동기 `UpdateContextStatus`와 경합 조건(Race Condition)이 발생하거나, 큰따옴표 대사(`"..."`) 내 3인칭 소문·차기 행선지 언급만으로 미대면 인물(`CHAR:`)·미방문 장소(`HOOK:`) 시드가 소모되고 `StatusAI`가 미대면 인물을 상태창에 선등록함.
* **익명 조우 후 지연 통성명 누락**: 첫 조우 턴에 익명 칭호로 등장한 뒤 다음 턴에 통성명하여 `Chars`에 등재된 인물의 `CHAR:` 시드가 미소모로 남음.
* **로어북 예산 역전 및 프롬프트 캐시 파손**: 활성 인물/장소 보충 로어북이 Sparse 직접 매칭을 밀어내거나 미등장 인물이 Dense 유사도로 유입되며, `SystemInstruction`에 동적 태그명이 포함될 경우 정적 캐시 불변식이 파손됨.
* **공백 변종 중복 배지**: `StatusAI`가 쉼표 뒤 공백이 포함된 키(`" 벨벳 호감도"`)를 반환할 때 기존 `"벨벳 호감도"`와 별개 키로 공존하여 UI 배지가 중복 생성됨.

### 2. 원인 (Root Cause)
* 타이핑 완료 시점(`AddModelResponse`)과 비동기 상태창 콜백(`UpdateContextStatus`) 간 시드 버퍼 수명주기가 분리되지 않았고, 지문(실제 조우)과 대사(소문/행선지 언급)의 소모 경계 및 딕셔너리 키 공백 정규화가 부재했음.

### 3. 해결책 및 불변식 (Resolution & Invariants)
* **시드 버퍼 분리 및 지문 조우·상태 전이 기반 소모 ([MemoryManagerService.cs](../../GRC/GRC/Services/MemoryManagerService.cs), [LoreDetails.cs](../../GRC/GRC/Models/LoreDetails.cs))**:
  * `_pendingStatusSeeds` 버퍼를 `AddModelResponse`에서 비우지 않고 `UpdateContextStatus`까지 유지.
  * `CHAR:` 시드는 지문(`narrationOnly`) 등장, 대사(`quotesOnly`) 내 1인칭 자기소개(`IsCharacterEncounteredInNarrative`), 또는 `Chars` 상태 등재(`SyncStateBasedSeedConsumptionLocked`) 시에만 소모하며, 미조우 로어북 인물의 `Chars`/`CustomStats` 등록을 차단.
  * `장소` 카테고리 `HOOK:` 시드는 `StatusAI.TriggeredSeeds`에서 제외하고 오직 실제 방문(`Places`/`currentPlace` 상태 전이)으로만 소모.
  * `NormalizeCustomStatsDictionary`로 `RestoreSession` 및 `UpdateContextStatus`에서 공백 변종 중복 키를 정화하고 절단된 게이지 분모를 복원.
* **로어북 인출 계층 우선순위 고정 ([HybridLorebookRetriever.cs](../../GRC/GRC/Services/Cognitive/HybridLorebookRetriever.cs))**:
  * 예산 배정 순서를 **시드 고정 로어북(3-H) ➔ Sparse 직접 매칭 ➔ 현재 장소·활성 인물 보충 ➔ Sticky(Sparse 전용 2턴) ➔ Dense(최대 3개, 미등장 인물 제외)** 순으로 고정.
* **핵심 부정 제약 (Never use X)**:
  * **절대 `AddModelResponse`에서 비동기 `UpdateContextStatus`가 참조할 `_pendingStatusSeeds` 버퍼를 조기 초기화하거나, 평문 불릿 정규식(`LegacyLoreParser`)/단어 출현 추측(`BondHookStopwords`)을 사용하지 말 것.**
  * **절대 큰따옴표 대사 내부의 3인칭 이름 언급만으로 `CHAR:` 시드를 소모하거나, `장소` 카테고리 `HOOK:` 시드를 `StatusAI.TriggeredSeeds`로 소모하지 말 것.**
  * **절대 `SystemInstruction`(`FormatRuleBlock` 포함)에 `<world_lore>` 같은 동적 태그명을 넣거나, 활성 인물·장소 보충 로어북이 Sparse 직접 매칭보다 앞서도록 순서를 변경하지 말 것.**

---

## 2026-11-04: [Resolved] IllustrationPromptService의 responseSchema 대소문자(CamelCase vs PascalCase) 불일치로 인한 API 400 실패 해결

### 1. 현상 (Symptom)
* `IllustrationStudioWindow`에서 로어북 인물 태그 추출(`ExtractCharacterTagsAsync`) 또는 장면 태그 추출(`ExtractSceneTagsAsync`) 실행 시 Gemini API가 `400 INVALID_ARGUMENT` 오류를 반환하며 실패함.

### 2. 원인 (Root Cause)
* [`GeminiApiService.cs`](../../GRC/GRC/Services/GeminiApiService.cs)의 `_jsonOptions`에 `PropertyNamingPolicy = JsonNamingPolicy.CamelCase`가 설정되어 있어 익명 객체로 선언한 `responseSchema.properties`의 키(`AppearanceTags`)는 `"appearanceTags"`(카멜케이스)로 직렬화되는 반면, 문자열 배열 값인 `required = new[] { "AppearanceTags", ... }`의 원소는 파스칼케이스 그대로 직렬화되어 스키마 불일치가 발생함.

### 3. 해결책 및 불변식 (Resolution & Invariants)
* **`responseSchema` 카멜케이스 키 통일 ([IllustrationPromptService.cs](../../GRC/GRC/Services/IllustrationPromptService.cs))**: `responseSchema` 내 `properties`와 `required` 배열 원소를 모두 소문자 카멜케이스(`appearanceTags`, `defaultOutfitTags`, `compositionTags`, `outfitAndExposureTags`, `poseAndVibeTags`, `backgroundTags`)로 통일.
* **핵심 부정 제약 (Never use X)**: **절대 `GeminiApiService`에 전달하는 `responseSchema`의 `properties`나 `required` 배열에 파스칼케이스(`PascalCase`) 키를 사용하지 말 것.**

---

## 2026-11-04: [Resolved] VirtualizingStackPanel 내부 오프셋 보정·Margin 애니메이션·뷰포트 축소에 의한 오토스크롤 풀림 및 1틱 탈출 불가 트랩 해결

### 1. 현상 (Symptom)
* 16ms 단위 토큰 스트리밍 중 마우스 휠을 위로 1틱(15px) 굴려도 24px 허용오차를 벗어나지 못해 최하단으로 재감금되거나, 컨텐츠 축소(`extentHeightChange < 0`) 시 상단 스크롤이 하이재킹됨.
* 반대로 사용자가 입력 장치를 건드리지 않았음에도 장문 스트리밍 도중 `VirtualizingStackPanel`(`ScrollUnit="Pixel"`)이 1행(`dV = -24.00px`) 자체 오프셋 보정을 수행할 때 오토스크롤이 풀려버리거나, 멀티라인 입력창 확장(`ViewportHeightChange < 0`) 시 마지막 대사가 가려지고 상향 스크롤 시 0.3초간 역보정 반동이 발생함.

### 2. 원인 (Root Cause)
1. **무상태 수식 및 `ScrollChanged` 내 사용자 의도 추측의 한계**: 무상태 바닥 판정식(`CalculateIsAtBottom`)만으로는 1틱(15px) 탈출이 불가능하며, `ScrollChanged` 내부에서 `e.VerticalChange < 0`이나 `500ms` 타이머(`LastUserInputTick`)로 상향 스크롤을 추측하면 프레임워크 자체 레이아웃 보정(`dV < 0`)을 사용자 조작으로 오판하여 오토스크롤을 해제함.
2. **`ChatMessageTemplate`의 `Margin` 레이아웃 애니메이션 ([ModernStyles.xaml](../../GRC/GRC/Themes/ModernStyles.xaml))**: `Loaded` 트리거가 레이아웃 측정 속성인 `Margin`을 0.3초간 애니메이션하여 매 프레임 `VirtualizingStackPanel.MeasureOverride`를 강제 호출하며 `ExtentHeight`와 `VerticalOffset`을 흔듦.

### 3. 해결책 및 불변식 (Resolution & Invariants)
* **물리적 상향 입력 전용 상태 전이 및 기하학 판정 ([AutoScrollHelper.cs](../../GRC/GRC/Helpers/AutoScrollHelper.cs), [ChatView.xaml.cs](../../GRC/GRC/Views/ChatView.xaml.cs))**:
  * `CalculateIsAtBottom`: `referenceHeight = extentHeightChange > 0 ? scrollableHeight - extentHeightChange : scrollableHeight` 수식으로 컨텐츠 축소 왜곡을 방지.
  * `IsAutoScrollActive = false` 해제는 오직 물리적 상향 입력 이벤트(`PreviewMouseWheel e.Delta > 0`, `ScrollBar.ScrollEvent`의 `SmallDecrement`/`LargeDecrement`/`First` 및 비하단 `ThumbTrack`, `PreviewKeyDown`의 `Up`/`PageUp`/`Home`)에서만 수행하고, `ScrollChanged` 내부에서는 절대 해제하지 않음.
  * `IsAutoScrollActive == true`일 때는 `ExtentHeightChange != 0`, `ViewportHeightChange != 0`, `e.VerticalChange < 0` 및 `ChatHistory.CollectionChanged` 모두 즉시 `ScrollToBottom()`(`DispatcherPriority.Render`)으로 보정하며, 상향 물리 입력 직후 `150ms`(`UpwardBounceCooldownMs`) 이내의 레이아웃 반동 재결합을 차단.
* **GPU `RenderTransform.(TranslateTransform.Y)` 교체 ([ModernStyles.xaml](../../GRC/GRC/Themes/ModernStyles.xaml))**: `MessageContainer`의 `Margin="15,8"`을 고정하고 `RenderTransform.(TranslateTransform.Y)`(`From="17" To="0"`)로 전환하여 레이아웃 재측정 비용과 오프셋 흔들림을 제거.
* **핵심 부정 제약 (Never use X)**:
  * **절대 무상태 수학식(`CalculateIsAtBottom`)만 단독으로 사용하거나, `ScrollViewer.ScrollChanged` 내부에서 `e.VerticalChange < 0`·타이머 휴리스틱으로 사용자 상향 스크롤을 추측해 `IsAutoScrollActive = false`로 전이하지 말 것.**
  * **절대 가상화 리스트(`VirtualizingStackPanel`) 아이템 템플릿의 `Loaded` 애니메이션에 `Margin`·`Height`·`Padding` 등 레이아웃 측정(Measure) 속성을 애니메이션하지 말 것.** 반드시 `RenderTransform`이나 `Opacity`만 사용할 것.

---

## 2026-11-04: [Resolved] 장문 세션 로딩 시 비가시 세그먼트 마크다운 4중 파싱, 커스텀 템플릿 바인딩 누락 및 64비트 WndProc 오버플로우 해결

### 1. 현상 (Symptom)
* 수백 턴 분량의 장문 세션을 로드할 때 초기 렌더링 부하로 UI 스레드가 일시 정지(Freeze)하거나 64비트 다중 모니터 환경 `WM_NCHITTEST` 처리 중 `OverflowException` 크래시가 발생함.

### 2. 원인 (Root Cause)
1. **비가시(`Collapsed`) `TextBlock` 4중 마크다운 파싱 ([ModernStyles.xaml](../../GRC/GRC/Themes/ModernStyles.xaml))**: 말풍선 내부의 세그먼트 `ItemsControl`은 비가상화 상태로 실현된 메시지의 모든 세그먼트를 생성하는데, 세그먼트당 4개의 `TextBlock` 중 1개만 표시(`Visible`)됨에도 `Collapsed` 상태인 나머지 3개까지 기본 속성으로 `SimpleMarkdownHelper.Text="{Binding Text}"`를 즉시 평가하여 정규식 파싱 및 `Inline` 객체 생성 비용이 4배로 증폭됨.
2. **커스텀 `ControlTemplate` 내 `ScrollViewer` `TemplateBinding` 누락 ([ChatView.xaml](../../GRC/GRC/Views/ChatView.xaml))**: `ChatHistoryItemsControl`에 `ScrollViewer.CanContentScroll="True"`를 선언했으나 내부 `<ControlTemplate>`의 `<ScrollViewer x:Name="ChatScrollViewer">`에 `CanContentScroll="{TemplateBinding ScrollViewer.CanContentScroll}"`가 누락됨. `ScrollViewer.CanContentScrollProperty`는 상속되지 않으므로(`Inherits = False`) 레이아웃 패스(`HookupScrollingComponents`) 연결 전까지 초기 기본값(`false`) 불일치가 발생하며 표준 템플릿 계약을 위반함.
3. **64비트 `WndProc` 포인터 변환 오버플로우 ([MainWindow.xaml.cs](../../GRC/GRC/Views/MainWindow.xaml.cs))**: `WM_NCHITTEST`에서 `lParam.ToInt32()` 호출 시 음수/확장 좌표에서 `OverflowException` 발생.

### 3. 해결책 및 불변식 (Resolution & Invariants)
* **조건부 마크다운 파싱 및 템플릿 바인딩 명시 ([ModernStyles.xaml](../../GRC/GRC/Themes/ModernStyles.xaml), [ChatView.xaml](../../GRC/GRC/Views/ChatView.xaml))**:
  * `ChatMessageTemplate`에서 `helpers:SimpleMarkdownHelper.Text="{Binding Text}"`를 각 `SegmentType`별 `DataTrigger` `Setter` 내부로 이동하여 실제 표시되는 단일 `TextBlock`에서만 마크다운 인라인을 생성(불필요한 파싱 오버헤드 75% 제거).
  * `ChatScrollViewer`에 `CanContentScroll="{TemplateBinding ScrollViewer.CanContentScroll}"` 및 스크롤바 가시성 `TemplateBinding`을 명시적으로 연결하여 레이아웃 초기화 전후의 템플릿 계약을 일관되게 보장.
* **64비트 `ToInt64()` 포인터 방어 ([MainWindow.xaml.cs](../../GRC/GRC/Views/MainWindow.xaml.cs))**: `WndProc` 내 `lParam`/`wParam`을 `ToInt64()`와 비트 마스킹으로 읽고 `PresentationSource.FromVisual(MaximizeButton) != null` 가드를 적용.
* **핵심 부정 제약 (Never use X)**:
  * **절대 다중 분기 `DataTemplate`에서 `Visibility="Collapsed"`인 비활성 요소에 비용이 큰 부착 속성(정규식 파싱 등) 바인딩을 기본값으로 걸어두지 말 것.** 반드시 활성 `DataTrigger`의 `Setter`에서만 바인딩할 것.
  * **절대 `ItemsControl.Template`을 커스텀 `<ScrollViewer>`로 재정의할 때 `CanContentScroll="{TemplateBinding ScrollViewer.CanContentScroll}"` 바인딩을 누락하지 말 것.**
  * **절대 64비트 Windows 환경의 `WndProc`에서 `lParam`/`wParam`에 `IntPtr.ToInt32()`를 호출하지 말 것.** 반드시 `ToInt64()`를 사용할 것.

---

> [!NOTE]
> **[정제 완료 기준선]** 2026-11-04 이전 상위 항목은 정제 완료됨. 신규 인시던트는 이 아래에 추가됩니다.

---

## 2026-11-05: [Resolved] Pool C 동료 유대 시드(`BOND:` / `BOND_SECRET:`) 원격 동료 역전, 3턴 창 만료 후 소진 판정 누락 및 1단계 발단 문구 중복 재연 해결

### 1. 현상 (Symptom)
1. **부재 동료(`Remote`)에 의한 현장 동료(`Local`) 시드 밀어내기**: 주인공이 성소에 머물며 현장 동료(오르페)와 교감 중임에도, 타 거점에 있는 동료(클라리스·시엘라)의 `SeedOfferCounts`가 더 낮다는 이유로 `Pool C` 단일 슬롯을 부재 동료 시드가 독점함.
2. **4~6턴 중장기 유대 에피소드 완수 시 `ConsumedSeeds` 누락**: 유대 사건이 개막된 후 4턴 이상 이어지면 3턴 서사 윈도우(`ActiveSeedKeys`)가 먼저 닫혀, 에피소드가 실제로 완수된 턴에 StatusAI의 `<offered_story_seeds>` 화이트리스트에서 해당 시드가 빠져 `ConsumedSeeds`에 기록되지 않음.
3. **2단계 `BOND_SECRET:{Name}`에서 1단계 발단 계기(`BondTrigger`) 반복 재연**: 이미 `BOND:오르페`가 완수되었음에도 2단계 `BOND_SECRET:오르페` 시드 문자열이 `LoreRenderer.BondClause`(1단계 `발단 계기: 날개죽지 상처 간호 -> 완수 후 변화: ...`) 원문을 그대로 재주입하여 서사 AI가 치료된 날개 상처 사건을 반복 묘사함.
4. **StatusAI `Rule 3`(`items`) 특정 아이템 과적합 및 복합 조사(`와의`/`과의`) 매칭 누락**: `StatusUpdateGuide`가 횃불·밧줄 등 특정 예시 단어에 과적합되어 소지품을 과잉 삭제하거나, `EntityNameResolver.KoreanParticlesPattern`에 복합 조사(`와의`, `과의`, `랑의` 등)가 빠져 `"오르페와의 유대"` 스탯 키가 인물 호감도 매칭에서 누락될 위험이 존재함.

### 2. 원인 (Root Cause)
* `ExtractAvailableStorySeeds`([MemoryManagerService.cs](../../GRC/GRC/Services/MemoryManagerService.cs))가 `Pool C`를 현장 동료와 원격 동료 구분 없이 1개만 뽑으면서 `OrderBy(OfferCount)`를 최우선 적용함.
* 서사 주입 기간(`ActiveSeedKeys`, 최대 3턴)과 StatusAI 완수 감시 기간을 동일한 리스트로 묶어, 3턴 경과 시 감시 목록까지 함께 소거됨.
* `AutoConsumeStage1BondsAt60Locked`가 미실행 1단계 시드를 호감도 60 도달만으로 자동 소진시키는 한편, 정작 `BOND_SECRET:` 프롬프트 조립 시에는 1단계 `BondTrigger`를 제거하지 않고 중복 결합함.

### 3. 해결책 및 불변식 (Resolution & Invariants)
* **Pool C 2슬롯 분리(`Slot C-Local` + `Slot C-Remote`) 및 제시 횟수 정밀화 ([MemoryManagerService.cs](../../GRC/GRC/Services/MemoryManagerService.cs), [ChapterContext.cs](../../GRC/GRC/Models/ChapterContext.cs))**:
  * 현장 동료(`IsCompanionPresentInCurrentSceneLocked == true`)는 `Slot C-Local`로 100% 보장하고, 부재 동료는 `Slot C-Remote`(`REMOTE_` 접두어, 동선 유도 라벨, `SeedOfferCounts` 동결)로 분리 주입함.
  * 창 개설 1턴째에 시드 미소진 상태로 즉시 장소를 이동하면 `RollbackCurrentWindowOfferCountsLocked`로 미소진 시드의 `SeedOfferCounts` 증가분을 롤백함.
* **서사 윈도우(`ActiveSeedKeys`)와 감시 윈도우(`MonitoredSeedKeys`) 분리 및 기조우 동료 상시 감시 ([MemoryManagerService.cs](../../GRC/GRC/Services/MemoryManagerService.cs))**:
  * 3턴 경과 시 `ActiveSeedKeys`만 닫고 `MonitoredSeedKeys`는 `sceneResolvedEffective` 시점까지 유지하며, 해당 턴 등장 동료(`_turnEncounteredChars`)의 미소진 `BOND:`/`BOND_SECRET:`을 `BuildStatusRequest`의 `<offered_story_seeds>`에 상시 합산(`CHAR:` 및 `장소` `HOOK:`은 StatusAI 감시에서 제외).
* **`AutoConsumeStage1BondsAt60Locked` 폐지 및 `BuildBondSeedPromptText` 단계별 분리 ([MemoryManagerService.cs](../../GRC/GRC/Services/MemoryManagerService.cs))**:
  * 1단계 미소진 시 호감도 60+에서도 `BOND:{Name}`을 유지하되 `(내면 단서 해금: ...)`을 합본 제시하며, `BOND:{Name}` 확정 소진 후 제시되는 `BOND_SECRET:{Name}`에서는 `BondTrigger`를 제거하고 `(기완수 유대: {BondPayoff}) -> 내면 단서 해금: ...`만 주입함.
* **복합 조사 정규식 보강 및 StatusAI `Rule 3` 구조적 분리 ([EntityNameResolver.cs](../../GRC/GRC/Helpers/EntityNameResolver.cs), [MemoryManagerService.cs](../../GRC/GRC/Services/MemoryManagerService.cs))**:
  * `KoreanParticlesPattern`에 `와의|과의|랑의|이랑의|에의|에서의|에게의|한테의|로부터의|으로부터의`를 추가하고, StatusAI `Rule 3`을 `[유지 원칙]`과 `[제거 조건]`으로 구조화하여 범용 소지품 보존율을 확보함.
* **핵심 부정 제약 (Never use X)**:
  * **절대 미실행된 `BOND:{Name}` 1단계 서사를 StatusAI 확정 판정 없이 호감도 수치만으로 자동 소진(`ConsumedSeeds`) 처리하지 말 것.**
  * **절대 `BOND_SECRET:{Name}` 2단계 시드 문구에 이미 완수된 1단계 `BondTrigger`(발단 계기) 원문을 재주입하지 말 것.**
  * **절대 서사 주입용 3턴 윈도우(`ActiveSeedKeys`) 만료 시 진행 중인 에피소드의 StatusAI 감시 목록(`MonitoredSeedKeys`)까지 함께 비우지 말 것.**

---

## 2026-11-06: [Resolved] 원장 불릿 초반 법칙·물품 고유명사 누락(A1), StatusAI 재시도·취소 토큰 분리(A2), NPC 착용 의복 역유입(A3) 및 MemoryLog 1:1 정밀 추적(B5~B12) 해결

### 1. 현상 (Symptom)
1. **`EventLedgerExtractor`(A1) 초반부 세계관 법칙·가설 및 물품 고유명사 누락**: 4턴(8메시지) 압축 구간에서 장소 이동이 없으면 중반부의 중요 세계관 법칙(대봉인 닻줄 이완·반발 자기장 감쇄 원리, 순결-정화 전도율 가설)이나 구체적 계획(비각 도끼 대여·나무 욕조 제작)이 후반부 스킨십 묘사에 밀려 누락(`LOW` 2개 오분류)되거나, 수선하여 양도한 `'삼베 가리개'`가 일반 명사 `'옷'`으로 추상화됨.
2. **`StatusAI`(A2) 일시 파싱 실패 시 무재시도 드롭 및 스트리밍 종료 후 `CancellationTokenSource` 조기 Dispose**: `Flash-Lite`가 간헐적으로 비정상 JSON을 반환할 때 재시도 없이 해당 턴 상태 갱신이 영구 유실되며, `ChatViewModel.SendMessageAsync`가 스트리밍 종료 시점에 `_cancellationTokenSource.Dispose()`를 즉시 호출하여 백그라운드에서 실행 중인 `statusTask` 및 `onDownloadComplete`와 수명주기가 충돌함.
3. **`StatusAI`(A3) `<canonical_names>` 등재 NPC 의복(`해진 삼베 가리개`)의 주인공 인벤토리(`items`) 역유입**: 로어북 `아이템` 카테고리에 등재된 NPC 착용 의복(`해진 삼베 가리개`)이 `<canonical_names>`의 `- 정식 아이템명:`에 노출될 때, 주인공이 오르페의 옷을 바늘로 꿰매어 수선해 주면 `Rule 3`(`원재료는 완성품으로 교체`)과 결합하여 타인 착용 의복이 주인공 `items`에 추가됨.

### 2. 원인 (Root Cause)
* `EventLedgerExtractor`의 `LOW/MEDIUM` 판정 기준이 물리적 장소 이동·신규 인물 대면 위주로만 기술되어 동일 장소 내 세계관 법칙·가설·소유권 이전·구체적 약속이 `LOW`로 격하되었고, 일반 명사 치환 금지 지침이 없었음.
* `ChatWorkflowService`의 `statusTask`에 지수 백오프 재시도 루프와 유효 페이로드(`isMeaningfulPayload`) 검증이 없었으며, 스트리밍 취소 토큰(`_cancellationTokenSource`)과 백그라운드 상태창 취소 토큰(`_statusUpdateCts`)이 분리되지 않았음.
* `BuildStatusRequest`의 `<canonical_names>`가 `- 정식 아이템명:`을 단순 나열하여 `StatusAI`가 이를 추적 대상 소지품 목록으로 오인했고, `Rule 3`에 타인 착용·귀속 의복을 수선해 준 경우 주인공 소유 완성품으로 추가하지 말라는 경계가 부재했음.

### 3. 해결책 및 불변식 (Resolution & Invariants)
* **원장 불릿 추출 지침 정밀화 및 세션 고유명사 비과적합 보장 ([EventLedgerExtractor.cs](../../GRC/GRC/Services/Cognitive/EventLedgerExtractor.cs))**:
  * 특정 세션 고유명사 예시 없이 일반 명사(`'옷'`, `'도구'`, `'연장'`, `'그녀'`) 치환을 금지하고 원문에 등장한 인물명·지명·아이템명 고유명사를 강제하며, 동일 장소 내 대화라도 세계관 법칙·봉인/저주 원리·가설 확인, 물품 소유 이전, 구체적 계획·약속이 포함된 구간은 `MEDIUM(3개)` 이상으로 분류하고 `HIGH(4~5개)`의 장기 시간 경과·주요 물품 획득/제작 기준을 보존함(프로덕션 동일 설정 `Temperature: null` 6개 프로브 × 5회 = 30회 더블 블라인드 라이브 벤치마크 `30/30 (100.0%)` 통과, 평균 `9.45 / 10.0`).
* **StatusAI 지수 백오프 재시도(`2000ms -> 5000ms`) 및 전용 `_statusUpdateCts` 수명주기 격리 ([ChatWorkflowService.cs](../../GRC/GRC/Services/ChatWorkflowService.cs), [ChatViewModel.cs](../../GRC/GRC/ViewModels/ChatViewModel.cs))**:
  * `StatusRetryDelaysMs = [2000, 5000]` 재시도 루프와 `isMeaningfulPayload` 검증을 도입하고, 재시도·최종 실패·취소 사유를 `MemoryLog.txt`에 기록.
  * `ChatViewModel`에서 스트리밍용 `_cancellationTokenSource`와 백그라운드 상태창용 `_statusUpdateCts`를 분리하고, `_isAwaitingPriorStatus`로 동기 재진입을 차단하며, `DeleteMessageAsync`(최신 턴 메시지 삭제 시에만 취소, 과거 메시지 삭제 시에는 `AwaitPendingStatusTaskAsync` 대기)/`ClearChatAsync`/`GoBack`/`InitializeWithSession`으로 백그라운드 상태 갱신 경합을 차단.
* **NPC 착용·수선 의복 역유입 차단 및 표기 사전 명확화 ([MemoryManagerService.cs](../../GRC/GRC/Services/MemoryManagerService.cs))**:
  * `<canonical_names>`의 아이템 항목에 `(<previous_state>의 items에 없던 타 인물 착용 의복·구속구나 이미 소멸한 초기 결박 잔해는 절대 items에 추가·유지 금지)` 경계를 명시하고, `Rule 3`을 `[타인 물품·수선품 추가 절대 금지]`와 `[제작 교체·소모품 및 장기 방치 잔해 삭제·도구 유지]`로 구조화하여 주인공 본인 물품 제작 교체, `days` 이상 경과 초기 결박·잔해 삭제, 본인 의복·도구 유지, 타인 의복 수선품 추가 차단을 동시에 달성(프로덕션 동일 설정 `Temperature: null`, `thinkingLevel: LOW` 핵심+Turn334 6대 프로브 × 5회 = 30회 더블 블라인드 라이브 벤치마크 `30/30 (100.0%)` 통과, 전체 35회 루브릭 `85.5 / 90.0`).
* **핵심 부정 제약 (Never use X)**:
  * **절대 스트리밍 완료 직후 `Dispose()`되는 `_cancellationTokenSource.Token`을 백그라운드 `statusTask`나 `onDownloadComplete` 추천 답변 프리페치에 그대로 전달하지 말 것.** 반드시 수명주기가 격리된 `_statusUpdateCts.Token` / `_suggestionCts.Token`을 사용할 것.
  * **절대 `StatusAI`의 `<canonical_names>`에 로어북 `아이템` 목록을 주입할 때 소유 판정 무관(표기 참조용 사전) 경계 문구를 생략하거나, 타인의 의복·장비를 수선해 준 행위를 주인공 소유 완성품 획득으로 취급하지 말 것.**

---
