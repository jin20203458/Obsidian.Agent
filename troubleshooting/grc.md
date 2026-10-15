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
* **`RepairTruncatedJson` 엔진 탑재**: 절단된 응답 감지 시 열린 따옴표를 닫고 dangling 키/콜론/쉼표를 정제한 후 스택에 보존된 여는 괄호의 짝(`]`, `}`)을 LIFO 역순으로 닫아 구조적 유효성 복원.

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

> [!NOTE]
> **[정제 완료 기준선]** 2026-10-14 이전 상위 항목은 정제 완료됨. 신규 인시던트는 이 아래에 추가됩니다.

---

## 2026-10-15: [Resolved] TensorPrimitives.CosineSimilarity의 영벡터(Zero-Vector) 연산 시 NaN 발생 및 방어 가드

### 1. 현상 (Symptom)
* `.NET 8.0 System.Numerics.Tensors.TensorPrimitives.CosineSimilarity`를 사용하여 벡터 코사인 유사도를 계산할 때, 입력 벡터 중 하나가 모든 요소가 0인 영벡터(Zero-Vector)인 경우 결과값으로 `float.NaN`이 반환됨.
* 반환된 `float.NaN`이 상위 유사도 랭킹 비교(`score >= threshold`)나 정렬 연산으로 전파되어 랭킹 알고리즘이 엉뚱한 결과를 산출하거나 오동작하는 잠재적 엣지 케이스 발생.

### 2. 원인 (Root Cause)
* 코사인 유사도 수학식 $$\\text{Similarity} = \\frac{A \\cdot B}{\\|A\\| \\|B\\|}$$ 에서 영벡터의 크기(L2 Norm, $\\|A\\|$)가 0이 되므로, 부동소수점 나눗셈에서 0으로 나누기(Division by Zero)가 발생하여 IEEE 754 부동소수점 규격에 따라 `NaN`이 산출됨.

### 3. 해결책 (Resolution)
* **`VectorMathHelper` 삼항 연산 방어 가드 구축**:
  `TensorPrimitives.CosineSimilarity` 연산 직후 `float.IsNaN`을 검사하여 `NaN` 발생 시 `0.0f`로 클램핑하여 안전하게 반환:
  ```csharp
  public static float CosineSimilarity(ReadOnlySpan<float> x, ReadOnlySpan<float> y)
  {
      if (x.IsEmpty || y.IsEmpty || x.Length != y.Length)
      {
          return 0.0f;
      }

      float similarity = TensorPrimitives.CosineSimilarity(x, y);
      return float.IsNaN(similarity) ? 0.0f : similarity;
  }
  ```
* **핵심 부정 제약 (Never use X)**: **절대 `TensorPrimitives.CosineSimilarity`의 반환값을 `float.IsNaN` 검증 없이 직접 비교식에 사용하지 말 것.** 빈 데이터베이스나 초기화되지 않은 임베딩 벡터가 유입될 경우 `NaN` 비교로 인해 의도치 않게 모든 필터링이 실패할 수 있음.

---

## 2026-10-15: [Resolved] Google text-embedding-004의 Vertex AI vs AI Studio API 엔드포인트 및 페이로드 스키마 비대칭 일원화

### 1. 현상 (Symptom)
* Google Cloud Vertex AI 환경과 Google AI Studio 환경 간에 `text-embedding-004` REST 호출 엔드포인트 URL 패턴, HTTP 인증 방식, 요청 JSON 스키마 및 응답 JSON 스키마가 완전히 달라 단일 DTO로 역직렬화할 경우 null 참조 예외 발생.

### 2. 원인 (Root Cause)
* **AI Studio (Gemini Developer API)**:
  - 엔드포인트: `models/text-embedding-004:embedContent?key={ApiKey}`
  - 요청 바디: `{"content": {"parts": [{"text": "..."}]}}`
  - 응답 바디: `{"embedding": {"values": [...]}}`
* **Vertex AI (Google Cloud Publisher API)**:
  - 엔드포인트: `publishers/google/models/text-embedding-004:predict` (전역 리전 `locations/global`)
  - 요청 바디: `{"instances": [{"content": "..."}]}`
  - 응답 바디: `{"predictions": [{"embeddings": {"values": [...]}}]}`
  - 인증 방식: OAuth2 Bearer 토큰 (`Authorization: Bearer {token}`)

### 3. 해결책 (Resolution)
* **`GeminiApiService.GetEmbeddingAsync` 내부 분기 일원화**:
  - `appSettings.UseVertexAI` 설정값에 따라 요청/응답 DTO(`EmbedContentRequest/Response` vs `VertexEmbedRequest/Response`)를 완전히 분리하여 직렬화.
  - 상위 계층(`ILorebookService`, `IMemoryManagerService`)에서는 서비스 분기를 알 필요 없이 `Task<float[]> GetEmbeddingAsync(string text)` 단일 시그니처만 호출하여 768차원 `float[]` 배열을 투명하게 수신하도록 추상화.
  - ProjectId/ApiKey 부재, HTTP 오류, 파싱 실패 시 `try-catch`로 Fail-safe 가드를 두고 `Array.Empty<float>()`를 안전하게 반환하여 대화 파이프라인 차단을 방지.

---

## 2026-10-15: [Resolved] Gemini API Part 빈 문자열 입력 시 HTTP 400 Bad Request 방어 및 Auto-Play 안전 Fallback

### 1. 현상 (Symptom)
* 플레이어가 입력을 입력하지 않고 `[계속 진행]`, `[Auto-Play]`를 실행하거나 빈 텍스트를 보낼 경우 `latestUserMessage.Text`가 빈 문자열(`""`) 또는 공백이 됨.
* Gemini API(Vertex AI / AI Studio)는 Content Part의 text가 비어있을 경우 `HTTP 400 Bad Request (contents.parts.text must not be empty)` 에러를 즉시 반환하고 대화 스트리밍이 크래시됨.

### 2. 원인 (Root Cause)
* Google Gemini API 규격상 `contents[].parts[].text` 필드는 null 또는 빈 문자열을 허용하지 않으며 최소 1자 이상의 유효한 텍스트를 요구함.

### 3. 해결책 (Resolution)
* **`CognitivePromptAssembler` 내 다단계 안전 Fallback 패턴 도입**:
  - `latestUserMessage.Text`가 `string.IsNullOrWhiteSpace`인 경우:
    1. `cognitiveContext.PlotCatalyst`가 존재하면 촉매 서술 지침으로 채택하여 주입.
    2. `PlotCatalyst`마저 비어있다면 `"(서사를 계속 진행하십시오.)"` 안전 Fallback 텍스트를 강제 주입하여 빈 Part 생성을 원천 차단.
  - **핵심 부정 제약 (Never use X)**: **절대 검증되지 않은 유저 텍스트를 검사 없이 그대로 `new Part(text)`에 전달하지 말 것.** 빈 입력 상황에서도 항상 유효한 대체 텍스트가 보장되어야 함.

---

## 2026-10-15: [Resolved] Sparse/Dense 병렬 합집합 로어북 검색 시 임베딩 Null-safe 가드 및 중복 제거

### 1. 현상 (Symptom)
* 로어북 엔트리 중 일부만 임베딩이 계산되어 있거나 `queryEmbedding`이 미생성(null)된 상태에서 시맨틱 검색을 수행할 경우 NullReferenceException이 발생하거나 검색이 누락됨.
* 키워드 정규식(Sparse)과 임베딩 유사도(Dense)에 동일한 로어북이 동시에 걸렸을 때 프롬프트에 중복 항목이 이중 주입되어 토큰을 낭비하고 어텐션을 교란함.

### 2. 원인 (Root Cause)
* 단순 병렬 결합 시 `queryEmbedding` 및 `entry.Embedding`의 null 가능성 검사가 누락되었고, 컬렉션 병합 후 고유 식별자(`Name`) 기준의 중복 제거 필터가 결여됨.

### 3. 해결책 (Resolution)
* **`HybridLorebookRetriever` 2단계 안전망 구현**:
  - `queryEmbedding == null`이거나 `entry.Embedding == null`인 경우 Dense 연산을 안전하게 0ms 스킵하고 Sparse 키워드 매칭 결과만 반환(Graceful Fallback).
  - 결합된 결과에 대해 `.DistinctBy(e => e.Name)`으로 중복 엔트리를 완벽 제거하고 `Priority` 내림차순 정렬 후 상위 `maxResults`개만 반환.

---

## 2026-10-15: [Resolved] 2계층 단서 추적기(ClueTracker)의 3중 안전망 및 20턴 TTL/슬롯 초과 시 Dormant 무손실 이관

### 1. 현상 (Symptom)
* 미회수 떡밥(`unresolvedClues`)을 세션 진행 중 무제한 축적할 경우 프롬프트 토큰이 팽창하고 모델 환각이 유발됨.
* 반대로 단순 삭제(Drop)할 경우 수십~수백 턴 뒤에 회수되어야 하는 대서사 복선이 영구 소실되는 인과율 파괴 발생.

### 2. 원인 (Root Cause)
* 단기 작업 기억(RAM)과 장기 영구 아카이브(Cold Storage) 간의 티어드 이관 파이프라인 부재.

### 3. 해결책 (Resolution)
* **`ClueTracker` 3중 결정론적 안전망 구축**:
  1. **슬롯 상한(Max 3) 무손실 이관**: 4번째 단서 유입 시 가장 오래된 1번 단서를 삭제하지 않고 `Status = ClueStatus.Dormant`로 전환하여 `_archivedClues`로 자동 이관(Paging-out).
  2. **20턴 TTL 만료 아카이빙**: `TickTurn(currentTurn)` 호출 시 20턴 동안 참조되지 않은 단서는 `Dormant` 상태로 아카이브에 안전 보존.
  3. **조건부 주입 (Gating)**: 유저가 직접 추궁/의심하지 않는 평소에는 프롬프트 주입 0토큰(0% 누설)을 유지하고, 추궁 시에만 `<latent_secrets>`로 조건부 주입.


