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

> [!NOTE]
> **[Superseded: 2026-10-18]** Google AI Studio의 `text-embedding-004` 서비스 종료(2026-01-14) 및 Vertex AI 레거시 지정에 따라, 본 엔드포인트 일원화 구조는 차세대 통합 모델 `gemini-embedding-001` (MRL 768차원)로 전면 대체되었습니다. (하단 2026-10-18 항목 참조)

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

> [!NOTE]
> **[Superseded: 2026-10-18]** `text-multilingual-embedding-002`는 차세대 통일 모델 `gemini-embedding-001` (MRL 768차원, 서울 asia-northeast3 리전)로 전면 통합 교체되었습니다. 단, `LLM(global) vs Embedding(asia-northeast3 서울) vs TTS(us-central1)` 3분할 리전 규격 및 `UnsafeRelaxedJsonEscaping` 불변식은 그대로 유지됩니다. (하단 2026-10-18 항목 참조)

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
- `MemoryManagerService.cs`에서 활성 인물 이름(`activeChars`)을 추출할 때 필터링 없이 상태창 `Chars` 딕셔너리 전체를 넘겨, 챕터 누적으로 인해 수십 턴 전 퇴장한 인물까지 활성 상태로 간주함.

### 3. 해결책 및 불변식 (Resolution & Invariants)
- **오타 조건식 제거**: `HybridLorebookRetriever.cs` L108에서 무차별 `Category == "인물" ||` 조건을 제거하고, `charNameSet.Contains(entry.Name) || (entry.Keywords != null && entry.Keywords.Any(k => charNameSet.Contains(k)))`로 한정하여 상태창 인물과 실제 매칭될 때만 후보로 등록되도록 수정.
- **광역 단기 기억 및 팩트 불릿 스캔**: 단순 4개 메시지 슬라이싱 시 대명사/주어 생략 구간에서 동료 로어북이 증발하는 회귀를 방지하기 위해, `MemoryManagerService.cs`에서 `_shortTermMemory` 전체(18~25개 메시지) 및 최근 팩트 불릿 3개(`EventLedgers.TakeLast(3)`)를 광역 스캔하여 현장 동료의 서사 연속성 보장과 초장기 세션 퇴장 조연의 로어북 예산 잠식 방지를 동시에 달성.
- **의무 검증 (Ground Truth)**: `MemoryManagerServiceTests` 내 단위 테스트(광역 단기 기억 및 팩트 불릿 기반 활성 인물 포함 및 퇴장 조연 배제 검증 신규 테스트 포함) 및 전체 단위 테스트 통과(Exit Code 0).

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

---

## 2026-10-17: [Resolved] 하이브리드 로어북 동적 패킹 시 우선순위(Priority 0 = 최우선) 역전 결함 해결

### 1. 현상 (Symptom)
- `EditLorebookWindow.xaml` UI 명세 및 툴팁에는 `"숫자가 낮을수록(0에 가까울수록) 스캔과 주입 우선순위가 높아집니다."`라고 정의되어 있고 기본값도 `Priority = 0`이었으나, 실제 인출 엔진(`HybridLorebookRetriever.cs`)에서는 기본값 0을 부여받은 최우선 로어북이 가장 후순위로 밀려 3,000토큰 예산 초과 시 조기 탈락하는 우선순위 역전 버그 발생.

### 2. 원인 (Root Cause)
- `HybridLorebookRetriever.cs`의 동적 패킹 4단계 정렬식에서 `OrderBy`가 아닌 `OrderByDescending(e => e.Priority)`를 호출하여, 숫자가 큰 항목(10, 9, 8...)이 먼저 패킹되고 0에 가까운 최우선 항목이 맨 뒤로 밀려남.

### 3. 해결책 및 불변식 (Resolution & Invariants)
- **오름차순 정렬 원복**: `HybridLorebookRetriever.cs` L123-L125의 정렬식을 `OrderBy(e => e.Priority)`로 수정하여, UI 설명 및 레코드 기본값(`Priority = 0`)과 일치하도록 0이 가장 높은 1순위로 패킹되도록 정렬 규칙 확립.
- **의무 검증 (Ground Truth)**: `HybridLorebookRetrieverTests`에 `Retrieve_SortsDirectMatches_ByPriorityAscending_ZeroIsHighestPriority` 신규 테스트를 추가하고 기존 테스트 케이스를 오름차순 명세로 정렬하여 전체 단위 테스트 131건 통과(Exit Code 0).

---

## 2026-10-17: [Resolved] LlmJsonParser의 전방 상태 머신, 인라인 주석/코드블록 격리 및 체크포인트 롤백을 통한 엣지 케이스 결함 전수 해결

### 1. 현상 (Symptom)
- `ExtractFirstValidJsonBlock`이 유효성 검사(`IsValidJson`)를 통과하지 못한 비-JSON 블록(예: 서두의 `[참고]`)을 반환하여 절단 복구 루틴 진입이 가로막힘.
- 서두 사설에 닫히지 않은 `{`나 `[`가 있을 때 괄호 카운터가 0으로 복귀하지 못해 뒤따르는 정상 JSON 블록 추출 실패.
- 주석(`//`, `/* */`) 내부에 포함된 따옴표나 괄호(`}`)로 인해 문자열/괄호 계수가 오염되거나, 주석 도중 절단 시 닫는 괄호가 주석에 삼켜지는 결함 발생.
- 미완성 유니코드 이스케이프(`\u12`), 이스케이프 따옴표가 포함된 키(`"my\"key":`), 미완성 키(`"user_na`), 미완성 리터럴/숫자(`tru`, `12.`, `-`) 절단 시 유효하지 않은 JSON이 생성되어 역직렬화 예외 발생.
- JSON 문자열 값 내부에 마크다운 백틱(` ```json {} ``` `)이 포함된 경우 코드블록 종료 위치로 오인되어 JSON이 중간에 절단되거나, 닫힌 코드블록 내부의 절단 JSON이 원본 텍스트 재탐색 시 하위 객체(`{"inner": 1}`)로 오추출되는 결함 발생.
- `DeserializeSafe<List<T>>` 호출 시 배열 후보군 탐색 실패 후 배열 절단 복구를 시도하지 않고 객체 절단 복구만 호출하여 역직렬화 실패.

### 2. 원인 (Root Cause)
1. 수동 스캐너와 절단 복구기에 주석 상태(`CommentState`) 추적이 결여되어 주석 내부 문자가 JSON 제어 문자로 오인됨.
2. `RepairTruncatedJson`의 역방향 문자열 조작(`CleanTrailingDanglingTokens`)이 이스케이프 따옴표(`\"`), 키/값 문맥(`ContainerKind`/`ContainerState`), 불완전 원시 토큰을 구별하지 못함.
3. 단순 Sliding Start Scan 도입 시 절단된 루트 JSON(`{"outer": {"inner": 1}, "trunc`)에서 내부 완성 하위 객체(`{"inner": 1}`)만 오추출하여 루트 데이터가 침묵 유실되는 트랩 존재.
4. `ExtractCodeFenceBlocks`가 단순 `IndexOf("```")`를 사용하여 JSON 문자열 리터럴 내부의 백틱을 코드블록 경계로 오인함.

### 3. 해결책 및 불변식 (Resolution & Invariants)
1. **전방 컨테이너 상태 머신 및 체크포인트 롤백 (`LlmJsonParser.cs`)**:
   - `ContainerFrame` 스택을 통해 객체/배열 문맥을 추적하고, 직전 완결 요소 직후의 안전 롤백 인덱스(`SafeRollbackLength`)를 기록.
   - EOF 도달 시 미완성 키, 콜론만 있는 프로퍼티, 미완성 리터럴/숫자(`tru`, `12.`)를 단일 인덱스 절삭(`sb.Length = top.SafeRollbackLength`)으로 $O(1)$ 안전 롤백.
2. **주석 스트리핑, 코드블록 문자열 격리 및 서로게이트 정제**:
   - 복구 버퍼 생성 시 문자열 외부 주석을 완전히 스트리핑하고, 리터럴/숫자 토큰 직후 공백 없이 주석이 시작될 때(`true//`, `42/*`) 토큰 완료 판정을 선행 수행.
   - `FindNextCodeFence`로 JSON 컨테이너의 문자열 리터럴 내부에 위치한 ` ``` `를 무시하고, 1단계 코드블록 탐색 완료 구간(`fenceSpans`)을 2단계 원본 스캔에서 건너뛰어 닫힌 코드블록 내 절단 JSON의 하위 객체 오추출을 원천 차단.
   - `RemoveIncompleteUnicodeEscape` 및 `RemoveTrailingUnpairedHighSurrogate`로 4자리 미달 `\u` 및 단독 상위 서로게이트(`\uD800`~`\uDBFF`)를 정제하여 UTF-8 트랜스코딩 크래시 방어.
3. **절단 루트 선점 가드 (Root Truncation Guard) 및 스키마 매칭**:
   - 미완성 블록 감지 시 현재 위치가 절단 복구 가능한 루트(`TryRepairFromIndex != null`)인지 검사하여, 하위 객체 오추출 및 $O(N^2)$ 스캔을 원천 차단.
   - 다중 JSON 블록(사설 내 예시 스키마 + 실제 페이로드) 존재 시 `HasMatchingSchemaProperties`(`[JsonPropertyName]` 및 제네릭 `IDictionary<,>` 포함)로 대상 DTO 스키마와 일치하는 후보를 우선 채택.
4. **의무 검증 (Ground Truth)**:
   - `LlmJsonParserTests` 총 58개 단위/스트레스 테스트 100% 통과 및 솔루션 전체 174개 단위 테스트 통과(Exit Code 0).

---

## 2026-10-17: [Resolved] dotnet test 백그라운드 실행 시 솔루션 단위 평가로 인한 MSBuild/VSTest I/O 파이프 블로킹 해결 및 테스트 프로젝트 직접 타겟팅 규격 확립

### 1. 현상 (Symptom)
- 백그라운드 태스크로 `dotnet test GRC.sln --filter "Category!=Integration" --no-build` 실행 시 테스트 실행 자체는 0.4초 만에 모두 완료(151/151 통과)되었으나, 프로세스가 종료되지 않고 셸이 무한 대기(RUNNING)에 머무는 현상 발생.

### 2. 원인 (Root Cause)
- `dotnet build`와 달리 `dotnet test` CLI 규격에서는 `-nodeReuse:false`가 VSTest(`vstest.console.exe`) 및 솔루션 그래프를 평가하는 MSBuild 백그라운드 작업자 노드에 완벽히 상속되지 않음.
- 자식 작업자 노드가 백그라운드 셸의 표준 출력(stdout) 스트림 핸들을 점유한 채 반환하지 않아 프로세스 종료 신호(EOF)가 셸에 전달되지 않음.

### 3. 해결책 및 불변식 (Resolution & Invariants)
- **테스트 프로젝트 직접 타겟팅 규격 확립**:
  솔루션(`.sln`) 대신 테스트 프로젝트를 명시적으로 직접 지정하고 일반 상세도(`-v n`)를 적용하여 불필요한 솔루션 중계 노드 생성을 차단:
  ```powershell
  dotnet test tests/GRC.Tests/GRC.Tests.csproj --filter "Category!=Integration" --no-build -v n
  ```
- **불변식**: `dotnet test` 실행 시 솔루션 파일(`.sln`)을 타겟으로 지정하지 말고, 반드시 해당 테스트 프로젝트 파일(`tests/GRC.Tests/GRC.Tests.csproj`)을 직접 지정하여 VSTest/MSBuild 파이프 데드락을 방지할 것.

---

## 2026-10-17: [Resolved] FormatRule 문체 지침 누락(독백 낫표 「」, 마크다운 기호 배제, 퓨샷) 복원 및 스트리밍 실시간 별표(*) 필터링을 통한 UI 불일치 해결

### 1. 현상 (Symptom)
- Cognitive Core 3.0 개편 이후 LLM이 속마음/독백에 낫표(`「」`)를 쓰지 않고 서술문으로 뭉뚱그려 UI의 F2(속마음 전용 스타일) 연동이 무력화됨.
- LLM이 간헐적으로 대사나 단어에 `**` 볼드를 출력할 때, 실시간 스트리밍 중에는 `**`가 화면에 그대로 노출되었다가 스트리밍 완료 후 정규식에 의해 기호가 툭 사라지거나 마크다운 서식으로 치환되는 시각적 깜빡임 및 파서 불일치 발생.

### 2. 원인 (Root Cause)
1. Cognitive Core 3.0 개편 커밋(`1796789`)에서 `MemoryManagerService.cs`에 존재하던 `<syntax_rules>`(독백 「 」, 마크다운/별표 금지)와 `<example>` 퓨샷이 `CognitivePromptAssembler.cs`로 이관되면서 단순 요약문으로 축약되어 세부 지침이 유실됨.
2. `ChatWorkflowService.cs`에서 마크다운 찌꺼기 정제를 스트리밍이 100% 완료된 뒤 사후 정규식(`Regex.Replace`)으로만 처리하여, 스트리밍 수신 중에는 `*` 기호가 UI 채널로 그대로 바이패스됨.

### 3. 해결책 및 불변식 (Resolution & Invariants)
1. **프롬프트단 문체 및 서식 퓨샷 복원 (`CognitivePromptAssembler.cs`)**:
   - `FormatRule`에 대사(`" "`), 독백 및 속마음(`「 」`), 행동/배경(기호 없는 평문)과 마크다운 강조(*별표* 등) 배제 지침을 명시하고, 직관적인 4줄 `<example>` 퓨샷을 복원.
2. **스트리밍 수신 루프 실시간 별표 필터링 (`ChatWorkflowService.cs`)**:
   - `producerTask` 스트리밍 청크 수신 루프(`foreach (char c in chunk)`)에서 `c == '*'` 감지 시 즉시 건너뛰도록(`continue`) 필터링하여, 스트리밍 화면과 완료 후 텍스트(`fullResponse`) 모두에서 별표 찌꺼기를 원천 차단하고 스트리밍 중/완료 후 텍스트를 100% 일치시킴.
3. **의무 검증 (Ground Truth)**:
   - `CognitivePromptAssemblerTests`에 `AssemblePrompt_SystemInstruction_ContainsDialogueThoughtAndMarkdownRulesWithExample` 테스트 추가.
   - `ChatWorkflowServiceTests`에 `ProcessChatStreamAsync_FiltersMarkdownAsterisksFromStreamAndFinalText` 테스트 추가.
   - 단위 테스트 170건 100% 통과(Exit Code 0, 541ms).

---

## 2026-10-17: [Resolved] 메타 지시문(MetaDirective) 합성에 따른 768차원 임베딩 쿼리 벡터 오염 해결, EventLedgerExtractor 정적 SystemInstruction KV Cache 분리 및 Thinking 자가 검증 기준 보강

### 1. 현상 (Symptom)
- 호감도 MAX 등 시스템 이벤트나 챕터 전환, 자동 이어쓰기 시 `[시스템 개입: ...]` 메타 텍스트가 `userMessage.Text`에 직접 병합되어 `GetEmbeddingAsync`로 전달됨에 따라, 순수 유저 의도 검색이어야 할 768차원 쿼리 임베딩 벡터가 메타 단어로 희석(Vector Dilution)되어 로어북/과거 기억 RAG 검색 정확도가 저하됨.
- 프롬프트 조립 시 `<current_action>` 태그 내부에 `<meta_directive>`가 중첩 유입되어 지시문-데이터 역전 및 프롬프트 인젝션 모호성 발생.
- 4턴 주기 백그라운드 사건 팩트 추출기(`EventLedgerExtractor`)가 `SystemInstruction: null`로 호출되어 약 400토큰의 정적 규칙과 스키마가 매번 캐시 미스(Cache Miss)를 일으켜 API 비용 및 지연이 누적되고, `Temperature: 0.1f` 하드코딩으로 인해 Flash-Lite 모델에서 무한 토큰 생성 루프 위험 노출.
- Gemini 3.8 Flash 추론 모델의 사고 과정(Hidden Thinking)을 위한 명시적 자가 검증 기준(Verification Criteria)이 `<final_instruction>`에 부재.

### 2. 원인 (Root Cause)
1. `MemoryManagerService.cs`에서 `effectiveText = !string.IsNullOrWhiteSpace(metaDirective) ? $"{userMessage.Text}\n{metaDirective}" : userMessage.Text;`로 단순 문자열 결합 처리.
2. `CognitivePromptAssembler.cs`가 `userActionText` 하나만 전달받아 `<current_action>` 태그 내부에 그대로 렌더링.
3. `EventLedgerExtractor.cs`가 단일 `user` 메시지에 모든 지침과 최근 대화 이력을 연결하여 `SystemInstruction` 캐싱 인프라 미활용.

### 3. 해결책 및 불변식 (Resolution & Invariants)
1. **메타 지시문 독립 데이터 파이프라인 및 순수 유저 임베딩 복원**:
   - `CognitiveContext`에 `MetaDirective` 프로퍼티를 신설하여 상위 서비스에서 메타 지시문을 독립 전달.
   - `MemoryManagerService.cs`에서 순수 `userMessage.Text`로만 768차원 임베딩 및 `loreScanQuery`를 생성하여 쿼리 벡터 희석 원천 차단.
   - `CognitivePromptAssembler.cs`에서 메타 지시문이 존재할 때만 `<current_action>` 앞쪽에 `<meta_directive>` 독립 상위 태그로 조건부 조립(미발생 시 완전 생략).
2. **EventLedgerExtractor SystemInstruction 분리 및 샘플링 최적화**:
   - 무인격 기능 명세, 5대 인과 규칙 및 TypeScript 인터페이스를 `SystemInstruction`으로 물리 분리하여 4턴 주기 KV Cache 100% 적중 달성.
   - `GenerationConfig.Temperature: null`로 교정하여 구글 Gemini 기본 최적화 샘플링 엔진에 위임(원칙 7 준수).
3. **Thinking 자가 검증 기준(Verification Criteria) 보강**:
   - `<final_instruction>`에 `- [자가 검증]: 최종 서사를 출력하기 전, 직전 턴의 인과 관계 및 배경 맥락과 상충되지 않는지 내부적으로 자체 검증하십시오.` 항목 추가.
4. **의무 검증 (Ground Truth)**:
   - `CognitivePromptAssemblerTests`에 `AssemblePrompt_WhenMetaDirectivePresent_RendersIndependentTagBeforeCurrentAction`, `AssemblePrompt_WhenMetaDirectiveNull_OmitsMetaDirectiveTag`, `AssemblePrompt_FinalInstruction_ContainsThinkingSelfVerification` 단위 테스트 추가.
   - 솔루션 전체 빌드(0 warning, 0 error) 및 Fast QA 177개 단위 테스트 100% 통과(Exit Code 0, 1.66s).

---

## 2026-10-17: [Resolved] 단위 테스트 환경에서 종료된 임시 STA 스레드의 Application.Current Dispatcher Invoke 호출로 인한 무한 대기(Deadlock) 해결

### 1. 현상 (Symptom)
- `dotnet test` 전체 테스트 스위트 실행 시 172개 테스트 통과 후 `DialogServiceTests`에서 프로세스가 멈추고 무한 대기(Hang)에 빠지는 결함 발생 (단독 실행 시에는 재현되지 않고 통과).

### 2. 원인 (Root Cause)
- 앞서 실행된 XAML 리소스 검증 테스트(`ThemeResourceTests`)가 임시 STA 스레드에서 `new Application()`을 호출하고 종료됨에 따라, 전역 `Application.Current.Dispatcher`가 이미 종료된 스레드(`Thread.IsAlive == false`)에 바인딩된 상태로 잔존함.
- 뒤이어 xUnit 작업자 스레드에서 `DialogService.RunOnUIThread`가 호출되었을 때, `Application.Current != null` 및 `CheckAccess() == false`로 인해 `dispatcher.Invoke(action)`를 호출했으나, 수신 스레드가 이미 사망하여 메시지 펌프가 돌지 못해 영원히 블로킹됨.

### 3. 해결책 및 불변식 (Resolution & Invariants)
1. **스레드 생존성 및 종료 상태 가드 보강 (`IDialogService.cs`)**:
   - `dispatcher == null || dispatcher.CheckAccess() || !dispatcher.Thread.IsAlive || dispatcher.HasShutdownStarted || dispatcher.HasShutdownFinished` 조건을 추가하여, UI 스레드가 사망했거나 셧다운된 테스트/헤드리스 환경에서는 백그라운드 스레드에서 즉시 안전하게 인라인 실행하도록 방어.
   - 예외 상황 대비 `try-catch` 가드를 추가하여 `Invoke` 실패 시에도 즉시 인라인 폴백 보장.
2. **의무 검증 (Ground Truth)**:
   - 솔루션 전체 빌드(0 warning, 0 error) 및 Fast QA 177개 단위 테스트 100% 통과(Exit Code 0, 1.52s 완료).

---

## 2026-10-17: [Resolved] GRC 런타임 6대 프롬프트(TTS, Lorebook, 상태창, 장기압축, 추천, 초기상황) 전수 최적화 및 Thinking 토큰 고갈 방지 아키텍처 수립

### 1. 현상 및 잠재적 장애 모드 (Symptom & Failure Modes)
- **TTS 실시간 번역 지연 병목**: 일본어 TTS 합성 직전 호출되는 Flash-Lite 번역 요청(`GeminiTtsService.cs`)에 `ThinkingConfig`가 누락되어 기본 사고(Thinking) 토큰이 폭주, 1~2초의 음성 출력 지연 발생 및 비격리 입력으로 인한 인젝션 취약점 노출.
- **초기 0번 턴 모델 스타일 전이 (Style Bleed)**: `MemoryManagerService.InjectInitialScenario`에서 0번 턴 모델 메시지에 `[시스템: 해당 세계관과 초기 상황을 완벽히 인지했습니다...]`라는 기계적 대괄호 문구가 주입되어, LLM이 향후 턴에서 문학적 산문 대신 기계적 시스템 말투를 학습·모방하는 위험 존재.
- **다크 판타지 로어북 추출 차단 위험**: `LorebookService.ExtractMemoryToLorebookAsync`에 `SafetySettings`가 `null`로 호출되어 전투/부상/저주 등 다크 판타지 지식 추출 시 API 필터 블록 및 빈 객체 폴백 위험 노출.
- **Thinking 모델의 MaxOutputTokens 축소 시 JSON 파손 위험**: Gemini 2.0/2.5 Thinking 아키텍처에서 `maxOutputTokens`는 생각 토큰(Thought Tokens)과 가시적 응답 토큰(Visible Output Tokens)의 통합 총합이므로, 이를 1,024~2,048로 강제 축소할 경우 생각 토큰에 예산이 잠식되어 JSON 페이로드가 중간에 잘려 파싱 에러(`LlmJsonParser` 예외)가 발생하는 장애 모드 존재.

### 2. 원인 (Root Cause)
1. 프롬프트 내 데이터 격리(Spotlighting) 및 TypeScript 인터페이스 계약 미비.
2. 백그라운드 단발성 작업(상태창 갱신, 로어북 추출, TTS 번역)에 Thinking 제어가 생략되거나 과도한 수준(`ThinkingLevel.medium`)으로 할당됨.
3. 0번 턴 In-context Demonstration의 모델 역할 발화가 시스템 문구로 오염됨.

### 3. 해결책 및 불변식 (Resolution & Invariants)
1. **Thinking 토큰 총합 상한 보존 및 저비용 추론 가드**:
   - 지연 시간 및 연산 비용 제어는 오직 `ThinkingConfig(ThinkingLevel.low)`로만 통제하며, `MaxOutputTokens`는 8,192의 넉넉한 상한을 유지하여 생각 토큰 소모로 인한 JSON 페이로드 잘림을 원천 방어.
2. **GeminiTtsService 초고속 단발 번역 파이프라인 및 순수 자연어 보존**:
   - `ThinkingConfig: new ThinkingConfig(ThinkingLevel.low)` 및 `MaxOutputTokens: 8192` 적용으로 번역 딜레이를 ~300ms로 단축.
   - **TTS 음성 변조 방어**: 음성 합성 모델(`gemini-3.1-flash-tts-preview`) 및 사전 번역기에 XML 태그나 불릿포인트를 주입하면 음성의 운율(Prosody) 왜곡, 기계음 발생, 태그 음독 등의 결함이 발생하므로, 사전 번역 프롬프트와 음성 합성 프롬프트(`dynamicPrompt`)는 일체의 태그/기호 없이 **100% 순수 자연어 구어체 지시문**으로만 유지.
3. **스타일 전이(Style Bleed) 원천 차단 및 테스트 동기화**:
   - `InjectInitialScenario`의 유저 메시지를 `<initial_scenario>` 태그로 감싸고, 모델 메시지를 자연스러운 서사 앵커(`"주어진 세계관과 초기 상황의 흐름을 받아들여, 첫 발걸음을 뗄 준비를 마쳤습니다."`)로 교체.
   - `MemoryManagerServiceTests.cs`의 단언문 동시 동기화.
4. **LorebookService 세이프티 가드 및 스키마 확립**:
   - `SafetySettings`에 4대 카테고리 `BLOCK_NONE` 적용으로 판타지 전투 서사의 오작동 차단.
   - `ThinkingLevel.low` 적용 및 `<output_format>`에 `interface LorebookEntry` TypeScript 인터페이스 명시.
5. **BuildStatusRequest & CompressToLongTerm & ReplySuggestion 정돈**:
   - `BuildStatusRequest`: `ThinkingLevel.low`로 매 턴 상태창 갱신 딜레이 단축, `<user_action>`, `<narrative_description>`, `<final_instruction>` 태그 표준화.
   - `CompressToLongTermAsync`: `<final_instruction>` 표준화 및 정중한 명령형 통일.
   - `ReplySuggestionService`: `systemInstruction` 내 `type ReplySuggestions = string[];` 스키마 추가.
6. **의무 검증 (Ground Truth)**:
   - 솔루션 전체 빌드(0 warning, 0 error) 및 Fast QA 177개 단위 테스트 100% 통과(Exit Code 0, 1.54s 완료).

---

## 2026-10-17: [Resolved] 로어북 레거시 문자열 직렬화(CurrentLorebookText)의 부분 문자열 오탐(False-Positive) 제거 및 강타입 불변 컬렉션(CurrentActiveLorebooks) 전환

### 1. 현상 (Symptom)
* RAG 인출 단계에서 `MemoryManagerService`가 인출된 로어북 객체 목록(`matchedLorebooks`)을 `CurrentLorebookText`라는 단일 문자열 덩어리로 직렬화해 두고, `ChatViewModel`의 TTS 프리페치 로직에서 프리셋의 전체 로어북($N$개)을 순회하며 `_memoryService.CurrentLorebookText.Contains($"[{l.Name}]")` 형태로 부분 문자열 일치 여부를 매 턴 $O(N \times M)$으로 검사.
* 이로 인해 인물 A의 본문 내용에 동료 B의 이름이 `[동료B]` 형태로 언급되어 있는 경우, 실제 해당 턴에 동료 B가 활성화되지 않았음에도 부분 문자열 검색에 걸려 활성 인물로 오인식(False-Positive)되어 동료 B의 로어북 정보가 TTS 음성 합성 프롬프트로 유입되는 결함 발생.

### 2. 원인 (Root Cause)
* Cognitive Core 3.0 하이브리드 RAG 전환 이전의 레거시 UI 바인딩/단일 텍스트 전달 인터페이스(`CurrentLorebookText`)가 잔존하여, 강타입 도메인 엔티티(`LorebookEntry`)를 비가역적 문자열로 병합했다가 다시 역추적하는 불필요한 연산 및 문자열 침범 위험 야기.

### 3. 해결책 및 불변식 (Resolution & Invariants)
1. **강타입 불변 컬렉션 전환 (`CurrentActiveLorebooks`)**:
   - `IMemoryManagerService` 및 `MemoryManagerService`의 `string CurrentLorebookText`를 완전 퇴출하고 `IReadOnlyList<LorebookEntry> CurrentActiveLorebooks { get; }`로 교체.
   - `BuildNarrativeRequestAsync` 완료 시 `CurrentActiveLorebooks = matchedLorebooks;`로 원자적 참조 스왑(Atomic Reference Swap).
   - 세션 초기화(`Clear()`) 및 로드(`RestoreSession()`) 시 빈 리스트(`[]`)로 무결성 보장.
2. **$O(\text{matched})$ 즉시 필터링 및 False-Positive 원천 차단**:
   - `ChatViewModel`에서 프리셋 전체 순회를 제거하고, `_memoryService.CurrentActiveLorebooks?.Where(l => l.Category == "인물")`로 즉각 필터링하여 문자열 침범 결함 원천 방어.
3. **가짜 목(Fake) 및 단위 테스트 동기화**:
   - `ChatWorkflowServiceTests.cs`의 `FakeMemoryManagerService`에 `CurrentActiveLorebooks` 프로퍼티 구현.
   - `MemoryManagerServiceTests.cs`에 `BuildNarrativeRequestAsync_PopulatesCurrentActiveLorebooks_WithMatchedLorebookObjects` 신규 단위 테스트 추가 및 동기화/초기화 라이프사이클 전수 검증.
4. **의무 검증 (Ground Truth)**:
   - 솔루션 전체 빌드(0 warning, 0 error) 및 Fast QA 178개 단위 테스트 100% 통과 (Exit Code 0).

---

## 2026-10-18: [Resolved] text-embedding-004(AI Studio EOL) 및 002(Vertex 레거시) 대응 차세대 통합 gemini-embedding-001(서울, 768 MRL) 전환 및 RAG 임계치(0.75f, 0.78f) 재보정

### 1. 현상 및 배경 (Symptom & Background)
* **구형 임베딩 모델의 서비스 종료 및 레거시 지정**:
  - Google AI Studio에서 구형 `text-embedding-004`가 2026년 1월 14일부로 서비스 종료(HTTP 404 / Model not found)됨.
  - Vertex AI의 `text-multilingual-embedding-002` 역시 구형 1세대 모델로 지정되어 향후 2027년 퇴역 예정 상태에 진입함.
  - 차세대 모델 `gemini-embedding-2`는 Vertex AI 정식 predict 경로에서 아직 404를 반환하는 프리뷰 단계임.
* **하이브리드 로어북의 이름 기반 누락 위험**:
  - 로어북 엔트리(`LorebookEntry`)에서 키워드(`Keywords`)가 비어 있는 경우, 캐릭터 이름(`Name`)이 본문에 직접 언급되더라도 1차 스파스 정규식 매칭을 건너뛰고 2차 덴스(임베딩) 유사도에만 의존하여 0ms 즉각 인출이 누락될 위험 존재.
* **임베딩 모델 교체 시 유사도 스케일 이동(Scale Shift)**:
  - `gemini-embedding-001`은 한국어 및 다국어 의미론적 표현력이 비약적으로 향상되면서, 기존 002 모델 대비 코사인 유사도 점수 분포가 전반적으로 약 +0.07~+0.09 상향 이동함.
  - 이에 따라 기존 임계치(로어북 0.68f, 렛저 0.71f)를 그대로 유지할 경우 하드 네거티브 및 무관한 일상 대화가 RAG로 과도하게 유입되는 오탐(False-Positive) 위험 발생.

### 2. 실측 벤치마크 및 원인 분석 (Ground Truth Benchmarks & Root Cause)
1. **엔드포인트 및 레이턴시 실측**:
   - `gemini-embedding-001`은 서울(`asia-northeast3`) 리전에서 안정적으로 `HTTP 200 OK` (약 570ms)로 정상 응답함.
   - Vertex AI와 Google AI Studio 양쪽 모두 `gemini-embedding-001` 단일 모델명으로 통일 지원됨.
2. **MRL(Matryoshka Representation Learning) 차원 축소 손실률 검증**:
   - `gemini-embedding-001`의 기본 3,072차원 대비 `outputDimensionality: 768` 지정 시, 코사인 유사도 차이는 `-0.006 ~ -0.010` (약 0.6%~1.1% 편차)에 불과하여 RAG 검색 품질의 손실 없이 C# 메모리 및 벡터 연산 효율(768-dim float)을 100% 보존함을 실측 입증.
3. **10문장 정탐 vs 비교군 분리 마진(Margin) 정밀 측정**:
   - **사건 불릿 렛저 RAG (부둣가 동맹 맹세)**:
     - 구 002 모델: 정탐 최저 0.7093 vs 비교군 최고 0.6822 $\implies$ 마진 `+0.0271`
     - 신 001 모델(768차원): 정탐 최저 0.7966 vs 비교군 최고 0.7488 $\implies$ 마진 `+0.0478` (마진 76% 대폭 향상)
     - $\implies$ **신규 임계치: `0.78f`** (비교군 0.7488을 완벽 차단하고 정탐 0.7966 이상 전원 합격).
   - **로어북 RAG (황녀 엘리시아)**:
     - 구 002 모델: 정탐 최저 0.7061 vs 비교군 최고 0.6401 $\implies$ 마진 `+0.0660`
     - 신 001 모델(768차원): 정탐 최저 0.7785 vs 비교군 최고 0.7196 $\implies$ 마진 `+0.0589`
     - $\implies$ **신규 임계치: `0.75f`** (비교군 0.7196을 완벽 차단하고 정탐 0.7785 이상 전원 합격).

### 3. 해결책 및 불변식 (Resolution & Invariants)
1. **차세대 통합 임베딩 모델 일원화 (`GeminiApiService.cs`, `GeminiApiDto.cs`)**:
   - Vertex AI: `asia-northeast3` 리전 `publishers/google/models/gemini-embedding-001:predict` 엔드포인트에 `outputDimensionality: 768` 파라미터 주입.
   - AI Studio: `v1beta/models/gemini-embedding-001:embedContent` 엔드포인트에 `outputDimensionality: 768` 주입.
   - C# 인터페이스 계약(`IGeminiApiService.cs`): `gemini-embedding-001, MRL 768차원` 명시.
2. **하이브리드 로어북 이름(Name) 1차 스파스 매칭 보강 (`HybridLorebookRetriever.cs`)**:
   - `entry.Keywords`가 비어 있거나 누락되어도 `entry.Name`에 대한 정규식 단어 경계(`\b`) 매칭을 기본 수행하여, 이름 등장 시 0ms 즉각 인출 보장.
   - 스파스와 덴스 양쪽에서 중복 인출된 경우 `entry.Name` 기준으로 자동 중복 제거(`DistinctBy`).
3. **RAG 임계치 전면 재보정**:
   - `GatedRAGRetriever.cs`: `SemanticSimilarityThreshold = 0.78f`
   - `HybridLorebookRetriever.cs`: `DenseThreshold = 0.75f`
4. **SSOT 문서 및 테스트 동기화**:
   - 단위 테스트(`HybridLorebookRetrieverTests`, `GatedRAGRetrieverTests`, `VectorMathHelperTests`) 및 통합 테스트(`CognitiveCoreIntegrationTests`)의 단언문과 임계치를 0.75f / 0.78f로 전수 동기화.
   - 기존 레거시 모델 탐색 스크래치 코드를 정리하고, 2대 핵심 RAG 20개 문장 실측 분리도 테스트(`RAG_SemanticSimilarity_RealEmbedding_EvaluationWithParaphrasedInputs`)를 100% 자동 통과하도록 보장.
5. **불변식 (Invariants)**:
   - **임베딩 모델은 반드시 `gemini-embedding-001` (서울 asia-northeast3 리전) 및 `outputDimensionality: 768`을 사용할 것.**
   - **로어북 시맨틱 임계치는 `0.75f`, 사건 렛저 RAG 임계치는 `0.78f`를 유지할 것.**
6. **의무 검증 (Ground Truth)**:
   - 솔루션 전체 빌드: 경고 0개, 오류 0개 (Exit Code 0).
   - Fast QA 180개 단위 테스트 100% 통과 (Exit Code 0, 1.98s).
   - Vertex AI 실서버 통합 QA 10개 테스트 100% 통과 (Exit Code 0, 36.56s).

