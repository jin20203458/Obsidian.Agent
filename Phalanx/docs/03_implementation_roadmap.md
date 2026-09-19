---
description: >-
  Phalanx 2계층 아키텍처 기반 단계별(Phase) 기능 구현 마일스톤, 선후 의존 관계 및 완료 정의(DoD). 개발 진행 시 태스크 트래킹 시 참조.
related:
  - ../README.md
  - ./00_project_overview.md
  - ./01_system_architecture.md
  - ./02_ai_agent_investigation_design.md
  - ./04_performance_benchmarks.md
---
# Phalanx Implementation Roadmap & Milestones

본 문서는 `Phalanx` 프로젝트의 2계층(C++ 네이티브 엔진 + C# AI 관제 콘솔) 아키텍처에 따른 단계별(Phase) 구현 목표, 완료 정의(Definition of Done, DoD) 및 기능 검증 계획을 정의합니다. 인위적인 날짜나 기간 대신, **선행 기능의 완성도와 기술적 검증 기준(DoD)**을 중심으로 진행 순서를 관리합니다.

---

## 1. 단계별 구현 마일스톤 흐름

```
[ Phase 1: Kernel Sensor & Telemetry ] ──▶ [ Phase 1.5: Atomic Freeze ] ──▶ [ Phase 2: In-Memory DAG & Rules ] ──▶ [ Phase 2.5: Defense Profiling Benchmark ] ──▶ [ Phase 3: AI Agent & Forensic Tools ] ──▶ [ Phase 3.5: Full-Chain E2E & Local FSM ] ──▶ [ Phase 4: Cockpit & Presentation ]
  • ETW 커널 수집 루프 (완료)   • NtSuspendProcess 동결 (완료)  • C++ 인메모리 프로세스 트리 (완료)   • 스크립트 150ms 웜업 vs 0.1ms 차단 (완료)    • Gemini ReAct 루프 (완료)                • C++ ➔ C# ➔ C++ 폐루프 E2E 실증          • ModernWpfUI 다크 대시보드
  • 락-스왑 무손실 버퍼 (완료)  • Toolhelp32 폴백 (완료)        • 로컬 룰 판정 (< 100μs) (완료)       • 네이티브 바이너리 2ms 실행 누수 계측 (완료) • 상용 1티어 5대 OS 도구 (완료)          • 로컬 FSM & 위험도 가중치 스코어링       • 인터랙티브 프로세스 트리 Canvas
  • ACTION_SUSPEND 대칭 (완료)  • 10초 세이프티 워치독 (완료)   • 0.1ms 현장 사살 & 24μs 동결 (완료)  • Canary 누수 0 Bytes 실증 (완료)          • 10s/50s SLA 연장 안전망 (완료)         • 정상 관리 족보 화이트리스트 가드        • QuestPDF 포렌식 리포트 출력
```

---

## 2. 단계별 세부 구현 태스크 및 완료 정의 (DoD)

### Phase 1: 고성능 커널 센서 및 텔레메트리 파이프라인 (Kernel Sensor & Telemetry) [완료]
* **목표**: Windows 커널 프로세스 이벤트를 유실 없이 수집하고 gRPC로 고속 송신하는 네이티브 C++ 파이프라인 구축.
* **주요 개발 내용**:
  * Visual Studio 2026 (MSVC v14.51, C++20) 및 Visual Studio 2022 기반 `Phalanx.Sensor` 프로젝트 스캐폴딩.
  * `krabs-etw` 라이브러리 연동 및 `Microsoft-Windows-Kernel-Process` ETW 프로바이더 리스너 구현.
  * `DoubleBufferedSwapQueue` 락-스왑 템플릿 구현 및 주기적 스왑 플러시 루프 계측.
  * Win32 `OpenThread` ➔ `SuspendThread` 및 `TerminateProcess` 안전 래퍼 함수 구현.
  * `phalanx.proto` 정의 및 `asio-grpc` 비동기 스트리밍 클라이언트 연동.
  * `EtwKernelCollector::Start()` 원자적 CAS 상태 전이 및 예외 안전성 롤백 적용 (`edc00bf`).
* **완료 정의 (DoD)**:
  * 로컬에서 `powershell.exe` 실행 시, C++ 센서가 이벤트를 드롭 없이 캡처하여 콘솔에 즉시 출력 (완료).
  * `SuspendThread` 호출 시 타깃 프로세스가 20ms 이내에 완전히 정지(Freeze)됨을 작업 관리자에서 확인 (완료).

---

### Phase 1.5: 원자적 고속 동결 엔진 및 안전 폴백 (Atomic Freeze & Fallback) [완료]
* **목표**: `ntdll.dll`의 미공개 커널 API(`NtSuspendProcess`/`NtResumeProcess`)를 `Common::UniqueHModule`로 동적 로드하여 동결 지연 시간을 100배 단축(수 ms ➔ 24μs)하고 스레드 레이스 컨디션을 원천 차단하며, 실패 시 기존 Toolhelp32 방식으로 우아하게 후퇴(Fallback)하는 2중 방어선 구축.
* **주요 개발 내용**:
  * `ProcessActuator` 내부에 `NtSuspendProcess` / `NtResumeProcess` 함수 포인터 시그니처 및 동적 로딩 구현 (`Common::UniqueHModule` 활용).
  * **1순위 (Primary)**: `NtSuspendProcess`를 통한 프로세스 레벨 원자적 동결 집행 (동결 도중 신규 스레드 생성 탈출 불가).
  * **2순위 (Fallback)**: API 로드 실패 또는 특정 OS 환경 비호환 시 기존 `CreateToolhelp32Snapshot` + `SuspendThread` 순회 루프로 즉각 자동 폴백(Graceful Degradation).
  * 복구(Resume) 시에도 동일하게 `NtResumeProcess` 1순위 시도 후 실패 시 스레드별 `ResumeThread` 2순위 폴백.
  * 데드락 방지용 세이프티 워치독(`SafetyWatchdog`, 기본 10초, 1회 한정 +50초 연장 가드, 만료 시 자동 Resume) 연동.
  * `phalanx.proto` 및 `GrpcStreamClient`에 `ACTION_SUSPEND = 4` 핸들러 추가로 프로토콜 대칭성 확립 (`89768b4`).
* **완료 정의 (DoD)**:
  * `NtSuspendProcess` 성공 시 프로세스 동결 소요 시간이 50μs 미만(실측 24~27μs)으로 단축됨을 단위 테스트에서 확인 (완료).
  * 가상 실패 주입 시에도 Win32 스냅샷 폴백이 즉각 작동하여 프로세스가 100% 정상 동결/복구됨을 확인 (완료, Exit Code 0).

---

### Phase 2: C++ 인메모리 프로세스 트리 및 100μs 로컬 룰 엔진 (In-Memory DAG & Local Rules) [완료]
* **목표**: C++ 네이티브 엔진 내부에서 활성 프로세스 트리(DAG)를 O(1)로 유지하고, 100μs 이내에 고위험 공격은 현장 즉시 사살(`0.1ms`), 회색지대 위협은 선제 동결(`24μs`)하는 자율 완결형 EDR 엔진 완성.
* **주요 개발 내용**:
  * **C++ 인메모리 프로세스 트리 (`ProcessTree`) 구현**:
    * `std::unordered_map<uint32_t, ProcessNode>` 기반 O(1) 부모-자식 관계 추적.
    * 기동 시 `InitializeFromSnapshot()`을 통한 335개 OS 프로세스 웜업 적재.
    * PID 재사용 대응 및 10,000개 Tombstone 메모리 바운딩 (고스트 부모 방지 양방향 링크 절단 완료).
    * 프로세스 족보 역추적(`GetAncestry`) 10,000회 평균 `0.436μs` 달성.
    * 실제 OS 프로세스 반복 생성/삭제 및 Toolhelp32 스냅샷 100% 동기화 검증 (`EngineTests` - Test 8).
  * **로컬 결정론적 룰 엔진 (`LocalRuleEngine`) 구현**:
    * 비할당 `std::string_view` 및 ASCII 고속 대소문자 무시 비교(< 20ns) 적용.
    * **경로 1 (고신뢰도 악성 ➔ 즉각 사살)**:
      * 규칙: `vssadmin.exe delete shadows`, `bcdedit /set`, `wbadmin delete catalog` 등.
      * 조치: 현장에서 `ProcessActuator::TerminateTargetProcess` 즉시 호출 (0.1ms 이내 사살, `is_terminated = true`).
    * **경로 2 (회색지대 위협 ➔ 선제 동결)**:
      * 규칙: `winword.exe` ➔ `powershell.exe`, `certutil.exe` (LOLBAS 다운로더/스폰) 행위.
      * 조치: 현장에서 `ProcessActuator::SuspendProcess` 호출 (24μs 원자적 동결) ➔ 세이프티 워치독(기본 10초) 등록 ➔ gRPC 스트림으로 `is_suspended = true` 보고하여 C# AI 에이전트에 수사 의뢰.
    * **경로 3 (정상 작업 ➔ 무간섭 패스스루)**:
      * 신뢰된 개발/시스템 도구 체인 통과 (`PASS_DEFAULT`).
* **완료 정의 (DoD)**:
  * 단위/벤치마크 테스트(`EngineTests.exe`)에서 50,000회 연속 룰 평가 시 평균 `0.354μs`(초당 257만 건, < 100μs 기준 통과), 10,000회 족보 역추적 시 평균 `0.436μs` 검증 완료.
  * 안전 픽스처 테스트에서 모의 고위험 프로세스 사살(`is_terminated = true`) 및 모의 회색지대 프로세스 24μs 동결(`is_suspended = true`) 확인 (Exit Code 0).
  * 실제 OS 프로세스 생성/삭제 동기화 검증(Test 8) 통과 확인 (Exit Code 0).
  * 벤치마크 보고서 [04_performance_benchmarks.md](./04_performance_benchmarks.md) 작성 및 커밋 완료 (`69930b3`).

---

### Phase 2.5: 방어 파이프라인 실측 및 공격 윈도우 벤치마크 (Defense Profiling Benchmark) [완료]
* **목표**: Phase 2에서 완성된 C++ 네이티브 엔진의 실시간 차단 능력에 대해, 실제 공격 시나리오(스크립트 기반 vs 네이티브 바이너리)를 대상으로 E2E 차단 시간과 실행 누수(Canary Execution Leak) 여부를 실측하고, 통합 벤치마크 레지스트리([04_performance_benchmarks.md](./04_performance_benchmarks.md))에 실측 데이터 기록.
* **주요 개발 내용**:
  * **[실험 1] 관리형 스크립트 공격 윈도우 검증**:
    * 모의 부모 프로세스 ➔ `powershell.exe -enc ...` (카나리 파일 생성 시도) 스폰.
    * .NET CLR 런타임 웜업 윈도우(약 624.50ms) 대비 Phalanx의 50.8μs 원자적 동결 실측 비교 (+624.45ms 안전 마진).
    * 카나리 파일 생성 전 100% 선제 차단(Zero Payload Execution) 성공 검증.
  * **[실험 2] 네이티브 바이너리 공격 윈도우 한계 측정**:
    * C/C++ 네이티브 모의 바이너리(`MockNativeRansomware.exe`, 진입점 0.8ms 윈도우) 실행.
    * C++ 로컬 룰 엔진(105.1μs 사살)에 의해 디스크 쓰기 전 카나리 파일 생성이 원천 차단(Zero Leak)됨을 실측 (+65.50ms 안전 마진).
  * **벤치마크 보고서 통합 기록**:
    * [04_performance_benchmarks.md](./04_performance_benchmarks.md)에 E2E 타임라인 간트 차트 및 카나리 누수 실측 데이터 기록.
* **완료 정의 (DoD)**:
  * 스크립트 및 네이티브 모의 공격 모두에서 1ms 미만의 선제 동결/사살로 카나리 파일 미생성(100% 방어, 누수 0건 / 0 Bytes) 확인 (`DefenseProfilingTest.exe` 통과, Exit Code 0).
  * [04_performance_benchmarks.md](./04_performance_benchmarks.md) 실측 결과 업데이트 및 커밋 완료.

---

### Phase 3: C# 자율 AI 위협 헌터 및 포렌식 도구 (AI Agent & Forensic Tools) [완료]
* **목표**: C++ 엔진이 동결해 둔 회색지대 타깃을 대상으로, C# AI 에이전트가 ReAct 루프를 돌며 5대 OS 도구를 직접 호출해 3초 이내에 심층 수사를 완료하고 사형/해제 최종 판결 도출.
* **주요 개발 내용**:
  * **[Step 1] gRPC 프로토콜 및 CQRS 트리 프로젝션 파이프라인 개통**:
    * `phalanx.proto`: `ProcessLifecycle` enum 및 PID 재사용 방지용 `process_guid` 필드 추가.
    * C++ `EtwKernelCollector`: `ProcessStop` 이벤트 발생 시 `LIFECYCLE_STOP`으로 `DoubleBufferedSwapQueue` 적재 누락 보완.
    * C++ `GrpcStreamClient`: C# 관제 콘솔 최초 접속 시 `ProcessTree::InitializeFromSnapshot()` 데이터를 `LIFECYCLE_SNAPSHOT`으로 1회 일괄 덤프 전송.
    * C# `ProcessTreeProjectionManager`: 수신된 스냅샷과 델타 이벤트를 바탕으로 로컬 메모리에 완전한 `ObservableCollection` 기반 프로세스 트리 DAG 구축 (C++ 역질의 없이 로컬 0초 족보 탐색).
  * **[Step 2] .NET 9 기반 `Phalanx.Cockpit` 내부 AI 에이전트 서브시스템 구축**:
    * Gemini 3.7 Flash 기반의 ReAct 추론 루프 (`Thought ➔ Tool Action ➔ Observation ➔ Final Verdict`) 구현.
    * 수사 개시 시 C++ 워치독 데드락 방지 1회성 타임아웃 연장 티켓(`ACTION_EXTEND_TIMEOUT`, +50초) 자동 발송.
    * 로컬 프로세스 트리를 기반으로 부모-자식-조부모 족보 문맥을 프롬프트에 무지연 주입.
  * **[Step 3] 5대 OS 수사 도구(Tool) 구현**:
    1. `DecodePayloadTool`: Base64 다단계 난독화 스크립트 해독.
    2. `ProcessMemoryScanTool`: P/Invoke `VirtualQueryEx`/`ReadProcessMemory` 기반 동결 타깃 RAM C2 URL/IP 정규식 스캔.
    3. `ThreatReputationTool`: 로컬 위협 DB 및 도메인 평판 조회.
    4. `MitreClassifierTool`: MITRE ATT&CK TTP 자동 매핑.
    5. `SystemFirewallTool`: Netsh/WFP 로컬 방화벽 IP 차단 룰 추가 (Loopback/Localhost Safety Clamp 적용).
  * 수사 결과에 따라 C++ 엔진으로 `ACTION_KILL` 또는 `ACTION_RESUME` gRPC 명령 하달.
  * 구조화 JSON 기반 침해사고 서사(Incident Narrative) 생성 및 `LiteDB 5.0.21` 포렌식 아카이브 영속화.
* **완료 정의 (DoD)**:
  * C# 접속 시 C++로부터 300여 개 초기 스냅샷이 수신되어 C# 로컬 트리가 즉각 완성되고, 신규 프로세스 생성/종료/동결/사살 이벤트가 실시간 반영됨을 확인 (`ProcessTreeProjectionTests` 통과).
  * 5대 OS 수사 도구 개별 동작 및 방화벽 안전 루프백 클램핑 검증 완료 (`InvestigationToolsTests` 통과).
  * 가상 동결 프로세스 인입 시, AI 에이전트가 로컬 트리의 족보 문맥을 바탕으로 메모리를 스캔하고 C2 평판을 확인하여 **331ms**(요구 기준 < 3,000ms 대비 9배 빠름) 만에 98% 확신도로 사살 명령(`ACTION_KILL`)과 JSON 서사를 도출하고 LiteDB 아카이브 저장 확인 (`AutonomousHunterAgentTests` 통과, Exit Code 0).
  * C++ 센서/엔진/E2E 테스트(`EngineTests`, `SensorTests`, `IpcE2ETest`, `DefenseProfilingTest`) 및 C# 테스트(`dotnet test`) 전원 100% Exit Code 0 통과 확인.

---

### Phase 3.5: 풀체인 E2E 실증 및 로컬 FSM 의사결정 고도화 (Full-Chain E2E & Local FSM Decision Engine)
* **목표**: C++ 커널 센서와 C# 관제 콕핏/AI 헌터를 실제 런타임 환경에서 결합하여 C++ ➔ C# ➔ C++ 폐루프(Closed-Loop) 전체 방어 서사를 자동 검증하고, 인터넷/LLM 단절 시 발동되는 로컬 오프라인 수사 엔진을 단순 키워드 매칭에서 상태 머신(FSM) 및 가중치 스코어링 모델로 격상하여 오탐을 원천 차단.
* **주요 개발 내용**:
  * **[태스크 1] C++ ➔ C# ➔ C++ 풀체인 라이브 통합 시스템 테스트 구축 (최우선)**:
    * Kestrel gRPC 서버(`Phalanx.Cockpit`, 포트 50051)와 C++ 센서(`Phalanx.Sensor.exe`)를 동시에 백그라운드 기동하여 양방향 스트리밍 핸드셰이크 수립.
    * 테스트 러너가 실제 OS에 외부 공격 프로세스(`powershell.exe -enc <C2 다운로더>`)를 독립 스폰.
    * C++ 센서 24μs 원자적 동결(`NtSuspendProcess`) ➔ gRPC 텔레메트리 전송 ➔ C# AI 에이전트 ReAct 수사 ➔ gRPC 사살 명령(`ACTION_KILL`) 역전송 ➔ C++ 액추에이터 현장 사살(`TerminateProcess`) 및 프로세스 강제 종료까지의 전체 닫힌 루프(Closed-Loop) 자동화 검증.
    * 각 단계별 타임스탬프 실측 및 종료 코드(Exit Code 0) 검증 스크립트 작성.
  * **[태스크 2] 로컬 오프라인 수사 '가중치 스코어링 & 상태 머신(FSM)' 고도화**:
    * `AutonomousHunterAgent.cs`의 `InvestigateOfflineDeterministicAsync` 내부 판정식을 단순 `Contains("-enc")`에서 다차원 누적 위험도(Risk Score) 모델로 전환:
      * 비정상 부모 프로세스(Office/HWP ➔ cmd/ps): +30점
      * 인라인 C2 다운로드 패턴: +35점
      * Unbacked 실행 메모리 주입: +40점
      * 사내 정상 서명/내부망 도메인: -50점
      * 총합 80점 초과 시에만 `ACTION_KILL` 집행.
    * 상태 머신 기반 조기 탈출(Early-Exit): 1단계 디코딩 결과 사내 정상 작업 확인 시 1ms 내 `ACTION_RESUME` 즉시 복구.
    * LLM 파이프라인(`InvestigateWithGeminiAsync`)은 코드 수정 없이 100% 격리 유지하되, LLM 장애/타임아웃 시 Fallback 안전벨트 품질 극대화.
  * **[태스크 3] 정상 관리 도구 및 시스템 프로세스 족보 화이트리스트 (Known-Good Baseline)**:
    * `explorer.exe ➔ powershell.exe` 등 사용자가 직접 기동한 터미널 및 윈도우 정상 관리 도구에 대한 동결 예외 필터링.
    * 오피스, 브라우저, PDF 등 취약 상위 앱에서 파생된 스크립트 실행기만 선별 동결하도록 로컬 룰 엔진 정밀화.
  * **[태스크 4] LLM 수사 퀄리티 2차 고도화 (상용 Copilot 수준 마감)**:
    * `<target_context>`에 실행 경로(Temp 폴더 여부), 디지털 서명 유무, 프로세스 무결성 레벨(Integrity Level) 메타데이터 추가 주입.
    * 프롬프트에 정상 관리 스크립트 방면(`ACTION_RESUME`) Few-shot 예시 1건 추가로 사살 편향(Confirmation Bias) 방지.
    * `AiInvestigationDecision` DTO에 `remediation_steps?: string[]` (전사 방화벽 차단, 계정 리셋 등 후속 조치 처방전) 필드 신설.
* **완료 정의 (DoD) - [2026-09-16 검증 완료]**:
  * 실제 OS 공격 프로세스 기동 시, 24μs 원자적 동결(`NtSuspendProcess`) ➔ Kestrel HTTP/2 gRPC 소켓 ➔ C# AI/FSM 수사 ➔ gRPC `ACTION_KILL` ➔ Win32 `TerminateProcess` 현장 사살 ➔ `targetProc.HasExited == true` 완전 닫힌 루프(Closed-Loop) 실측 자동화 완주 (`LiveFullChainE2ETests`, 555ms, Exit Code 0).
  * 로컬 오프라인 수사에서 정상 사내 스크립트(`*.internal`, `*.corp.local`) 인입 시 1ms 조기 탈출(`ACTION_RESUME`), 악성 인라인 다운로더 인입 시 누적 위험도 105점(> 80점)으로 `ACTION_KILL` 및 방화벽 C2 차단, 볼륨 섀도 복사본 삭제(`vssadmin delete shadows`) 파괴 명령 시 +80점 즉각 사살 판결 확인 (`AutonomousHunterAgentTests`).
  * 취약 부모 프로세스 감시망(`AcroRd32.exe`, `Acrobat.exe`, `hwp.exe`) 확장 및 C++ 회귀 단위 테스트 통과 (`EngineTests.exe`).
  * LLM 2-Shot 균형 프롬프트(`<example type="verdict_resume">`), `<target_context>` 3대 메타데이터 주입 및 사후 조치 처방전(`remediation_steps`) DTO / LiteDB 아카이브 완비.
  * 통합 테스트 러너(`run_fullchain_test.ps1`) 5대 전 단계(C# 순서 실측, C++ 센서/엔진 벤치, C++ gRPC 루프백, 실제 OS Live E2E, C++ ➔ C# ➔ C++ 크로스 랭귀지 E2E) 100% Exit Code 0 통과 확인.

---

### Phase 4: WPF 관제 콘솔 및 포트폴리오 에셋화 (Cockpit & Presentation)
* **목표**: SOC 관제 표준 다크 테마 감각을 적용하여 실시간 프로세스 트리와 AI 사고 피드를 시각화하고, 원클릭 PDF 리포트 출력 및 데모 에셋 제작.
* **주요 개발 내용**:
  * `Phalanx.Cockpit` WPF 프로젝트 UI 완성 (ModernWpfUI 다크 테마).
  * 인터랙티브 프로세스 공격 트리 Canvas 렌더링 (안전 초록, 동결 파랑 펄스, 사살 빨강 배지).
  * 실시간 AI 에이전트 사고 스트리밍 터미널 패널 구현 (이탤릭 슬레이트 블루 타이포그래피).
  * `QuestPDF` 기반 공식 침해사고 A4 포렌식 리포트 출력 템플릿 완성.
  * Windows 토스트 알림 클릭 시 조사실 창으로 바로 진입하는 UX 연결.
* **완료 정의 (DoD)**:
  * 전체 공격 및 방어 시나리오가 WPF 화면에 매끄럽게 렌더링되고, 버튼 클릭 시 포렌식 PDF 보고서가 정상 출력.
  * 모의 시연 영상(MP4) 및 고화질 GIF 에셋 녹화 완료.
  * GitHub용 영문/국문 README.md 및 아키텍처 다이어그램 게시.

---

## 3. 기술 스택 및 개발 환경 요구사항

| 구분 | 기술 스택 및 라이브러리 | 용도 및 비고 |
| :--- | :--- | :--- |
| **IDE / 컴파일러** | Visual Studio 2026 (MSVC v14.51, C++20) / VS 2022 | 윈도우 네이티브 개발 표준 |
| **C++ 라이브러리** | `Microsoft.krabs-etw`, `asio-grpc`, `Boost.Asio` | 커널 수집, 인메모리 트리, 비동기 gRPC |
| **C# 런타임** | .NET 10.0 SDK (.NET 9.0 / 10.0 호환) | 관제 콘솔 및 AI 에이전트 스튜디오 |
| **C# 패키지** | `Grpc.Net.Client`, `LiteDB 5.0.21`, `QuestPDF` | 통신, 포렌식 아카이브, 리포팅 |
| **WPF UI** | `CommunityToolkit.Mvvm`, `ModernWpfUI` | MVVM 다크 테마 관제 인터페이스 |
| **AI LLM** | `Google.Apis.Auth` / Gemini 3.7 Flash / Ollama | 구조화 JSON 모드 및 Tool Calling |

---

## 4. 참조 로컬 코드 자산 및 차용 원칙 (Reference Assets & Clean-Room Principles)

> **참조 원칙 (Clean-Room Implementation Rule)**:
> * 본 참조 자산은 **'아키텍처 패턴(Boilerplate)', '동시성 알고리즘 뼈대', 'UI 디자인 토큰(XAML 스타일)'**만을 학습·차용하기 위한 것입니다.
> * 기존 프로젝트의 **파일 통째 복사, 비즈니스 도메인 모델(게임 NPC/대화, 정적분석 진단 등), 고유 네임스페이스를 복제하는 행위는 엄격히 금지**됩니다.
> * 모든 코드는 Phalanx의 보안/EDR 도메인(`ProcessEvent`, `ProcessTree`, `MitigationCommand`)에 맞추어 **새롭게 독립 구현(Clean-Room)**되어야 합니다.

1. **C++ 락-스왑 큐 & 비동기 gRPC 클라이언트**:
   * 저장소 경로: `../MundusVivens.GameServer.Cpp`
   * **참조 범위 (Pattern Only)**: `AsyncGrpcClient.cpp`의 `agrpc::ClientRPC` + `boost::asio::co_spawn` 비동기 호출 **패턴 구조** 및 락-스왑 템플릿 알고리즘 (게임 로직 복제 금지).
2. **C# Gemini API 호출, gRPC 수신 서비스 & 계층형 메모리**:
   * 저장소 경로: `../MundusVivens`
   * **참조 범위 (Pattern Only)**:
     - `GeminiApiService.cs`: Google Gemini REST API 호출, JSON 모드 강제, 토큰 로깅 및 오류 핸들링 **통신 패턴**.
     - `Grpc.AspNetCore` 양방향 스트리밍 수신 파이프라인 및 `Channel<T>` 기반 백그라운드 LiteDB 비동기 쓰기(Write-Behind) **패턴** (게임 세이브/에이전트 모델 복제 금지).
3. **AI 실시간 사고(Thinking) 스트리밍 타이포그래피**:
   * 저장소 경로: `../GRC`
   * **참조 범위 (Tokens Only)**: `GRC/Themes/ModernStyles.xaml`의 폰트 크기, 행간, 이탤릭 슬레이트 블루(`#A2B9D8`) 등 **순수 텍스트 스타일 정의** (롤플레잉 시나리오/뷰모델 복제 금지).
4. **엔터프라이즈 대시보드 레이아웃 & 캡슐 버튼 스타일**:
    * 참조 에셋: `ArqaStatic/Themes/DarkTheme.xaml` (엔터프라이즈 WPF UI 디자인 에셋)
    * **참조 범위 (Tokens Only)**: `ArqaStatic/Themes/DarkTheme.xaml`의 캡슐형 플랫 버튼(`CornerRadius="24"`), 다크 타이틀바, 다크 팔레트 브러시 **키값** (정적분석 진단 로직 및 다국어 번역 복제 금지).
