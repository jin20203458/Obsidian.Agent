---
description: Mundus Vivens C# AI 서버 & C++ 물리 서버 트러블슈팅 런북.
related:
  - ../README.md
  - ../MundusVivens/README.md
---

# Mundus Vivens Troubleshooting

본 문서는 Mundus Vivens 프로젝트 개발 및 통합 테스트 중 발생하는 시스템별 예외 현상과 해결 시나리오를 상세히 기록합니다. 주로 시스템 환경 오류, 빌드 에러, gRPC 프로토콜 및 아키텍처 정합성 관련 문제를 누적하여 다룹니다.

---

## 2026-07-12: [Resolved] 생체 위기 로컬 BT 처리 시 틱 갱신 교착 상태 및 Job 덮어쓰기 결함

### 1. 현상 (Symptom)
* NPC 생체 욕구 위기로 로컬 BT 가상 Job(999000/999001) 실행 시, 20Hz 물리 틱마다 `[목적지 도착 - Direct Seek]` 및 `[Toil Transition]` 로그가 폭발적으로 반복 출력되며 시뮬레이션 틱이 정체되는 현상.

### 2. 원인 (Root Cause)
* C# `GetPendingJobsAsync` gRPC 콜백이 로컬 생존 위기 처리 상태(`is_resolving_survival == true`)를 검사하지 않아 C# 정기 스케줄 Job으로 매 틱 덮어써지며 `toil.state`가 `Idle`로 강제 리셋됨.
* `ActionMoveToTarget`이 목적지 도착 완료를 오직 구역명 문자열 일치(`loc.location_name == job.target_location`)에만 의존하여 황무지 등에서 불일치 시 영구 실패.

### 3. 해결책 (Resolution)
1. **Pending Job 갱신 스킵**: `GetPendingJobsAsync` 콜백에서 `is_resolving_survival == true`인 NPC는 C# 스케줄 주입 대상에서 즉시 제외.
2. **좌표 기반 도착 판정**: `ActionMoveToTarget`에 목적지 좌표와의 물리적 거리 연산(`dist < 0.8f`)을 추가하여 구역명 불일치 시에도 `Success` 반환 보장.
3. **가상 Job 타겟 초기화**: 가상 Job 주입 시 `target_location = ""` 및 좌표를 초기화하여 가구 탐색 및 모닥불 자율 야영 정상 스폰 보장.

---

## 2026-07-14: [Resolved] Tracy Profiler 비활성화(ENABLE_PROFILING=OFF) 빌드 시 헤더 컴파일 실패

### 1. 현상 (Symptom)
* `ENABLE_PROFILING=OFF` 옵션으로 빌드 시 `#include <tracy/Tracy.hpp>`를 참조하는 소스 파일들에서 `fatal error C1083: 포함 파일을 열 수 없습니다. 'tracy/Tracy.hpp'` 컴파일 에러 발생.

### 2. 원인 (Root Cause)
* Tracy 헤더가 `#ifdef TRACY_ENABLE` 가드 없이 직접 인클루드되어, `ENABLE_PROFILING=OFF` 시 CMake의 include 경로에 Tracy가 추가되지 않아 컴파일러가 헤더를 찾지 못함.

### 3. 해결책 (Resolution)
* 단일 래퍼 헤더 `TracyIntegration.h`를 신설하여 `TRACY_ENABLE` 미정의 시 모든 프로파일링 매크로(`FrameMark`, `ZoneScoped` 등)를 no-op으로 치환하고, 소스 파일의 직접 include를 래퍼 헤더 참조로 일원화:
  ```cpp
  #pragma once
  #ifdef TRACY_ENABLE
  #   define TRACY_ON_DEMAND
  #   include <tracy/Tracy.hpp>
  #else
  #   define FrameMark
  #   define ZoneScoped
  #   define ZoneScopedN(name)
  #   define TracyLockable(type, name) type name
  #   define LockableBase(type) type
  #endif
  ```

---

## 2026-07-14: [Resolved] Tracy Profiler 활성화 시 main() 정적 초기화 단계 즉시 종료 (Exit Code 1)

### 1. 현상 (Symptom)
* `ENABLE_PROFILING=ON` 빌드 후 실행 시 `main()` 첫 배너조차 출력되지 않고 콘솔 출력 0바이트 상태로 프로세스가 즉시 종료(`Exit Code 1`).

### 2. 원인 (Root Cause)
* `TRACY_ENABLE` 정의 시 생성되는 전역 정적 객체 `tracy::Profiler`가 `main()` 진입 전 정적 초기화(Static Initialization) 단계에서 소켓 인프라(TCP 8086) 바인딩 실패 시 내부에서 `exit(1)`을 호출함.

### 3. 해결책 (Resolution)
* `CMakeLists.txt`에서 `TRACY_ON_DEMAND` 매크로를 빌드 타겟에 주입하여 프로파일러 GUI가 접속하기 전까지 소켓 인프라 초기화를 지연.

---

## 2026-07-14: [Resolved] Tracy 온디맨드 매크로 유실 및 방화벽 UDP 바인딩 충돌 해결 (소스 내재화)

### 1. 현상 (Symptom)
* `TRACY_ON_DEMAND` 지정에도 불구하고 서버 기동 즉시 `exit(1)` 크래시 발생 및 Windows 방화벽 UDP 브로드캐스트 차단.

### 2. 원인 (Root Cause)
1. vcpkg 사전 컴파일 라이브러리(`TracyClient.lib`)는 `TRACY_ON_DEMAND`가 꺼진 채 빌드되어 헤더 매크로 주입이 무효화됨.
2. 미적용 상태의 기동 즉시 활성화된 소켓 스레드가 UDP 브로드캐스트를 시도하다 OS 보안 정책에 의해 강제 종료됨.

### 3. 해결책 (Resolution)
1. **의존성 소스 격리 내재화**: vcpkg 라이브러리 링크를 배제하고 Tracy 클라이언트 소스(`thirdparty/tracy/TracyClient.cpp`)를 프로젝트 내부로 직접 임포트.
2. **타겟 매크로 격리 주입**: `CMakeLists.txt`에서 `TRACY_ENABLE`, `TRACY_ON_DEMAND`, `TRACY_NO_BROADCAST`를 주입하여 UDP 브로드캐스트를 원천 차단하고 순수 TCP(8086)만 오픈.

---

## 2026-07-19: [Resolved] 인과 캐스케이드 지수형 노드 폭발로 인한 시스템 OOM 및 순환 참조 방어

### 1. 현상 (Symptom)
* 인과 캐스케이드 벤치마크(깊이 10, 분기 5) 실행 시 RAM 사용량이 10GB 이상으로 급증하며 가비지 컬렉터(GC) 한계 도달 및 시스템 OOM 크래시.

### 2. 원인 (Root Cause)
* 등비수열 $\sum_{k=0}^D C^k$ 공식에 의해 1,220만 개 노드가 일시에 인메모리 할당되고, 부모 ID 재귀 중첩 문자열로 인해 메모리 소비가 폭증함.

### 3. 해결책 (Resolution)
1. **깊이 제한 가드 (Depth Clamping = 5)**: `BeliefEngine.cs` 내부 `PropagateCausalCascade`에서 전파 깊이 5레벨 초과 시 조기 종료하여 지수 폭발 차단.
2. **순환 참조 방지 가드**: `HashSet`을 통한 방문 검증으로 재귀 순환 궤도(A ➔ B ➔ A) 즉각 탈출(`StackOverflowException` 예방).

---

## 2026-07-19: [Resolved] Windows OS 기본 클럭 주기(15.6ms)로 인한 C++ 물리 틱레이트 지연 해결

### 1. 현상 (Symptom)
* 20Hz(50.0ms) 메인 게임 루프의 실제 틱 주기가 약 61.3ms로 늘어지며 시뮬레이션 물리 속도가 약 22% 감속되는 현상.

### 2. 원인 (Root Cause)
* Windows OS 스케줄러 기본 클럭 주기가 15.6ms로 설정되어 있어, `std::this_thread::sleep_for`가 15.6ms 단위로 양자화(Rounding)되며 오차 누적.

### 3. 해결책 (Resolution)
* `winmm` 라이브러리를 링크하고, `main()` 진입 시 RAII 클래스 `WindowsTimerResolutionRaii` 가드로 `timeBeginPeriod(1)`을 호출하여 타이머 해상도를 1ms 단위로 상향 조정 (프로세스 종료 시 `timeEndPeriod(1)` 복구).

---

## 2026-07-20: [Resolved] 대량 기억 도태(Eviction Storm) 시 메인 스레드 4.4초 I/O 멈춤 해결

### 1. 현상 (Symptom)
* 단시간 대량 기억 도태(Eviction Storm) 시 C# AI 서버 전체가 약 4.4초 동안 멈추는 프레임 드랍 발생.

### 2. 원인 (Root Cause)
* `OnBeliefEvicted` 핸들러가 메인 스레드상에서 디스크 파일(`GameData.db`)에 동기식(Synchronous) 쓰기를 수행.
* `MemoryBox._lock`을 잡은 상태에서 `PersistenceService._dbLock`까지 순차 획득하는 이중 락 중첩(Double-Lock Nesting)으로 디스크 I/O 대기 동안 메인 루프 블로킹.

### 3. 해결책 (Resolution)
* `System.Threading.Channels` 기반 Async Write-Behind Queue(`Channel<ArchiveEntry>`, 용량 2048, `DropOldest`) 구축.
* 동기 직접 쓰기 대신 O(1) 메모리 큐 삽입(`EnqueueArchive`)으로 전환하고, 백그라운드 워커 스레드(`RunArchiveWorkerAsync`)가 LiteDB 비동기 일괄 기록 수행.

---

> [!NOTE]
> **[정제 완료 기준선]** 2026-10-06 이전 상위 항목은 정제 완료됨. 신규 인시던트는 이 아래에 추가됩니다.

---
