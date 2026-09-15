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

### 2. 프로세스 원자적 동결 데드락 예외 및 안전 복구 (Safety Watchdog)
* **현상**: 타깃 프로세스가 크리티컬 섹션이나 ntdll 로더 락(`LdrpLoaderLock`)을 쥐고 있는 상태에서 비동기 동결 호출 시 시스템 전역 리소스 경합 또는 데드락 발생 가능성.
* **대응책**:
  * 동결 API(`ntdll!NtSuspendProcess` 및 폴백 `SuspendThread`)는 자체 타임아웃 파라미터가 없으므로, 센서 내부에 **비동기 안전 타이머(Safety Watchdog, 기본 10,000ms)**를 운영하여 C# 대뇌가 크래시되거나 네트워크가 두절되어 응답이 없는 비정상 상태(Orphan Freeze) 감지 시 자동으로 `NtResumeProcess`(폴백 시 `ResumeThread`)를 호출하여 시스템 프리징을 해제하는 안전 폴백 메커니즘을 구비할 것.
  * **AI 수사 1회성 타임아웃 연장 티켓 (One-shot Extension Ticket: `ACTION_EXTEND_TIMEOUT`)**: 100μs(실측 0.354μs) 초고속 로컬 룰 엔진으로 즉각 판정되지 않고 AI 심층 조사(2~5초 소요)로 넘어갈 경우, C# 코어는 조사 개시 시점에 단 1회 타임아웃 연장 티켓(`ACTION_EXTEND_TIMEOUT`)을 발송하여 워치독 마감 시한을 10,000ms 연장할 수 있다.
  * **절대 상한선 (Hard Ceiling / Fail-Safe)**: C++ 워치독은 시스템 데드락(로더 락 등)을 원천 차단하기 위해 **타임아웃 연장을 최대 1회로 엄격히 제한**한다. 1회를 초과하는 추가 연장 요청은 즉시 거부되며, 최초 동결 시점으로부터 최대 20초(기본 10초 + 연장 10초)를 초과하면 워치독이 자동으로 `NtResumeProcess`(폴백 시 `ResumeThread`)를 강제 집행하여 OS 안정성을 보장한다.
  * 타깃 프로세스가 완전히 안전하거나 정상으로 판정된 경우 즉시 `MitigationCommand(ACTION_RESUME)`를 하달하여 프로세스를 정상 복구할 것.

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

---

## 2026-09-10: [Resolved] MitigationCommand 프로토콜 비대칭 해소 및 ACTION_SUSPEND 동결 핸들러 연동

### [현상 (Symptom)]
* C# 코어에서 모호한 위협을 감지했을 때 AI 심층 수사를 위해 프로세스를 선제 동결(`NtSuspendProcess`)하려 해도, gRPC 프로토콜 `MitigationCommand`에 `ACTION_SUSPEND` 명령이 누락되어 있어 원격 동결 명령을 하달할 수 없는 비대칭성 존재.
* C++ 센서 수신 루프(`GrpcStreamClient`)에 `ACTION_SUSPEND` 분기 핸들러가 부재하여 프로세스 동결 액추에이터를 원격 트리거할 수 없었음.

### [원인 (Root Cause)]
* 기획 초기 "C++ 센서가 모든 프로세스를 선제 동결하고 C#은 해제/사살만 판단한다"는 단방향 가정으로 인해 `ACTION_RESUME`과 `ACTION_KILL`만 정의되었음.

### [해결책 (Resolution)]
1. `proto/phalanx.proto`의 `MitigationCommand::ActionType`에 `ACTION_SUSPEND = 4` 추가.
2. `GrpcStreamClient.cpp`의 수신 루프에 `case phalanx::MitigationCommand::ACTION_SUSPEND:`를 추가하고 `impl_->actuator->SuspendProcess(cmd.target_pid())` 연동.
3. `build.ps1`을 통해 Protobuf 코드 생성 및 빌드 성공(Exit Code 0), `SensorTests.exe` 및 `IpcE2ETest.exe` 통과 확인 (커밋 `89768b4`).

---

## 2026-09-10: [Resolved] Phase 2 C++ 인메모리 프로세스 트리(DAG) 및 100μs 초고속 로컬 규칙 엔진 구축

### [현상 (Symptom)]
* 센서 기동 전 이미 실행 중이던 Office/브라우저 프로세스에 대한 정보가 부재할 경우, 후속 LOLBAS 스폰 시 부모 PID를 찾지 못해 동결 규칙이 미탐(False Negative)될 수 있는 콜드 스타트 취약점 존재.
* 장시간 운영 시 종료된 프로세스 노드가 메모리에 무한 누적되거나 Windows PID 재사용(PID Reuse) 시 이전 부모-자식 관계가 왜곡될 위험.
* 표준 정규식(`std::regex`) 사용 시 수십~수백 μs가 소모되어 EDR 반사신경 요구 예산(100μs)을 초과할 수 있는 지연 병목 위험.

### [원인 (Root Cause)]
* ETW 커널 프로세스 이벤트는 센서 세션이 기동된 이후의 이벤트만 인입되므로 기존 OS 활성 프로세스에 대한 인메모리 스냅샷 부재.
* Windows 커널의 PID 고속 재할당 메커니즘 및 힙 파편화를 유발하는 동적 할당/정규식 평가 구조.

### [해결책 (Resolution)]
1. **스냅샷 웜업(Snapshot Warm-up)**: `ProcessTree::InitializeFromSnapshot()`을 기동 시 1회 호출(`CreateToolhelp32Snapshot`)하여 수 ms 만에 OS 상의 수백 개 활성 프로세스를 트리에 사전 탑재.
2. **PID 재사용 & Tombstone 메모리 바운딩**: `OnProcessStart` 시 동일 PID 노드 즉시 덮어쓰기 및 부모 링크 갱신, 종료 노드는 최대 10,000개로 상한선(FIFO Eviction)을 엄격히 제한하여 메모리 30MB 이하 보장.
3. **비할당 고속 문자열 정규화**: `std::string_view` 기반 파일명 추출 및 ASCII 대소문자 무시 비교(< 20ns)를 적용하여 정규식 오버헤드 원천 배제.
4. **0.1ms 현장 사살 & 24μs 선제 동결 파이프라인 연동**:
   - 볼륨 섀도 복사본 파괴 명령(`vssadmin.exe delete shadows`, `bcdedit`, `wbadmin`): 0.1ms 즉각 사살(`TerminateProcess`) 및 `is_terminated = true` 설정.
   - Office/Browser ➔ LOLBAS 스폰: 24μs 원자적 동결(`NtSuspendProcess`), 10초 세이프티 워치독 등록 및 `is_suspended = true` 설정.
5. **실측 벤치마크 결과 (`EngineTests.exe`)**:
   - 족보 역추적(`GetAncestry`) 10,000회 실측: **평균 0.436μs** (요구 기준 < 10μs 대비 22배 고속).
   - 로컬 규칙 평가 50,000회 실측: **평균 0.354μs, P99 0.7μs, 처리량 2,578,183 evals/sec** (요구 기준 < 100μs 대비 280배 여유).
   - `build.ps1`, `EngineTests.exe`, `SensorTests.exe`, `IpcE2ETest.exe` 전원 통과 (Exit Code 0).

---

## 2026-09-10: [Resolved] Phase 2.5 방어 파이프라인 E2E 실측 벤치마크 및 카나리 누수 제로(Zero Leak) 증빙

### [현상 (Symptom)]
* 이론적인 룰 엔진 마이크로초 지연시간(< 100μs)이 실제 OS 환경에서 PowerShell 스크립트 실행 또는 컴파일된 C/C++ 네이티브 랜섬웨어 공격을 마주했을 때, 디스크 쓰기(Canary File Write) 이전에 프로세스를 선제 차단할 수 있는지에 대한 실증 데이터 부재.
* 공격 윈도우(공격자가 디스크에 최초 바이트를 기록하기까지의 시간) 대비 EDR 파이프라인의 실질 안전 마진(Safety Margin) 불명확.

### [원인 (Root Cause)]
* 관리형 런타임(PowerShell/.NET CLR)의 웜업 지연과 초경량 네이티브 바이너리의 실행 진입점 속도는 수백 배 차이가 나므로, 단일 룰 엔진으로 양극단의 위협 윈도우를 모두 방어할 수 있음을 입증하는 E2E 통합 테스트 하네스 부재.

### [해결책 (Resolution)]
1. **초경량 모의 공격 바이너리 구축 (`MockNativeRansomware.exe`)**:
   - Release 최적화 빌드로 0.8ms 이내에 카나리 파일 생성을 시도하는 고속 공격 윈도우 시뮬레이터 구현.
2. **E2E 방어 벤치마크 하네스 구축 (`DefenseProfilingTest.exe`)**:
   - **실험 0 (무방비 대조군)**: PowerShell 카나리 생성 시간 **646.41 ms**, MockRansomware 카나리 생성 시간 **88.62 ms** 실측.
   - **실험 1 (스크립트 선제 동결)**: Office(`winword.exe`) 하위의 `powershell.exe` 스폰 감지 즉시 **53.2 μs** 만에 `NtSuspendProcess` 원자적 동결 집행 ➔ **카나리 파일 미생성 (Zero Payload Execution, +646.36 ms 안전 마진)**.
   - **실험 2 (네이티브 현장 사살)**: `MockNativeRansomware.exe vssadmin delete shadows` 감지 즉시 **95.9 μs** 만에 `TerminateProcess` 즉각 사살 집행 ➔ **카나리 파일 미생성 (Zero Leak Defense, +88.53 ms 골든타임 사살)**.
3. **종합 결과**:
   - 총 2회 실전 모의 공격 시도 중 2회 완벽 선제 차단 (방어율 100.0%).
   - 디스크 누수 용량: **0 Bytes (Zero Leak 공인)**.
   - `DefenseProfilingTest.exe`, `EngineTests.exe`, `SensorTests.exe`, `IpcE2ETest.exe` 전원 통과 (Exit Code 0).
   - 통합 벤치마크 레지스트리 `04_performance_benchmarks.md` Section 6 업데이트 완료.

---

## 2026-09-14: [Resolved] ProcessTree PID 재사용 시 유령 부모(Ghost Parent) 족보 왜곡 방어

### [현상 (Symptom)]
* 윈도우 OS는 종료된 프로세스의 PID를 빠른 속도로 재할당함.
* 부모 프로세스 A(PID: 1000)가 자식 B(PID: 2000, `ppid = 1000`)를 생성한 후 A가 먼저 종료되고 자식 B는 계속 실행 중인 상태에서, OS가 동일한 PID 1000을 전혀 무관한 새 프로세스 C에 재할당하는 경우.
* 이때 C++ `ProcessTree`가 PID 1000 노드를 새 프로세스 C로 덮어쓰면, 기존 자식 B의 `ppid`가 여전히 1000을 가리키고 있어 B가 엉뚱한 새 프로세스 C를 자기 부모로 오인하고 족보를 거슬러 올라가는 **유령 부모(Ghost Parent) 족보 왜곡** 취약점 발견.

### [원인 (Root Cause)]
* 윈도우 OS 커널은 부모 프로세스가 종료되어도 고아 자식 프로세스의 `ParentProcessId`를 0으로 재설정해주지 않음 (죽은 부모 PPID 영구 보존).
* 기존 `ProcessTree::InsertOrOverwriteNodeInternal`은 PID 재사용 시 이전 노드의 부모(`old_ppid`)와의 링크만 절단하고, **이전 노드가 낳았던 자식들(`it->second.children_pids`)의 부모 링크(`child.ppid = 0`) 절단 처리가 누락**되어 있었음.

### [해결책 (Resolution)]
1. **자식 노드 고아 처리 및 부모 링크 원자적 절단 (`ProcessTree.cpp`)**:
   * **[경로 1: 즉각 재사용 덮어쓰기 (`InsertOrOverwriteNodeInternal`)]**: PID 덮어쓰기 직전, 이전 프로세스의 자식 노드들을 순회하여 `child.ppid == pid`인 경우 `ppid = 0`으로 재설정하여 엉뚱한 새 프로세스로의 유령 입양 원천 차단 (`8161881`).
   * **[경로 2: 10,000개 상한선 영구 퇴출 (`EvictOldestTombstoneInternal`)]**: 톰스톤 노드가 메모리에서 완전히 삭제(Evict)될 때도, 상향 링크(부모의 `children_pids`에서 나를 제거)뿐만 아니라 하향 링크(자식 노드들의 `ppid = 0` 고아 처리)를 양방향으로 원자적 절단 (`7cb4554`).
2. **검증 및 회귀 테스트 통과**:
   * `build.ps1`, `EngineTests.exe`, `DefenseProfilingTest.exe`, `SensorTests.exe`, `IpcE2ETest.exe` 4대 테스트 스위트 전원 통과 (Exit Code 0).

---

## 2026-09-14: [Verified] 실제 OS 프로세스 계층 반복 생성/삭제 & ProcessTree vs OS 실시간 동기화 검증

### [현상 및 검증 목적 (Objective)]
* ProcessTree가 인메모리에서 관리하는 증분(Incremental) 상태와 실제 Windows OS 커널의 `EPROCESS` 테이블(`CreateToolhelp32Snapshot`) 간에 프로세스가 동적으로 생성되고 종료되는 동안 상태 불일치(State Drift)가 발생하는지 여부를 실제 OS 레벨에서 검증할 필요성 대두.

### [검증 설계 (Verification Design)]
1. **실제 OS 프로세스 계층(부모-자식) 동적 기동 (`EngineTests` - Test 8)**:
   - `CreateProcessA`로 실제 `cmd.exe /c timeout /t 10 > nul` 자식 프로세스 2개를 OS에 동적 스폰 (`UniqueHandle` RAII 보호).
2. **실시간 OS 스냅샷 vs 증분 트리 상호 비교 (State Reconciliation)**:
   - 증분 트리(`incremental_tree`)에 시작 이벤트 반영 후, OS 커널 전체 스냅샷(`os_snapshot_tree.InitializeFromSnapshot()`)을 동시 채취하여 대조.
   - PID 존재 여부, 부모 PPID 일치성, 부모의 `children_pids` 목록, 직계 족보 체인(`GetAncestry`)의 100% 완전 일치 확인.
3. **종료 후 사망자 격리 검증**:
   - `TerminateProcess`로 자식 1을 사살한 뒤 OS 스냅샷 채취.
   - OS 테이블에서는 자식 1이 완전히 소멸했음을 확인하고, 증분 트리에서는 사후 포렌식을 위한 Tombstone(`is_alive = false`)으로 안전하게 격리 보존됨을 확인.
   - 아직 살아있는 자식 2는 OS와 증분 트리 모두에서 `is_alive = true`로 유지됨을 확인.
4. **반복 사이클 신뢰성**:
   - 총 3회 반복 사이클 동안 누적 생성 6건, 종료 6건을 수행하며 단 1건의 메모리 누수나 링크 왜곡 없이 100% 동기화 유지 입증.

### [해결 및 증빙 (Resolution & Verification)]
* `EngineTests.exe`에 `TestRealOSProcessTreeSynchronization` (Test 8) 구현 및 커밋 (`020101a`).
* `build.ps1`, `EngineTests.exe`, `DefenseProfilingTest.exe`, `SensorTests.exe`, `IpcE2ETest.exe` 전원 Exit Code 0 통과 완료.

---

## 2026-09-14: [Resolved] Phase 3 CQRS 프로젝션 파이프라인 개통 및 초기 스냅샷 핸드셰이크 구축

### [현상 (Symptom)]
* C# Cockpit이 가동되었을 때 C++ 센서로부터 실시간 증분 이벤트만 수신할 경우, 센서 기동 전이나 Cockpit 기동 전부터 실행 중이던 프로세스(약 300여 개)의 계층 관계를 알지 못해 자식 프로세스 인입 시 족보 추적(`GetAncestry`)이 루트에서 단절되는 콜드 스타트 문제 발생.
* C++ `EtwKernelCollector`에서 프로세스 종료 이벤트(`ProcessStop`) 발생 시 내부 옵저버(`observer_->OnProcessStop`)에게만 통지하고 gRPC 락-스왑 큐 푸시가 누락되어, C# 프로젝션 트리가 종료된 프로세스를 인지하지 못하고 영구 활성 상태로 방치하는 메모리/상태 누수 존재.

### [원인 (Root Cause)]
* 1단계 프로토콜 설계 시 `ProcessEvent`에 프로세스 생명주기 구분이 없었고, 센서-클라이언트 간 gRPC 스트림 연결 시 초기 상태 동기화(Initial State Synchronization) 프로토콜 규약이 부재했음.

### [해결책 (Resolution)]
1. **`phalanx.proto` 생명주기 및 GUID 확장**:
   - `ProcessLifecycle` enum 추가: `LIFECYCLE_UNKNOWN(0)`, `LIFECYCLE_SNAPSHOT(1)`, `LIFECYCLE_START(2)`, `LIFECYCLE_STOP(3)`, `LIFECYCLE_SUSPENDED(4)`, `LIFECYCLE_TERMINATED(5)`.
   - `ProcessEvent`에 `lifecycle`, `process_guid`, `parent_process_guid`, `exit_code` 필드 확장.
2. **C++ `EtwKernelCollector`의 `ProcessStop` 큐 푸시 연동**:
   - 커널 `ProcessStop` 이벤트 수신 시 `ProcessTree::OnProcessStop`을 통해 종료 노드의 메타데이터를 획득하고, `LIFECYCLE_STOP` 태깅 및 종료 코드(`exit_code`)를 포함하여 락-스왑 큐에 원자적 푸시.
3. **`GrpcStreamClient` 초기 300여 개 스냅샷 일괄 덤프 핸드셰이크**:
   - `ProcessTree::GetActiveSnapshotEvents()` 메서드를 구축하여 활성 노드들의 스냅샷 이벤트를 생성.
   - gRPC 스트림 연결 직후 1회 한정으로 활성 프로세스 스냅샷 배치(`LIFECYCLE_SNAPSHOT`)를 C# Cockpit으로 일괄 전송하는 핸드셰이크 구현.
4. **`tests/IpcE2ETest/CMakeLists.txt` 빌드 종속성 보완**:
   - `GrpcStreamClient`의 `ProcessTree` 참조에 따라 `E2E_SOURCES`에 `ProcessTree.cpp` 추가하여 링크 에러 방지.
5. **검증**:
   - `build.ps1`, `EngineTests.exe`, `SensorTests.exe`, `IpcE2ETest.exe`, `DefenseProfilingTest.exe` 전원 Exit Code 0 통과 확인.

---

## 2026-09-14: [Resolved] C# ProcessTreeProjectionManager의 PID 재사용 및 선제 조치 상태 전이 안전성 확보

### [현상 (Symptom)]
* C++ 센서에서 선제 동결(`LIFECYCLE_SUSPENDED`) 또는 즉각 사살(`LIFECYCLE_TERMINATED`) 이벤트를 수신했을 때, 신규 GUID가 생성되거나 활성 PID 매핑이 갱신되면서 기존 프로세스 노드와 분리되어 상태가 전이되지 않는 현상.
* Windows OS의 빈번한 PID 재사용 환경에서 종료된 이전 프로세스의 잔존 포인터로 인해 직계 족보 체인이 왜곡될 위험.

### [원인 (Root Cause)]
* 이벤트 처리기가 들어오는 모든 이벤트를 단순히 PID 기준으로 신규 등록하거나, 생명주기 전이(Start ➔ Suspend/Resume ➔ Stop/Terminate)의 원자적 상태 머신 검증 없이 처리함.

### [해결책 (Resolution)]
1. **상태 전이 라우팅 분기 구현 (`ProcessTreeProjectionManager.cs`)**:
   - `LIFECYCLE_SUSPENDED`, `LIFECYCLE_TERMINATED` 수신 시 `_activePidToGuid`를 우선 조회하여 기존 활성 노드의 `IsSuspended`, `IsTerminated` 플래그를 원자적으로 갱신.
   - `LIFECYCLE_STOP` 수신 시 활성 노드를 Tombstone화(`IsAlive = false`, `ExitCode` 반영)하고 `_activePidToGuid`에서 안전하게 퇴출.
   - `LIFECYCLE_START` 수신 시 동일 PID의 활성 노드가 존재하면 이전 노드를 즉시 Tombstone 처리하고 신규 GUID 노드로 덮어씌워 유령 족보 연결 원천 차단.
2. **0초 인메모리 족보 탐색 보장**:
   - `_nodesByGuid` 딕셔너리를 활용한 O(Depth) 고속 상향 순회로 C++ 센서에 대한 IPC 왕복 지연 없이 즉각적으로 직계 선조 체인(`GetAncestry`) 획득 가능.
3. **단위 테스트 검증**:
   - `ProcessTreeProjectionTests.cs`를 구축하여 350개 노드 스냅샷 일괄 인입, 족보 상향 추적, 델타 생명주기 이벤트(Start/Suspend/Stop), PID 재사용 시 유령 부모 절단 등 4대 핵심 시나리오 100% 통과 (Exit Code 0).

---

## 2026-09-15: [Resolved] Gemini 2.0 Flash REST API 실제 연동 및 하이브리드 ReAct 무중단 폴백(Fallback) 엔진 구축

### [현상 (Symptom)]
* `AutonomousHunterAgent.cs`에 `_geminiApiKey` 필드는 선언되어 있었으나, 실제 외부 Google Gemini REST API(`generativelanguage.googleapis.com`) 호출 통신 클라이언트가 부재하여 하드코딩된 규칙 기반 5단계 시뮬레이터(더미 스크립트)로만 동작하는 한계 존재.
* 외부 네트워크 API 호출을 단순 추가할 경우, API Key가 없는 환경이나 네트워크 단절 환경에서 단위 테스트가 실패하거나 C++ 센서 워치독 SLA(3초)를 초과하여 타임아웃이 발생할 수 있는 잠재적 취약점 존재.

### [원인 (Root Cause)]
* Phase 3 초기 구현 시 로드맵 문서의 참조 자산 링크 누락으로 인해 MundusVivens의 `GeminiApiService.cs` 통신 패턴이 이식되지 않았고, 단위 테스트 고속 통과만을 위해 로컬 오프라인 시뮬레이터로만 작성되었음.

### [해결책 (Resolution)]
1. **Gemini REST API 클라이언트 및 DTO 신설 (`Agent/Gemini/`)**:
   - `GeminiApiDto.cs`: Gemini 2.0 Flash REST 표준 스키마 및 구조화 출력(`AiInvestigationDecision`) 선언.
   - `LlmJsonParser.cs`: 마크다운 코드블록 정제 및 중첩 중괄호 균형 탐색을 통한 안전한 JSON 파서 구현.
   - `GeminiRestClient.cs`: 2.5초 내부 SLA Linked CTS가 결합된 비동기 HTTP 통신 클라이언트 구축.
2. **하이브리드 ReAct 아키텍처 구축 (`AutonomousHunterAgent.cs`)**:
   - `GEMINI_API_KEY` 존재 시: 실제 Gemini 2.0 Flash 호출을 통해 프로세스 족보 및 5대 도구 동적 실행, 실시간 서사 도출 (`InvestigateWithGeminiAsync`).
   - `GEMINI_API_KEY` 부재 또는 네트워크 장애/타임아웃 시: 기존 23ms 오프라인 결정론적 엔진(`InvestigateOfflineDeterministicAsync`)으로 무중단 자동 폴백(Graceful Degradation).
3. **검증 및 무결성 확인 (Ground Truth)**:
   - `AutonomousHunterAgentTests.cs`에 `TestGeminiLiveModeWithMockHttp` 및 `TestGeminiFallbackToOfflineOnNetworkFailure` 추가.
   - 단위 테스트 11종 전원 통과 (`Exit Code 0`), C++ 4대 테스트 스위트 전원 통과 확인.



