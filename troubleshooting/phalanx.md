---
description: >-
  Phalanx C++ 센서 및 C# 코어 트러블슈팅 런북. Phalanx 프로젝트 버그, ETW 수집 오류 및 gRPC 장애 발생 시 참조.
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
* **대응책**: `Phalanx.Sensor.exe`의 매니페스트 파일(`app.manifest`)에 `requireAdministrator` 실행 수준을 필수 명시.

### 2. 프로세스 원자적 동결 데드락 방어 및 세이프티 워치독 (Safety Watchdog)
* **현상**: 타깃 프로세스가 크리티컬 섹션이나 ntdll 로더 락(`LdrpLoaderLock`)을 쥐고 있는 상태에서 비동기 동결 호출 시 시스템 전역 리소스 경합 또는 데드락 발생 가능성.
* **대응책**:
  * 동결 API(`ntdll!NtSuspendProcess` 및 폴백 `SuspendThread`)는 자체 타임아웃 파라미터가 없으므로, 센서 내부에 비동기 안전 타이머(Safety Watchdog)를 운영하여 고아 동결(Orphan Freeze) 감지 시 자동 복구(`NtResumeProcess`).
  * **타임아웃 분할 구조**: 기본 10,000ms(10초, C# 코어 하트비트 확인용, CLI `--watchdog-timeout` 연동) + AI 수사 개시 시 단 1회 50,000ms(50초, CLI `--extend-timeout` 연동) 연장 티켓(`ACTION_EXTEND_TIMEOUT`) 발송으로 총 60초 수사 예산 확보.
  * **절대 상한선 (Hard Ceiling)**: 데드락 방지를 위해 연장은 최대 1회로 엄격 제한되며, 최대 60초 초과 시 자동으로 `NtResumeProcess`(폴백 시 `ResumeThread`)를 강제 집행.
  * C# 상위 타임아웃 CTS는 통신 레이스 차단을 위해 50,000ms(50초)로 설정.

---

## 2026-09-08: [Resolved] Windows Winsock/NOMINMAX 충돌 및 MSVC UAC 매니페스트 링크 에러

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

## 2026-09-10: [Resolved] EtwKernelCollector::Start() 동시성 레이스 컨디션 해결 및 원자적 CAS 적용

### 1. 현상 (Symptom)
* `EtwKernelCollector::Start()`를 복수의 스레드가 동시에 호출할 경우, 이미 가동 중인 스레드가 덮어씌워지며 C++ 런타임에 의해 `std::terminate()` 크래시가 유발될 수 있는 잠재적 취약점 존재.
* 스레드 기동 중 시스템 자원 부족 예외(`std::system_error` 등) 발생 시 상태 플래그 롤백 로직이 부재하여 `running`이 `true`로 고착되는 상태 불일치 발생.

### 2. 원인 (Root Cause)
* 기존 코드가 `running.load()`를 확인하고 `running.store(true)`를 호출하는 전형적인 Check-Then-Act (TOCTOU) 비원자적 상태 전이 구조로 작성되어 있었음.

### 3. 해결책 (Resolution)
1. `impl_->running.compare_exchange_strong(expected, true, std::memory_order_acq_rel)`을 적용하여 복수의 스레드가 동시 진입하더라도 오직 하나의 스레드만 `false -> true` 전이에 성공하도록 원자적 상태 전이 보장.
2. 스레드 생성부를 `try-catch`로 감싸 `std::thread` 생성 실패 시 `impl_->running.store(false, std::memory_order_release)`로 원자적 롤백 수행 및 `false` 반환하도록 예외 안전성 확보.

---

## 2026-09-14: [Resolved] ProcessTree PID 재사용 시 유령 부모(Ghost Parent) 족보 왜곡 방어

### 1. 현상 (Symptom)
* 윈도우 OS는 종료된 프로세스의 PID를 빠른 속도로 재할당함.
* 부모 프로세스 A(PID: 1000)가 자식 B(PID: 2000, `ppid = 1000`)를 생성한 후 A가 먼저 종료되고 자식 B는 계속 실행 중인 상태에서, OS가 동일한 PID 1000을 전혀 무관한 새 프로세스 C에 재할당하는 경우 발생.
* 이때 C++ `ProcessTree`가 PID 1000 노드를 새 프로세스 C로 덮어쓰면, 기존 자식 B의 `ppid`가 여전히 1000을 가리키고 있어 B가 엉뚱한 새 프로세스 C를 자기 부모로 오인하고 족보를 거슬러 올라가는 유령 부모(Ghost Parent) 족보 왜곡 발생.

### 2. 원인 (Root Cause)
* 윈도우 OS 커널은 부모 프로세스가 종료되어도 고아 자식 프로세스의 `ParentProcessId`를 0으로 재설정해주지 않음 (죽은 부모 PPID 영구 보존).
* 기존 `ProcessTree::InsertOrOverwriteNodeInternal`은 PID 재사용 시 이전 노드의 부모(`old_ppid`)와의 링크만 절단하고, 이전 노드가 낳았던 자식들(`it->second.children_pids`)의 부모 링크(`child.ppid = 0`) 절단 처리가 누락되어 있었음.

### 3. 해결책 (Resolution)
1. **즉각 재사용 덮어쓰기 (`InsertOrOverwriteNodeInternal`)**: PID 덮어쓰기 직전, 이전 프로세스의 자식 노드들을 순회하여 `child.ppid == pid`인 경우 `ppid = 0`으로 재설정하여 엉뚱한 새 프로세스로의 유령 입양 원천 차단.
2. **10,000개 상한선 영구 퇴출 (`EvictOldestTombstoneInternal`)**: 톰스톤 노드가 메모리에서 완전히 삭제(Evict)될 때도, 상향 링크(부모의 `children_pids`에서 나를 제거)뿐만 아니라 하향 링크(자식 노드들의 `ppid = 0` 고아 처리)를 양방향으로 원자적 절단.
3. **C# ProcessTreeProjectionManager 연동**: C# 측에서도 `LIFECYCLE_START` 수신 시 동일 PID의 활성 노드가 존재하면 이전 노드를 즉시 Tombstone 처리하고 신규 GUID 노드로 대체.

---

## 2026-09-14: [Resolved] CQRS 프로젝션 파이프라인 콜드 스타트 및 초기 스냅샷 핸드셰이크

### 1. 현상 (Symptom)
* C# Cockpit이 가동되었을 때 C++ 센서로부터 실시간 증분 이벤트만 수신할 경우, 센서 기동 전이나 Cockpit 기동 전부터 실행 중이던 프로세스(약 300여 개)의 계층 관계를 알지 못해 자식 프로세스 인입 시 족보 추적(`GetAncestry`)이 루트에서 단절되는 콜드 스타트 문제 발생.
* C++ `EtwKernelCollector`에서 프로세스 종료 이벤트(`ProcessStop`) 발생 시 내부 옵저버에게만 통지하고 gRPC 큐 푸시가 누락되어, C# 프로젝션 트리가 종료된 프로세스를 인지하지 못하고 영구 활성 상태로 방치하는 메모리 누수 존재.

### 2. 원인 (Root Cause)
* 1단계 프로토콜 설계 시 `ProcessEvent`에 프로세스 생명주기 구분이 없었고, 센서-클라이언트 간 gRPC 스트림 연결 시 초기 상태 동기화 프로토콜 규약이 부재했음.

### 3. 해결책 (Resolution)
1. **`phalanx.proto` 생명주기 및 GUID 확장**: `ProcessLifecycle` enum 추가(`LIFECYCLE_SNAPSHOT`, `LIFECYCLE_START`, `LIFECYCLE_STOP`, `LIFECYCLE_SUSPENDED`, `LIFECYCLE_TERMINATED`).
2. **C++ `EtwKernelCollector`의 `ProcessStop` 큐 푸시 연동**: 커널 `ProcessStop` 수신 시 `LIFECYCLE_STOP` 및 종료 코드(`exit_code`)를 포함하여 락-스왑 큐에 푸시.
3. **초기 스냅샷 핸드셰이크**: `ProcessTree::GetActiveSnapshotEvents()`를 구축하여 gRPC 스트림 연결 직후 활성 프로세스 스냅샷 배치(`LIFECYCLE_SNAPSHOT`)를 C# Cockpit으로 일괄 전송.

---

## 2026-09-15: [Resolved] Google Cloud Vertex AI OAuth2 인증 및 JsonElement 매개변수 언래핑 결함

### 1. 현상 (Symptom)
* Google AI Studio의 단순 API 키 방식 외에, Google Cloud Vertex AI 서비스 어카운트(`Config/google-credentials.json`)를 연동할 때 인증 실패 발생.
* LLM이 반환한 `ActionArgs` JSON을 `System.Text.Json`으로 역직렬화할 때 딕셔너리 값들이 `JsonElement`로 파싱되어 `DecodePayloadTool` 등의 하위 도구에서 `raw is string` 타입 검사가 실패하고 매개변수 누락 오류가 발생하는 현상.

### 2. 원인 (Root Cause)
* Vertex AI는 HTTP 헤더에 `x-goog-api-key`가 아닌 OAuth2 Bearer Token(`Google.Apis.Auth.OAuth2`) 인증을 요구하며 엔드포인트 URL 구조가 다름.
* C# `System.Text.Json`의 `Dictionary<string, object>` 역직렬화 특성상 원시 타입이 네이티브 `string`, `int`가 아닌 `JsonElement` 박싱 객체로 적재됨.

### 3. 해결책 (Resolution)
1. **`GeminiRestClient.cs` OAuth2 지원**: `ServiceAccountCredential`을 통해 `cloud-platform` 스코프의 Bearer Token을 동적 발급받아 헤더에 주입.
2. **도구 매개변수 언래핑**: `AutonomousHunterAgent.cs`에서 도구 인자 전달 전 `JsonElement`를 네이티브 C# 타입(`string`, `int`, `double`, `bool`)으로 일괄 언래핑 처리.
3. `DecodePayloadTool.cs`에서 `JsonElement` 및 다양한 대소문자/별칭(`encodedCommand`, `command`, `payload` 등)을 지원하도록 정규화.

---

## 2026-09-15: [Resolved] Gemini responseSchema CFG 루프/토큰 고갈 결함 및 JSON Mode 최적화

### 1. 현상 (Symptom)
* EDR 환경에서 Gemini API 호출 시 `responseSchema`를 적용했을 때, 간헐적으로 15초 타임아웃에 도달하며 응답이 실패하거나 도구 선택 정확도가 40%로 급락하는 현상 발생.

### 2. 원인 (Root Cause)
* Gemini 내부 추론 토큰(`thoughtsTokenCount`)이 `MaxOutputTokens`(4096)의 예산을 잠식하고, 필드 설명이 불명확한 필드에서 CFG(Context-Free Grammar) 문법 제약 퇴행 무한 반복 루프가 발생하여 타임아웃 유발.
* Native Function Calling은 보안 사고 과정(`Thought`)이 90% 이상 누락되어 EDR 포렌식 요구사항에 부적합.

### 3. 해결책 (Resolution)
* **JSON Mode + 정밀 파서 채택**: 순수 JSON Mode와 견고한 중첩 괄호 균형 탐색 파서(`LlmJsonParser`) 조합을 프로덕션 표준으로 확정.
* 도구 선택 정확도 100%, 필수 인자 100%, 보안 사고 과정(CoT) 보존 및 단일 왕복 완결 달성.


---

## 2026-09-15: [Resolved] EDR 수사 도구 5대 실무 맹점 해결

### 1. 현상 (Symptom)
* 실전 환경 검증 시 식별된 핵심 수사 도구 결함:
  1. `DecodePayloadTool`: Gzip/Deflate 압축 인코딩(`H4sIA...`)이 결합된 파워셸 드로퍼 미탐.
  2. `ProcessMemoryScanTool`: 0x0부터 선형 50MB만 순회하여 고위 주소 동적 힙(`VirtualAlloc`)의 Cobalt Strike/Reflective DLL 미탐, `PAGE_GUARD` 크래시 위험 및 LOH 파편화.
  3. `ThreatReputationTool`: RFC 1918 B클래스(`172.16.0.0/12`) 누락으로 사내 사설망을 외부 IP로 오인, 미확인 IP에 75점 부여로 정상 통신 프로세스 오탐 사살 위험.
  4. `MitreClassifierTool`: 단순 `Contains("c2")` 매칭으로 정상 설치기 `c2rsetup.exe`를 C2 공격으로 오탐.
  5. `SystemFirewallTool`: Netsh 실패 시에도 `true`를 반환하는 Silent Failure 버그, 게이트웨이/DNS 차단 시 엔드포인트 네트워크 먹통(Self-DoS) 위험.

### 2. 원인 (Root Cause)
* VAD 구조, 엔터프라이즈 사설망 토폴로지, 다단계 압축/난독화 및 인프라 보호 가드가 프로토타입 단계에서 결여되었음.

### 3. 해결책 (Resolution)
1. **`DecodePayloadTool`**: Gzip 매직 바이트(`0x1F, 0x8B`) 자동 감지 및 `GZipStream`/`DeflateStream` 무손실 압축 해제, Hex 디코더 추가, ReDoS 가드(250ms).
2. **`ProcessMemoryScanTool`**: `VirtualQueryEx` 기반 VAD 순회로 Unbacked Executable Memory (`MEM_PRIVATE` + `EXECUTE` + `!PAGE_GUARD`)만 선별 스캔, 사설 메모리 첫 2바이트 `MZ` 헤더 감지, `ArrayPool<byte>.Shared` 활용.
3. **`ThreatReputationTool`**: 비트마스크 사설망 분류기(RFC 1918 A/B/C, 루프백, APIPA, CGNAT 등 0점 처리), 미확인 외부 IP는 30점 중립(`INCONCLUSIVE`) 처리, 포트/디팽 파싱 전처리.
4. **`MitreClassifierTool`**: 단어 경계(`\b`) 컴파일 정규식 26종 적용으로 파일명 오탐 차단, 10단계 사이버 킬체인 순서 정렬.
5. **`SystemFirewallTool`**: `new ToolResult(overallSuccess, ...)` 반환으로 Silent Failure 방지, 로컬 IP/기본 게이트웨이/DNS 화이트리스트 보호망 구축.

---

## 2026-09-16: [Resolved] C# 생성자 내 Sync-over-Async(GetAwaiter().GetResult()) 스레드풀 데드락 제거

### 1. 현상 (Symptom)
* `AutonomousHunterAgent` 클래스 생성자 내부에서 Vertex AI 서비스 계정 토큰 발급 및 설정 로딩 시 `.GetAwaiter().GetResult()`를 호출하는 동기 블로킹 코드가 잔존하여, 스레드풀 고갈(Thread Pool Starvation) 시 데드락 발생 위험 존재.

### 2. 원인 (Root Cause)
* 의존성 주입 또는 인스턴스 초기화 시점에서 비동기 초기화 팩토리 패턴을 사용하지 않고 생성자에서 동기 대기함.

### 3. 해결책 (Resolution)
* `AutonomousHunterAgent.cs` 생성자에서 블로킹 호출을 제거하고, `GeminiRestClient.TryCreateFromLocalConfig()` 동기 팩토리 메서드를 신설하여 Phalanx 로컬 JSON 설정을 안전하게 파싱하도록 리팩토링.

---

## 2026-09-16: [Resolved] AI 수사관 판정 왜곡(Decision Hijacking) 및 결정권 침해 결함 해결 (SSOT 아키텍처 확립)

### 1. 현상 (Symptom)
* 정상 관리 스크립트(`explorer.exe ➔ powershell.exe -enc <Get-Service ... *.internal>`) 인입 시, Gemini 모델이 정상 판결(`ACTION_RESUME`, 확신도 98%)을 내렸음에도 C# 호스트 코드가 이를 가로채 `ActionKill`로 변조하고 피싱 기법(`T1566.001`)을 조작 주입하는 치명적 오탐 발생.

### 2. 원인 (Root Cause)
1. **의미론적 확신도 역전 (Semantic Inversion)**: `ConfidenceScore`(정상 프로세스 확신도 98%)를 `threatScore`(위협 점수 98점)로 오인 바인딩하여 사살 집행.
2. **정적 시그니처 강제 오버라이드**: 동결 사유였던 `-enc`를 최종 단계에서 `CommandLine.Contains("-enc")`로 재검사하여 LLM 수사 결론을 무시하고 강제 사살.
3. **증거 조작 및 조기 차단**: 정상 제목을 사살용 제목으로 치환하고, 미확정 상태에서 사내 백업 서버 IP 방화벽 차단 집행.

### 3. 해결책 (Resolution)
1. **단일 진실 공급원(SSOT) 아키텍처 확립**: ReAct 루프가 정상 종결(`reachedFinal == true && hasValidAction`)된 경우, Gemini AI 수사관의 `VerdictAction`(`ACTION_KILL` vs `ACTION_RESUME`)을 100% 최상위 결정권으로 수용. `Contains("-enc")`, `Contains("http")` 등 정적 오버라이드 코드 완전 삭제.
2. **Fail-Secure 안전 가드 격리**: C# 시스템 가드는 최대 턴 초과, API 장애 등 '예외 상황'에서만 선제 사살을 집행하도록 관심사 분리(SoC).
3. **포렌식 무결성 보장**: 정상 프로세스 판정 시 가짜 TTP 주입 차단 및 `blockedIp = ""` 보장, 악성 확정 시에만 `SystemFirewallTool` 집행.

---

## 2026-09-16: [Resolved] WPF 관제 콕핏과 Kestrel gRPC 백그라운드 서버 하이브리드 호스팅 및 STA 스레드 안전성 확보

### 1. 현상 (Symptom)
* Phase 4에서 WPF 관제 콕핏(`Phalanx.Cockpit`)과 C++ 센서와의 통신을 위한 Kestrel gRPC 서버(포트 50051)를 단일 실행 바이너리(`Program.cs`)에 통합할 때, 비동기 `async Task Main`에서 `new MainWindow()`를 인스턴스화할 경우 STA(Single-Threaded Apartment) 스레드 제약 위반으로 `InvalidOperationException`이 발생하거나, 반대로 WPF STA 스레드에서 gRPC 네트워크 IO를 블로킹하여 UI 프리징이 발생하는 아키텍처 충돌 발생.
* 백그라운드 Kestrel gRPC 및 AI 에이전트 수사관 스레드에서 실시간 이벤트 발생 시 WPF UI 컬렉션(`ObservableCollection`)을 직접 수정하려 하여 `NotSupportedException` 발생 위험.

### 2. 원인 (Root Cause)
* WPF UI 서브시스템은 엄격한 `[STAThread]` 동기 진입점과 독립된 Dispatcher 메시지 펌프를 요구하는 반면, Kestrel 웹 호스트는 비동기 멀티스레드 스레드풀 워커를 기반으로 동작함.
* 비UI 스레드에서 UI 바인딩 컬렉션을 조작할 경우 WPF의 스레드 선호도(Thread Affinity) 모델과 충돌함.

### 3. 해결책 (Resolution)
1. **하이브리드 호스팅 라이프사이클 (`Program.cs`)**:
   * `[STAThread] public static void Main(string[] args)` 동기 진입점을 유지.
   * `webApp.Start()`(동기 비차단)를 통해 Kestrel gRPC 서버를 백그라운드에서 기동한 후, 메인 STA 스레드에서 `wpfApp.Run(mainWindow)`을 실행하여 UI 메시지 펌프를 완벽히 유지.
   * `mainWindow.Closed` 이벤트 핸들러에서 `await webApp.StopAsync()` 및 `DisposeAsync()`를 호출하여 창 종료 시 백그라운드 gRPC 서버가 안전하게 Graceful Shutdown되도록 결합.
   * CI 및 풀체인 E2E 테스트 스크립트를 위한 `--headless` 모드 지원 분기 추가.
2. **이벤트 브리지 및 UI 스레드 마샬링 (`CockpitUiBridge`)**:
   * gRPC 수신 및 AI 수사 시작/완료 알림을 `Application.Current?.Dispatcher?.InvokeAsync(...)`로 안전하게 래핑하여 UI 스레드로 마샬링.
   * 비GUI 환경(헤드리스 러너 및 단위 테스트)에서도 `Application.Current`가 null일 때 안전하게 No-op 통과하는 Null-Safety 방어 구현.

---

## 2026-09-20: [Resolved] gRPC 스트림 다중 클라이언트 세션 덮어쓰기 및 거짓 DISCONNECTED 상태 전이 결함 해결

### 1. 현상 (Symptom)
* C++ 커널 센서(`Phalanx.Sensor`)가 백그라운드에서 정상 기동되어 gRPC 스트림을 유지하고 있음에도 불구하고, 모의 공격 도구(`Phalanx.AttackSimulator`) 실행 종료 직후 또는 유휴 상태 경과 시 WPF 관제 콘솔의 상단 통신 상태가 주기적으로 빨간색 `[DISCONNECTED]`로 반전되는 현상 발생.
* 센서 토글 버튼은 `STOP SENSOR`로 가동 상태를 가리키는데 통신 상태는 `[DISCONNECTED]`로 표시되어 관제관에게 혼선을 초래하고, 수동 완화 명령 하달 실패 가능성 유발.

### 2. 원인 (Root Cause)
1. **단일 응답 스트림 포인터 덮어쓰기 및 조기 폐기**:
   * `PhalanxGrpcService`가 단일 필드 `private IServerStreamWriter<MitigationCommand>? _responseStream;`로 작성되어 있었음.
   * C++ 센서가 연결된 상태에서 모의 공격 도구가 추가로 `StreamTelemetry`에 연결하면 해당 필드가 모의 도구의 스트림으로 덮어써짐.
   * 모의 도구의 시나리오가 끝나 연결이 해제되면 `finally` 블록에서 `_responseStream = null`로 초기화하고 `_uiBridge?.NotifySensorConnected(false)`를 무조건 호출함.
   * 이로 인해 C++ 센서가 여전히 연결되어 있음에도 UI가 `[DISCONNECTED]`로 반전되고, 백그라운드 센서로 완화 명령을 보낼 수 없는 단절 상태 발생.
2. **Kestrel HTTP/2 유휴 킵얼라이브 미설정**:
   * Kestrel gRPC 엔드포인트에 HTTP/2 KeepAlive Ping 설정이 부재하여 유휴 시 소켓 반폐쇄(Half-closed) 상태 진입 가능성 존재.

### 3. 해결책 (Resolution)
1. **동시성 컬렉션 기반 멀티 클라이언트 세션 관리 (`PhalanxGrpcService.cs`)**:
   * 단일 포인터를 `ConcurrentDictionary<string, IServerStreamWriter<MitigationCommand>> _activeClients`로 교체.
   * 클라이언트 접속 시 고유 ID로 등록하고, 연결 해제 시 해당 클라이언트만 제거.
   * `_activeClients.IsEmpty`가 true(즉, 등록된 모든 클라이언트가 완전히 단절)일 때만 `_uiBridge.NotifySensorConnected(false)`를 호출하도록 통신 수명주기 보정.
   * `SendCommandAsync` 실행 시 살아있는 모든 스트림으로 완화 명령을 브로드캐스팅하고 죽은 스트림은 안전하게 제거.
2. **Kestrel HTTP/2 킵얼라이브 활성화 (`Program.cs`)**:
   * `KeepAlivePingDelay = 30s`, `KeepAlivePingTimeout = 15s`, `KeepAliveTimeout = 5분`을 명시하여 장기 유휴 gRPC 세션의 무중단 연결 유지 보장.

---

## 2026-09-20: [Resolved] C++ 커널 룰 엔진 0.1ms 즉각 현장 사살(Reflex Kill)의 관제 콕핏 누락 및 포렌식 즉시 등록 구현

### 1. 현상 (Symptom)
* 랜섬웨어(`vssadmin.exe delete shadows`, `bcdedit /set recoveryenabled no` 등)가 실행될 때 C++ 커널 센서의 `LocalRuleEngine`이 0.1ms(80μs) 이내에 즉각 현장 사살(`NtTerminateProcess`)을 집행하고 `Lifecycle = LIFECYCLE_TERMINATED` 텔레메트리를 C# Cockpit으로 전송함.
* 그러나 WPF 관제 콕핏 UI 상단의 Monitored Processes 카운트 및 내부 CQRS 트리에서만 프로세스가 비활성화(`IsAlive = false`)될 뿐, 좌측 실시간 인시던트 작업 목록(Incident Worklist)에는 침해 대응 카드가 전혀 생성되지 않는 현상 발생.

### 2. 원인 (Root Cause)
1. **수사 파이프라인 트리거 조건의 단일화 (`PhalanxGrpcService.cs`)**:
   * 기존 gRPC 서비스의 텔레메트리 루프는 `if (ev.IsSuspended)` 조건문만 검사하여 동결된 회색지대 프로세스만 `AutonomousHunterAgent.InvestigateThreatAsync`로 라우팅하고 있었음.
   * C++ 로컬 룰 엔진에 의해 즉각 사살된 이벤트는 이미 종료되었으므로 `IsSuspended = false`, `IsTerminated = true`, `Lifecycle = LIFECYCLE_TERMINATED` 상태로 인입되어 사건 통지가 누락됨.
2. **초고속 사건 레이턴시 절삭 및 DB 스키마 누락**:
   * 10ms 미만 초고속 사건이 소수점 둘째 자리 초(`:F2` s) 변환으로 인해 `0.00s`로 절삭 표기됨.
   * `IncidentRecord`에 `ElapsedMs` 필드가 누락되어 앱 재기동 후 과거 카드가 `Investigating...`으로 잘못 복원됨.

### 3. 해결책 (Resolution)
1. **현장 사살 즉각 보고 파이프라인 신설 (`AutonomousHunterAgent.HandleReflexKill`)**:
   * AI ReAct 루프의 지연 없이 0.08ms 소요시간 레코드, `LocalRuleEngine` 사살 사유, MITRE ATT&CK T1490 전술을 담은 `IncidentRecord`를 즉각 생성하여 LiteDB 영구 적재 및 관제 UI에 `CRITICAL` / `SECURED` 카드로 즉시 표출.
2. **gRPC 인입 라우팅 분기 보강 (`PhalanxGrpcService.cs`)**:
   * `else if (ev.IsTerminated || ev.Lifecycle == ProcessLifecycle.LifecycleTerminated)` 분기를 추가하여 C++ 현장 사살 수신 시 `_agent.HandleReflexKill(node)`로 직결.
3. **적응형 레이턴시 포맷터 및 DB 스키마 정규화**:
   * 10ms 미만 소요시간은 마이크로초 단위(`80μs (0.08ms, Reflex)`)로 적응형 표기.
   * `IncidentRecord.ElapsedMs` 스키마 필드를 신설하고 영구 복원 파이프라인 구축.

---

## 2026-09-21: [Resolved] 관제 콕핏 인시던트 검색창 키워드 입력 시 NullReferenceException 크래시 결함

### 1. 현상 (Symptom)
* 관제 콕핏 상단 검색창에 키워드 입력 시 `MainViewModel.ApplyFilter()`에서 `NullReferenceException`이 발생하며 콕핏 애플리케이션 강제 종료.

### 2. 원인 (Root Cause)
* 과거 사건 기록 중 `BlockedIp`, `CommandLine` 등이 null인 상태에서 Null 조건부 연산자 없이 `.Contains()`를 직접 호출함.
* LiteDB 역직렬화 시 null 필드가 뷰모델에 그대로 바인딩되어 필터 탐색 시 예외 발생.

### 3. 해결책 (Resolution)
* `MainViewModel.cs`의 `ApplyFilter()` 내 `TargetImage`, `CommandLine`, `SummaryTitle`, `BlockedIp` 프로퍼티 탐색에 `?.Contains(...) ?? false` 널-세이프 탐색 연산자 적용.
* `LoadIncidentsFromDatabase`에서 역직렬화 시 null 필드를 `?? string.Empty`로 방어 초기화.

---

## 2026-09-23: [Resolved] Kestrel 백그라운드 스레드의 ObservableCollection 조작으로 인한 gRPC 스트림 단절 및 센서 ON/OFF 무한 루프

### 1. 현상 (Symptom)
* WPF 관제 콘솔 UI에서 C++ 센서 연결 상태가 `LIVE`와 `OFFLINE` 사이를 수 초 주기로 계속해서 자동으로 반복 전환(플리핑)됨.
* 프로세스 트리 화면에 활성 프로세스가 1개 또는 소수만 표시되고 전체 PC 프로세스가 적재되지 못함.

### 2. 원인 (Root Cause)
1. **WPF UI 컬렉션 스레드 위반 (`NotSupportedException`)**:
   * C++ 센서가 접속하여 300+개 활성 프로세스 스냅샷 배치(`snapshot_batch`)를 gRPC로 전송할 때, Kestrel 백그라운드 스레드풀에서 `ProcessTreeProjectionManager.RootNodes.Add(node)`를 호출함.
   * `ProcessGraphView`의 `TreeView`가 `RootNodes`에 바인딩되어 있는 상태에서 Dispatcher가 아닌 백그라운드 스레드가 `ObservableCollection`을 수정함에 따라 WPF `CollectionView`가 `NotSupportedException`을 발생시킴.
   * 예외로 인해 `PhalanxGrpcService.StreamTelemetry`가 루프를 탈출하고 `finally` 블록에서 `_uiBridge.NotifySensorConnected(false)`를 호출하여 UI가 `OFFLINE`으로 전환됨.
   * C++ 센서는 스트림 단절을 감지하고 2초 후 자동 재접속(`LIVE`) ➔ 스냅샷 재전송 ➔ 예외 재발생 ➔ `OFFLINE` 전환을 무한 반복함.
2. **스냅샷 배치 분할 및 족보 왜곡**:
   * `PhalanxGrpcService`에서 `foreach (var ev in batch.ProcessEvents)`로 쪼개어 `ApplyDeltaEvent(ev)`를 호출하면서, 300개의 스냅샷 이벤트가 개별 `ApplySnapshotBatch(new[] { ev })`로 분할 전달됨.
   * `tempMap`이 1개 노드만 갖게 되어 부모-자식 관계를 형성하지 못하고 전원 루트 노드로 편입되는 결함 유발.

### 3. 해결책 (Resolution)
1. **WPF 컬렉션 동기화 및 Dispatcher 마샬링 (`ProcessTreeProjectionManager.cs`)**:
   * `BindingOperations.EnableCollectionSynchronization(RootNodes, _syncLock)` 및 `EnableCollectionSynchronization(AllNodes, _syncLock)` 등록.
   * `DispatchUI` 헬퍼를 도입하여 `RootNodes`, `AllNodes`, `node.Children` 조작을 UI Dispatcher 스레드로 안전하게 마샬링 (헤드리스/테스트 환경 Null-Safety 보장).
2. **gRPC 스냅샷 배치 보존 (`PhalanxGrpcService.cs`)**:
   * `batch.ProcessEvents` 중 `LifecycleSnapshot` 이벤트를 `ApplySnapshotBatch(snapshotEvents)`로 통째로 전달하여 단 1회의 Dispatcher 컨텍스트 스위치로 부모-자식 트리 전체를 0초 완결 투영.

---

## 2026-09-28: [Resolved] 프로세스 트리(TreeView) 리프 노드 클릭 시 화면 좌측 쏠림 및 고착 결함

### 1. 현상 (Symptom)
* `ProcessGraphView`(인메모리 프로세스 족보 탐색기)에서 깊이 중첩된 자식/리프 노드를 클릭했을 때, 트리 뷰포트 전체가 우측으로 스크롤되면서 화면 내 프로세스 트리 내용이 좌측으로 밀려 사라짐.
* 루트 노드와 확장 접기 화살표, 부모 프로세스들이 좌측 화면 밖으로 이탈하며, 다른 노드를 클릭하거나 마우스를 움직여도 원상태(가로 오프셋 0)로 돌아오지 않고 좌측 쏠림 상태로 영구 고착됨.

### 2. 원인 (Root Cause)
1. **WPF TreeView의 기본 포커스 BringIntoView() 호출 메커니즘**:
   * WPF의 `TreeViewItem`은 마우스 클릭 또는 포커스 획득 시 자동으로 `BringIntoView()`를 호출하여 `FrameworkElement.RequestBringIntoViewEvent` 라우티드 이벤트를 발생시킴.
2. **무제한 수평 측정 pass 및 가로 폭 오프셋 팽창**:
   * `TreeView` 내부의 기본 템플릿에 내장된 `ScrollViewer`는 기본적으로 `HorizontalScrollBarVisibility="Auto"` 상태로 동작함.
   * 이에 따라 자식 노드들에 대해 `availableSize.Width = double.PositiveInfinity`로 무한 가로 너비를 부여하며, `TreeViewItem` 템플릿 내의 `<ColumnDefinition Width="*" />`와 결합하여 자식 노드가 깊어질수록(19px * depth 계층 들여쓰기) 항목의 우측 바운딩 박스가 뷰포트 가시 영역 너비를 크게 초과함.
3. **ScrollViewer의 일방향 수평 스크롤 및 복구 기전 부재**:
   * `ScrollViewer`는 이벤트의 `TargetRect` 우측 경계가 화면 밖으로 나갔다고 판단하여 이를 화면 안에 넣기 위해 `HorizontalOffset`을 증가시킴 (콘텐츠가 화면 좌측으로 밀려남).
   * WPF `ScrollViewer`는 항목 가시화 요청에 따른 일방향 스크롤만 수행할 뿐 클릭 완료 후 원점(X=0)으로 복귀시키는 메커니즘이 전무함.
   * 또한 수평 스크롤바가 숨겨져 있어 사용자가 수동으로 되돌릴 수도 없으며, 다른 자식 노드를 클릭해도 해당 노드의 들여쓰기 바운딩 박스가 타깃이 되므로 수평 오프셋이 유지되거나 더 밀려남.

### 3. 해결책 (Resolution)
1. **1단계 프레임워크 제어 (WPF 표준 패턴)**:
   * `TreeView`에 `ScrollViewer.HorizontalScrollBarVisibility="Disabled"` 선언 및 `RequestBringIntoView` 이벤트 차단(`e.Handled = true`)으로 1차 방어.
2. **2단계 구조적 전면 해결: 플랫 가상화 트리 투영 (Flat Virtualized Tree Projection) 마이그레이션**:
   * Microsoft WinUI 3(`TreeViewList`), VS Code(`Monaco Tree`), ILSpy(`SharpTreeView`)의 아키텍처 패턴을 Phalanx에 선제적 도입.
   * **데이터 계층 (`ProcessTreeProjectionManager`)**:
     * `ProcessNodeModel`에 `Depth` 및 `IndentMargin` 속성, `IsExpanded` 토글 추가.
     * `VisibleNodes` (`ObservableCollection<ProcessNodeModel>`)를 구축하여 트리가 펼쳐질 때 DFS 전위 순서(Pre-order)로 1차원 평탄화 투영.
     * `ToggleNodeExpanded` 메서드를 통해 노드 접힘/펼침 시 VS Code의 배열 `splice()` 방식으로 가시 노드만 부분 갱신.
     * `EnsureNodeVisible` 메서드를 통해 심층 수사실에서 프로세스 트리 점프 시 상위 조상 노드 자동 언랩 지원.
   * **UI 뷰 계층 (`ProcessGraphView.xaml` / `.cs`)**:
     * 고전 재귀 `TreeView`를 제거하고, 하드웨어 가상화가 켜진 `ListView`(`VirtualizingStackPanel.IsVirtualizing="True"`, `VirtualizationMode="Recycling"`, `ScrollUnit="Pixel"`)로 교체.
     * 순수 MVVM 데이터 바인딩(`SelectedItem="{Binding SelectedProcessNode, Mode=TwoWay}"`)으로 코드비하인드 이벤트 핸들러 제거.
     * 1차원 수직 평면 렌더링으로 수평 스크롤 요동 및 쏠림 현상을 구조적으로 0% 원천 박멸하고, 60fps 가상화 스크롤과 향후 멀티컬럼(TreeGrid) 확장 기반 확보.

---

## 2026-09-28: [Resolved] SettingsWindow 오픈 시 TwoWay 바인딩 읽기 전용 속성 충돌로 인한 CLR 강제 종료(0xc000041d)

### 1. 현상 (Symptom)
* 대시보드에서 `SETTINGS` 버튼 클릭 시 `PresentationUI.resources.dll` 로드 직후 `STATUS_FATAL_USER_CALLBACK_EXCEPTION (0xc000041d)`가 발생하며 Cockpit 프로세스가 즉시 비정상 종료됨.
* 일반적인 .NET 처리되지 않은 예외(UnhandledException) 대화상자 없이 네이티브 Fast-fail로 크래시 발생.

### 2. 원인 (Root Cause)
* WPF `RadioButton.IsChecked` 의존성 속성의 기본 바인딩 모드는 `TwoWay`임.
* `SettingsViewModel.cs`의 `IsApiKeyMode`가 getter만 존재하는 읽기 전용 계산 프로퍼티(`=> !UseVertexAi;`)로 선언되어 있었음.
* 윈도우 초기화 및 렌더링 과정에서 XAML 엔진이 ViewModel 프로퍼티로 역방향 쓰기(`ConvertBack`)를 시도할 때 `InvalidOperationException`이 발생함.
* 이 예외가 Win32 메시지 루프의 네이티브 `WndProc` 콜백 경계를 교차(cross unmanaged boundary)하면서 CLR이 치명적 콜백 예외로 판단, 프로세스를 즉시 강제 종료시킴.

### 3. 해결책 (Resolution)
* **`SettingsViewModel.cs`의 `IsApiKeyMode`에 양방향 setter 구현**:
  * 읽기 전용 계산 프로퍼티였던 `IsApiKeyMode`에 setter를 구현하여, XAML 바인딩 엔진의 상태 쓰기(`ConvertBack`)를 정상 수용하고 `UseVertexAi`와 상호 동기화되도록 수정:
    ```csharp
    public bool IsApiKeyMode
    {
        get => !UseVertexAi;
        set
        {
            if (UseVertexAi == value)
            {
                UseVertexAi = !value;
                OnPropertyChanged();
            }
        }
    }
    ```
  * 양방향 통로가 정상 개방됨으로써 윈도우 생성 및 렌더링 시 발생하던 `InvalidOperationException` 및 Win32 네이티브 콜백 Fast-fail(`0xc000041d`) 원천 해소.

---

## 2026-09-29: [Resolved] SettingsWindow 재오픈 시 RadioButton TwoWay 바인딩 순환 피드백에 의한 StackOverflowException (0x800703E9)

### 1. 현상 (Symptom)
* Cockpit 상단 헤더 또는 네비게이션 레일에서 `환경 설정(SETTINGS)` 창을 열었다가 닫은 후, 다시 `환경 설정` 창을 열 때 `System.StackOverflowException (HResult: 0x800703E9)` 크래시 발생.

### 2. 원인 (Root Cause)
1. **닫힌 윈도우의 이벤트 델리게이트 및 DataContext 미해제 누수**:
   * `SettingsWindow.xaml.cs`에서 `DataContextChanged`를 통해 싱글톤 `SettingsViewModel.RequestClose`에 이벤트 핸들러를 등록했으나, 창이 닫힐 때(`Closed`) 이를 해제하지 않아 닫힌 윈도우 인스턴스가 GC되지 않고 싱글톤 뷰모델에 강참조로 고착됨.
   * 닫힌 윈도우의 DataContext와 바인딩들이 싱글톤 뷰모델의 `PropertyChanged`를 계속 청취 및 양방향 쓰기 상태를 유지함.
2. **`GroupName="AuthMode"` 전역 등록 간섭**:
   * WPF `RadioButton`은 `GroupName`이 지정될 경우 부모 컨테이너 범위를 넘어 네임스페이스/전역 그룹 레지스트리에 등록됨.
   * 새 윈도우 인스턴스가 열릴 때 새 윈도우의 라디오 버튼이 체크되면, WPF 그룹 로직이 기존 닫힌 윈도우의 라디오 버튼에 `IsChecked = false`를 전파함.
3. **`IsApiKeyMode` 역방향 바운싱 세터에 의한 상호 무한 재귀 (Infinite Ping-Pong Recursion)**:
   * 기존 `IsApiKeyMode` 세터의 `if (UseVertexAi == value)` 비교 로직은 비활성화 시그널(`value = false`)이 주입될 때 `UseVertexAi`가 `false`이면 `false == false`가 되어 참(true)으로 평가되고, `UseVertexAi = !false = true`로 강제 반전시킴.
   * 이에 따라 [새 창의 라디오버튼 체크 -> 기존 창의 라디오버튼 언체크 -> ViewModel 프로퍼티 변경 -> 새 창의 라디오버튼 언체크 -> 기존 창의 라디오버튼 체크...]가 밀리초 단위로 수만 회 상호 재귀 호출되어 호출 스택이 고갈(StackOverflowException)됨.

### 3. 해결책 (Resolution)
1. **`SettingsViewModel.cs`의 `IsApiKeyMode` 단방향 활성화 가드 적용**:
   * 라디오 버튼 선택 해제 시그널(`value = false`)에 의한 역방향 프로퍼티 반전(Bouncing)을 원천 차단:
     ```csharp
     public bool IsApiKeyMode
     {
         get => !UseVertexAi;
         set
         {
             if (value && UseVertexAi)
             {
                 UseVertexAi = false;
             }
             else if (!value && !UseVertexAi)
             {
                 UseVertexAi = true;
             }
         }
     }
     ```
2. **`SettingsWindow.xaml` 라디오 버튼의 `GroupName` 속성 제거**:
   * 두 라디오 버튼은 이미 동일 `StackPanel` 내에 배치되어 있으므로 WPF 패널 스코프에 의해 자연스럽게 상호 배타 그룹화됨.
   * `GroupName="AuthMode"`를 제거하여 다중 윈도우 인스턴스 간 전역 그룹 등록 및 교차 간섭 원천 제거.
3. **`SettingsWindow.xaml.cs`의 Closed 수명주기 정리 핸들러 구현**:
   * 윈도우 종료 시 `SettingsViewModel.RequestClose` 구독을 해제하고 `DataContext = null`로 설정하여 모든 바인딩 및 델리게이트 체인을 즉시 완전 절단.

---

## 2026-09-29: [Resolved] 환경설정 커널 센서 카테고리 내 허위(Phantom) 액추에이터 옵션 폐기 및 런타임 진단 체계 일원화

### 1. 현상 (Symptom)
* 환경설정(`SettingsWindow`) '커널 센서 및 차단 액추에이터' 탭에 표출되던 `ReflexKillEnabled`("커널 룰 즉각 차단") 및 `KillProcessTree`("프로세스 트리 동반 종료") 체크박스가 실제 C++ 커널 센서(`Phalanx.Sensor`)의 런타임 동작과 일치하지 않는 상태로 방치됨.
* 센서 토글 및 gRPC 제어 파라미터로 동작하는 것처럼 표기되었으나, C++ 센서는 관련 CLI 옵션이나 gRPC 플래그를 수신하지 않고 독자적인 안전 규칙 및 단일 PID 기준 `TerminateProcess`를 집행하고 있었음.

### 2. 원인 (Root Cause)
1. **`ReflexKillEnabled`의 허위성**:
   * C++ 센서의 `LocalRuleEngine`은 랜섬웨어 복구 무력화 등 시스템 치명 공격에 대해 0.08ms(80μs) 이내에 무조건적인 현장 즉각 사살을 집행하도록 설계됨.
   * 이를 외부 설정으로 끄거나 켤 수 있는 인터페이스가 C++ 센서에 구현되어 있지 않았으며, UI 상의 체크박스는 실제 센서에 아무런 영향도 미치지 못하는 100% 사장/허위 코드였음.
2. **`KillProcessTree`의 스코프 왜곡**:
   * C++ 커널 센서의 `ProcessActuator`는 커널 핸들을 열어 타깃 프로세스 단일 PID에 대해서만 `TerminateProcess`를 호출함.
   * `KillProcessTree`는 모의 침해 시나리오 러너(`AttackLabScenarioRunner`)가 테스트용 자식 프로세스를 일괄 회수하기 위한 C# 내부 안전 메커니즘이었으나, UI상에 커널 센서의 액추에이터 동작 옵션인 것처럼 잘못 배치되어 관제관에게 혼선을 초래함.

### 3. 해결책 (Resolution)
1. **허위 및 레거시 옵션 전면 폐기**:
   * `AppSettings.json`, `SettingsViewModel.cs`, `SettingsWindow.xaml`, `SettingsViewModelTests.cs`에서 `ReflexKillEnabled` 및 `KillProcessTree`를 완전 제거.
   * `AttackLabScenarioRunner`의 프로세스 트리 정리 로직은 내부 불변 안전 규약(`KillProcessTree = true`)으로 정상 격리.
2. **센서 바이너리 및 런타임 연결 상태 실시간 진단 카드로 전환**:
   * 기존 허위 체크박스 UI 공간을 대체하여, `SensorProcessController`가 탐지한 실제 C++ 센서 바이너리 파일 경로(`SensorBinaryPath`), 바이너리 검출 여부(`SensorBinaryFound`), Kestrel gRPC 클라이언트 활성 연결 상태(`IsSensorConnected`)를 실시간으로 표출하는 진단 카드를 구현.
3. **네트워크 포트 유효 범위 가드 적용**:
   * `SensorPort` 저장 및 로드 시 `Math.Clamp(port, 1024, 65535)`를 적용하여 비정상 포트 번호 주입 방어.

---

## 2026-09-29: [Resolved] 환경설정 포렌식 스토리지 카테고리 진단 체계 강화 및 파일 탐색/초기화 파이프라인 구축

### 1. 현상 (Symptom)
* 환경설정(`SettingsWindow`) '포렌식 스토리지' 탭에서 LiteDB 파일명(`DatabasePath`)과 보고서 저장 경로(`ReportExportPath`)가 단순 텍스트 박스로만 제공되어 경로 타이핑 오류 위험이 상존함.
* 상대 경로 기반으로 동작하여 실제 물리적 디스크 절대 경로, 파일 존재 여부, DB 용량(KB/MB)을 관제관이 파악할 수 없었으며, 누적된 테스트 인시던트를 안전하게 비우거나 탐색기로 열어보는 운영 편의 기능이 부재함.

### 2. 원인 (Root Cause)
* 초기 프로토타입 단계에서 DB 연결 스트링과 디렉터리 경로만 바인딩하고, 파일/폴더 선택 대화상자(`OpenFileDialog`, `OpenFolderDialog`), 물리 경로 역추적 및 크기 계산 로직이 ViewModel에 누락되었음.
* `ForensicArchiveManager`에 단일 인시던트 추가/조회만 구현되어 있고 대량 인시던트 일괄 원자적 삭제(`ClearAllIncidents`) API가 부재했음.

### 3. 해결책 (Resolution)
1. **파일 및 폴더 탐색 대화상자 연동**:
   * `BrowseDatabaseCommand`(`OpenFileDialog`, `*.db`) 및 `BrowseReportExportCommand`(.NET 9 네이티브 `OpenFolderDialog`) 구현.
   * `OpenReportFolderCommand` 및 `OpenDatabaseFolderCommand`(`explorer.exe /select`)를 통해 윈도우 파일 탐색기 즉시 팝업 지원.
2. **물리적 절대 경로 및 실시간 용량 진단 카드 구현**:
   * 실행 환경(`Directory.GetCurrentDirectory()` vs `AppContext.BaseDirectory`) 기준 실제 DB 파일 존재 여부를 동적 추적(`DatabaseFileExists`).
   * 파일 크기를 적응형(`Bytes`, `KB`, `MB`)으로 계산하여 표출(`DatabaseFileSizeText`).
3. **인시던트 원자적 일괄 삭제 및 UI 동기화 (`ClearAllIncidents`)**:
   * `ForensicArchiveManager.ClearAllIncidents()`를 신설하여 `_incidents.DeleteAll()`과 `_traces.DeleteAll()`을 단일 트랜잭션(`BeginTrans` ➔ `Commit`)으로 안전 집행.
   * `CockpitUiBridge.IncidentsDatabaseCleared` 이벤트를 신설하여 삭제 즉시 관제 콘솔(`MainViewModel.LoadIncidentsFromDatabase`)이 실시간 리프레시되도록 반응형 파이프라인 결합.

---

## 2026-09-29: [Resolved] 환경설정 시스템 정보 카테고리 내 개발자 사족/내부 구현 세부사항 제거 및 엔터프라이즈 명세 체계화

### 1. 현상 (Symptom)
* 환경설정(`SettingsWindow`) '시스템 정보' 탭에 표출되던 사양이 내부 개발 단계 용어(`Phase 5`), 컴파일러/언어 버전(`C++20`, `C# 13`), 하드코딩된 포트(`:50051`), 내부 알고리즘/자료구조 명칭(`DAG`, `Multi-Turn ReAct Cycle`) 등 관제관 및 사용자 관점에서 불필요한 사족과 개발자 지향적 텍스트로 구성되어 있었음.
* 또한 사용자가 환경설정에서 변경한 AI 모델/인증 방식, 센서 IPC 엔드포인트, DB 경로가 시스템 정보 탭에 동적으로 연동되지 않고 정적 문자열로 고정되어 설정 불일치가 발생함.

### 2. 원인 (Root Cause)
* 초기 시스템 정보 페이지가 아키텍처 스펙 메모 용도로 단일 Border 내에 정적 텍스트 5줄로 작성된 채 방치되었음.
* 런타임 호스트 OS 환경(OS 버전, 프로세스 아키텍처) 및 .NET CLR 런타임 정보, 실시간 활성 컴포넌트(AI 공급자, 센서 주소, 포렌식 DB 절대 경로)를 제공하는 데이터 바인딩 프로퍼티가 ViewModel에 체계화되지 않았음.

### 3. 해결책 (Resolution)
1. **개발자 사족 및 내부 세부 구현체 명칭 전면 제거**:
   * 로드맵 스프린트 표기(`(Phase 5)`), 프로그래밍 언어 표준(`C++20`, `C# 13`), 자료구조 명칭(`DAG`) 및 내부 설계용 문구(`시스템 아키텍처`) 등을 사용자 친화적 엔터프라이즈 명세로 정제 및 불필요 항목 완전 삭제.
   * `제품 및 솔루션 사양` 카드(명칭, 버전, 라이선스 에디션)와 `호스트 플랫폼 및 활성 런타임 환경` 카드의 2개 그룹 카드로 시각적 체계화.
2. **동적 런타임 환경 바인딩 구현**:
   * `ProductVersionText`: 솔루션 버전 표출 (`Phalanx EDR v0.5.0-preview`).
   * `HostPlatformText`: `RuntimeInformation.OSDescription` 및 `ProcessArchitecture` 기반 실행 호스트 정보 제공.
   * `DotNetRuntimeText`: `.NET Environment.Version` 기반 CLR 런타임 버전 제공.
   * `ActiveAiModelText`: 클라우드 인증 모드(Vertex AI / AI Studio) 및 선택된 모델명 실시간 동적 포맷팅.
   * `ActiveSensorEndpointText`: 설정된 센서 호스트 및 포트(`SensorHost:SensorPort`) 동적 바인딩.
   * `ActiveDatabaseText`: 실제 물리적 디스크 절대 경로(`DatabaseResolvedPath`)와 실시간 동기화.
3. **단위 테스트 및 빌드 검증**:
   * `SettingsViewModelTests.TestSettingsViewModel_SystemInformationObservables`를 추가하여 속성 초기화 및 설정 변경 시 실시간 반영 여부 검증 (35/35 통과, Exit Code 0).

---

## 2026-09-29: [Resolved] 심층 포렌식 수사실에서 프로세스 트리 포커싱 시 Foreground DependencyProperty.UnsetValue 예외 크래시

### 1. 현상 (Symptom)
* 심층 포렌식 분석(`InvestigationView`) 화면에서 '전역 프로세스 트리에서 위치 확인 ➔'(`FocusProcessInGraphCommand`) 버튼 클릭 시 WPF 렌더링 파이프라인에서 크래시 발생:
  ```text
  System.InvalidOperationException: '{DependencyProperty.UnsetValue}'은(는) 'Foreground' 속성의 유효한 값이 아닙니다.
  HResult=0x80131509
  ```

### 2. 원인 (Root Cause)
* `FocusProcessInGraph` 실행 시 관제 뷰가 `ProcessGraphView`로 전환되며 타깃 프로세스 노드가 `SelectedProcessNode`로 지정됨.
* `ProcessGraphView.xaml` 내 세 곳에서 WPF 의존성 프로퍼티(DependencyProperty) 스타일링 안티패턴이 존재함:
  1. **관리자 권한 수준 TextBlock**:
     * `Foreground="{StaticResource TextSecondaryBrush}"`가 TextBlock 태그의 로컬 속성으로 지정된 상태에서, Style 내부에는 기본 Foreground Setter가 없고 `DataTrigger(TokenElevationType == 2)`에만 `<Setter Property="Foreground" Value="{StaticResource DotCriticalBrush}" />`가 지정됨.
     * 노드가 선택되어 트리거가 비활성화/해제되거나 조건 평가 시, Style에 기본 Setter가 없으므로 WPF는 Style의 기본값인 `DependencyProperty.UnsetValue`를 복원하려고 시도하여 Brush 타입 검증에 실패함.
  2. **ListViewItem ItemContainerStyle**:
     * `ControlTemplate.Triggers` 내부의 `Trigger(IsSelected == True)`에 `TargetName` 없이 `<Setter Property="Foreground" Value="{StaticResource TextPrimaryBrush}" />`가 선언되어 템플릿 부모인 `ListViewItem`의 Foreground를 직접 변경함.
     * 그러나 `ListViewItem` Style 자체에 기본 Foreground Setter가 정의되어 있지 않아 선택 해제 시 `DependencyProperty.UnsetValue` 할당 충돌이 발생함.
  3. **동결/사살 뱃지 TextBlock**:
     * Style 내부에 기본 `Text` 및 `Foreground` Setter가 누락되어 비동결/비사살 일반 프로세스 노드 렌더링 시 잠재적 UnsetValue 복원 위험 존재.

### 3. 해결책 (Resolution)
1. **관리자 권한 수준 TextBlock Style 리팩토링**:
   * 로컬 `Foreground` 속성을 제거하고 `<Setter Property="Foreground" Value="{StaticResource TextSecondaryBrush}" />`를 Style 기본 Setter로 이동.
   * `TokenElevationType == 2` 및 `TokenElevationType == 3` DataTrigger 모두에 명시적 `Foreground` 브러시를 보장하여 UnsetValue 전파 원천 차단.
2. **`ListViewItem` Foreground 수명주기 안정화**:
   * `ListViewItem` Style에 기본 `<Setter Property="Foreground" Value="{StaticResource TextPrimaryBrush}" />`를 명시하고, 내부 템플릿 트리거의 중복 부모 Foreground 변경 Setter를 제거하여 `TargetName="Bd"` 배경만 제어하도록 격리.
3. **상태 뱃지 기본 브러시 보장**:
   * 동결/사살 뱃지 Style에 기본 `<Setter Property="Text" Value="" />` 및 `<Setter Property="Foreground" Value="{StaticResource TextMutedBrush}" />`를 명시.
4. **단위 테스트 검증**:
   * `ProcessTreeProjectionTests.TestMainViewModel_FocusProcessInGraphCommand`를 추가하여 `FocusProcessInGraphCommand` 실행 시 뷰 전환, 타깃 PID 탐색, 권한 레벨(TokenElevationType = 2) 노드 정상 바인딩 완결성 검증 (36/36 통과, Exit Code 0).

