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
   * 이에 따라 [새 창의 라디오버튼 체크 -> 기존 창의 라디오버튼 언체크 -> ViewModel 프로퍼티 변경 -> 새 창의 라디오버튼 언체크 -> 기존 창의 라디오버튼 체크...]가 밀리초 단위로 수만 회 상호 재귀 호출되어 호출 스택이 고갈됨.

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

## 2026-09-29: [Resolved] 설정 창(SettingsWindow) 진입 시 테마 RadioButton 읽기 전용 속성 바인딩 충돌 및 0xc000041d 크래시

### 1. 현상 (Symptom)
* 메인 관제 콘솔 좌측 네비게이션 레일에서 [환경 설정] 버튼 클릭 시 `Phalanx.Cockpit.exe`가 즉각 비정상 종료됨.
* 종료 코드: `3221226525 (0xc000041d)` (`STATUS_FATAL_USER_CALLBACK_EXCEPTION`).

### 2. 원인 (Root Cause)
1. **WPF RadioButton의 기본 TwoWay 바인딩 특성**:
   * WPF `RadioButton.IsChecked` 의존성 속성은 메타데이터 상 `BindsTwoWayByDefault = true`로 구성됨.
   * `SettingsWindow.xaml`의 테마 선택 라디오 버튼에 `{Binding IsThemeSystem}`, `{Binding IsThemeDark}`, `{Binding IsThemeLight}`를 바인딩했으나, `SettingsViewModel.cs`의 세 프로퍼티는 게터 전용 람다 프로퍼티(`=> SelectedThemeMode == "..."`)로 선언되어 있었음.
2. **Win32 네이티브 콜백 내 예외 전파**:
   * 윈도우 생성 및 렌더링 초기화 단계(`HwndSource.SetLayoutSize` ➔ `ContextLayoutManager.UpdateLayout`)에서 WPF 바인딩 엔진의 `PropertyPathWorker.CheckReadOnly`가 호출되며 `System.InvalidOperationException: TwoWay 또는 OneWayToSource 바인딩은 'Phalanx.Cockpit.ViewModels.SettingsViewModel' 형식의 읽기 전용 속성 'IsThemeSystem'에서 작동하지 않습니다.` 예외를 발생시킴.
   * 해당 예외가 Win32 메시지 디스패치 루프(`WM_CREATE` / `WM_SHOWWINDOW`) 내부에서 처리되지 않고 탈출하면서 Windows 커널에 의해 `STATUS_FATAL_USER_CALLBACK_EXCEPTION` (`0xc000041d`)으로 프로세스가 강제 사살됨.

### 3. 해결책 (Resolution)
1. **ViewModel 양방향 세터 및 상태 안전성 구축 (`SettingsViewModel.cs`)**:
   * `IsThemeSystem`, `IsThemeDark`, `IsThemeLight`에 명시적 `set` 블록을 구현.
   * 비활성화(`value == false`) 시그널이 주입될 때는 상태를 덮어쓰지 않고, 오직 활성화(`value == true`) 시그널일 때만 `SelectedThemeMode`를 원자적으로 변경하도록 방어 로직 적용.
2. **XAML 바인딩 모드 방어 강화 (`SettingsWindow.xaml`)**:
   * 라디오 버튼의 `IsChecked` 바인딩에 `Mode=OneWay`를 명시적으로 부여하여 WPF 바인딩 엔진의 소스 갱신 시도를 원천 차단하고, 변경은 `Command="{Binding SetThemeModeCommand}"`로만 통제하도록 2중 방어선 확립.
3. **STA 윈도우 인스턴스화 회귀 테스트 완비 (`SettingsViewModelTests.cs`)**:
   * `TestSettingsWindow_InstantiationAndThemeToggle` 단위 테스트 추가: STA 스레드에서 `SettingsWindow`를 실제 생성, 렌더링(`Show()`), 테마 카테고리 전환 및 다크/라이트/시스템 모드 동적 토글 후 정상 종료(`Close()`)까지 전 구간 무결성 검증.

---

## 2026-09-29: [Resolved] ProcessGraphView 내 WPF DataTrigger 기본값 부재 및 유령 리소스 키(ThreatCriticalBrush)로 인한 DependencyProperty.UnsetValue 크래시

### 1. 현상 (Symptom)
* 심층 포렌식 분석(`InvestigationView`) 화면에서 '전역 프로세스 트리에서 위치 확인 ➔'(`FocusProcessInGraphCommand`) 버튼 클릭 시 크래시 발생.
* 관제 콘솔(`ProcessGraphView`)에서 프로세스를 선택하고 `[원자적 동결 (Suspend)]` 버튼을 클릭하는 즉시 프로그램이 비정상 종료되며 동일 예외 발생:
  ```text
  System.InvalidOperationException: '{DependencyProperty.UnsetValue}'은(는) 'Foreground' 속성의 유효한 값이 아닙니다.
  HResult=0x80131509
  ```

### 2. 원인 (Root Cause)
1. **WPF 의존성 프로퍼티(DependencyProperty) Style 기본 Setter 누락**:
   * `ProcessGraphView.xaml`의 관리자 권한 수준 TextBlock 및 `ListViewItem` ItemContainerStyle 내부에 기본 `Foreground` Setter가 누락되어 있었음.
   * `TokenElevationType == 2` DataTrigger 또는 `IsSelected` Trigger가 비활성화/해제될 때, WPF는 Style의 기본값을 복원하려고 시도하나 기본 Setter가 없어 `DependencyProperty.UnsetValue`를 반환하였고, Brush 타입 유효성 검사에 실패함.
2. **미존재 유령 리소스 키 (Phantom Resource Key)**:
   * 상태 배지 `<TextBlock.Style>`에서 `IsSuspended == True` 트리거에 지정된 `{StaticResource ThreatCriticalBrush}`가 테마 사전(`EnterpriseTheme.xaml`)에 존재하지 않는 유령 키였음.
   * 수동 제어 후 낙관적 UI 갱신(`UpdateStatus`)이 추가되면서 버튼 클릭 즉시 DataTrigger가 활성화되어 미존재 키를 조회하였고, `DependencyProperty.UnsetValue`가 반환되어 런타임 크래시를 유발함.

### 3. 해결책 (Resolution)
1. **Style 기본 Setter 명시 및 템플릿 트리거 격리**:
   * TextBlock Style 및 `ListViewItem` Style에 기본 `<Setter Property="Foreground" Value="{DynamicResource TextPrimaryBrush}" />`를 명시하여 트리거 조건 해제 시 UnsetValue 전파 원천 차단.
   * `ListViewItem` 내부 템플릿 트리거가 부모의 Foreground를 직접 덮어쓰지 않도록 `TargetName="Bd"` 배경만 제어하도록 스코프 격리.
2. **유령 리소스 키 완전 제거 및 공식 테마 브러시 매핑**:
   * `[동결]` 트리거: `{StaticResource ThreatCriticalBrush}` ➔ `{DynamicResource SeveritySuspendedTextBrush}` (`#FBBF24` Amber Gold)
   * `[사살]` 트리거: `{StaticResource TextMutedBrush}` ➔ `{DynamicResource SeverityCriticalTextBrush}` (`#F87171` Crimson Red)
3. **단위 테스트 검증**:
   * `ProcessTreeProjectionTests.TestMainViewModel_FocusProcessInGraphCommand`를 통해 뷰 전환, 타깃 PID 탐색, 권한 레벨 바인딩 무결성을 검증 (Exit Code 0).

---

## 2026-09-29: [Resolved] 프로세스 트리 정상 종료 노드의 EDR 사살(빨간색 DotCriticalBrush) 오표출 시각화 결함 해결 및 4색 상태 정규화

### 1. 현상 (Symptom)
* Obsidian의 백그라운드 Git 동기화 루틴 등 정상적인 백그라운드 프로세스가 완료 후 종료되었을 때, 프로세스 트리의 상태 점이 전부 크리티컬 위협 색상인 빨간색(`DotCriticalBrush`)으로 표출되어 정상 종료 프로세스가 EDR에 의해 사살된 침해 사고로 심각하게 오인되는 시각화 결함 발생.

### 2. 원인 (Root Cause)
* `ProcessGraphView.xaml`의 `<Ellipse.Style>` 트리거에서 `IsAlive == False` 조건에 무조건 `DotCriticalBrush`를 할당하여, EDR 긴급 사살(`IsTerminated == True`)과 OS 정상 종료(`LifecycleStop`)의 시각적 상태가 구별되지 않고 동일하게 처리됨.

### 3. 해결책 (Resolution)
1. **4단계 상태 점 시각화 정규화**:
   * 기본값: `DotLiveBrush` (초록 `#10B981` - 정상 실행 중)
   * `IsAlive == False`: `DotBenignBrush` (회색 `#71717A` - 정상 종료 프로세스 명확 분리)
   * `IsSuspended == True`: `DotSuspendedBrush` (호박색 `#D97706` - 원자적 동결)
   * `IsTerminated == True`: `DotCriticalBrush` (빨강 `#E11D48` - EDR 긴급 사살 최우선 덮어쓰기)
   * `ImageName` TextBlock에 `IsAlive == False` 시 `TextMutedBrush` 딤 처리를 적용하여 시각적 가독성 개선.
2. **회귀 검증**:
   * `ProcessTreeProjectionTests`를 통해 프로세스 상태별 브러시 매핑 및 족보 가시성 무결성 확인 (Exit Code 0).

---

## 2026-09-29: [Resolved] FullChainSystemTests 동시성 타이밍 결함 및 Category=Live 벤치마크 미분리로 인한 테스트 지연

### 1. 현상 (Symptom)
* `FullChainSystemTests.TestPhalanxGrpcService_MultiClientConcurrentStreams_MaintainsConnectionState`에서 `Assert.True(lastReportedConnection)` (line 572) 간헐적 실패 (`Expected: True, Actual: False`, 615ms 시점).
* 카테고리 필터 없이 `dotnet test` 실행 시 콘솔에 아무런 진척 없이 3분 이상 멈춰 있는 현상 발생.

### 2. 원인 (Root Cause)
1. **gRPC 다중 스트림 테스트의 500ms 협소 타임아웃 및 자원 미회수**:
   * `for (int i = 0; i < 20; i++) await Task.Delay(25)` 구조로 최대 대기시간이 500ms에 불과하여, CPU 스레드풀 지연 시 이벤트 수신 전에 조기 단언문 실패 발생.
   * `finally` 블록의 부재로 인해 단언문 실패 시 백그라운드 `StreamTelemetry` 태스크와 채널 리더가 정상 회수되지 않고 잔존.
   * `MockAsyncStreamReader.Complete()`가 `ChannelWriter.Complete()`를 호출하여 중복 완료 시 `ChannelClosedException` 유발.
2. **무필터 테스트 실행 시 실제 클라우드 AI 벤치마크 트리거**:
   * 프로젝트 규약상 기본 단위 테스트는 `dotnet test tests/Phalanx.Agent.Tests/ --filter "Category=Unit"` (2.8초 소요).
   * 필터 생략 시 `Category=Live`에 속한 `TestLive_MultiScenario_AverageTurnAndLatencyBenchmark`가 실행됨.
   * 해당 테스트는 실제 Google Cloud Vertex AI / Gemini 3.7 Flash와 10대 복합 위협 시나리오에 대해 멀티턴 ReAct 통신을 수행하며, RPM 버퍼링(시나리오당 1.5초)을 포함하여 2~3분이 소요되는 대규모 엔드투엔드 AI 벤치마크임.
   * xUnit 기본 콘솔 출력 정책으로 인해 중간 진행 상황이 보이지 않아 무한 대기/프리징으로 오인됨.

### 3. 해결책 (Resolution)
1. **`FullChainSystemTests.cs` 동시성 및 진단 내구성 강화**:
   * `[Fact(Timeout = 10000)]` 10초 타임아웃 속성 부여.
   * 500ms 하드코딩 루프를 `WaitForConditionAsync` (최대 5초 적응형 폴링 및 실패 컨텍스트 출력)로 교체.
   * `try ... finally` 블록을 구성하여 `req1.Complete()`, `req2.Complete()`, `cts.Cancel()`, `await Task.WhenAll(task1, task2)`를 통해 모든 백그라운드 태스크의 완벽한 생명주기 회수 보장.
   * `MockAsyncStreamReader<T>.Complete()`를 멱등한 `ChannelWriter.TryComplete()`로 수정하여 중복 채널 닫힘 예외 방지.
   * 단계별(1~5단계) 상세 진단 로깅(`_output.WriteLine`) 탑재.
2. **`AutonomousHunterAgentTests.cs` Live 테스트 방어 및 가시성 개선**:
   * `agent.IsOnlineGemini`를 검증하여 로컬 인증 정보 부재 시 안전하게 건너뛰도록 방어 가드 장착.
   * `[Fact(Timeout = 300000)]` 5분 타임아웃 부여 및 각 시나리오별 실시간 진행 상황 및 예외 출력(`try-catch`).
   * 수사 결과 상세 내역(Trace, Thought, Verdict)을 Assertion 전에 선제 출력하도록 순서 재정렬.
3. **규약 표준화 및 테스트 가이드 문서화**:
   * `[`.agents/AGENTS.md`](../../../Phalanx/.agents/AGENTS.md)` 및 `[`Phalanx/README.md`](../../../Phalanx/README.md)`에 Fast QA 단위 테스트 커맨드(`--filter "Category=Unit"`, 2.8s)와 Live 클라우드 벤치마크 커맨드(`--filter "Category=Live" --logger "console;verbosity=normal"`)를 공식 분리 명시.
4. **검증**:
   * `dotnet build Phalanx.sln -c Release` ➔ Exit Code 0 (경고 0, 오류 0).
   * `dotnet test tests/Phalanx.Agent.Tests/ -c Release --no-build --filter "Category=Unit"` ➔ 총 46개 단위 테스트 전원 통과 (2.8초 소요, Exit Code 0).
   * `dotnet test tests/Phalanx.Agent.Tests/ -c Release --no-build` ➔ 총 51개 전체 테스트(Live 벤치마크 포함) 전원 통과 (Exit Code 0).

---

## 2026-09-30: [Resolved] 관제 콘솔 수동 프로세스 제어(동결/해제/사살) 시 하단 글로벌 텔레메트리 카운터 미갱신 및 ETW 사살 배지 덮어쓰기 결함

### 1. 현상 (Symptom)
* Phalanx Cockpit 프로세스 트리 뷰에서 관제사가 [원자적 동결], [동결 해제], [프로세스 사살] 버튼을 수동 클릭하여 조치하더라도 하단 글로벌 상태 표시줄의 `FROZEN`, `RESTORED`, `TERMINATED`, `MONITORED` 카운터가 전혀 갱신되지 않고 0으로 고정되는 현상.
* 관제사 또는 자율 AI 수사관에 의해 현장 사살(`IsTerminated == true`, `[현장 사살]`)된 프로세스가 OS 상에서 완전히 소멸될 때 커널 ETW로부터 `LifecycleStop` 이벤트가 도착하면 `targetNode.UpdateStatus(LifecycleStop)`에 의해 `[현장 사살]` 배지가 `[정상 종료]`로 다운그레이드 덮어쓰기되는 레이스 컨디션 발생.

### 2. 원인 (Root Cause)
1. `MainViewModel.cs`의 `SuspendSelectedProcessAsync`, `ResumeSelectedProcessAsync`, `TerminateSelectedProcessAsync` 커맨드 내부에서 `SelectedProcessNode.UpdateStatus`만 호출하고 `ActiveSuspendedCount`, `TotalRestoredCount`, `TotalTerminatedCount`, `MonitoredProcessCount` 속성을 증감하는 로직이 완전히 누락되어 있었음.
2. `ProcessTreeProjectionManager.HandleStop`에서 노드의 기존 사살 여부(`IsTerminated`)를 확인하지 않고 무조건 `UpdateStatus(LifecycleStop)`를 호출하여 사살 표식이 유실됨.

### 3. 해결책 (Resolution)
1. **`MainViewModel.cs` 수동 조치 커맨드 3종 카운터 실시간 동기화**:
   * 비동기 gRPC 통신 대기 중 사용자 노드 선택 변경에 의한 타깃 역전 방지를 위해 메서드 진입 시 `var node = SelectedProcessNode;` 로컬 캡처 및 `!node.IsAlive` 사망 가드 완비.
   * `SuspendSelectedProcessAsync`: `!wasSuspended` 시 `ActiveSuspendedCount++`.
   * `ResumeSelectedProcessAsync`: `wasSuspended` 시 `ActiveSuspendedCount--` (0 하한 가드) 및 `TotalRestoredCount++`.
   * `TerminateSelectedProcessAsync`: `wasSuspended` 시 `ActiveSuspendedCount--`, `TotalTerminatedCount++`, `MonitoredProcessCount--` (0 하한 가드), `HideTerminatedProcesses` 토글 시 `_treeManager.RebuildVisibleNodes()` 연동.
2. **`ProcessTreeProjectionManager.cs` 사살 배지 불변성 보장**:
   * `HandleStop` 내 UI Dispatcher 블록에서 `bool wasTerminated = targetNode.IsTerminated;` 검사를 수행하고 `!wasTerminated` 조건부로만 `UpdateStatus(LifecycleStop)`를 호출하여 `[현장 사살]` 배지 보존.
3. **회귀 검증**:
   * `ProcessTreeProjectionTests.TestMainViewModel_ManualActuation_SuspendResumeTerminateCommands`를 전면 보강하여 동결 ➔ 해제 ➔ 재동결 ➔ 동결 중 직접 사살(콤보) ➔ 사망 노드 조작 차단 ➔ ETW `LifecycleStop` 수신 시 사살 배지 보존까지 전 시나리오 단위 테스트 구축.
   * `dotnet build Phalanx.sln -c Release` ➔ Exit Code 0 (경고 0, 오류 0).
   * `dotnet test tests/Phalanx.Agent.Tests/ -c Release --no-build --filter "Category=Unit"` ➔ 46개 단위 테스트 전원 통과 (Exit Code 0).

---

## 2026-09-30: [Resolved] 단일 프로세스 반복 동결/해제 시 RESTORED 카운터 무한 증가 결함 및 엔터프라이즈 Gauge 모델 전면 통일

### 1. 현상 (Symptom)
* 단일 프로세스에 대해 관제사가 [원자적 동결]과 [동결 해제]를 반복 클릭할 때마다 `RESTORED` 카운터가 무한히 증가(`1 -> 2 -> 3... -> 10`)하는 현상 발생.
* 이는 CrowdStrike Falcon, SentinelOne, Defender XDR 등 엔터프라이즈 EDR의 상황 인식(Situational Awareness) 상태 표시줄 표준인 Gauge(현재 활성 상태 유지 수) 방식과 불일치하여 관제사에게 혼선을 유발함.

### 2. 원인 (Root Cause)
1. `TotalRestoredCount`가 단순 누적 액션 카운터(`++`)로만 동작하여, 프로세스가 다시 동결되거나 사살되거나 OS 상에서 자연 종료되더라도 복원 수치가 회수되지 않고 무한 누적됨.
2. `ProcessNodeModel`에 현재 복원 상태인지 여부를 나타내는 상태 플래그(`IsRestored`)가 부재하였음.
3. `ProcessTreeProjectionManager.HandleStop`에서 상태 리셋 전 사전 플래그(`wasRestored`, `wasSuspended`) 캡처 및 UI 스레드 마샬링 파이프라인이 부재하여 자연 종료 시 카운터 차감이 누락됨.

### 3. 해결책 (Resolution)
1. **`ProcessNodeModel.cs` 상태 플래그 확장**:
   * `[ObservableProperty] private bool _isRestored;` 추가 및 동결/사살/종료 시 리셋.
2. **`ProcessTreeProjectionManager.cs` 이벤트 시그니처 및 사전 캡처 확장**:
   * `OnProcessStopped`를 `Action<ProcessNodeModel, bool, bool>` (`node, wasSuspended, wasRestored`)로 확장하고, `HandleStop`에서 UI Dispatcher 진입 전 원자적으로 플래그를 캡처하여 전달.
3. **`MainViewModel.cs` Gauge 라이프사이클 및 안전 마샬링 완비**:
   * `ActiveRestoredCount`를 신설하고 `TotalRestoredCount` 양방향 호환 브리지를 제공하여 CS0200 에러 및 XAML 하위 호환성 100% 보장.
   * `HandleProcessStopped`에서 `Application.Current?.Dispatcher` 안전 마샬링 및 Headless 환경 분기를 구성하여, 프로세스 자연 종료 시 `ActiveRestoredCount--`, `ActiveSuspendedCount--`, `MonitoredProcessCount` 동기화 보장.
   * `SuspendSelectedProcessAsync`: 복원 상태 노드 재동결 시 `ActiveRestoredCount--` 회수 및 `ActiveSuspendedCount++`.
   * `ResumeSelectedProcessAsync`: 동결 노드 해제 시 `ActiveSuspendedCount--`, `ActiveRestoredCount++`, `node.IsRestored = true`.
   * `TerminateSelectedProcessAsync`: 복원/동결 상태 노드 사살 시 각 Gauge 회수, `TotalTerminatedCount++`, `MonitoredProcessCount--`.
   * AI Hunter 수사 라이프사이클(`OnInvestigationStarted`, `OnInvestigationCompleted`) 내 `FindNodeByPid` 역참조 및 `ActiveRestoredCount` 동기화.
   * `LoadIncidentsFromDatabase`: Gauge 모델 원칙에 따라 과거 DB 수치 덮어쓰기 배제.
4. **`MainWindow.xaml` 바인딩 정규화**:
   * 하단 RESTORED 텍스트 바인딩을 `ActiveRestoredCount`로 정규화.
5. **회귀 검증**:
   * `ProcessTreeProjectionTests.cs` 단위 테스트에 3회 추가 동결/해제 루프(`RESTORED: 1` 상한 유지, 무한 증가 차단), 복원 상태 자연 종료 회수, 동결 중 사살 회수 검증을 추가하여 46개 단위 테스트 전원 통과 (Exit Code 0).

---

## 2026-09-30: [Resolved] AI 자율 수사 진행 중 시각적 정체감 해소 및 실시간 ReAct 턴 스트리밍 생동감 UX 파이프라인 구축

### 1. 현상 (Symptom)
* C++ 커널 센서가 선제 동결한 회색지대 프로세스를 AI 헌터(`AutonomousHunterAgent`)가 수사하는 동안(3초~36초), 관제 UI 상의 사건 카드가 단일 정적 텍스트로 고정되어 에이전트의 작동 여부를 체감하기 어려움.
* 심층 포렌식 수사 기록(ReAct 추론 단계별 Thought, Action, Observation)이 수사가 모두 종료된 후에야 일괄 갱신되어, 관제사가 수사관의 실시간 추론 과정을 지켜볼 수 없는 정체된 UX 발생.

### 2. 원인 (Root Cause)
1. `AutonomousHunterAgent` 내에서 각 도구 호출(`IInvestigationTool.ExecuteAsync`) 및 판결 추론 시점에 외부로 진행 상태를 통지하는 실시간 이벤트 파이프라인이 부재하여, 오직 수사 종결 시점의 `OnInvestigationCompleted` 이벤트에만 의존함.
2. `CockpitUiBridge` 및 `MainViewModel`에 개별 ReAct 턴 단위의 UI 스레드 안전 마샬링 및 수사 경과시간 타이머가 구현되어 있지 않았음.
3. `IncidentItemViewModel` 및 XAML 뷰템플릿(`IncidentsView.xaml`, `InvestigationView.xaml`)에 수사 진행 중 상태(`IsInvestigating`), 실시간 진행 텍스트, 펄스 애니메이션, 아코디언 양방향 펼침 제어(`IsExpanded`)가 결여되어 있었음.

### 3. 해결책 (Resolution)
1. **`AutonomousHunterAgent.cs` 실시간 턴 이벤트 신설**:
   * `public event Action<string, ReActTraceRecord>? OnReActStepProgress;` 추가.
   * 온라인 Gemini ReAct 루프(최종 판결, 각 도구 완료, 최대 루프 소진 Fail-Secure 스텝, 방화벽 후속 조치) 및 오프라인 결정론적 추론 5개 단계 전역에서 턴 완료 시 즉시 이벤트 발생.
2. **`CockpitUiBridge.cs` UI 안전 마샬링**:
   * `ReActStepCompleted` 이벤트 및 `NotifyReActStepCompleted` 구현 (UI 스레드 안전 Dispatcher 마샬링 및 Headless null 가드 적용).
3. **`Program.cs` 이벤트 배선**:
   * `agent.OnReActStepProgress` ➔ `uiBridge.NotifyReActStepCompleted` 연결.
4. **뷰모델 상태 전이 및 100ms 경과 타이머 구현**:
   * `ReActStepViewModel`: `[ObservableProperty] private bool _isExpanded;` 추가.
   * `IncidentItemViewModel`: `_investigationProgressText`, `_isInvestigating`, `_activeElapsedSeconds` 프로퍼티 추가 및 `[NotifyPropertyChangedFor(nameof(FormattedLatency))]` 어트리뷰트 적용으로 타이머 틱 시 실시간 지연시간 UI 갱신 보장.
   * `MainViewModel`:
     * `OnInvestigationStarted`: `IsInvestigating = true`, 100ms 간격 `DispatcherTimer` 가동 (`Application.Current == null` 가드 포함).
     * `OnReActStepCompleted`: 이전 턴 자동 접힘, 신규 턴 `IsExpanded = true` 생성 및 증식, `InvestigationProgressText = "Turn N • [ActionTool] 완료"` 갱신.
     * `OnInvestigationCompleted`: `IsInvestigating = false`, 다중 수사 안전 가드 타이머 정지, `Traces.Clear()`를 지양하고 누락분만 보충 동기화하여 아코디언 펼침 상태 보존.
5. **WPF XAML 3계층 생동감 UI 고도화**:
   * `IncidentsView.xaml`: 사건 카드 좌측 점에 `IsInvestigating=True` 시 부드러운 Glow 호흡 펄스(Opacity 0.35 ↔ 1.0, 0.8초 주기, `AutoReverse="True"`) 적용. Line 2 TextBlock에서 태그 내 로컬 하드코딩 속성을 제거하고 `Style.Setters` 및 `DataTrigger`로 완전 위임하여 의존성 프로퍼티 우선순위 충돌 방지.
   * `InvestigationView.xaml`: ReAct 아코디언 `<Expander>`를 `IsExpanded="{Binding IsExpanded, Mode=TwoWay}"`로 양방향 바인딩하여 턴 증식 시 최신 턴 자동 펼침 지원.
6. **회귀 및 수명주기 검증**:
   * `ProcessTreeProjectionTests.cs`에 `TestInvestigation_RealtimeProgressAndTraceStreaming_Lifecycle` 단위 테스트 추가: 수사 개시 ➔ 100ms 지연시간 통지 ➔ Turn 1 스트리밍 및 자동 펼침 ➔ Turn 2 스트리밍(Turn 1 자동 접힘 및 Turn 2 자동 펼침) ➔ 수사 완료 후 아코디언 상태 보존 전 과정 검증.
   * `dotnet build Phalanx.sln -c Release -m:1` ➔ Exit Code 0 (경고 0, 오류 0).
   * `dotnet test tests/Phalanx.Agent.Tests/ -c Release --no-build --filter "Category=Unit"` ➔ 47개 단위 테스트 전원 통과 (Exit Code 0, 908ms).

---

## 2026-09-30: [Resolved] AI 수사 완료 시 심층수사실 빈 화면(SelectedIncident null) 유실 버그 및 최종 판결 턴 'NONE' 표기 개선

### 1. 현상 (Symptom)
* AI 자율 수사관의 실시간 ReAct 턴 스트리밍 중 마지막 판결 단계 헤더에 날것의 `PHASE 03 : NONE`이 노출되어 미구현 또는 오류처럼 보이는 현상 발생.
* 직후 수사가 완전히 종료되는 순간 심층수사실(`InvestigationView.xaml`)의 모든 3-Panel 데이터(서사, 런북, MITRE 전술, 아코디언 추적)가 한순간에 증발하여 화면 전체가 완전한 빈 화면(Blank Screen)으로 변하는 심각한 화면 유실 버그 발생.

### 2. 원인 (Root Cause)
1. **WPF `ListBox` 컬렉션 Reset에 의한 `SelectedItem` 강제 Coercion (`null` 주입)**:
   * `MainViewModel.OnInvestigationCompleted`에서 수사 결과를 반영한 뒤 사건 목록 필터를 갱신하기 위해 `ApplyFilter()`를 호출함.
   * `ApplyFilter()` 내부에서 `FilteredIncidents.Clear()`가 호출되는 순간, WPF `ListBox`(`IncidentsView.xaml`)가 `ItemsSource`의 `Reset`을 감지하여 자신의 `SelectedItem`을 `null`로 강제 초기화함.
   * `ListBox.SelectedItem="{Binding SelectedIncident}"`의 기본 양방향(Two-Way) 바인딩에 의해 `MainViewModel.SelectedIncident`가 즉각 `null`로 덮어씌워짐.
   * 그 직후 `if (SelectedIncident == existing)` 조건 검사가 이미 `null == existing`으로 평가되어 `false`로 실패하고, `SelectedIncident`가 영구히 `null`로 방치됨.
   * `InvestigationView.xaml`의 모든 UI 컨트롤이 `{Binding SelectedIncident.*}`를 바라보고 있어 전체 화면이 공백으로 증발함.
2. **ReAct 최종 판결 도구 부재 표기 가공 누락**:
   * Gemini ReAct 루프가 최종 판결(Final Verdict)에 도달하면 더 이상 도구를 호출하지 않으므로 `ActionTool = "None"`으로 기록됨.
   * `ReActStepViewModel.FormattedStep`이 이를 `$"PHASE {StepNumber:D2} : {ActionTool.ToUpperInvariant()}"`로 단순 변환하여 `PHASE 03 : NONE`이 아코디언 헤더에 그대로 노출됨.

### 3. 해결책 (Resolution)
1. **`MainViewModel.cs` 선택 상태 원자적 보존 및 확정 할당**:
   * `ApplyFilter()` 시작 시 `var previousSelected = SelectedIncident;`로 원자적 스냅샷을 캡처하고, 필터링 루프 완료 후 `if (previousSelected != null && FilteredIncidents.Contains(previousSelected)) { SelectedIncident = previousSelected; }`를 통해 ListBox 초기화로 인한 null 코어션을 원천 차단.
   * `OnInvestigationCompleted` 종료 시 `ApplyFilter()` 직후 `SelectedIncident = existing; OnPropertyChanged(nameof(SelectedIncident));`를 명시적으로 실행하여 수사가 완료된 사건의 상세 포렌식 데이터가 심층수사실에 100% 온전히 유지되도록 보장.
2. **`ReActStepViewModel.cs` 최종 판결 엔터프라이즈 용어 정규화**:
   * `FormattedStep`에서 `string.Equals(ActionTool, "None", StringComparison.OrdinalIgnoreCase)` 분기를 적용하여, 최종 턴일 경우 날것의 `NONE` 대신 `PHASE {02} : FINAL VERDICT`로 품격 있게 렌더링되도록 개선.
3. **회귀 및 수명주기 검증**:
   * `ProcessTreeProjectionTests.cs` 내 `TestInvestigation_RealtimeProgressAndTraceStreaming_Lifecycle`에 "None" 턴의 `FINAL VERDICT` 표기 검증 및 수사 완료 후 `SelectedIncident` 인스턴스 동일성 보존(`Assert.Same(incident, vm.SelectedIncident)`) 검증 추가.
   * `dotnet build Phalanx.sln -c Release -m:1` ➔ Exit Code 0 (경고 0, 오류 0).
   * `dotnet test tests/Phalanx.Agent.Tests/ -c Release --no-build --filter "Category=Unit"` ➔ 47개 단위 테스트 전원 통과 (Exit Code 0, 916ms).

---

## 2026-09-30: [Resolved] 어택랩(Attack Lab) 프로세스 트리 고부하 스트레스 시나리오 제거 및 고난도 의심/정상 시나리오 4종 확충 개편

### 1. 현상 및 과업 배경 (Background)
* 어택랩 구 7번 시나리오("Process Tree DAG Burst", 50개 프로세스 연속 생성/종료 주입)는 단순 인메모리 프로세스 트리 및 UI 렌더링 성능(60FPS) 검증용 스트레스 테스트였음.
* 실제 AI 자율 위협 헌터(Autonomous Hunter Agent)의 ReAct 수사 파이프라인(`_agent.InvestigateAsync`)을 거치지 않고 `if (sc.Id == 7)` 분기에서 단순 레이턴시 어설션만 수행하고 즉시 리턴하는 구조로 인해, AI 수사관의 실질적인 탐지/오탐 방지 역량을 시험하지 못함.
* 보안 운영 환경에서 엔드포인트 보안 시스템을 우회하거나 오탐을 빈번하게 유발하는 고난도 침해 및 합법적 워크플로우에 대한 방어 검증 역량이 요구됨.

### 2. 해결책 및 신규 시나리오 4종 상세 구현 (Resolution)
1. **구 7번 DAG 스트레스 시나리오 및 전용 분기 완전 제거**:
   * `AttackScenarioRegistry.cs`: 구 7번 DAG Burst 시나리오 제거.
   * `AttackLabScenarioRunner.cs`: `if (sc.Id == 7)` 하드코딩 분기 완전 삭제. 모든 표준 시나리오가 Reflex Kill(#2) 또는 NtSuspendProcess 선제 동결 ➔ ReAct 심층 수사관 파이프라인으로 일관되게 진입하도록 일원화.
2. **고난도 정상(Tricky Benign, ACTION_RESUME) 2종 신설 (오탐 방지 및 엔지니어링 연속성)**:
   * **[시나리오 #7] SCCM Maintenance Script**:
     * 부모 `taskhostw.exe`가 `powershell.exe -ExecutionPolicy Bypass -NoProfile -enc <Base64 WMI Hotfix Audit>` 실행.
     * 정규 시스템 호스트의 난독화 파워셸 스폰으로 인해 단순 룰(T1059.001)에서 24μs 선제 동결 유발.
     * `DecodePayloadTool`로 Base64 해독 시 사내 핫픽스 점검 및 인트라넷 로그(`\\corp-sccm.internal\patches\audit.log`) 기록 확인 ➔ `IsKnownInternalOrTrusted` 1ms 조기 복구(`ACTION_RESUME`).
   * **[시나리오 #8] Developer Toolchain Loopback IPC**:
     * 개발 도구(`code.exe`)가 `cmd.exe /c curl.exe -s http://127.0.0.1:8080/health` 실행.
     * 개발자 IDE의 셸 스폰 및 네트워크 유틸리티 실행으로 의심 동결 유발.
     * `DecodePayloadTool`에서 외부 C2가 아닌 로컬 루프백(`127.0.0.1`/`localhost`) IPC 통신임을 식별하고 `ACTION_RESUME` 조기 복구 집행.
3. **고난도 비정상(Tricky Malicious, ACTION_KILL) 2종 신설 (탐지 회피 및 은닉 공격 분쇄)**:
   * **[시나리오 #9] LOLBAS Rundll32 Proxy Execution (T1218.011)**:
     * 공격자가 파워셸 감시 및 AMSI를 우회하기 위해 `powershell.exe` 없이 정품 서명 바이너리 `rundll32.exe`의 `RunHTMLApplication`을 악용하여 인라인 자바스크립트로 원격 COM 스크립트릿(`http://185.220.101.5/beacon.sct`) 로드.
     * `DecodePayloadTool` ➔ `ThreatReputationTool`(APT29 Cobalt Strike 98점 확증) ➔ `MitreClassifierTool`(T1218.011 자동 분류) ➔ `SystemFirewallTool` 차단 ➔ `ACTION_KILL`.
   * **[시나리오 #10] Process Injection via Unbacked Memory (T1055)**:
     * 정상 윈도우 인쇄 스풀러 `spoolsv.exe (C:\Windows\System32\spoolsv.exe)`의 완전한 정상 명령줄로 위장하여 정적 커맨드라인 검사를 100% 무력화.
     * 인메모리 VAD 가상 스캔(`ProcessMemoryScanTool`)을 통해 `PAGE_EXECUTE_READWRITE` 코드 케이브, 리플렉티브 DLL MZ 헤더, LockBit C2(`194.165.16.11`) 적발 ➔ `MitreClassifierTool`(T1055 매핑) ➔ `ACTION_KILL`.
4. **Clean-Room VAD 가상 메모리 스캔 엔진 및 누수 방지 생명주기 구현**:
   * `ProcessMemoryScanTool.cs`: `SimulatedMemoryEntry` 레코드 및 `ConcurrentDictionary<uint, ...> SimulatedMemoryDb` 구현.
   * `AttackLabScenarioRunner.ExecuteScenarioAsync`의 `finally` 블록에서 `ProcessMemoryScanTool.ClearSimulatedMemory(targetPid)`를 호출하여 정적 캐시 메모리 누수를 원천 차단.
5. **FSM 다차원 위험도 및 로컬 루프백 예외 로직 정밀화 (`AutonomousHunterAgent.cs`)**:
   * `HasInlineC2Pattern`: `127.0.0.1`/`localhost` 단독 로컬 쿼리는 외부 C2 지표에서 제외.
   * `InvestigateOfflineDeterministicAsync`:
     * `memoryObservation`을 `combinedBehavior`에 병합하여 결백한 명령줄의 인메모리 공격도 MITRE T1055로 정확히 분류.
     * 위험도 평가 시 `hasUnbackedMemory`(+50점)와 `threatScore >= 0.85`(+40점)를 분리 가산하여 인메모리 주입 단독으로 90점 도달 ➔ 즉각 사살 확정.
     * 시스템 바이너리 위장 드로퍼(T1036.005) 검출 시 +50점 가산 적용.
6. **커스텀 공작소 상수 격리 및 1~10번 전 시나리오 일괄 회귀 파이프라인 (`MainViewModel.cs`, `AttackLabWindow.xaml`)**:
   * `public const int CustomScenarioId = 99;` 정의.
   * `AttackLabWindow.xaml` 라인 723: `<DataTrigger Binding="{Binding Id}" Value="99">` 설정으로 8번 정규 시나리오 라벨 오염 해소.
   * `RunAllScenariosBatchAsync`: `Scenarios.Where(s => s.Id != CustomScenarioId)`로 1~10번 10개 전 시나리오 일괄 회귀 테스트 및 100% PASS 검증 자동화.
7. **단위 테스트 및 QA 검증 결과**:
   * `ProcessTreeProjectionTests.cs`:
     * `TestAttackLab_NewScenarios_EvasionAndDetection_Verdicts`: 신규 4종 시나리오 개별 판결 정합성 실측 통과.
     * `TestAttackLab_AllTenScenarios_FullChainBatch_Passes100Percent`: 10개 전 시나리오 일괄 회귀 100% 통과 실측.
   * `dotnet build Phalanx.sln -c Release -m:1` ➔ Exit Code 0 (경고 0, 오류 0).
   * `dotnet test tests/Phalanx.Agent.Tests/ -c Release --no-build --filter "Category=Unit"` ➔ 49개 단위 테스트 전원 통과 (Exit Code 0, 1.0s).

---

## 2026-09-30: [Resolved] WinVerifyTrust 64비트 P/Invoke 비관리 메모리 누수 방지 및 10초 세이프티 워치독 SLA 보호를 위한 FileInspectionTool 구현

### 1. 현상 (Symptom)
* 복합 회피 공격(T1036.005 Masquerading, Temp 디렉터리 내 가짜 svchost.exe 등) 인입 시, 디스크 파일 정밀 검증 도구의 부재로 인해 AI 수사관이 `ProcessMemoryScanTool`로 우회하면서 Turn 3에서 21.7초간 지연되고 최대 5턴(36.5초)을 전소하는 수사 병목 발생.
* `WinVerifyTrust` 호출 시 네트워크 CRL/CTL 조회가 발생할 경우 최대 15~30초간 동기 블로킹되어 C++ 커널 센서의 기본 10초 세이프티 워치독 SLA 데드라인을 침해하고 프로세스가 조기 복구될 위험 존재.
* 64비트 환경에서 `WINTRUST_DATA` 및 `WINTRUST_FILE_INFO` 구조체 포인터 비관리 메모리 할당(`Marshal.AllocHGlobal`) 후 해제 누락 시 반복 호출에 따른 메모리 누수 발생 위험.

### 2. 원인 (Root Cause)
1. EDR 수사 도구 체계에 디지털 서명(Authenticode), 시스템 바이너리 비인가 경로 배치 감별, Shannon 엔트로피 분석 도구가 결여되어 있었음.
2. `WinVerifyTrust`의 기본 플래그는 인터넷 연결을 통한 인증서 폐기 목록(CRL/CTL) 조회를 시도하여 고립망 또는 지연 네트워크에서 스레드를 장시간 블로킹함.
3. 무서명/위조 파일에 대해 `X509Certificate.CreateFromSignedFile` 호출 시 `CryptographicException`이 발생하며, 0바이트 빈 파일 엔트로피 연산 시 분모 0 나눗셈으로 `NaN`이 유발됨.

### 3. 해결책 (Resolution)
1. **`FileInspectionTool.cs` 구현 및 WinVerifyTrust P/Invoke 무누수 라이프사이클 확립**:
   * `WINTRUST_DATA` 및 `WINTRUST_FILE_INFO` 64비트 구조체 마샬링.
   * `Marshal.AllocHGlobal` 할당 및 이중 중첩 `try...finally` 블록 내 `Marshal.FreeHGlobal` 호출로 비관리 메모리 100% 안전 회수.
   * `WTD_STATEACTION_IGNORE` 지정으로 상태 데이터 캐시 누수 차단.
2. **세이프티 워치독 10초 SLA 보호**:
   * `dwProvFlags`에 `WTD_CACHE_ONLY_URL_RETRIEVAL = 0x00001000` 및 `fdwRevocationChecks = WTD_REVOCATION_CHECK_NONE`을 명시하여 네트워크 CRL 조회를 원천 차단, 서명 검증을 수십 마이크로초 단위로 즉각 완결.
3. **엣지 케이스 및 예외 방어**:
   * `X509Certificate2` 서명자 주체(`SignerSubject`) 추출 시 `CryptographicException` 방어 핸들러 장착.
   * 0바이트 파일 시 `Entropy = 0.0` 조기 반환(Division by Zero NaN 원천 차단).
   * DOS 헤더 64바이트 미만 및 `e_lfanew` 경계 초과 검증으로 손상된 바이너리의 `IndexOutOfRangeException` 크래시 방어.
   * 실행 중인 페이로드 잠금 충돌 방지를 위해 `FileShare.ReadWrite` 모드로 `FileStream` 개방.
4. **Clean-Room 모의 DB 및 어택랩 시나리오 #5 연동**:
   * `SimulatedFileEntry` 및 `RegisterSimulatedFile`/`ClearSimulatedFiles`를 제공하여 가상 환경에서도 디스크 IO 없이 무서명 svchost 위장 텔레메트리 주입 및 사후 안전 회수 완비.
5. **자율 AI 헌터 FSM 및 수사 파이프라인 연계**:
   * `AutonomousHunterAgent.cs`: 시스템 프롬프트에 6번째 도구 등록 및 파일 우선 수사 지침 명시.
   * 오프라인 결정론적 엔진에 Step 1.5로 `FileInspectionTool`을 연계하여 `IsPathMasqueraded == true` 또는 `AnomalyScore >= 80` 시 `riskScore += 50` 가산 후 파이프라인(C2 IP 방화벽 차단 및 MITRE 매핑) 완주 보장.
6. **회귀 및 QA 검증 결과**:
   * `FileInspectionToolTests.cs`: 7대 검증 시나리오(정규 서명, 무서명 위장 100점, .dat 확장자 위장, 고엔트로피, 0바이트 빈 파일, 결측/무효 인자, Clean-Room 모의 주입) 전원 통과.
   * `dotnet build Phalanx.sln`: 경고 0, 오류 0 (Exit Code 0).
   * `dotnet test tests/Phalanx.Agent.Tests/ --filter "Category=Unit"`: 총 56개 단위 테스트 전원 통과 (Exit Code 0, 1.0s).
