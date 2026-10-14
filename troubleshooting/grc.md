---
description: GRC(GenAI Roleplay Chat) C# WPF 데스크톱 클라이언트 트러블슈팅 런북.
related:
  - ../README.md
  - ../GRC/README.md
---
# GRC Troubleshooting

본 문서는 GRC(GenAI Roleplay Chat) 데스크톱 클라이언트(C#/.NET 8.0 WPF, CommunityToolkit.Mvvm, Google Gemini API, Vertex AI TTS, System.Threading.Channels 스트리밍 및 O(1) 파일 로거) 개발 및 운영 중 발생하는 시스템 예외, 동시성 버그, 라이브러리 충돌 및 트러블슈팅 내역을 누적하여 관리하는 중앙 기술 런북입니다.

---

## 사전 주의사항 및 알려진 기술적 제약 (Known Constraints)

### 1. WPF UI 스레드 (STA) 비차단 불변식
* **현상**: 대규모 LLM 토큰 스트리밍 또는 디스크 쓰기 작업 시 UI 스레드 블로킹으로 인한 창 멈춤(Freezing) 발생 가능성.
* **대응책**: 모든 API 호출 및 I/O는 `System.Threading.Channels` 기반 백그라운드 큐로 오프로드하고, UI 렌더링은 디스패처 배치 업데이트를 통해서만 수행.

---

## 2026-10-14: [Resolved] Vertex AI 최신 주력 모델의 특정 리전 엔드포인트 404 오류 및 글로벌 엔드포인트 단일화

### 1. 현상 (Symptom)
* Google Cloud Vertex AI 연동 시 UI 설정에서 특정 로컬 리전(예: `asia-northeast3`, `us-central1`)을 지정한 상태에서 최신 주력 모델(`gemini-3.8-flash`, `gemini-4-argon`)을 호출할 경우 `404 Not Found` 또는 리전 미지원 예외 발생.
* 사용자가 설정창에서 리전(`Location`) 값을 직접 입력하거나 임의 변경하여 세션 통신이 마비되는 혼선 발생.

### 2. 원인 (Root Cause)
1. Google Cloud Vertex AI의 서비스 정책상 최신 주력 모델(Gemini 3.x / 4.x)은 전역 엔드포인트(`locations/global`)에서만 우선 서비스되며, 로컬 리전에는 배포되지 않음.
2. 기존 GRC 클라이언트는 `AppSettings.Location` 값을 사용자 입력에 의존하여 URL을 동적 구성하도록 설계되어 있었음 (단, Vertex AI TTS 음성 모델의 경우 `us-central1` 등 특정 리전에서만 지원됨).

### 3. 해결책 (Resolution)
1. **UI 및 설정 리전 필드 완전 제거**: `SettingsView.xaml`, `SettingsViewModel.cs`, `AppSettings.cs`에서 `Location` 입력 컨트롤과 설정 프로퍼티를 완전히 제거하여 사용자 혼선을 원천 방지.
2. **LLM 엔드포인트 전역(Global) 고정**: `GeminiApiService.cs`에서 LLM 추론 엔드포인트를 `aiplatform.googleapis.com/.../locations/global/...` 단일 엔드포인트로 하드코딩 고정.
3. **TTS 음성 모델 전용 리전 분리**: `GeminiTtsService.cs`는 Google Cloud TTS 지원 리전인 `us-central1`로 내부 자동 고정.
4. **활성 4대 모델 표준화**: 지원 종료 대상 레거시 모델을 정리하고 `gemini-3.8-flash`, `gemini-3.5-flash`, `gemini-3.5-flash-lite`, `gemini-4-argon`의 4대 모델로 시스템 전반을 표준화.

### 4. 검증 (Verification)
* `tests/GRC.Tests/ModelVerificationTests.cs`를 구현하여 실제 Google Cloud Service Account 토큰 기반으로 활성 4개 모델에 대한 Global 엔드포인트 호출 검증: 전 모델 정상 응답(Pass) 확인.
* 비활성/폐기 모델(`gemini-3.6-flash`, `gemini-3.1-pro-preview`) 차단 검증 통과.

---

## 2026-10-14: [Resolved] LlmJsonParser의 사전 잡담 괄호 오인식, 문자열형 숫자/Enum 역직렬화 실패 및 스트리밍 절단 복구

### 1. 현상 (Symptom)
* LLM이 JSON 페이로드 앞뒤에 설명용 템플릿 괄호(예: `Format {schema_template}. Payload: {"name": "Hero"}`)를 출력할 때 단순 `rawText.IndexOf('{')` 방식이 `{schema_template}`을 JSON 시작점으로 오인하여 역직렬화 실패(`null` 반환).
* JSON 문자열 내부에 큰따옴표와 괄호가 포함된 경우(예: `"instruction \"{task}\": {"id": 1}"`), 따옴표 내부의 괄호가 파서의 시작점으로 잘못 인식됨.
* LLM이 숫자 필드를 큰따옴표로 감싸서 반환하거나(`"count": "42"`, `"ratio": "0.75"`), DTO의 `enum`을 문자열(`"Planning"`)로 반환할 때 `System.Text.Json.JsonException` 발생.
* `DeserializeSafe<T>` 매 호출마다 `new JsonSerializerOptions`를 생성하여 힙 메모리 할당 및 리플렉션 캐시 재생성 오버헤드 발생.
* 긴 서술문이나 배열 출력 도중 토큰 제한(max_tokens)이나 네트워크 끊김으로 JSON이 중간에 절단(Truncated)되면 파싱이 불가능하여 전체 데이터가 유실됨.

### 2. 원인 (Root Cause)
1. 괄호 계수기(bracket-counter)가 인덱스 0부터 문자열 리터럴/이스케이프(`\"`) 상태를 추적하지 않고 단순 `IndexOf('{')`로 개시 지점을 탐색하여 사설 내부 괄호에 취약함.
2. `JsonSerializerOptions`에 `JsonNumberHandling.AllowReadingFromString` 및 `JsonStringEnumConverter`가 등록되지 않았고, 제네릭 제약이 `where T : class`로 고정되어 struct/record 지원이 불가능했음.
3. 미완성 스트림에 대한 LIFO 괄호 닫기 및 불완전 trailing 토큰(`"key":`, `,`) 정제 복구 로직이 부재함.

### 3. 해결책 (Resolution)
1. **`LenientOptions` 정적 캐싱**: `AllowReadingFromString`, `AllowTrailingCommas`, `ReadCommentHandling = Skip`, `JsonStringEnumConverter`를 사전 구성하여 정적 캐싱하고 제네릭 제약 조건(`where T : class`)을 완화하여 struct, record 역직렬화 지원.
2. **마크다운 코드블록 우선 파싱**: ```` ```json ```` 블록을 우선적으로 검출하여 본문 외부의 서술형 잡담을 원천 배제.
3. **Index 0 기준 문자열 리터럴 추적 및 다중 후보군(Candidate) 순차 역직렬화**: 문자열 내부의 괄호 오인식을 방지하고, 설명용 템플릿 괄호가 선행하더라도 구문/스키마 유효성을 순차 검증하여 실제 페이로드를 역직렬화.
4. **`RepairTruncatedJson` 복구 엔진 탑재**: 절단된 응답 감지 시 열린 따옴표를 닫고, dangling 키/콜론/쉼표를 정제한 후 스택에 보존된 여는 괄호의 짝(`]`, `}`)을 LIFO 역순으로 닫아 구조적 유효성을 복원.

### 4. 검증 (Verification)
* `tests/GRC.Tests/LlmJsonParserTests.cs` 17개 유닛 테스트 케이스 전량 통과:
  - 정상 객체/배열 추출, 마크다운 코드블록 파싱, 이스케이프 및 중첩 괄호 보존.
  - 사전 잡담 괄호 무시 및 실제 페이로드 선택.
  - 문자열형 숫자 변환, 대소문자 무관 Enum 파싱, 주석 및 trailing comma 허용.
  - struct 및 record 타입 역직렬화.
  - 잘린 JSON 객체 및 배열 복구 역직렬화.
* 솔루션 전체 빌드(`dotnet build GRC.sln`) 및 테스트 스위트 24/24 통과 (**Exit Code 0**).
