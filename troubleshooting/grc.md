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

## 2026-10-15: [Resolved] TensorPrimitives.CosineSimilarity의 영벡터(Zero-Vector) 연산 시 NaN 발생 및 방어 가드

### 1. 현상 (Symptom)
* `.NET 8.0 TensorPrimitives.CosineSimilarity` 연산 시 영벡터(Zero-Vector) 입력에 대해 `float.NaN`이 반환되어 상위 유사도 랭킹 및 필터링 비교식이 침묵 파손되는 결함.

### 2. 원인 (Root Cause)
* 코사인 유사도 연산 중 영벡터의 크기(L2 Norm = 0)로 인한 0 나누기(Division by Zero) 발생.

### 3. 해결책 및 불변식 (Resolution & Invariants)
* **`VectorMathHelper` 삼항 연산 가드**: `float.IsNaN(similarity) ? 0.0f : similarity` 클램핑 적용.
* **핵심 부정 제약 (Never use X)**: **절대 `TensorPrimitives.CosineSimilarity` 반환값을 `float.IsNaN` 검증 없이 직접 비교식에 사용하지 말 것.**

---

## 2026-10-15: [Resolved] Google text-embedding-004의 Vertex AI vs AI Studio API 엔드포인트 및 페이로드 스키마 비대칭 일원화

### 1. 현상 (Symptom)
* Google Cloud Vertex AI와 AI Studio 간의 REST 엔드포인트 URL, 인증 헤더(OAuth2 Bearer vs API Key), 요청/응답 JSON 스키마 비대칭으로 인한 역직렬화 실패 및 null 참조 예외 발생.

### 2. 원인 (Root Cause)
* AI Studio는 `content.parts` 구조 및 ApiKey 쿼리를 사용하나, Vertex AI는 `instances[].content` 구조 및 Bearer 토큰을 요구하여 단일 DTO로 역직렬화 불가.

### 3. 해결책 및 불변식 (Resolution & Invariants)
* **내부 분기 추상화 (`GeminiApiService.GetEmbeddingAsync`)**: `UseVertexAI` 설정에 따라 요청/응답 DTO를 분리 직렬화하고, 상위 계층에는 `Task<float[]> GetEmbeddingAsync(string text)` 단일 시그니처로 768차원 배열 투명 반환.
* **불변식**: 외부 API 공급자 간 스키마 차이를 상위 도메인 서비스로 노출하지 말고 API 서비스 계층에서 단일 인터페이스로 격리할 것.

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

## 2026-10-15: [Resolved] MemoryManagerService V3 컷오버 시 metaDirective 합성 및 GenerationConfig/ThinkingAnchor 무손실 보존

### 1. 현상 (Symptom)
* 순수 DTO 프롬프트 조립기(`CognitivePromptAssembler`) 분리 후 상위 모델 파라미터(`GenerationConfig`, `ThinkingConfig(medium)`) 및 시스템 지침(`metaDirective`)이 유실되는 결함.

### 2. 원인 (Root Cause)
* 조립기는 프롬프트 텍스트 배치만 담당하므로 API 레벨 실행 설정 및 시스템 지침 합성은 상위 서비스(`MemoryManagerService`)의 명시적 재부착 책임임.

### 3. 해결책 및 불변식 (Resolution & Invariants)
* **지침 합성 및 C# `with` 식 재부착**: 유저 텍스트에 `metaDirective`를 합성하고, `assembledRequest with { SafetySettings = ..., GenerationConfig = ... }` 패턴으로 모델 실행 파라미터 재부착.
* **불변식**: 프롬프트 조립기 분리 시 API 모델 실행 파라미터가 DTO 전환 과정에서 누락되지 않도록 상위 서비스에서 불변식으로 보존할 것.

---

## 2026-10-15: [Resolved] Vertex AI 한국어 임베딩의 토큰 길이 더미 벡터 침묵 결함 및 LLM(global) vs Embedding(asia-northeast3 서울) 리전 분리 규격

### 1. 현상 (Symptom)
* **더미 벡터 침묵 실패**: Vertex AI `text-embedding-004`에 한국어 인입 시 에러 없이 `HTTP 200 OK`와 함께 토큰 길이에 비례하는 고정 더미 벡터가 반환되어 무관한 문장 간 유사도 `0.9995` 오판정 발생.
* **LLM 리전 404 오류**: 최신 주력 LLM(`gemini-3.8-flash`, `gemini-4-argon`)을 로컬 리전(`us-central1`, `asia-northeast3`)으로 호출 시 `404 NotFound` 발생.

### 2. 원인 (Root Cause)
1. Vertex AI의 `text-embedding-004`는 영어/코드 전용 모델로 한글 인입 시 정적 더미 벡터를 조용히 반환함.
2. 최신 Gemini LLM은 로컬 리전에 배포되지 않고 전역 프록시 엔드포인트(`locations/global`)에서만 서비스됨.
3. `System.Text.Json` 기본 직렬화기의 한글 `\uXXXX` 유니코드 이스케이프 왜곡.

### 3. 해결책 및 아키텍처 SSOT 규격 (Resolution & Invariants)
1. **한국어 다국어 모델 전환**: **`text-multilingual-embedding-002`**로 교체하고 `JavaScriptEncoder.UnsafeRelaxedJsonEscaping` 적용.
2. **리전 분리 아키텍처 불변식**:
   * **최신 주력 LLM**: 반드시 `locations/global` 단일 엔드포인트 사용 (`us-central1`/`asia-northeast3` 호출 금지, 404 에러 방어).
   * **임베딩 모델**: 반드시 `asia-northeast3` (서울 리전 물리 직결, 약 400~500ms로 2배 고속화).
   * **TTS 모델**: `us-central1` 엔드포인트 유지.

---

## 2026-10-16: [Resolved] 단일 불릿 원장(Single Event Ledger) 일원화 및 [제N장] 시간축 앵커 도입을 통한 서사 시간 왜곡, 81턴 불릿 증발 및 240턴 장기 기억 지연 결함 완전 해결

### 1. 현상 (Symptom)
* 챕터 전환(81턴) 시 `EventLedgers`가 빈 리스트로 초기화되어 최근 15개 불릿이 프롬프트에서 즉시 소멸하는 기억 단절 발생.
* 3개 챕터(240턴) 대기 큐로 인해 2~4장 진행 중 장기 줄거리가 프롬프트 어디에도 존재하지 않는 거대한 기억 공백 발생.
* 시스템 턴 번호(`[Turn X]`) 노출로 인한 서사 시간 괴리 및 행동 묘사(평서문) 시 RAG 검문소 오차단.

### 2. 원인 (Root Cause)
* v1.0 레거시 3챕터 대기 큐(`_mediumBuffers`)와 v3.0 불릿 시스템의 중복 및 상태창 단서 수집(`unresolvedClues`) 모델의 옥상옥 구조로 인한 파편화.

### 3. 해결책 및 불변식 (Resolution & Invariants)
1. **단일 불릿 원장(Single Event Ledger) 일원화**:
   * 챕터 전환 시 `EventLedgers`를 무손실 누적 승계하고, 상시 15개는 언제나 `TakeLast(15)`, 과거 RAG 풀은 `Take(Count - 15)`로 단일 슬라이싱하여 81턴 불릿 증발 원천 차단.
   * 1개 챕터(80턴) 완료 즉시 백그라운드 Pro 모델 장기 요약을 가동하여 기억 공백 해소.
2. **시간축 앵커 및 RAG 인과 정렬**:
   * 프롬프트 내 `[Turn X]`를 제거하고 장 번호 변경 시에만 `[제N장]` 그룹 헤더를 1회 출력. RAG 인출 결과는 `item.Turn` 오름차순으로 재정렬하여 인과 역전 방지.
3. **핵심 부정 제약 (Never use X)**:
   * **절대 매 턴 상태창 모델에 단서(`unresolvedClues`) 수집 슬롯을 두지 말 것(Never collect clues via turn status model).** 복선과 비밀은 잡음 생성과 우선순위 역전을 유발하므로 독자적 슬롯을 두지 말고 `EventLedger` 팩트 단일 원장으로 일원화하여 관리할 것.

---

> [!NOTE]
> **[정제 완료 기준선]** 2026-10-16 이전 상위 항목은 정제 완료됨. 신규 인시던트는 이 아래에 추가됩니다.

---

## 2026-10-16: [Resolved] 로어북 3순위 상태창 인물 인출 조건식 오타 버그 수정 및 활성 인물 윈도우 슬라이싱 최적화

### 1. 현상 (Symptom)
- 상태창에 인물이 1명이라도 존재할 경우, 로어북에 등록된 카테고리 "인물" 항목 전체가 무차별 매칭되어 최대 3,000토큰 예산이 조기에 소진되는 결함 발생.
- 챕터 누적으로 인해 상태창 인물 사전(`Chars`)이 비대해졌을 때, 과거에 이미 퇴장한 조연 로어북까지 상시 인출 후보군으로 포함되어 현재 장면의 로어북 인출을 저해할 위험 존재.

### 2. 원인 (Root Cause)
- `HybridLorebookRetriever.cs` 내 3단계 조건식이 `string.Equals(entry.Category, "인물", ...) || charNameSet.Contains(entry.Name)...`로 작성되어 논리합(`||`)에 의해 카테고리가 "인물"인 모든 항목이 `charNameSet` 포함 여부와 무관하게 무조건 참(True)으로 평가됨.
- `MemoryManagerService.cs`에서 활성 인물 이름(`activeChars`)을 추출할 때 단기 기억 전체(`_shortTermMemory`)를 대상으로 검색하여 오래 전 퇴장한 인물까지 활성 상태로 간주함.

### 3. 해결책 및 불변식 (Resolution & Invariants)
- **오타 조건식 제거**: `HybridLorebookRetriever.cs` L108에서 무차별 `Category == "인물" ||` 조건을 제거하고, `charNameSet.Contains(entry.Name) || (entry.Keywords != null && entry.Keywords.Any(k => charNameSet.Contains(k)))`로 한정하여 상태창 인물과 실제 매칭될 때만 후보로 등록되도록 수정.
- **최근 4개 메시지 슬라이싱**: `MemoryManagerService.cs`에서 `_shortTermMemory.TakeLast(4)` 및 현재 입력 텍스트에 이름이 직접 언급된 인물만 `activeChars`로 엄격히 필터링하여 프롬프트 예산 잠식 방지.
- **의무 검증 (Ground Truth)**: `MemoryManagerServiceTests` 내 단위 테스트 12건(최근 4개 메시지 외 인물 배제 검증 신규 테스트 포함) 및 전체 단위 테스트 127건, 통합 테스트 9건 전원 통과(Exit Code 0).

---

## 2026-10-17: [Resolved] 구버전 세션 로어북 백그라운드 임베딩 백필(Backfill) 중 세션 전환 및 로어북 수정 시 상태 오염/데이터 손실 경합 조건 해결

### 1. 현상 (Symptom)
- 임베딩이 없는 구버전 세션 로드 시 백그라운드에서 임베딩을 비동기 생성하는 도중(약 1~2초), 사용자가 즉시 로어북 편집창을 열어 새 항목을 등록하거나 뒤로 가기 후 다른 세션을 열면:
  - 신규 등록한 로어북이 백그라운드 태스크 완료 시점에 구버전 목록으로 덮어써져 영구 증발.
  - 뒤로 가기를 누르고 다른 세션 파일을 열었을 때, 이전 세션의 로어북 데이터가 현재 열린 다른 세션 파일에 덮어써져 저장되는 파일 오염 위험 발생.

### 2. 원인 (Root Cause)
- `ChatViewModel.cs`의 `Task.Run` 내부에서 시작 시점의 스냅샷 변수 없이 인스턴스 멤버(`CurrentPreset`, `_currentFileName`)를 직접 참조하고, 비동기 호출 완료 후 무조건 `CurrentPreset with { Lorebooks = updatedLores }`로 전체를 덮어썼기 때문.

### 3. 해결책 및 불변식 (Resolution & Invariants)
- **세션 활성 수명주기 앵커링**: `private bool _isSessionActive;` 플래그 및 `string targetFileName = _currentFileName;` 스냅샷을 캡처하고, `GoBack()` 호출 시 `_isSessionActive = false;`로 전환. 비동기 완료 후 `!_isSessionActive || _currentFileName != targetFileName`일 경우 즉시 작업을 폐기하여 파일 오염 차단.
- **비파괴적 선택적 임베딩 병합(Selective Merge)**: 전체 리스트를 덮어쓰지 않고, 완료된 768차원 임베딩을 `(Name, Content)` 키 딕셔너리로 구축한 뒤 현재 `CurrentPreset.Lorebooks`의 항목 중 임베딩이 결여된 항목에만 선택적으로 주입. 백그라운드 실행 중 유저가 추가/수정한 로어북 무손실 보존.
- **WPF STA 스레드 안전 디스패칭**: `Application.Current?.Dispatcher` 가드를 통해 UI 스레드 위반 없이 안전하게 상태를 반영하고 세션 파일과 동기화.


