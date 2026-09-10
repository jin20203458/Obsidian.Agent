---
description: >-
  Phalanx C++ 센서 및 C# 코어 트러블슈팅 런북. Phalanx 프로젝트 버그, ETW 수집 오류 및 gRPC 장애 발생 시 참조.
related:
  - ../README.md
  - ../Phalanx/README.md
---
# Phalanx Troubleshooting Runbook

본 문서는 `Phalanx` EDR 솔루션(C++ 센서, gRPC 스트리밍, C# 코어 및 AI 에이전트) 개발 및 실전 모의 침투 테스트 중 발생하는 시스템 예외 현상과 해결 방안을 상세히 기록하는 중앙 런북입니다.

---

## 트러블슈팅 기록 템플릿 (작성 표준)

```markdown
## YYYY-MM-DD: [발생 이슈 요약]

### [현상 (Symptom)]
* 오류 메시지, 로그 내용, 비정상 동작 양상

### [원인 (Root Cause)]
* 코드, 시스템 콜, 동시성 또는 OS API 동작 메커니즘 차원의 근본 원인 분석

### [해결책 (Resolution)]
* 적용된 코드 패치, 구조 변경 및 해결 증빙 (Exit Code 0 / 빌드 확인)
```

---

## 사전 주의사항 및 알려진 기술적 고려점 (Known Constraints)

### 1. ETW 커널 세션 생성 권한 (Administrator Elevation)
* **현상**: 관리자 권한이 없는 일반 사용자 권한으로 센서 실행 시 `krabs-etw` 세션 생성 단계에서 `ACCESS_DENIED (0x5)` 예외 발생.
* **대응책**: `Phalanx.Sensor.exe`의 매니페스트 파일(`app.manifest`)에 `requireAdministrator` 실행 수준을 필수 명시할 것.

### 2. SuspendThread 데드락 예외 및 안전 복구 (Safety Watchdog)
* **현상**: 타깃 프로세스가 크리티컬 섹션이나 ntdll 로더 락(`LdrpLoaderLock`)을 쥐고 있는 상태에서 비동기 `SuspendThread` 호출 시 시스템 전역 리소스 경합 또는 데드락 발생 가능성.
* **대응책**:
  * `SuspendThread`는 자체 타임아웃 파라미터가 없으므로, 센서 내부에 **비동기 안전 타이머(Safety Watchdog, 기본 10,000ms)**를 운영하여 C# 대뇌가 크래시되거나 네트워크가 두절되어 응답이 없는 비정상 상태(Orphan Freeze) 감지 시 자동으로 `ResumeThread`를 호출하여 시스템 프리징을 해제하는 안전 폴백 메커니즘을 구비할 것.
  * **AI 수사 1회성 타임아웃 연장 티켓 (One-shot Extension Ticket: `ACTION_EXTEND_TIMEOUT`)**: 50ms 결정론적 룰 엔진으로 즉각 판정되지 않고 AI 심층 조사(2~5초 소요)로 넘어갈 경우, C# 코어는 조사 개시 시점에 단 1회 타임아웃 연장 티켓(`ACTION_EXTEND_TIMEOUT`)을 발송하여 워치독 마감 시한을 10,000ms 연장할 수 있다.
  * **절대 상한선 (Hard Ceiling / Fail-Safe)**: C++ 워치독은 시스템 데드락(로더 락 등)을 원천 차단하기 위해 **타임아웃 연장을 최대 1회로 엄격히 제한**한다. 1회를 초과하는 추가 연장 요청은 즉시 거부되며, 최초 동결 시점으로부터 최대 20초(기본 10초 + 연장 10초)를 초과하면 워치독이 자동으로 `ResumeThread`를 강제 집행하여 OS 안정성을 보장한다.
  * 타깃 프로세스가 완전히 안전하거나 정상으로 판정된 경우 즉시 `MitigationCommand(ACTION_RESUME)`를 하달하여 스레드를 정상 복구할 것.

---

## 2026-09-08: [Resolved] Windows Winsock/NOMINMAX 충돌 및 MSVC UAC 매니페스트 링크 에러

### [현상 (Symptom)]
* `ws2ipdef.h` / `ws2tcpip.h` 컴파일 시 `error C2011: 'ip_mreq': 'struct' type redefinition`, `error C2065: 'PADDRINFOA'`, `error C3861: 'WSAIoctl'` 등 100여 건의 Winsock 심볼 충돌 발생.
* `grpcpp/impl/generic_serialize.h` 및 `grpc/event_engine/memory_request.h`에서 Windows 매크로 `min`/`max` 간섭으로 구문 에러 발생.
* `Phalanx.Sensor.exe` 링크 시 `manifest authoring error c1010001: Values of attribute "level" not equal in different manifest snippets (LNK1327)` 발생.

### [원인 (Root Cause)]
1. `windows.h`가 `winsock2.h`보다 먼저 인클루드되어 구형 `winsock.h` (Winsock 1)와 `winsock2.h` (Winsock 2)가 중복 로드됨.
2. `NOMINMAX` 및 `UNICODE` / `_UNICODE` 매크로 부재로 `std::min`/`std::max` 파괴 및 `krabs-etw`의 `KERNEL_LOGGER_NAME` (`TEXT(...)`) 와이드 문자열 불일치 발생.
3. CMake의 `target_sources`에 `app.manifest`를 직접 전달하면서 MSVC 기본 생성 매니페스트(`asInvoker`)와 병합 충돌 발생.

### [해결책 (Resolution)]
1. 루트 `CMakeLists.txt`에 전역 컴파일 정의 `add_compile_definitions(UNICODE _UNICODE NOMINMAX WIN32_LEAN_AND_MEAN _WIN32_WINNT=0x0A00)` 적용.
2. 모든 C++ 헤더에서 `<windows.h>` 호출 전 `<winsock2.h>`와 `<ws2tcpip.h>`를 선행 인클루드하도록 구조화.
3. CMake 링크 플래그에 MSVC 네이티브 UAC 임베딩 지시어 `/MANIFEST:EMBED /MANIFESTUAC:"level='requireAdministrator' uiAccess='false'"` 적용하여 `mt.exe` 충돌 없이 PE 바이너리에 권한 임베딩 완료 (Ninja 빌드 및 `IpcE2ETest.exe` Exit Code 0 통과).

---

## 2026-09-10: [Resolved] Phase 1.5 원자적 프로세스 동결 엔진(NtSuspendProcess) 및 2중 안전 폴백 체계 구축

### [현상 (Symptom)]
* 기존 Win32 `CreateToolhelp32Snapshot` + `SuspendThread` 스레드 순회 방식은 스냅샷 생성 및 스레드 오픈 순회 과정에서 수십 ms의 지연이 발생(약 35ms 계측).
* 순회 도중 타깃 악성 프로세스가 신규 워커 스레드를 즉각 분기(`CreateThread`)하여 페이로드를 실행하고 탈출할 수 있는 미세한 동시성 레이스 컨디션(Race Window) 취약점 존재.

### [원인 (Root Cause)]
* Win32 공개 API군에는 단일 호출로 프로세스 내 모든 스레드를 일괄 정지시키는 표준 인터페이스가 부재하여, 유저모드 스레드 열거 순회 방식에 의존함.

### [해결책 (Resolution)]
1. `ProcessActuator`에 `ntdll.dll`의 미공개 커널 네이티브 API `NtSuspendProcess` 및 `NtResumeProcess`를 동적으로 바인딩하여 1순위 원자적(Atomic) 동결 파이프라인 구축.
2. 단위 테스트 계측 결과, 동결 소요 시간이 기존 35,281μs(~35ms)에서 **23μs(마이크로초, 1000배 이상 단축)**로 극적으로 단축되었으며 스레드 탈출 레이스 윈도우 원천 차단 확인.
3. 권한 부족, 특정 OS 비호환 환경 또는 결함 발생 시 즉시 기존 `Toolhelp32` 방식으로 우아하게 자동 후퇴(Graceful Fallback)하는 2중 방어선 구현.
4. `SensorTests`에 결함 주입(`force_fallback = true`) 테스트 케이스를 포함하여 2순위 폴백 경로에서도 정상 동결/복구됨을 완전 검증 (Exit Code 0).

---

## 2026-09-10: [Resolved] EtwKernelCollector::Start() 동시성 레이스 컨디션 해결 및 원자적 CAS 적용

### [현상 (Symptom)]
* `EtwKernelCollector::Start()`를 복수의 스레드가 동시에 호출할 경우, 이미 가동 중인 스레드가 덮어씌워지며 C++ 런타임에 의해 `std::terminate()` 크래시가 유발될 수 있는 잠재적 취약점 존재.
* 스레드 기동 중 시스템 자원 부족 예외(`std::system_error` 등) 발생 시 상태 플래그 롤백 로직이 부재하여 `running`이 `true`로 고착되는 상태 불일치 발생.

### [원인 (Root Cause)]
* 기존 코드가 `running.load()`를 확인하고 `running.store(true)`를 호출하는 전형적인 **Check-Then-Act (TOCTOU) 비원자적 상태 전이** 구조로 작성되어 있었음.

### [해결책 (Resolution)]
1. `impl_->running.compare_exchange_strong(expected, true, std::memory_order_acq_rel)`을 적용하여 복수의 스레드가 동시 진입하더라도 오직 하나의 스레드만 `false -> true` 전이에 성공하도록 원자적 상태 전이 보장.
2. 스레드 생성부를 `try-catch`로 감싸 `std::thread` 생성 실패 시 `impl_->running.store(false, std::memory_order_release)`로 원자적 롤백 수행 및 `false` 반환하도록 예외 안전성 확보.
3. `build.ps1` 재빌드, `SensorTests.exe` 및 `IpcE2ETest.exe`를 실행하여 정상 동작 및 Exit Code 0 통과 확인.
