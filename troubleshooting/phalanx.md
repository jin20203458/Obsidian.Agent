---
description: Phalanx C++ 센서 및 C# 코어 트러블슈팅 런북.
related:
  - ../README.md
  - ../Phalanx/README.md
---
# Phalanx Troubleshooting

본 문서는 Phalanx EDR 솔루션(C++ 센서, gRPC 스트리밍, C# 코어 및 AI 에이전트) 개발 및 실전 운영 중 발생하는 시스템 예외 현상과 해결 방안을 기록하는 중앙 기술 런북입니다.

---

## 사전 주의사항 및 알려진 기술적 제약 (Known Constraints)

### 1. ETW 커널 세션 생성 권한 (Administrator Elevation)
* **현상**: 관리자 권한이 없는 일반 사용자 권한으로 센서 실행 시 `krabs-etw` 세션 생성 단계에서 `ACCESS_DENIED (0x5)` 예외 발생.
* **대응책**: `src/Phalanx.Sensor/CMakeLists.txt`의 MSVC 링크 플래그(`/MANIFEST:EMBED /MANIFESTUAC:"level='requireAdministrator' uiAccess='false'"`)로 `Phalanx.Sensor.exe`에 `requireAdministrator` 실행 수준을 임베딩.

### 2. 프로세스 원자적 동결 데드락 방어 및 세이프티 워치독 (Safety Watchdog)
* **현상**: 타깃 프로세스가 크리티컬 섹션이나 ntdll 로더 락(`LdrpLoaderLock`)을 쥐고 있는 상태에서 비동기 동결 호출 시 시스템 전역 리소스 경합 또는 데드락 발생 가능성.
* **대응책**:
  * 동결 API(`ntdll!NtSuspendProcess` 및 폴백 `SuspendThread`)는 자체 타임아웃 파라미터가 없으므로, 센서 내부에 비동기 안전 타이머(Safety Watchdog)를 운영하여 고아 동결(Orphan Freeze) 감지 시 자동 복구(`NtResumeProcess`).
  * **타임아웃 분할 구조**: 기본 10,000ms(10초, C# 코어 하트비트 확인용, CLI `--watchdog-timeout` 연동) + AI 수사 개시 시 단 1회 50,000ms(50초, CLI `--extend-timeout` 연동) 연장 티켓(`ACTION_EXTEND_TIMEOUT`) 발송으로 총 60초 수사 예산 확보.
  * **절대 상한선 (Hard Ceiling)**: 데드락 방지를 위해 연장은 최대 1회로 엄격 제한되며, 최대 60초 초과 시 자동으로 `NtResumeProcess`(폴백 시 `ResumeThread`)를 강제 집행.
  * C# 상위 타임아웃 CTS는 통신 레이스 차단을 위해 50,000ms(50초)로 설정.

---

## 2026-09-29: [Resolved] Windows Winsock/NOMINMAX 충돌 및 MSVC UAC 매니페스트 링크 에러

### 1. 현상 (Symptom)
* `ws2ipdef.h` / `ws2tcpip.h` 컴파일 시 `error C2011: 'ip_mreq': 'struct' type redefinition`, `error C2065: 'PADDRINFOA'`, `error C3861: 'WSAIoctl'` 등 100여 건의 Winsock 심볼 충돌 발생.
* `grpcpp/impl/generic_serialize.h` 및 `grpc/event_engine/memory_request.h`에서 Windows 매크로 `min`/`max` 간섭으로 구문 에러 발생.
* `Phalanx.Sensor.exe` 링크 시 `manifest authoring error c1010001: Values of attribute "level" not equal in different manifest snippets (LNK1327)` 발생.

### 2. 원인 (Root Cause)
1. `windows.h`가 `winsock2.h`보다 먼저 인클루드되어 구형 `winsock.h` (Winsock 1)와 `winsock2.h` (Winsock 2)가 중복 로드됨.
2. `NOMINMAX` 및 `UNICODE` / `_UNICODE` 매크로 부재로 `std::min`/`std::max` 파괴 및 `krabs-etw`의 `KERNEL_LOGGER_NAME` (`TEXT(...)`) 와이드 문자열 불일치 발생.
3. CMake의 `target_sources`에 `app.manifest`를 직접 전달하면서 MSVC 기본 생성 매니페스트(`asInvoker`)와 병합 충돌 발생.

### 3. 해결책 (Resolution)
1. 루트 `CMakeLists.txt`에 전역 컴파일 정의 `add_compile_definitions(UNICODE _UNICODE NOMINMAX WIN32_LEAN_AND_MEAN _WIN32_WINNT=0x0A00)` 적용.
2. 모든 C++ 헤더에서 `<windows.h>` 호출 전 `<winsock2.h>`와 `<ws2tcpip.h>`를 선행 인클루드하도록 구조화.
3. CMake 링크 플래그에 MSVC 네이티브 UAC 임베딩 지시어 `/MANIFEST:EMBED /MANIFESTUAC:"level='requireAdministrator' uiAccess='false'"` 적용하여 `mt.exe` 충돌 없이 PE 바이너리에 권한 임베딩 완료.

---

## 2026-09-29: [Resolved] EtwKernelCollector::Start() 동시성 레이스 컨디션 해결 및 원자적 CAS 적용

### 1. 현상 (Symptom)
* `EtwKernelCollector::Start()`를 복수의 스레드가 동시에 호출할 경우, 이미 가동 중인 스레드가 덮어씌워지며 C++ 런타임에 의해 `std::terminate()` 크래시가 유발될 수 있는 잠재적 취약점 존재.
* 스레드 기동 중 시스템 자원 부족 예외(`std::system_error` 등) 발생 시 상태 플래그 롤백 로직이 부재하여 `running`이 `true`로 고착되는 상태 불일치 발생.

### 2. 원인 (Root Cause)
* 기존 코드가 `running.load()`를 확인하고 `running.store(true)`를 호출하는 Check-Then-Act (TOCTOU) 비원자적 상태 전이 구조로 작성됨.

### 3. 해결책 (Resolution)
1. `impl_->running.compare_exchange_strong(expected, true, std::memory_order_acq_rel)`을 적용하여 복수의 스레드가 동시 진입하더라도 오직 하나의 스레드만 `false -> true` 전이에 성공하도록 원자적 상태 전이 보장.
2. 스레드 생성부를 `try-catch`로 감싸 `std::thread` 생성 실패 시 `impl_->running.store(false, std::memory_order_release)`로 원자적 롤백 수행 및 `false` 반환하도록 예외 안전성 확보.

---

## 2026-09-30: [Resolved] ProcessTree PID 재사용 시 유령 부모(Ghost Parent) 족보 왜곡 방어

### 1. 현상 (Symptom)
* 윈도우 OS가 종료된 프로세스의 PID를 빠른 속도로 재할당할 때, 종료된 부모 A(PID: 1000)의 자식 B(PID: 2000, `ppid = 1000`)가 살아있는 상태에서 신규 프로세스 C에 동일 PID 1000이 부여될 경우 자식 B가 무관한 C를 부모로 오인하는 유령 부모(Ghost Parent) 족보 왜곡 발생.

### 2. 원인 (Root Cause)
* Windows 커널은 부모 프로세스 종료 시 고아 자식 프로세스의 `ParentProcessId`를 0으로 초기화하지 않음.
* `ProcessTree`가 PID 재사용 시 이전 노드의 자식들(`children_pids`)의 부모 링크(`child.ppid = 0`) 절단 처리가 누락되어 있었음.

### 3. 해결책 (Resolution)
1. **즉각 재사용 덮어쓰기 (`InsertOrOverwriteNodeInternal`)**: PID 덮어쓰기 직전 이전 자식 노드들의 `ppid = 0` 재설정으로 유령 입양 원천 차단.
2. **상한선 영구 퇴출 (`EvictOldestTombstoneInternal`)**: 톰스톤 노드 메모리 해제 시 상향 링크와 하향 링크(`child.ppid = 0`)를 양방향으로 원자적 절단.
3. **C# ProcessTreeProjectionManager 연동**: `LIFECYCLE_START` 수신 시 동일 PID 활성 노드가 존재하면 이전 노드를 즉시 Tombstone 처리하고 신규 GUID 노드로 대체.

---

## 2026-09-30: [Resolved] CQRS 프로젝션 파이프라인 콜드 스타트 및 초기 스냅샷 핸드셰이크

### 1. 현상 (Symptom)
* C# Cockpit 기동 시 센서 기동 전부터 실행 중이던 프로세스(약 300여 개)의 계층 관계를 알지 못해 족보 추적이 루트에서 단절되는 콜드 스타트 문제 발생.
* 커널 `ProcessStop` 이벤트의 gRPC 큐 푸시 누락으로 C# 프로젝션 트리에 종료 프로세스가 영구 활성 상태로 잔존하는 메모리 누수 발생.

### 2. 원인 (Root Cause)
* `phalanx.proto` 내 프로세스 생명주기 구분 부재 및 gRPC 스트림 연결 시 초기 상태 동기화 핸드셰이크 프로토콜 결여.

### 3. 해결책 (Resolution)
1. **`phalanx.proto` 생명주기 확장**: `ProcessLifecycle` enum 추가(`LIFECYCLE_SNAPSHOT`, `LIFECYCLE_START`, `LIFECYCLE_STOP`, `LIFECYCLE_SUSPENDED`, `LIFECYCLE_TERMINATED`).
2. **`ProcessStop` 큐 푸시 연동**: 커널 `ProcessStop` 수신 시 `LIFECYCLE_STOP` 및 종료 코드를 락-스왑 큐에 푸시.
3. **초기 스냅샷 핸드셰이크**: `ProcessTree::GetActiveSnapshotEvents()`로 gRPC 연결 직후 활성 프로세스 스냅샷 배치(`LIFECYCLE_SNAPSHOT`) 일괄 전송.

---

## 2026-09-30: [Resolved] Google Cloud Vertex AI OAuth2 인증 및 JsonElement 매개변수 언래핑 결함

### 1. 현상 (Symptom)
* Google Cloud Vertex AI 서비스 어카운트(`Config/google-credentials.json`) 연동 시 인증 실패.
* `System.Text.Json` 역직렬화 시 `Dictionary<string, object>` 내부 원시 타입이 `JsonElement`로 박싱되어 도구 매개변수 타입 검사 실패.

### 2. 원인 (Root Cause)
* Vertex AI는 `x-goog-api-key` 대신 OAuth2 Bearer Token(`cloud-platform` 스코프) 인증을 요구함.
* C# `System.Text.Json` 역직렬화 특성상 동적 객체가 네이티브 타입이 아닌 `JsonElement` 구조체로 박싱됨.

### 3. 해결책 (Resolution)
1. **OAuth2 Bearer Token 주입**: `ServiceAccountCredential`을 통해 동적 Bearer Token 발급 및 요청 헤더 주입.
2. **도구 매개변수 언래핑**: `AutonomousHunterAgent.cs`에서 도구 인자 전달 전 `JsonElement`를 네이티브 C# 타입(`string`, `int`, `double`, `bool`)으로 일괄 언래핑 처리.

---

## 2026-09-30: [Resolved] Gemini responseSchema CFG 루프/토큰 고갈 결함 및 JSON Mode 최적화

### 1. 현상 (Symptom)
* EDR 환경에서 Gemini API 호출 시 `responseSchema` 적용 시 15초 타임아웃 응답 실패 또는 도구 선택 정확도 급락 현상 발생.

### 2. 원인 (Root Cause)
* Gemini 내부 추론 토큰(`thoughtsTokenCount`)이 출력 예산을 잠식하고, 모호한 필드 설명으로 인해 CFG(Context-Free Grammar) 문법 제약 퇴행 무한 루프 발생.

### 3. 해결책 (Resolution)
* **[Negative Constraint] 절대 EDR 자율 에이전트 ReAct 루프에 `responseSchema`를 직접 바인딩하지 말 것.**
* 순수 JSON Mode와 중첩 괄호 균형 탐색 파서(`LlmJsonParser`) 조합으로 단일 왕복 완결 및 추론 사고(CoT) 보존.

---

## 2026-09-30: [Resolved] EDR 수사 도구 핵심 메모리 크래시 및 방화벽 Self-DoS 방어

### 1. 현상 (Symptom)
* `ProcessMemoryScanTool`: 미보호 메모리 순회 시 `PAGE_GUARD` 접근 예외 프로세스 크래시 발생 위험.
* `SystemFirewallTool`: Netsh 실패 시에도 `true` 반환 Silent Failure 및 로컬 게이트웨이/DNS 차단 시 엔드포인트 네트워크 단절(Self-DoS) 위험.

### 2. 원인 (Root Cause)
* VAD 메모리 속성 검증 부재 및 호스트 통신 필수 인프라 IP 화이트리스트 보호 가드 결여.

### 3. 해결책 (Resolution)
1. **`ProcessMemoryScanTool` PAGE_GUARD 크래시 방어**: `VirtualQueryEx` 기반 VAD 순회로 Unbacked Executable Memory(`MEM_PRIVATE` + `EXECUTE` + `!PAGE_GUARD`)만 선별 스캔.
2. **`SystemFirewallTool` Self-DoS 방어**: `ToolResult(overallSuccess)` 반환으로 Silent Failure를 차단하고, 로컬 IP/기본 게이트웨이/DNS 주소 차단 시도를 원천 차단하는 보호망 구축.

---

## 2026-10-01: [Resolved] C# 생성자 내 Sync-over-Async(GetAwaiter().GetResult()) 스레드풀 데드락 제거

### 1. 현상 (Symptom)
* 클래스 생성자 내부에서 토큰 발급 및 설정 로딩 시 `.GetAwaiter().GetResult()`를 동기 호출하여 스레드풀 고갈(Thread Pool Starvation) 시 데드락 발생 위험 존재.

### 2. 원인 (Root Cause)
* 비동기 초기화 팩토리 패턴 대신 생성자에서 비동기 작업을 동기 블로킹 대기함.

### 3. 해결책 (Resolution)
* **[Negative Constraint] 생성자 내부에서 `.GetAwaiter().GetResult()` 또는 `.Result`를 호출하지 말 것.**
* 생성자 내 블로킹 호출을 완전 제거하고, 동기 팩토리 메서드(`TryCreateFromLocalConfig()`) 또는 비동기 팩토리로 리팩토링.

---

## 2026-10-01: [Resolved] AI 수사관 판정 왜곡(Decision Hijacking) 및 결정권 침해 결함 해결 (SSOT 아키텍처 확립)

### 1. 현상 (Symptom)
* 정상 관리 스크립트 실행 시 Gemini 모델이 정상 판결(`ACTION_RESUME`, 확신도 98%)을 내렸음에도 C# 호스트 코드가 이를 가로채 강제 사살하고 피싱 기법(`T1566.001`)을 조작 주입하는 치명적 오탐 발생.

### 2. 원인 (Root Cause)
1. `ConfidenceScore`(정상 확신도 98%)를 `threatScore`(위협 점수 98점)로 오인 바인딩.
2. `CommandLine.Contains("-enc")` 정적 검사로 LLM 수사 결론을 강제 오버라이드.

### 3. 해결책 (Resolution)
* **[Negative Constraint] LLM ReAct 루프가 종결된 후 호스트 정적 시그니처 검사로 판결(`VerdictAction`)을 임의 오버라이드하지 말 것.**
* ReAct 루프 정상 종결 시 Gemini AI 수사관의 판결을 단일 진실 공급원(SSOT)으로 100% 수용하고, 시스템 가드는 최대 턴 초과 등 예외 상황에서만 격리 집행.

---

## 2026-10-01: [Resolved] WPF 관제 콕핏과 Kestrel gRPC 백그라운드 서버 하이브리드 호스팅 및 STA 스레드 안전성 확보

### 1. 현상 (Symptom)
* WPF 콕핏과 Kestrel gRPC 서버를 단일 바이너리에서 구동할 때, `async Task Main`에서 윈도우 인스턴스화 시 STA 스레드 제약 위반(`InvalidOperationException`) 발생 또는 STA 스레드에서 gRPC IO 블로킹으로 UI 프리징 발생.
* 백그라운드 스레드에서 `ObservableCollection` 조작 시 `NotSupportedException` 발생.

### 2. 원인 (Root Cause)
* WPF UI 서브시스템의 `[STAThread]` Dispatcher 메시지 펌프와 Kestrel 멀티스레드 비동기 호스트 간 스레드 선호도(Thread Affinity) 충돌.

### 3. 해결책 (Resolution)
1. **하이브리드 라이프사이클**: `[STAThread]` 동기 진입점에서 `webApp.Start()`(비차단)로 gRPC 서버를 기동하고 메인 스레드에서 `wpfApp.Run(mainWindow)` 실행. 창 종료 시 `webApp.StopAsync()`로 Graceful Shutdown.
2. **UI 스레드 마샬링**: gRPC 수신 및 AI 수사 알림을 `Application.Current?.Dispatcher?.InvokeAsync(...)`로 안전하게 래핑.

---

## 2026-10-01: [Resolved] gRPC 스트림 다중 클라이언트 세션 덮어쓰기 및 거짓 DISCONNECTED 상태 전이 결함 해결

### 1. 현상 (Symptom)
* C++ 센서가 정상 기동 중임에도 모의 공격 도구 종료 후 또는 유휴 상태 경과 시 관제 콘솔 상단 통신 상태가 주기적으로 `[DISCONNECTED]`로 오표시됨.

### 2. 원인 (Root Cause)
1. `PhalanxGrpcService`가 단일 필드로 응답 스트림을 유지하여, 신규 클라이언트 접속 시 기존 스트림을 덮어쓰고 한 클라이언트 종료 시 연결 상태를 `false`로 일괄 반전시킴.
2. Kestrel HTTP/2 KeepAlive Ping 부재로 유휴 시 소켓 반폐쇄 상태 진입.

### 3. 해결책 (Resolution)
1. **동시성 컬렉션 세션 관리**: `ConcurrentDictionary<string, IServerStreamWriter<MitigationCommand>>`로 멀티 세션 관리. 모든 세션이 비었을 때만 `Disconnected` 통지.
2. **Kestrel HTTP/2 킵얼라이브 활성화**: `KeepAlivePingDelay = 30s`, `KeepAlivePingTimeout = 15s` 명시.

---

## 2026-10-01: [Resolved] Kestrel 백그라운드 스레드의 ObservableCollection 조작으로 인한 gRPC 스트림 단절 및 센서 ON/OFF 무한 루프

### 1. 현상 (Symptom)
* WPF 관제 콘솔에서 센서 연결 상태가 `LIVE`와 `OFFLINE` 사이를 수 초 주기로 무한 반복 전환(플리핑)되며 프로세스 트리가 갱신되지 못함.

### 2. 원인 (Root Cause)
* C++ 센서 스냅샷 수신 시 Kestrel 백그라운드 스레드가 `ObservableCollection`을 직접 수정하여 WPF `CollectionView`에서 `NotSupportedException` 발생, 이로 인해 스트림이 종료되고 센서가 무한 재접속 루프에 진입.

### 3. 해결책 (Resolution)
1. **WPF 컬렉션 동기화**: `BindingOperations.EnableCollectionSynchronization(RootNodes, _syncLock)` 등록 및 UI Dispatcher 스레드로 마샬링.
2. **스냅샷 일괄 전달**: 300여 개 프로세스 이벤트를 단일 `ApplySnapshotBatch`로 일괄 전달하여 1회의 Dispatcher 컨텍스트 스위치로 트리 투영 완료.

---

## 2026-10-02: [Resolved] SettingsWindow 오픈 시 TwoWay 바인딩 읽기 전용 속성 충돌로 인한 CLR 강제 종료(0xc000041d)

### 1. 현상 (Symptom)
* `SETTINGS` 버튼 클릭 시 예외 대화상자 없이 `STATUS_FATAL_USER_CALLBACK_EXCEPTION (0xc000041d)` 네이티브 Fast-fail로 Cockpit 프로세스 즉시 강제 종료.

### 2. 원인 (Root Cause)
* WPF `RadioButton.IsChecked` 기본 바인딩은 `TwoWay`이나, ViewModel의 프로퍼티(`IsApiKeyMode` 및 테마 라디오버튼 `IsThemeSystem`, `IsThemeDark`, `IsThemeLight`)가 getter 전용 계산 프로퍼티로 작성됨.
* XAML 엔진이 역방향 쓰기(`ConvertBack`) 시도 시 발생한 예외가 Win32 메시지 루프의 네이티브 `WndProc` 콜백 경계를 교차(cross unmanaged boundary)하면서 CLR이 치명적 예외로 판단하고 프로세스를 강제 종료시킴.

### 3. 해결책 (Resolution)
* **[Negative Constraint] WPF XAML TwoWay 바인딩이 연결되는 ViewModel 프로퍼티를 getter 전용으로 선언하지 말 것.**
* `IsApiKeyMode` 및 테마 프로퍼티에 명시적 `set` 블록을 구현하여 XAML 엔진의 역방향 쓰기를 정상 수용하고, 라디오 버튼 바인딩에 `Mode=OneWay`를 명시하여 이중 방어선 확립.

---

## 2026-10-02: [Resolved] SettingsWindow 재오픈 시 RadioButton TwoWay 바인딩 순환 피드백에 의한 StackOverflowException (0x800703E9)

### 1. 현상 (Symptom)
* `환경 설정` 창을 닫은 후 재오픈 시 `System.StackOverflowException (0x800703E9)` 크래시 발생.

### 2. 원인 (Root Cause)
1. 닫힌 윈도우 인스턴스가 DataContext 미해제로 인해 싱글톤 ViewModel에 고착되어 GC되지 않음.
2. `GroupName` 전역 등록으로 인해 새 윈도우와 이전 윈도우의 라디오 버튼 간에 역방향 바운싱 세터가 트리거되어 밀리초당 수만 회 상호 재귀 호출 발생.

### 3. 해결책 (Resolution)
1. **단방향 세터 가드**: 비활성화 시그널(`value = false`) 시 역방향 반전을 차단하고, 상태가 실제 변경될 때만 세터 로직 실행.
2. **전역 GroupName 제거**: 패널 스코프 격리를 활용하여 `GroupName` 전역 간섭 제거.
3. **Closed 핸들러 정리**: 윈도우 종료 시 `DataContext = null` 설정으로 델리게이트 및 바인딩 체인 완전 절단.

---

## 2026-10-02: [Resolved] ProcessGraphView 내 WPF DataTrigger 기본값 부재 및 유령 리소스 키로 인한 DependencyProperty.UnsetValue 크래시

### 1. 현상 (Symptom)
* 프로세스 트리에서 `[원자적 동결]` 클릭 즉시 `InvalidOperationException: '{DependencyProperty.UnsetValue}'은(는) 'Foreground' 속성의 유효한 값이 아닙니다` 런타임 크래시 발생.

### 2. 원인 (Root Cause)
1. TextBlock Style에 기본 `Foreground` Setter가 누락되어 DataTrigger 해제 시 기본값 복원 실패로 `DependencyProperty.UnsetValue` 반환.
2. `IsSuspended == True` 트리거에 지정된 `{StaticResource ThreatCriticalBrush}`가 테마 딕셔너리에 존재하지 않는 유령 키였음.

### 3. 해결책 (Resolution)
1. **기본 Setter 명시**: Style에 `<Setter Property="Foreground" Value="{DynamicResource TextPrimaryBrush}" />`를 명시하여 트리거 해제 시 안전 복원 보장.
2. **유령 키 제거**: 미존재 정적 리소스 키를 공식 테마 동적 리소스(`{DynamicResource SeveritySuspendedTextBrush}`)로 전면 교체.

---

## 2026-10-02: [Resolved] AI 수사 완료 시 심층수사실 빈 화면(SelectedIncident null) 유실 버그

### 1. 현상 (Symptom)
* AI 자율 수사 완료 즉시 심층수사실(`InvestigationView.xaml`)의 모든 상세 포렌식 데이터가 공백(Blank Screen)으로 증발하는 화면 유실 발생.

### 2. 원인 (Root Cause)
* 수사 완료 후 목록 필터 갱신(`FilteredIncidents.Clear()`) 시, WPF `ListBox`가 `ItemsSource`의 Reset을 감지하고 `SelectedItem`을 `null`로 강제 코어션하여 ViewModel의 `SelectedIncident`가 영구히 null로 덮어써짐.

### 3. 해결책 (Resolution)
* `ApplyFilter()` 시작 시 `previousSelected = SelectedIncident;` 로컬 스냅샷을 캡처하고, 필터링 루프 완료 후 `previousSelected`를 안전 복원하여 ListBox 초기화로 인한 null 코어션 원천 차단.

---

## 2026-10-03: [Resolved] QuestPDF 2026.9+ 시스템 폰트 로드 예외 및 UI 디커플링

### 1. 현상 (Symptom)
* QuestPDF 기반 수사 보고서 생성 단위 테스트 시 `DocumentDrawingException: font families that are not available: 'Segoe UI'` 예외 발생.
* 백그라운드 PDF 생성 중 사용자 UI 선택 변경 시 `SelectedIncident` 동시 참조 경합 위험.

### 2. 원인 (Root Cause)
* QuestPDF v2026.9.0부터 OS 시스템 폰트 자동 조회가 기본 비활성화(`Settings.UseSystemFonts = false`)되고 미등록 폰트 참조 시 예외를 던지도록 정책 변경됨.

### 3. 해결책 (Resolution)
1. **폰트 안전성 설정**:
   ```csharp
   QuestPDF.Settings.License = LicenseType.Community;
   QuestPDF.Settings.UseSystemFonts = true;
   QuestPDF.Settings.ThrowOnMissingFontFamilies = false;
   ```
2. **로컬 스냅샷 캡처**: `ExportForensicPdfAsync` 진입 즉시 `var incident = SelectedIncident;` 스냅샷을 캡처하여 비동기 작업 중 UI 참조 경합 차단.


---

> [!NOTE]
> **[정제 완료 기준선]** 2026-10-06 이전 상위 항목은 정제 완료됨. 신규 인시던트는 이 아래에 추가됩니다.

---
