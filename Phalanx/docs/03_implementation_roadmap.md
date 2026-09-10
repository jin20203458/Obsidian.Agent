---
description: >-
  Phalanx 단계별(Phase) 기능 구현 마일스톤, 선후 의존 관계 및 완료 정의(DoD). 개발 진행 시 태스크 트래킹 시 참조.
related:
  - ../README.md
  - ./00_project_overview.md
  - ./01_system_architecture.md
  - ./02_ai_agent_investigation_design.md
  - ./04_concurrency_queue_benchmark.md
  - ./05_edr_reflex_pipeline_profiling.md
---
# Phalanx Implementation Roadmap & Milestones

본 문서는 `Phalanx` 프로젝트의 아키텍처 의존성에 따른 단계별(Phase) 구현 목표, 완료 정의(Definition of Done, DoD) 및 기능 검증 계획을 정의합니다. 인위적인 날짜나 기간 대신, **선행 기능의 완성도와 기술적 검증 기준(DoD)**을 중심으로 진행 순서를 관리합니다.

---

## 1. 단계별 구현 마일스톤 흐름

```
[ Phase 1: Sensor & IPC ] ──▶ [ Phase 1.5: Atomic Freeze ] ──▶ [ Phase 2: Core & Reflex ] ──▶ [ Phase 2.5: Reflex Profiling ] ──▶ [ Phase 3: AI Agent & Tools ] ──▶ [ Phase 4: Cockpit & Presentation ]
  • ETW 커널 수집 루프         • NtSuspendProcess 원자적 동결   • gRPC 양방향 수신 파이프라인 • E2E 왕복 지연 실측 (RTT)    • ReAct 추론 루프              • WPF 노드 그래프 UI
  • 락-스왑 무손실 버퍼        • 100배 가속 (10~20μs)           • LiteDB 프로세스 트리        • 오피스-파워셸 공격 시나리오 • 5대 OS 도구 호출 체계        • QuestPDF 침해사고 리포트
  • Suspend/Kill 액추에이터    • Toolhelp32 우아한 폴백         • 로컬 결정론적 룰 엔진       • CLR 웜업 대비 차단 입증     • Fallback 모드 전환 검증      • E2E 차단 시나리오 데모화
```

---

## 2. 단계별 세부 구현 태스크 및 완료 정의 (DoD)

### Phase 1: 고성능 센서 및 통신 파이프라인 (Sensor & IPC)
* **목표**: Windows 커널 프로세스 이벤트를 유실 없이 수집하고 gRPC로 고속 송신하는 네이티브 C++ 파이프라인 구축.
* **주요 개발 내용**:
  * Visual Studio 2022 기반 C++20 `Phalanx.Sensor` 프로젝트 스캐폴딩.
  * `krabs-etw` 라이브러리 연동 및 `Microsoft-Windows-Kernel-Process` ETW 프로바이더 리스너 구현.
  * `DoubleBufferedSwapQueue` 락-스왑 템플릿 구현 및 주기적 스왑 플러시 루프 계측.
  * Win32 `OpenThread` ➔ `SuspendThread` 및 `TerminateProcess` 안전 래퍼 함수 구현.
  * `phalanx.proto` 정의 및 `asio-grpc` 비동기 스트리밍 클라이언트 연동.
* **완료 정의 (DoD)**:
  * 로컬에서 `powershell.exe` 실행 시, C++ 센서가 이벤트를 드롭 없이 캡처하여 콘솔에 즉시 출력.
  * `SuspendThread` 호출 시 타깃 프로세스가 20ms 이내에 완전히 정지(Freeze)됨을 작업 관리자에서 확인.

---

### Phase 1.5: 원자적 고속 동결 엔진 및 안전 폴백 (Atomic Freeze & Fallback)
* **목표**: `ntdll.dll`의 미공개 커널 API(`NtSuspendProcess`/`NtResumeProcess`)를 `Common::UniqueHModule`로 동적 로드하여 동결 지연 시간을 100배 단축(수 ms ➔ 10~20μs)하고 스레드 레이스 컨디션을 원천 차단하며, 실패 시 기존 Toolhelp32 방식으로 우아하게 후퇴(Fallback)하는 2중 방어선 구축.
* **주요 개발 내용**:
  * `ProcessActuator` 내부에 `NtSuspendProcess` / `NtResumeProcess` 함수 포인터 시그니처 및 동적 로딩 구현 (`Common::UniqueHModule` 활용).
  * **1순위 (Primary)**: `NtSuspendProcess`를 통한 프로세스 레벨 원자적 동결 집행 (동결 도중 신규 스레드 생성 탈출 불가).
  * **2순위 (Fallback)**: API 로드 실패 또는 특정 OS 환경 비호환 시 기존 `CreateToolhelp32Snapshot` + `SuspendThread` 순회 루프로 즉각 자동 폴백(Graceful Degradation).
  * 복구(Resume) 시에도 동일하게 `NtResumeProcess` 1순위 시도 후 실패 시 스레드별 `ResumeThread` 2순위 폴백.
  * `SensorTests`에 `NtSuspendProcess` 원자적 동결 검증 및 강제 폴백(Fault Injection) 테스트 케이스 추가.
* **완료 정의 (DoD)**:
  * `NtSuspendProcess` 성공 시 프로세스 동결 소요 시간이 50μs 미만으로 단축됨을 단위 테스트에서 확인.
  * 가상 실패 주입 시에도 Win32 스냅샷 폴백이 즉각 작동하여 프로세스가 100% 정상 동결/복구됨을 확인 (Exit Code 0).

---

### Phase 2: 코어 엔진 및 결정론적 1차 방어 (Core & Reflex)
* **목표**: C# 백엔드에서 텔레메트리를 수신해 인과 그래프를 구성하고, 룰 엔진을 통해 0.05초 이내에 자동 차단하는 닫힌 루프(Closed-Loop) 완성.
* **주요 개발 내용**:
  * .NET 8/9 C# `Phalanx.Core` 프로젝트 생성 및 gRPC 수신 서비스 구축.
  * `LiteDB 5.0` 기반의 임베디드 Threat Graph 메모리 구축 (프로세스 부모-자식 트리 및 인과 관계망 매핑).
  * gRPC `StreamTelemetry` 양방향 스트림의 역방향 응답 채널을 통한 `MitigationCommand` 하달 연동.
  * **결정론적 1차 룰 엔진 (Deterministic Rule Engine)** 구축:
    * *규칙 1: `winword.exe`, `excel.exe` ➔ 자식 `powershell.exe`, `cmd.exe` 스폰 감지.*
    * *규칙 2: 명령줄 인자에 `-enc`, `-EncodedCommand`, `DownloadString` 포함 여부 판별.*
  * 규칙 충족 시 C++ 센서로 `ACTION_KILL` 명령을 자동 하달.
* **완료 정의 (DoD)**:
  * 테스트 매크로 스크립트 실행 시, 사람의 개입 없이 0.05초(50ms) 이내에 파워셸 프로세스가 강제 종료되어야 함.

---

### Phase 2.5: 반사신경 파이프라인 E2E 실측 및 프로파일링 (Reflex Pipeline Profiling)
* **목표**: Phase 2에서 완성된 C++ 센서와 C# 코어 간의 양방향 닫힌 루프(Closed-Loop) 파이프라인을 바탕으로, 실제 공격 시나리오(오피스 매크로 난독화 파워셸 스폰)에 대한 엔드투엔드(E2E) 반사신경 차단 시간(Round-Trip Time, RTT)을 C++ 관점에서 실측하고, 파이프라인 지연 시간을 프로파일링하여 문서화(`05_edr_reflex_pipeline_profiling.md`).
* **주요 개발 내용**:
  * **C++ 센서 관점의 RTT 계측 체계 구축**:
    * 텔레메트리 패킷 발송 시점(`t0 = high_resolution_clock::now()`)과 C# 코어로부터 `ACTION_KILL` 응답 수신 후 프로세스 강제 종료(`TerminateProcess`) 집행 완료 시점(`t1`) 간의 순수 왕복 시간(Round-Trip Time) 계측.
    * 프로세스 간 시계 동기화 오버헤드를 배제하고 외부 관측자(C++ 센서) 기준의 단일 타임라인 확립.
  * **C# 코어 내부 경량 진단 로깅**:
    * `Stopwatch`를 활용하여 gRPC 수신 시점부터 50ms 결정론적 룰 엔진 판정 완료 시점까지의 순수 판단 소요 시간 보조 로깅.
  * **실제 악성 모의 공격 시나리오 검증**:
    * 부모 프로세스(`winword.exe` 모사) ➔ 자식 프로세스(`powershell.exe -WindowStyle Hidden -EncodedCommand ...`) 스폰 공격 체인 실행.
    * 파워셸의 .NET CLR 로딩 및 런타임 웜업 윈도우(약 150~250ms) 대비 Phalanx의 반사신경 차단 완료 시점 실측치 비교 분석.
  * **벤치마크 보고서 문서화**:
    * `05_edr_reflex_pipeline_profiling.md` 작성 (구간별 소요 시간 표, 파이프라인 타임라인 간트 차트, 공격자 실행 윈도우 대비 차단 시점 분석 수록).
* **완료 정의 (DoD)**:
  * 모의 악성 스크립트 실행 시, C++ 단독 RTT 기준 전체 E2E 차단 소요 시간이 50ms 미만(실측 목표 15~20ms)으로 검증됨을 확인.
  * `Obsidian.Agent/Phalanx/docs/05_edr_reflex_pipeline_profiling.md` 보고서가 작성되어 커밋됨.

---

### Phase 3: 자율 AI 에이전트 및 심층 조사 루프 (AI Agent & Tools)
* **목표**: ReAct 루프를 통해 에이전트가 OS 도구를 직접 호출하며 미지의 위협을 심층 분석하고 자연어 서사 리포트를 도출.
* **주요 개발 내용**:
  * Gemini API 및 로컬 LLM(Ollama) 통신 서비스 인터페이스 구축.
  * **5대 OS 도구(Tool) 구현**:
    1. `DecodePayloadTool`: Base64 다단계 난독화 스크립트 해독.
    2. `ProcessMemoryScanTool`: 프로세스 메모리 내 C2 IP/URL 정규식 스캔.
    3. `ThreatReputationTool`: 로컬 악성 IP/도메인 블랙리스트 조회.
    4. `MitreClassifierTool`: MITRE ATT&CK TTP 매핑.
    5. `SystemFirewallTool`: Netsh 로컬 방화벽 IP 차단 룰 추가.
  * ReAct 추론 루프 (`Thought ➔ Tool Action ➔ Observation ➔ Final Verdict`) 파이프라인 완성.
  * 구조화 JSON 기반 침해사고 서사(Incident Narrative) 생성 엔진 완성.
  * API 키 부재 시 자동으로 룰 엔진으로만 동작하는 **Graceful Degradation** 모드 전환 검증.
* **완료 정의 (DoD)**:
  * 모의 침투 페이로드 실행 시, AI 에이전트가 도구를 호출하여 C2 IP를 스스로 알아내고 자연어 분석 보고서 JSON을 도출. (전형적 위협 시나리오 1~2회 반복 기준 3초 이내 도출, 복합 다단계 심층 분석 시 5~8초 허용).

---

### Phase 4: WPF 관제 콘솔 및 포트폴리오 에셋화 (Cockpit & Presentation)
* **목표**: SOC 관제 표준 다크 테마 감각을 적용하여 실시간 프로세스 트리와 AI 사고 피드를 시각화하고, 원클릭 PDF 리포트 출력 및 데모 에셋 제작.
* **주요 개발 내용**:
  * .NET 8/9 `Phalanx.Cockpit` WPF 프로젝트 생성 (ModernWpfUI 다크 테마 적용).
  * 인터랙티브 프로세스 공격 트리 Canvas 렌더링 (부모-자식 노드 및 KILLED 뱃지 가시화).
  * 실시간 AI 에이전트 사고 스트리밍 터미널 패널 구현.
  * `QuestPDF` 기반 공식 침해사고 A4 포렌식 리포트 출력 템플릿 완성.
  * Windows 토스트 알림 클릭 시 조사실 창으로 바로 진입하는 UX 연결.
* **완료 정의 (DoD)**:
  * 전체 공격 및 방어 시나리오(3초 컷)가 WPF 화면에 매끄럽게 렌더링되고, 클릭 한 번으로 PDF 보고서가 출력.
  * 모의 시연 영상(MP4) 및 고화질 GIF 에셋 녹화 완료.
  * GitHub용 영문/국문 README.md 및 아키텍처 다이어그램 게시.

---

## 3. 기술 스택 및 개발 환경 요구사항

| 구분 | 기술 스택 및 라이브러리 | 용도 및 비고 |
| :--- | :--- | :--- |
| **IDE / 컴파일러** | Visual Studio 2022 (MSVC v143, C++20) | 윈도우 네이티브 개발 표준 |
| **C++ 라이브러리** | `Microsoft.krabs-etw`, `asio-grpc`, `Boost.Asio` | 커널 수집 및 비동기 gRPC |
| **C# 런타임** | .NET 8.0 또는 .NET 9.0 SDK | 코어 엔진 및 데스크톱 콘솔 |
| **C# 패키지** | `Grpc.AspNetCore`, `LiteDB 5.0.21`, `QuestPDF` | 통신, 그래프 DB, 리포팅 |
| **WPF UI** | `CommunityToolkit.Mvvm`, `ModernWpfUI` | MVVM 다크 테마 관제 인터페이스 |
| **AI LLM** | `Google.Apis.Auth` / Gemini 2.0 Flash / Ollama | 구조화 JSON 모드 및 Tool Calling |

---

## 4. 참조 로컬 코드 자산 및 차용 원칙 (Reference Assets & Clean-Room Principles)

> **참조 원칙 (Clean-Room Implementation Rule)**:
> * 본 참조 자산은 **'아키텍처 패턴(Boilerplate)', '동시성 알고리즘 뼈대', 'UI 디자인 토큰(XAML 스타일)'**만을 학습·차용하기 위한 것입니다.
> * 기존 프로젝트의 **파일 통째 복사, 비즈니스 도메인 모델(게임 NPC/대화, 정적분석 진단 등), 고유 네임스페이스를 복제하는 행위는 엄격히 금지**됩니다.
> * 모든 코드는 Phalanx의 보안/EDR 도메인(`ProcessEvent`, `ThreatGraph`, `MitigationCommand`)에 맞추어 **새롭게 독립 구현(Clean-Room)**되어야 합니다.

1. **C++ 락-스왑 큐 & 비동기 gRPC 클라이언트**:
   * 저장소 경로: `C:\Users\user\Documents\GitHub\MundusVivens.GameServer.Cpp`
   * **참조 범위 (Pattern Only)**: `AsyncGrpcClient.cpp`의 `agrpc::ClientRPC` + `boost::asio::co_spawn` 비동기 호출 **패턴 구조** 및 락-스왑 템플릿 알고리즘 (게임 로직 복제 금지).
2. **C# gRPC 수신 서비스 & 계층형 메모리**:
   * 저장소 경로: `C:\Users\user\Documents\GitHub\MundusVivens`
   * **참조 범위 (Pattern Only)**: `Grpc.AspNetCore` 양방향 스트리밍 수신 파이프라인 및 `Channel<T>` 기반 백그라운드 LiteDB 비동기 쓰기(Write-Behind) **패턴** (게임 세이브/에이전트 모델 복제 금지).
3. **AI 실시간 사고(Thinking) 스트리밍 타이포그래피**:
   * 저장소 경로: `C:\Users\user\Documents\GitHub\GRC`
   * **참조 범위 (Tokens Only)**: `GRC/Themes/ModernStyles.xaml`의 폰트 크기, 행간, 이탤릭 슬레이트 블루(`#A2B9D8`) 등 **순수 텍스트 스타일 정의** (롤플레잉 시나리오/뷰모델 복제 금지).
4. **엔터프라이즈 대시보드 레이아웃 & 캡슐 버튼 스타일**:
   * 저장소 경로: `C:\clang-lab\UI_WPF\ArqaStatic`
   * **참조 범위 (Tokens Only)**: `ArqaStatic/Themes/DarkTheme.xaml`의 캡슐형 플랫 버튼(`CornerRadius="24"`), 다크 타이틀바, 다크 팔레트 브러시 **키값** (정적분석 진단 로직 및 다국어 번역 복제 금지).
