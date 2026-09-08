---
description: >-
  Phalanx 단계별(Phase) 기능 구현 마일스톤, 선후 의존 관계 및 완료 정의(DoD). 개발 진행 시 태스크 트래킹 시 참조.
related:
  - ../README.md
  - ./00_project_overview.md
  - ./01_system_architecture.md
  - ./02_ai_agent_investigation_design.md
---
# Phalanx Implementation Roadmap & Milestones

본 문서는 `Phalanx` 프로젝트의 아키텍처 의존성에 따른 단계별(Phase) 구현 목표, 완료 정의(Definition of Done, DoD) 및 기능 검증 계획을 정의합니다. 인위적인 날짜나 기간 대신, **선행 기능의 완성도와 기술적 검증 기준(DoD)**을 중심으로 진행 순서를 관리합니다.

---

## 1. 단계별 구현 마일스톤 흐름

```
[ Phase 1: Sensor & IPC ] ──▶ [ Phase 2: Core & Reflex ] ──▶ [ Phase 3: AI Agent & Tools ] ──▶ [ Phase 4: Cockpit & Presentation ]
  • ETW 커널 수집 루프       • gRPC 양방향 수신 파이프라인 • ReAct 추론 루프              • WPF 노드 그래프 UI
  • 락-스왑 무손실 버퍼      • LiteDB 프로세스 트리        • 5대 OS 도구 호출 체계        • QuestPDF 침해사고 리포트
  • Suspend/Kill 액추에이터  • 로컬 결정론적 룰 엔진       • Fallback 모드 전환 검증      • E2E 차단 시나리오 데모화
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

## 4. 참조 로컬 코드 자산 (Reference Code Assets)

본 프로젝트 개발 시 새로운 패턴을 바닥부터 구현하지 않고, 동일 로컬 환경에 검증된 기존 코드베이스의 핵심 구현체를 직접 참조·재활용합니다:

1. **C++ 락-스왑 큐 & 비동기 gRPC 클라이언트**:
   * 저장소 경로: `C:\Users\user\Documents\GitHub\MundusVivens.GameServer.Cpp`
   * 핵심 참조: `AsyncGrpcClient.cpp` (`agrpc::ClientRPC` + `boost::asio::co_spawn` 패턴) 및 메인 스레드 락-스왑 스왑 큐 메커니즘
2. **C# gRPC 수신 서비스 & 계층형 메모리**:
   * 저장소 경로: `C:\Users\user\Documents\GitHub\MundusVivens`
   * 핵심 참조: `Grpc.AspNetCore` 양방향 스트리밍 수신 파이프라인 및 `LiteDB` 기반 Hot/Cold 캐시 아키텍처
3. **AI 실시간 사고(Thinking) 스트리밍 타이포그래피**:
   * 저장소 경로: `C:\Users\user\Documents\GitHub\GRC`
   * 핵심 참조: `GRC/Themes/ModernStyles.xaml` (`StreamingThoughtTextStyle` 이탤릭 슬레이트 블루, `StreamingNarrativeTextStyle`)
4. **엔터프라이즈 대시보드 레이아웃 & 캡슐 버튼 스타일**:
   * 저장소 경로: `C:\clang-lab\UI_WPF\ArqaStatic`
   * 핵심 참조: `ArqaStatic/Themes/DarkTheme.xaml`, 캡슐형 플랫 버튼(`CornerRadius="24"`), 커스텀 윈도우 다크 타이틀바(`WindowTitleBarBehavior`)
