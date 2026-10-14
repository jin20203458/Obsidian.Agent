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

---

## 2026-10-14: [Resolved] 설정 화면 내 LLM 엔드포인트 실시간 연결 시험(Test Connection) 부재 및 통신 가용성 사전 검증 결여

### 1. 현상 (Symptom)
* 사용자가 API Key 오타, Project ID 누락, 비인가 Service Account 키 파일 등록 등 설정 오류를 범하더라도 설정 화면에서 이를 즉각 검증할 방법이 없음.
* 실제 롤플레잉 세션을 시작하고 첫 대화를 전송한 뒤에야 통신 예외가 발생하여 사용자 경험이 훼손되고 디버깅 시간이 지연되는 결함 발생.

### 2. 원인 (Root Cause)
1. `SettingsViewModel`에 엔드포인트 가용성 및 RTT 지연시간을 사전 점검하는 비동기 핑(Ping) 로직이 결여되어 있었음.
2. Google Cloud Vertex AI 및 AI Studio의 인증 토큰 유효성, Project ID 정합성, 모델명 지원 여부를 실시간 진단할 UI 피드백 파이프라인 부재.

### 3. 해결책 (Resolution)
1. **`Phalanx.Cockpit` 아키텍처 이식 (`TestConnectionCommand`)**: `SettingsViewModel.cs`에 15초 타임아웃 기반 초경량 핑 호출(`"Respond with 'OK'."`, maxOutputTokens: 16) 로직을 탑재.
2. **듀얼 모드 실시간 분기 진단**:
   * **Vertex AI**: Project ID 및 키 파일 존재 여부 검사 -> OAuth2 Bearer 토큰 획득 -> `global` 엔드포인트 호출.
   * **AI Studio**: API 키 유효성 검사 -> `generativelanguage.googleapis.com` 엔드포인트 호출.
3. **지연시간 및 구글 API 에러 메시지 상세 추출**: `Stopwatch` 기반 왕복 시간(ms) 측정 및 비정상 응답 시 Google API 에러 JSON에서 `error.message`를 추출하여 사용자에게 구체적 조치 방안 노출.
4. **UI 바인딩 및 상태 피드백**: [SettingsView.xaml](file:///c:/Users/user/Documents/GitHub/GRC/GRC/Views/SettingsView.xaml)에 진행 상태, 비활성화 가드(`CanExecute`), 상태별 전용 색상 피드백(성공 녹색, 실패 적색, 진행 황색) 패널 배치.

### 4. 검증 (Verification)
* `tests/GRC.Tests/SettingsViewModelTests.cs` 전용 5개 테스트 작성 및 검증 통과:
  - Vertex AI 모드 Project ID 누락 방어 (`FAIL: Project ID 미입력`) 검증.
  - Vertex AI 키 파일 미등록 방어 (`FAIL: google-credentials.json 미등록`) 검증.
  - AI Studio API 키 누락 방어 (`FAIL: Gemini API 키 미입력`) 검증.
  - Mock HTTP 200 응답 시 PASS 판정, 전역 리전 표기 및 지연시간 측정 검증.
  - Mock HTTP 403 응답 시 구글 에러 JSON 본문 메시지(`API key not valid`) 추출 검증.
* 솔루션 전체 빌드(`dotnet build GRC.sln`) 및 테스트 스위트 29/29 통과 (**Exit Code 0**).

---

## 2026-10-14: [Resolved] Google Cloud TTS 음성 캐릭터 하드코딩 해제, 동적 보이스 선택 및 오디오 미리듣기(Preview) 기능 구현

### 1. 현상 (Symptom)
* Google Cloud TTS 음성 합성이 `GeminiTtsService.cs` 내부의 `"Zephyr"` 단일 보이스로 하드코딩되어 있어 사용자가 원하는 캐릭터 목소리(성별, 톤, 음색)를 선택할 수 없음.
* 설정 화면(`SettingsView.xaml`)에 TTS 음성을 사전에 테스트하고 청취할 수 있는 미리듣기(Preview) 기능이 부재하여, 실제 인게임 대사 발화 전까지 음성 정상 동작 여부나 음색을 확인할 수 없는 문제 발생.

### 2. 원인 (Root Cause)
1. `GeminiTtsService.GenerateVoiceAsync` 호출 시 요청 바디의 `voiceName` 필드가 상수로 고정되어 있었고, `AppSettings` 모델에 음성 선택 저장 프로퍼티(`SelectedTtsVoice`)가 정의되지 않았음.
2. `IGoogleTtsService` 인터페이스에 개별 보이스 오버라이드 및 테스트용 음성 미리듣기 비동기 메서드가 결여되어 있었음.
3. `SettingsViewModel`에 사용 가능한 14대 구글 표준 TTS 보이스 메타데이터 목록 및 미리듣기 실행 커맨드(`TestTtsCommand`), 상태 통지 프로퍼티가 구현되지 않았음.

### 3. 해결책 (Resolution)
1. **TTS 보이스 모델 신설 및 영속화**:
   * `GRC/Models/TtsVoiceOption.cs` 레코드를 신설하여 보이스 고유 ID, 한글 표시명 및 음색/성별 상세 설명(`Description`)을 캡슐화.
   * `AppSettings.cs`에 `SelectedTtsVoice = "Zephyr"` 필드를 추가하여 선택된 보이스를 설정 파일에 영속화.
2. **동적 보이스 오버라이드 및 미리듣기 파이프라인 구현**:
   * `IGoogleTtsService.cs` 및 `GeminiTtsService.cs`에 `voiceOverride` 및 `languageOverride` 매개변수를 지원하도록 확장.
   * `PreviewSpeechAsync(text, voiceOverride, languageOverride, ct)` 메서드를 신설하여 단문 샘플 음성을 생성하고 임시 파일 경로를 반환하도록 구현.
3. **설정 UI 연동 및 실시간 재생 피드백**:
   * `SettingsViewModel.cs`에 `AvailableTtsVoices`(Zephyr, Achernar, Aoede, Autonoe, Callirrhoe, Despina, Erinome, Gacrux, Kore, Laomedeia, Leda, Pulcherrima, Sulafat, Vindemiatrix 등 14종) 컬렉션 및 `SelectedTtsVoiceOption` 바인딩 속성 구성.
   * `TestTtsCommand`를 통해 `PreviewSpeechAsync` 호출 -> `IAudioService.PlayVoiceSoundAsync`로 비동기 오디오 재생 -> 상태 메시지 및 성공/실패 색상 피드백 파이프라인 탑재.
   * `SettingsView.xaml`에 2줄 형태(보이스명 + 음색 설명)의 ComboBox 및 "음성 미리듣기" 버튼/상태 패널 배치.

### 4. 검증 (Verification)
* `tests/GRC.Tests/SettingsViewModelTests.cs`에 신규 TTS 테스트 케이스 4종 추가 검증 통과:
  - `TestTtsAsync_Success_PlaysVoiceSoundAndSetsStatusMessage`: 선택한 보이스(`Aoede`) 및 언어로 미리듣기 생성, 오디오 플레이어 호출, 초록색 완료 메시지 검증.
  - `TestTtsAsync_WhenTtsDisabled_CanTestTtsReturnsFalse`: TTS 비활성화 시 커맨드 실행 가드(`CanExecute = false`) 검증.
  - `TestTtsAsync_Failure_SetsErrorMessage`: TTS 실패 시 빨간색 에러 메시지 갱신 검증.
  - `LoadSettingsAsync_LoadsVoiceOptionCorrectly`: 설정 파일 로드 시 저장된 보이스(`Sulafat`)가 ViewModel 옵션 객체로 정확히 매핑되는지 검증.
* 솔루션 전체 빌드(`dotnet build GRC.sln`) 및 테스트 스위트 33/33 전량 통과 (**Exit Code 0**).

---

## 2026-10-14: [Resolved] XAML 버튼 호버(MouseOver) 시 Windows Aero 테마 점등 및 시맨틱 색상 변색 결함 해결

### 1. 현상 (Symptom)
* 다크 테마 기반 데스크톱 클라이언트의 다수 버튼(`SettingsView.xaml`, `CustomMessageBoxWindow.xaml`, `EditInitialScenarioWindow.xaml`, `EditWorldviewWindow.xaml`, `NewSessionSetupWindow.xaml`, `SessionListView.xaml`)에 마우스를 올릴 때, 지정된 배경색이 무시되고 Windows OS 기본 테마(`PresentationFramework.Aero2`)의 밝은 하늘색(#BEE6FD) 유리 질감 그라데이션이 점등되는 UI 파손 발생.
* `SessionArchitectWindow.xaml`의 '승인, 다음 단계'(에메랄드 그린 `#142F24`) 및 '중단'(크림슨 레드 `#421A1A`) 버튼에 마우스를 올리면 `HeaderActionButton` 템플릿의 하드코딩된 트리거로 인해 고유 색상이 지워지고 칙칙한 진회색(`#353535`)으로 강제 변색되는 결함 발생.
* 버튼 비활성화(`IsEnabled="False"`) 시 투명도 감쇄 등의 시각 피드백이 부재하고 `Cursor="Hand"` 설정이 누락된 버튼 다수 존재.

### 2. 원인 (Root Cause)
1. `Themes/ModernStyles.xaml`에 전역 기본 암시적 스타일(`<Style TargetType="Button">`)이 누락되어 있어, 인라인으로 배경색만 준 버튼들이 WPF 시스템 테마 템플릿으로 fallback됨.
2. `HeaderActionButton`의 `ControlTemplate.Triggers`가 `TargetName="border"`의 `Background`를 `#353535` 단일 상수로 강제 치환하여, 상속받은 버튼의 고유 색상(녹색/적색)을 파괴함.
3. 시맨틱 버튼 스타일 계층(Primary, Success, Danger, Secondary)이 체계화되지 않아 각 뷰마다 인라인 하드코딩이 파편화됨.

### 3. 해결책 (Resolution)
1. **버튼 디자인 시스템 3계층 아키텍처 정립 (`ModernStyles.xaml`)**:
   * **BaseButtonStyle**: 통합 ControlTemplate, `CornerRadius="4"`, `Cursor="Hand"`, `IsPressed`(Opacity 0.75), `IsEnabled`(Opacity 0.40) 상태 머신 내장.
   * **글로벌 암시적 스타일 (`<Style TargetType="Button" BasedOn="{StaticResource BaseButtonStyle}" />`)**: 앱 전체에 암시적 적용하여 향후 추가되는 모든 버튼의 Windows Aero fallback 원천 차단.
   * **시맨틱 변형 스타일 확립**:
     - `PrimaryActionButton`: 청록색 포인트 (`#008B99` -> `#00A5B5` 호버, `#33E1F5` 보더)
     - `SuccessActionButton`: 에메랄드 그린 (`#142F24` -> `#1D4333` 호버, `#3EBA8F` 보더)
     - `DangerActionButton`: 크림슨 레드 (`#421A1A` -> `#5A2323` 호버, `#C24848` 보더)
     - `SecondaryActionButton`: 뉴트럴 다크 그레이 (`#252528` -> `#35353A` 호버, `#525258` 보더)
     - `AccentBlueActionButton`: 온보딩 블루 (`#2563EB` -> `#3B82F6` 호버)
     - `HeaderActionButton`: `SecondaryActionButton` 상속 기반 콤팩트 패딩(`8,4`) 하위 호환성 유지.
     - `GhostButton`: 투명 버튼 전용 `Cursor="Hand"` 및 `IsEnabled` 상태 피드백 보강.
2. **뷰 전수조사 및 17개 인라인 버튼 일괄 리팩터링**:
   * `SettingsView.xaml`, `CustomMessageBoxWindow.xaml`, `EditInitialScenarioWindow.xaml`, `EditWorldviewWindow.xaml`, `NewSessionSetupWindow.xaml`, `SessionArchitectWindow.xaml`, `SessionListView.xaml`의 인라인 하드코딩 속성을 제거하고 시맨틱 스타일로 표준화.

### 4. 검증 (Verification)
* 전수조사 스크립트 실행으로 13개 XAML 내 총 33개 버튼 전량의 스타일 바인딩 상태 확인 (미스타일 0건, 인라인 배경 충돌 0건).
* 단위 테스트 스위트 전량 통과 (`dotnet test tests/GRC.Tests/GRC.Tests.csproj` - 33/33 Pass, **Exit Code 0**).
* 솔루션 전체 빌드 검증 (`dotnet build GRC.sln` - 0 Warning, 0 Error, **Exit Code 0**).


