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
  * **AI 수사 킵얼라이브 (Investigation Keep-Alive)**: 50ms 결정론적 룰 엔진으로 즉각 킬되지 않고 AI 심층 조사(2~5초 소요)로 넘어갈 경우, C# 코어는 조사 개시 신호 또는 주기적 하트비트를 통해 워치독 타이머를 갱신(Keep-Alive)함으로써 수사 도중 타깃이 조기 해제되는 참사를 방지할 것.
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

