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
  * **타임아웃 분할 구조**: 기본 10,000ms(10초, C# 코어 하트비트 확인용) + AI 수사 개시 시 단 1회 50,000ms(50초) 연장 티켓(`ACTION_EXTEND_TIMEOUT`) 발송으로 총 60초 수사 예산 확보.
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

## 2026-09-10: [Resolved] 원자적 프로세스 동결 엔진(NtSuspendProcess) 24μs 집행 및 폴백 체계

### 1. 현상 (Symptom)
* 기존 Win32 `CreateToolhelp32Snapshot` + `SuspendThread` 스레드 순회 방식은 스냅샷 생성 및 스레드 오픈 순회 과정에서 수십 ms의 지연이 발생(약 35ms 계측).
* 순회 도중 타깃 악성 프로세스가 신규 워커 스레드를 즉각 분기(`CreateThread`)하여 페이로드를 실행하고 탈출할 수 있는 미세한 동시성 레이스 컨디션(Race Window) 취약점 존재.

### 2. 원인 (Root Cause)
* Win32 공개 API군에는 단일 호출로 프로세스 내 모든 스레드를 일괄 정지시키는 표준 인터페이스가 부재하여, 유저모드 스레드 열거 순회 방식에 의존함.

### 3. 해결책 (Resolution)
1. `ProcessActuator`에 `ntdll.dll`의 미공개 커널 네이티브 API `NtSuspendProcess` 및 `NtResumeProcess`를 동적으로 바인딩하여 1순위 원자적(Atomic) 동결 파이프라인 구축.
2. 동결 소요 시간이 기존 35,281μs(~35ms)에서 23μs(마이크로초, 1000배 이상 단축)로 단축되어 스레드 탈출 레이스 윈도우 원천 차단.
3. 권한 부족, 특정 OS 비호환 환경 또는 결함 발생 시 즉시 기존 `Toolhelp32` 방식으로 자동 후퇴(Graceful Fallback)하는 2중 방어선 구현.

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

## 2026-09-15: [Resolved] SafetyWatchdog 타임아웃 연장 마감시간 누적 가산 수식 불일치 버그

### 1. 현상 (Symptom)
* C# AI 에이전트가 수사 개시 즉시 1회성 타임아웃 연장 패킷(`ACTION_EXTEND_TIMEOUT`)을 발송했을 때, C++ 센서 내부에서 기존 마감 기한(30초)에 30초가 가산되어 60초가 되는 것이 아니라, 호출 시점(`steady_clock::now()`)으로부터 30초로 리셋되어 총 동결 시간이 약 30.1초에 머무는 SLA 레이스 위험 발생.

### 2. 원인 (Root Cause)
* `SafetyWatchdog.cpp`의 `ExtendTimeout` 메서드 내부에서 `it->second.deadline = std::chrono::steady_clock::now() + extend_by;`로 작성되어, 기존 `deadline`에 가산하지 않고 현재 시각을 기준으로 덮어쓰고 있었음.

### 3. 해결책 (Resolution)
* `it->second.deadline = (std::max)(it->second.deadline, std::chrono::steady_clock::now()) + extend_by;`로 수정하여, 수사 개시 직후 패킷이 도착하더라도 기존 마감시간에 연장 시간이 정확히 누적 가산되도록 교정.

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

