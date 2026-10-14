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
4. **UI 바인딩 및 상태 피드백**: [SettingsView.xaml](../../GRC/GRC/Views/SettingsView.xaml)에 진행 상태, 비활성화 가드(`CanExecute`), 상태별 전용 색상 피드백(성공 녹색, 실패 적색, 진행 황색) 패널 배치.

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

---

## 2026-10-14: [Resolved] XAML 벡터 아이콘 지오메트리 아키텍처 구축 및 장식용 이모지 전면 제거

### 1. 현상 (Symptom)
* UI 전반에 장식용 유니코드 이모지(`🔄`, `📋`, `🗑`, `🌱`, `💾`, `🔑`, `📁`, `⚙`, `✨`, `🎭`, `💬`, `💭`, `📍`, `❖`, `▣`, `🔹`, `✕`, `▶`, `+`, `←`)가 인라인 `TextBlock` 또는 `Button.Content`로 혼용되어 데스크톱 테마 일관성을 저해하고 OS/폰트 렌더러에 따라 색상 왜곡 및 픽셀 뭉개짐 발생.
* 인라인 SVG 문자열(`M12,2L14.6...`, `M2.01 21L...`)이 뷰 파일마다 중복 정의되어 유지보수성이 저하되고 전역 스타일 변경에 취약함.
* 가이드라인([Modern_AI_Desktop_UI_Guidelines.md](../ai_agent_refs/Modern_AI_Desktop_UI_Guidelines.md) §4)에서 규정한 "순수 XAML `<Geometry>` 벡터 리소스 사전 및 장식용 이모지 제로 원칙"에 미달함.

### 2. 원인 (Root Cause)
1. 중앙 벡터 아이콘 리소스 사전(`Themes/VectorIcons.xaml`)의 부재로 인해 각 뷰 개발 시 손쉬운 이모지 텍스트나 인라인 중복 Path 문자열로 임시 구현됨.
2. `App.xaml`에서 공용 벡터 지오메트리를 `MergedDictionaries`로 등록하지 않아 컴포넌트 간 재사용 불가능.

### 3. 해결책 (Resolution)
1. **중앙 집중식 벡터 아이콘 리소스 사전 구축 (`Themes/VectorIcons.xaml`)**:
   * 외부 이미지 파일(PNG), 글꼴(Font Awesome) 의존성을 완전 배제하고 HiDPI 환경에서도 선명한 순수 `<Geometry x:Key="Icon...">` 27종 정의:
     - 범용 액션: `IconPlay`, `IconStop`, `IconStopSquare`, `IconTerminal`, `IconCheck`, `IconClose`, `IconPlus`, `IconAlertTriangle`, `IconCopy`, `IconRefresh`, `IconSearch`, `IconChevronDown`, `IconChevronRight`, `IconChevronLeft`, `IconArrowLeft`, `IconTool`, `IconSparkles`, `IconActivity`, `IconUser`, `IconShield`
     - 특화 도메인: `IconSend`, `IconEdit`, `IconBook`, `IconHistory`, `IconSettings`, `IconKey`, `IconFolder`, `IconTrash`, `IconBranch`, `IconSave`, `IconMessageSquare`, `IconBrain`, `IconHourglass`, `IconMapPin`, `IconDiamond`, `IconSquare`
2. **`App.xaml` 리소스 통합 등록**:
   * `Application.Resources`의 `MergedDictionaries` 최상단에 `Themes/VectorIcons.xaml`을 링크하여 모든 뷰/컨트롤에서 `{StaticResource Icon...}` 단일 심볼로 즉시 참조 가능하도록 확립.
3. **7개 XAML 뷰 및 테마 전수조사 및 일괄 치환**:
   * `Themes/ModernStyles.xaml`: 우클릭 컨텍스트 메뉴 내 5대 액션(재전송, 복사, 삭제, 세션 분기, 기억 저장) 이모지를 단일 컬러 벡터 패스로 전면 교체.
   * `Views/SettingsView.xaml`: `← 뒤로` 및 `🔑`, `📁` 이모지를 `IconChevronLeft`, `IconKey`, `IconFolder`로 교체.
   * `Views/SessionListView.xaml`: 헤더 설정 버튼 텍스트 제거 및 `IconSettings` 단독 아이콘 버튼화(`ToolTip="설정"`), `+ 새 대화 시작` -> `IconPlus`, 아키텍트 중복 Path 제거, 웰컴 카드(`🎭`, `⚙`, `✨`), 세션 삭제 버튼(`✕`)을 `{StaticResource Icon...}`으로 교체.
   * `Views/ChatView.xaml`: 상단 우측 '상태창' 및 '비우기' 버튼을 중앙 도구들과 동일한 무테두리/투명 배경 `GhostButton` 스타일로 통일하고, 기존 텍스트 색상과 동일한 차분한 뉴트럴 그레이(`#AAAAAA`)의 `IconShield`(RPG 방패/스탯) 및 `IconTrash`(휴지통) 16x16 순수 아이콘으로 교체(`ToolTip` 연동), 상단 세션 도구 및 하단 입력창 컨트롤을 순수 지오메트리 패스로 교체.
   * `Views/SessionArchitectWindow.xaml`: 타이틀바 스파클, 승인 버튼(`▶`), 자동 실행 토글(`↻`)을 `IconSparkles`, `IconPlay`, `IconRefresh`로 교체.
   * `Views/StatusWindow.xaml`: 위치(`📍`), 커스텀 스탯(`❖`), 상태(`▣`), 인벤토리 항목(`🔹`)을 `IconMapPin`, `IconDiamond`, `IconSquare`, `IconChevronRight`로 교체.
   * `Views/EditWorldviewWindow.xaml` & `Views/NewSessionSetupWindow.xaml`: 스탯 추가(`+`) 및 스탯 삭제(`✕`) 버튼을 `IconPlus` 및 `IconClose` 벡터 패스로 치환.

### 4. 검증 (Verification)
* 전체 XAML 코드 정규식 스캔 결과: XML 주석을 제외한 XAML 코드/텍스트 내 장식용 이모지 및 유니코드 특수문자 **0건 (완전 소멸)** 확인.
* 단위 테스트 스위트 전량 통과 (`dotnet test tests/GRC.Tests/GRC.Tests.csproj` - 33/33 Pass, **Exit Code 0**).
* 솔루션 전체 빌드 검증 (`dotnet build GRC.sln` - 0 Warning, 0 Error, **Exit Code 0**).

---

## 2026-10-14: [Resolved] WindowChrome 기반 일체형 커스텀 셸 및 Windows 11 Snap Layouts 연동 구축

### 1. 현상 (Symptom)
* 메인 윈도우(`MainWindow.xaml`)가 Windows OS 표준 네이티브 캡션 바를 사용하고 있어, 내부 다크 테마 캔버스와 상단 타이틀바 사이에 시각적 단절감이 존재하고 프레임리스(Frameless) 현대적 AI 클라이언트 룩앤필에 미달함.
* 커스텀 타이틀바 도입 시 발생할 수 있는 최대화 시 화면 밖 잘림(Taskbar Over-spill) 및 Windows 11 Snap Layouts 플라이아웃(최대화 버튼 호버 분할 메뉴) 무반응 우려 존재.

### 2. 해결책 (Resolution)
1. **WindowChrome 셸 프레임워크 탑재 (`MainWindow.xaml`)**:
   * `WindowStyle="SingleBorderWindow"` 유지로 네이티브 DWM 윈도우 그림자 및 둥근 모서리 보존.
   * `shell:WindowChrome` 속성 정의: `CaptionHeight="36"`, `GlassFrameThickness="-1"`, `ResizeBorderThickness="6"`, `UseAeroCaptionButtons="False"`.
   * 최상단 36px 높이의 일체형 커스텀 캡션 바 구축: 좌측 공식 고해상도 앱 아이콘(`/Resources/GRC.ico` 16x16) + 앱 타이틀 브랜딩("GRC" + "GenAI Roleplay Chat"), 중앙 드래그 가능 캡션 영역, 우측 시스템 창 제어 버튼군(최소화, 최대화, 닫기) 배치.
   * 닫기 버튼 전용 Windows 표준 레드 호버 액센트(`Background="#E81123"`) 적용.
2. **Win32 DWM, Snap Layouts 및 시스템 메뉴 메시지 훅 (`MainWindow.xaml.cs`)**:
   * `DWMWA_USE_IMMERSIVE_DARK_MODE` 및 `DWMWCP_ROUND` 연동.
   * Windows 11 22H2+ Mica/Backdrop 투명화 파이프라인 탑재 (`DWMSBT_MAINWINDOW`).
   * `WM_NCHITTEST` 트랩: `MaximizeButton.PointFromScreen`으로 로컬 DIP 좌표 변환 후 `HTMAXBUTTON`(9)을 반환하여 Windows 11 네이티브 Snap Layouts 플라이아웃 메뉴 완벽 활성화.
   * `WM_NCLBUTTONDOWN` / `WM_NCLBUTTONUP` 메시지 중계 및 `ToggleMaximize` 핸들러 연동.
   * `StateChanged` 이벤트에 따른 최대화(`☐`) / 복원(`M3 1h6v6H9 M1 3h6v6H1z`) 아이콘 동적 전환 및 툴팁 갱신.
   * 좌측 공식 앱 아이콘 클릭 시 Windows 시스템 메뉴(복원/이동/크기/최소화/최대화/닫기) 호출 및 더블클릭 시 창 닫기 핸들러 연동. 타이틀바 우클릭 시스템 메뉴 호출 연동.

### 3. 검증 (Verification)
* 단위 테스트 스위트 전량 통과 (`dotnet test GRC.sln` - 33/33 Pass, **Exit Code 0**).
* 솔루션 전체 빌드 검증 (`dotnet build GRC.sln` - 0 Warning, 0 Error, **Exit Code 0**).

---

## 2026-10-14: [Resolved] WindowChrome 커스텀 타이틀바 내 앱 아이콘 미표시 및 레이아웃 절단 버그 해결

### 1. 현상 (Symptom)
* `MainWindow.xaml`에 커스텀 타이틀바를 적용한 후, 좌측 상단 시스템 메뉴 영역에 배치한 공식 앱 아이콘(`GRC.ico`)이 화면에 렌더링되지 않고 공백으로 남는 현상 발생.

### 2. 원인 (Root Cause)
1. **상대 URI 해석 경로 오류**: WPF XAML에서 `<Image Source="/Resources/GRC.ico" />`로 선언 시, BAML 컴파일 컨텍스트에 따라 pack URI가 아닌 로컬 파일시스템 루트(`C:\Resources\GRC.ico`)로 오인식되어 `DirectoryNotFoundException` 유발.
2. **글로벌 암시적 버튼 스타일(Padding) 간섭에 의한 너비 절단**: `Themes/ModernStyles.xaml`의 `BaseButtonStyle`이 전역 `Style TargetType="Button"`으로 적용되어 `Padding="12,6"`을 강제 상속함. 너비 22px인 아이콘 버튼 내부에서 좌우 패딩 24px(12+12)이 차감되어 내부 `ContentPresenter`의 가용 폭이 `-2px`로 음수 붕괴하며 16x16 아이콘이 강제 절단(Clipped to 0px).
3. **Button의 마우스 다운 이벤트 삼킴(Swallowing)**: 기본 WPF `Button` 클래스가 내부 `OnMouseLeftButtonDown`에서 `e.Handled = true`를 설정하여 `MouseLeftButtonDown="AppIcon_MouseLeftButtonDown"` 시스템 메뉴 호출 핸들러가 단일 클릭 시 격발되지 않음.

### 3. 해결책 (Resolution)
1. **정규 Pack URI 명시 (`pack://application:,,,/GRC;component/Resources/GRC.ico`)**: 어셈블리 리소스 식별자를 명시한 완전한 Pack URI로 변경하여 런타임 BAML 및 리소스 로더가 안정적으로 `GRC.ico`를 디코딩하도록 확립.
2. **`Border` 기반 경량 컨테이너 전환 및 시인성 개선**: 암시적 `Button` 스타일 및 패딩 상속을 원천 차단하고 `Border`(`Width="28"`, `Height="28"`, `CornerRadius="4"`, `Cursor="Hand"`)에 투명 배경 및 마우스 호버 트리거(`#252528`)를 구성하여 아이콘(22x22)이 36px 높이 타이틀바에서 최적의 비율로 온전하게 렌더링되도록 구현.
3. **직접 라우티드 이벤트 연동**: `Border.MouseLeftButtonDown`을 통해 단일 클릭 시 시스템 메뉴(`SystemCommands.ShowSystemMenu`) 호출 및 더블 클릭 시 창 닫기(`Close()`)가 누락 없이 격발되도록 보장.
4. **회귀 방지 단위 테스트 추가 (`tests/GRC.Tests/IconResourceTests.cs`)**:
   - `AppIcon_ResourceStream_ExistsAndDecodesSuccessfully`: 리소스 스트림 및 256x256 아이콘 디코딩 검증.
   - `AppIcon_ImageSourceConverter_ResolvesPackUri`: `ImageSourceConverter`의 Pack URI 변환 유효성 검증.
   - `AppIcon_ImageElementOnStaThread_RendersWithoutClipping`: STA 스레드 상에서 `Border`(28x28) 내부 `Image` 배치 및 22x22 렌더 크기 유지(절단 방지) 검증.

5. **미니멀 타이포그래피 브랜딩 디자인 결정**: 이후 세련된 미니멀 프레임리스 데스크톱 룩앤필(VS Code, Windows Terminal 스타일)을 위해 커스텀 타이틀바 내부의 앱 아이콘 렌더링을 배제하고, 좌측 16px 여백의 순수 텍스트 브랜딩(`GRC` 볼드 + 서브타이틀) 및 전역 캡션 드래그(`IsHitTestVisible="False"`)로 통일함. OS 작업 표시줄 및 Alt+Tab용 `Window.Icon`은 정규 Pack URI로 유지.

### 4. 검증 (Verification)
* `tests/GRC.Tests/IconResourceTests.cs` 포함 단위 테스트 스위트 36/36 전량 통과 (**Exit Code 0**).
* 솔루션 전체 빌드 검증 (`dotnet build GRC.sln` - 0 Warning, 0 Error, **Exit Code 0**).

---

## 2026-10-14: [Resolved] 테스트 스위트 전수 점검, 일회성 검증 테스트 정리 및 유닛/통합(Trait) 분리

### 1. 현상 (Symptom)
* 일상적인 빌드 QA 검증 시 `dotnet test GRC.sln`이 전체 36개 테스트를 일괄 실행하면서, 외부 네트워크 HTTP 통신을 수행하는 `ModelVerificationTests`로 인해 수십 초~1분 이상 지연되거나 외부 구글 서버 지연 시 타임아웃(30초) 실패 위험이 상존함.
* 과거 모델 진단용 일회성 음성/모델 폐기 검증 및 타이틀바 디버깅용 임시 테스트가 테스트 스위트에 잔류함.

### 2. 해결책 (Resolution)
1. **일회성/임시 테스트 정리**:
   - `ModelVerificationTests.cs`: 구글 서버 폐기 모델(`gemini-3.6-flash`, `gemini-3.1-pro-preview`) 대상 음성 호출 시험 케이스 제거.
   - `IconResourceTests.cs`: 타이틀바 아이콘 디버깅용 일회성 `ImageSourceConverter` 테스트 제거 및 프로덕션 `Window.Icon`의 `BitmapFrame` 256x256 디코딩 검증으로 정돈.
2. **xUnit `[Trait("Category", "Integration")]` 카테고리 분리**:
   - 실제 Google Cloud Vertex AI 실시간 엔드포인트를 호출하는 `ModelVerificationTests`에 `[Trait("Category", "Integration")]` 속성 부여.
   - 일상 빌드 QA 시 순수 로컬 인메모리 유닛 테스트(29개)만 즉시 실행되도록 분리:
     - 일상 빌드 QA: `dotnet test GRC.sln --filter "Category!=Integration"` (~40ms 실행)
     - 외부 API 가용성 시험: `dotnet test GRC.sln --filter "Category=Integration"`
3. **프로젝트 실행 프로토콜 갱신 (`.agents/AGENTS.md`)**:
   - 일상 테스트 명령어를 `--filter "Category!=Integration"`으로 등록하여 40ms 단위의 초고속 피드백 루프 확립.

### 3. 검증 (Verification)
* 일상 로컬 유닛 테스트 29/29 통과 (소요 시간: **42ms**, Exit Code 0).
* 외부 API 통합 테스트 4/4 통과 (소요 시간: **5s**, Exit Code 0).
* 솔루션 빌드 검증 (`dotnet build GRC.sln` - 0 Warning, 0 Error, **Exit Code 0**).

---

## 2026-10-14: [Resolved] 백그라운드 스레드 및 단위 테스트 환경 내 WPF STA 스레드 위반 방지를 위한 AudioService / DialogService Dispatcher 사전 캡처

### 1. 현상 (Symptom)
* `AudioService`의 `MediaPlayer` 또는 `DialogService`의 윈도우 생성/표시 로직이 비동기 작업(`Task.Run` 또는 스레드 풀 스레드)에서 호출될 때 `System.InvalidOperationException: 다른 스레드가 이 개체를 소유하고 있어 호출한 스레드가 해당 개체에 액세스할 수 없습니다.`(STA 스레드 선점 위반)가 발생하거나 UI 스레드가 블로킹되는 현상 발생.
* 단위 테스트 러너(xUnit)와 같이 메시지 펌프가 없는 환경에서 `Application.Current`가 `null`이거나 `Dispatcher.Invoke()` 호출 시 데드락이 발생할 위험 상존.

### 2. 원인 (Root Cause)
1. WPF의 `MediaPlayer` 및 `Window` 파생 컨트롤은 생성된 STA 스레드에 엄격한 스레드 친화성(Thread Affinity)을 지님.
2. 서비스 인스턴스가 생성되는 시점에 UI 디스패처를 포착하지 않고, 런타임에 임의의 스레드에서 직접 `Application.Current.Dispatcher`를 조회하거나 직접 접근할 경우 null 참조 또는 스레드 액세스 위반 발생.

### 3. 해결책 (Resolution)
1. **생성자 내 Dispatcher 사전 캡처**:
   * `AudioService` 및 `DialogService` 생성자에서 `_dispatcher = Application.Current?.Dispatcher ?? Dispatcher.CurrentDispatcher;`를 캡처하여 보관.
2. **`RunOnUIThread` 스레드 스위칭 캡슐화**:
   * UI 조작 전 `_dispatcher.CheckAccess()`를 검사하여 이미 UI 스레드인 경우 즉시 동기 실행하고, 백그라운드 스레드인 경우에만 `_dispatcher.Invoke(action)` 또는 `_dispatcher.InvokeAsync(action)`로 전환하는 헬퍼 메서드로 감쌈.
   * `DialogService`의 `SetOwner(Window window)` 역시 UI 스레드 상에서 `owner.IsLoaded` 상태를 확인한 후 `dialog.Owner`를 안전하게 바인딩하도록 보호.

### 4. 검증 (Verification)
* `tests/GRC.Tests/AudioServiceTests.cs` (STA 및 MTA 스레드 안전성 2종 테스트): MTA 스레드 풀에서 재생 메서드 호출 시 예외 없이 통과 검증.
* `tests/GRC.Tests/DialogServiceTests.cs` (스레드 안전성 및 다이얼로그 서비스 동작 2종 테스트) 검증 통과.
* 테스트 스위트 전량 통과 (**Exit Code 0**).

---

## 2026-10-14: [Resolved] xUnit 테스트 러너 환경에서 WPF Application 인스턴스 중복 생성(AppDomain Singleton Trap) 해결

### 1. 현상 (Symptom)
* WPF 리소스 딕셔너리 또는 뷰를 검증하는 단위 테스트(`ThemeResourceTests`, `IconResourceTests`)를 xUnit 병렬 러너로 동시 실행할 때, `System.InvalidOperationException: 동일한 AppDomain에서 둘 이상의 Application 개체를 만들 수 없습니다.` 오류가 발생하며 테스트 실패.

### 2. 원인 (Root Cause)
1. WPF `System.Windows.Application` 클래스는 단일 `AppDomain` 내에서 단 하나의 인스턴스만 허용하는 싱글톤 불변식을 지님.
2. 여러 테스트 클래스에서 독립적으로 `new Application()`을 호출하거나, 테스트 순서에 따라 이미 생성된 `Application.Current`가 존재하는 상태에서 재할당을 시도함.

### 3. 해결책 (Resolution)
1. **`EnsureApplication` 정적 가드 패턴 도입**:
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
2. **리소스 테마 사전 동기화**:
   * 애플리케이션 초기화 시 리소스 딕셔너리가 비어있는 경우에만 `PrimitiveTokens.xaml`, `DarkTheme.xaml`을 순차적으로 병합하도록 방어.

### 4. 검증 (Verification)
* `tests/GRC.Tests/ThemeResourceTests.cs` 및 `tests/GRC.Tests/IconResourceTests.cs` 동시/반복 실행 시 싱글톤 충돌 없이 100% 통과 검증.
* 테스트 스위트 전량 통과 (**Exit Code 0**).

---

## 2026-10-14: [Resolved] SmartAutoScroll의 컨텐츠 축소 및 LLM 토큰 스트리밍 시 상단 스크롤 하이재킹(Scroll Hijacking) 방어 수학식 교정

### 1. 현상 (Symptom)
* LLM 텍스트가 빠르게 실시간 스트리밍되거나, 접기/펼치기 및 메시지 삭제 등으로 스크롤 뷰어의 내용물 높이(`ExtentHeight`)가 줄어들 때(`extentHeightChange < 0`), 과거 대화를 읽기 위해 스크롤을 올려둔 사용자의 화면이 강제로 최하단으로 끌려 내려가는 스크롤 하이재킹(Scroll Hijacking) 현상 발생.
* `ScrollViewer.Tag` 속성에 불리언 값을 임의 저장하여 스크롤 상태를 추적하던 기존 레거시 코드가 레이스 컨디션 및 뷰 가상화 재활용 시 상태 불일치를 유발함.

### 2. 원인 (Root Cause)
1. 기존 하단 도달 판정식에서 높이 변화량(`extentHeightChange`)의 절댓값(`Math.Abs`)을 차감할 경우, 컨텐츠가 축소(`extentHeightChange < 0`)될 때 기준 높이가 비정상적으로 왜곡되어 유저가 최하단에 있지 않음에도 하단에 머무르고 있는 것으로 오판단함.
2. `Tag` 프로퍼티 기반의 상태 저장은 순수 렌더링 파이프라인의 수학적 뷰포트 위치와 비동기 이벤트 사이의 불일치를 유발함.

### 3. 해결책 (Resolution)
1. **`ScrollViewer.Tag` 제거 및 순수 수학적 위치 판정 함수로 전환 (`AutoScrollHelper.cs`)**:
   ```csharp
   public static bool CalculateIsAtBottom(double verticalOffset, double scrollableHeight, double extentHeightChange, double tolerance = 24.0)
   {
       if (scrollableHeight <= 0) return true;
       // 컨텐츠 확장 시에는 확장 직전의 기준 높이와 비교, 축소 또는 변화 없을 시 현재 최대 스크롤 가능 높이와 직접 비교
       double referenceHeight = extentHeightChange > 0 ? scrollableHeight - extentHeightChange : scrollableHeight;
       return verticalOffset >= (referenceHeight - tolerance);
   }
   ```
2. **24px 허용오차(Tolerance) 및 컨텐츠 축소 분기 완벽 분리**:
   * 토큰 단위 스트리밍 팽창 시에만 직전 프레임 오프셋을 역산하여 바닥 추적 여부를 판정하고, 축소 시에는 절댓값 왜곡 없이 현재 스크롤 위치를 절대 비교하여 상단 탐색 중인 사용자의 뷰포트를 절대 침범하지 않도록 보장.

### 4. 검증 (Verification)
* `tests/GRC.Tests/SmartAutoScrollTests.cs` 6개 경계값 단위 테스트 전량 통과:
  - 최하단 위치 판정, 24px 허용오차 경계 판정.
  - 상단 스크롤 유저 대상 토큰 스트리밍 팽창 시 하이재킹 차단(`false`).
  - 접기/삭제 등 컨텐츠 축소(`extentHeightChange < 0`) 시 상단 스크롤 뷰포트 보존(`false`).
* 테스트 스위트 전량 통과 (**Exit Code 0**).

---

## 2026-10-14: [Resolved] Application.LoadComponent 사용 시 절대 Pack URI 지정으로 인한 ArgumentException 및 상대 URI 표준화

### 1. 현상 (Symptom)
* 단위 테스트 또는 동적 리소스 로딩 코드에서 `Application.LoadComponent(new Uri("pack://application:,,,/GRC;component/Themes/PrimitiveTokens.xaml", UriKind.Absolute))` 호출 시 `System.ArgumentException: 절대 URI를 사용할 수 없습니다.` 예외 발생.

### 2. 원인 (Root Cause)
1. WPF 내부의 `Application.LoadComponent` API는 BAML 리소스 로딩 규격상 반드시 상대 URI(`UriKind.Relative`)를 요구함.
2. XAML 마크업 내 `ResourceDictionary.Source`는 절대 Pack URI를 지원하지만, C# 코드-비하인드에서 직접 `Application.LoadComponent`를 호출할 때는 Pack 스킴이 포함된 절대 URI를 전달하면 런타임 인자 검증에서 거부됨.

### 3. 해결책 (Resolution)
1. **상대 URI 표준화 (`UriKind.Relative`)**:
   * C# 코드에서 `Application.LoadComponent` 호출 시 어셈블리 상대 경로로 일원화:
     ```csharp
     var uri = new Uri("/GRC;component/Themes/PrimitiveTokens.xaml", UriKind.Relative);
     var dict = (ResourceDictionary)Application.LoadComponent(uri);
     ```
2. **PackUriHelper 스킴 사전 등록**:
   * WPF `Application`을 생성하지 않는 헤드리스 테스트 환경에서는 `_ = PackUriHelper.UriSchemePack;`를 호출하여 URI 파서에 pack 스킴을 사전 등록.

### 4. 검증 (Verification)
* `tests/GRC.Tests/ThemeResourceTests.cs`에서 상대 URI 기반 컴포넌트 로드 및 동결 브러시 유효성 검증 통과.
* 테스트 스위트 전량 통과 (**Exit Code 0**).

---

## 2026-10-14: [Resolved] 스트리밍 및 가상화 스크롤 도중 마우스 휠 상향 회전 시 최하단 강제 스크롤 하이재킹 및 1틱 탈출 트랩 해결

### 1. 현상 (Symptom)
* 과거 대화를 읽기 위해 마우스 휠을 위로 빠르게 굴릴 때, 화면이 상단으로 올라가지 못하고 자동으로 다시 최하단으로 끌려 내려가는 스크롤 하이재킹(Scroll Hijacking) 현상 발생.

### 2. 원인 (Root Cause)
1. **무상태(Stateless) 판정의 탈출 불가능 트랩**:
   * 최하단(예: 1000px)에서 휠을 1~2틱 위로 올렸을 때(예: 985px), 16ms 주기로 AI 스트리밍 토큰이 들어오면 `extentHeightChange`가 발생하여 판정 기준선이 올라감.
   * 사용자의 뷰포트가 24px 허용오차 이내에 머무는 순간 `CalculateIsAtBottom`이 `true`를 반환하여 즉시 `ScrollToEnd()`가 호출됨. 16ms 이내에 24px 이상 벗어나지 못해 최하단에 갇힘.
2. **가상화 패널(`VirtualizingStackPanel` with `ScrollUnit="Pixel"`) 아이템 실체화 변동**:
   * 상단 스크롤 시 이전에 가상화되어 측정되지 않았던 대화 아이템이 렌더 트리에 진입하면서 `ExtentHeightChange`가 수백 픽셀 단위로 급증함.
   * 이에 따라 `referenceHeight`가 현재 `scrollableHeight`보다 수백 픽셀 낮아져 사용자가 한참 위에 머물러도 바닥인 것으로 오판단함.
3. **비동기 디스패처 큐잉(`DispatcherPriority.Loaded`) 지연 실행**:
   * 직전 프레임에서 적재된 `ScrollToEnd()` 비동기 태스크가 유저가 휠을 위로 돌린 직후 뒤늦게 실행되어 화면을 바닥으로 강제 텔레포트시킴.

### 3. 해결책 (Resolution)
1. **사용자 의도 기반 상태 머신 (`IsAutoScrollActive` Attached DependencyProperty) 도입**:
   * `PreviewMouseWheel`: 마우스 휠을 위로 회전(`e.Delta > 0`)하는 순간 즉시 `SetIsAutoScrollActive(sv, false)` 호출.
   * `ScrollChanged`: 순수 사용자 조작(`isPureUserScroll = e.ExtentHeightChange == 0 && e.ViewportHeightChange == 0`)이면서 뷰포트가 위로 이동한 경우(`e.VerticalChange < 0`) 즉각 비활성화.
   * 창 최대화/리사이징(`ViewportHeightChange > 0`)으로 인한 클램핑 역방향 오프셋 변동은 `isPureUserScroll` 가드로 사용자 조작에서 격리.
2. **지연 실행 검증 (Lazy Guard)**:
   * 디스패처에 대기 중이던 `ScrollToEnd()` 실행 시점에 `if (GetIsAutoScrollActive(sv))`를 2차 확인하여 큐 대기 시간 중 발생한 사용자 조작을 보호.
3. **능동적 하향 조작 시에만 재활성화**:
   * 토큰 유입 시(`VerticalChange == 0`)에는 재활성화되지 않도록 차단하고, 사용자가 능동적으로 휠을 아래로 굴려 바닥에 도달하거나(`e.VerticalChange > 0 && isAtBottomNow`), 스크롤바가 없는 경우(`sv.ScrollableHeight <= 0`), 또는 메시지 전송 시(`ScrollToEnd`)에만 `IsAutoScrollActive = true`로 복구.

### 4. 검증 (Verification)
* `tests/GRC.Tests/SmartAutoScrollTests.cs`에 STA 스레드 기반 `AutoScrollHelper_StateTransitions_ExecuteOnStaThread` 단위 테스트 추가.
* 기본 6종 경계값 테스트 및 상태 전이 테스트 전량 통과 (총 78/78 통과, **Exit Code 0**).






