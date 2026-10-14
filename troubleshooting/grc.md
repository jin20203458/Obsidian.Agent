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
