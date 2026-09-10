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
[ Phase 1: Kernel Sensor & Telemetry ] ──▶ [ Phase 1.5: Atomic Freeze ] ──▶ [ Phase 2: In-Memory DAG & Rules ] ──▶ [ Phase 2.5: Defense Profiling Benchmark ] ──▶ [ Phase 3: AI Agent & Forensic Tools ] ──▶ [ Phase 4: Cockpit & Presentation ]
  • ETW 커널 수집 루프 (완료)   • NtSuspendProcess 동결 (완료)  • C++ 인메모리 프로세스 트리 DAG     • 스크립트 150ms 웜업 vs 0.1ms 차단 실측       • Gemini 2.0 Flash ReAct 루프             • ModernWpfUI 다크 대시보드
  • 락-스왑 무손실 버퍼 (완료)  • Toolhelp32 폴백 (완료)        • 로컬 룰 판정 (< 100μs)              • 네이티브 바이너리 2ms 실행 누수 계측        • 5대 OS 수사 도구 (메모리 스캔 등)        • 인터랙티브 프로세스 트리 Canvas
  • ACTION_SUSPEND 대칭 (완료)  • 10초 세이프티 워치독 (완료)   • 0.1ms 현장 사살 & 24μs 선제 동결     • Canary 파일 생성 차단 여부 실증          • 동결 타깃 수사 ➔ 사형/해제 최종 판결    • QuestPDF 포렌식 리포트 출력
```

---

## 2. 단계별 세부 구현 태스크 및 완료 정의 (DoD)

### Phase 1: 고성능 커널 센서 및 텔레메트리 파이프라인 (Kernel Sensor & Telemetry) [완료]
* **목표**: Windows 커널 프로세스 이벤트를 유실 없이 수집하고 gRPC로 고속 송신하는 네이티브 C++ 파이프라인 구축.
* **주요 개발 내용**:
  * Visual Studio 2022 기반 C++20 `Phalanx.Sensor` 프로젝트 스캐폴딩.
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
  * 데드락 방지용 10초 `SafetyWatchdog` (1회 한정 +10초 연장 가드, 자동 Resume) 연동.
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
    * PID 재사용 대응 및 10,000개 Tombstone 메모리 바운딩.
    * 프로세스 족보 역추적(`GetAncestry`) 10,000회 평균 `0.436μs` 달성.
  * **로컬 결정론적 룰 엔진 (`LocalRuleEngine`) 구현**:
    * 비할당 `std::string_view` 및 ASCII 고속 대소문자 무시 비교(< 20ns) 적용.
    * **경로 1 (고신뢰도 악성 ➔ 즉각 사살)**:
      * 규칙: `vssadmin.exe delete shadows`, `bcdedit /set`, `wbadmin delete catalog` 등.
      * 조치: 현장에서 `ProcessActuator::TerminateTargetProcess` 즉시 호출 (0.1ms 이내 사살, `is_terminated = true`).
    * **경로 2 (회색지대 위협 ➔ 선제 동결)**:
      * 규칙: `winword.exe` ➔ `powershell.exe`, `certutil.exe` (LOLBAS 다운로더/스폰) 행위.
      * 조치: 현장에서 `ProcessActuator::SuspendProcess` 호출 (24μs 원자적 동결) ➔ 10초 `SafetyWatchdog` 등록 ➔ gRPC 스트림으로 `is_suspended = true` 보고하여 C# AI 에이전트에 수사 의뢰.
    * **경로 3 (정상 작업 ➔ 무간섭 패스스루)**:
      * 신뢰된 개발/시스템 도구 체인 통과 (`PASS_DEFAULT`).
* **완료 정의 (DoD)**:
  * 단위/벤치마크 테스트(`EngineTests.exe`)에서 50,000회 연속 룰 평가 시 평균 `0.354μs`(초당 257만 건, < 100μs 기준 통과), 10,000회 족보 역추적 시 평균 `0.436μs` 검증 완료.
  * 안전 픽스처 테스트에서 모의 고위험 프로세스 사살(`is_terminated = true`) 및 모의 회색지대 프로세스 24μs 동결(`is_suspended = true`) 확인 (Exit Code 0).
  * 벤치마크 보고서 `docs/04_performance_benchmarks.md` 작성 및 커밋 완료 (`69930b3`).

---

### Phase 2.5: 방어 파이프라인 실측 및 공격 윈도우 벤치마크 (Defense Profiling Benchmark)
* **목표**: Phase 2에서 완성된 C++ 네이티브 엔진의 실시간 차단 능력에 대해, 실제 공격 시나리오(스크립트 기반 vs 네이티브 바이너리)를 대상으로 E2E 차단 시간과 실행 누수(Canary Execution Leak) 여부를 실측하고, 통합 벤치마크 레지스트리(`04_performance_benchmarks.md`)에 실측 데이터 기록.
* **주요 개발 내용**:
  * **[실험 1] 관리형 스크립트 공격 윈도우 검증**:
    * 모의 부모 프로세스 ➔ `powershell.exe -enc ...` (카나리 파일 생성 시도) 스폰.
    * .NET CLR 런타임 웜업 윈도우(약 150~250ms) 대비 Phalanx의 0.1ms 현장 사살 실측 비교.
    * 카나리 파일 생성 전 100% 선제 차단(Zero Payload Execution) 성공 여부 검증.
  * **[실험 2] 네이티브 바이너리 공격 윈도우 한계 측정**:
    * C/C++ 네이티브 모의 바이너리(`MockNativeRansomware.exe`, 진입점 0.5~2ms 이내 디스크 쓰기) 실행.
    * C++ 로컬 룰 엔진(0.1ms)에 의해 카나리 파일 생성이 원천 차단되는지 실측.
  * **벤치마크 보고서 통합 기록**:
    * `04_performance_benchmarks.md`에 E2E 타임라인 간트 차트 및 카나리 누수 실측 데이터 기록.
* **완료 정의 (DoD)**:
  * 스크립트 및 네이티브 모의 공격 모두에서 1ms 미만의 현장 사살로 카나리 파일 미생성(100% 방어) 확인.
  * `Obsidian.Agent/Phalanx/docs/04_performance_benchmarks.md` 실측 결과 업데이트 및 커밋 완료.

---

### Phase 3: C# 자율 AI 위협 헌터 및 포렌식 도구 (AI Agent & Forensic Tools)
* **목표**: C++ 엔진이 동결해 둔 회색지대 타깃을 대상으로, C# AI 에이전트가 ReAct 루프를 돌며 5대 OS 도구를 직접 호출해 3초 이내에 심층 수사를 완료하고 사형/해제 최종 판결 도출.
* **주요 개발 내용**:
  * .NET 9 기반 `Phalanx.Cockpit` 내부 AI 에이전트 서브시스템 구축.
  * Gemini 2.0 Flash 기반의 ReAct 추론 루프 (`Thought ➔ Tool Action ➔ Observation ➔ Final Verdict`) 구현.
  * **5대 OS 수사 도구(Tool) 구현**:
    1. `DecodePayloadTool`: Base64 다단계 난독화 스크립트 해독.
    2. `ProcessMemoryScanTool`: 동결된 타깃 RAM 영역에서 C2 URL/IP 정규식 스캔.
    3. `ThreatReputationTool`: 로컬 위협 DB 및 도메인 평판 조회.
    4. `MitreClassifierTool`: MITRE ATT&CK TTP 자동 매핑.
    5. `SystemFirewallTool`: Netsh/WFP 로컬 방화벽 IP 차단 룰 추가.
  * 수사 결과에 따라 C++ 엔진으로 `ACTION_KILL` 또는 `ACTION_RESUME` gRPC 명령 하달.
  * 구조화 JSON 기반 침해사고 서사(Incident Narrative) 생성 및 `LiteDB` 포렌식 아카이브 영속화.
* **완료 정의 (DoD)**:
  * 가상 동결 프로세스 인입 시, AI 에이전트가 메모리를 스캔하고 C2 평판을 확인하여 3초 이내에 사살 명령과 JSON 서사를 도출함을 확인 (Exit Code 0).

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
| **IDE / 컴파일러** | Visual Studio 2022 (MSVC v143, C++20) | 윈도우 네이티브 개발 표준 |
| **C++ 라이브러리** | `Microsoft.krabs-etw`, `asio-grpc`, `Boost.Asio` | 커널 수집, 인메모리 트리, 비동기 gRPC |
| **C# 런타임** | .NET 9.0 SDK | 관제 콘솔 및 AI 에이전트 스튜디오 |
| **C# 패키지** | `Grpc.Net.Client`, `LiteDB 5.0.21`, `QuestPDF` | 통신, 포렌식 아카이브, 리포팅 |
| **WPF UI** | `CommunityToolkit.Mvvm`, `ModernWpfUI` | MVVM 다크 테마 관제 인터페이스 |
| **AI LLM** | `Google.Apis.Auth` / Gemini 2.0 Flash / Ollama | 구조화 JSON 모드 및 Tool Calling |
