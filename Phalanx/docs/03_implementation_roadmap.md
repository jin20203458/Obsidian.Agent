---
description: >-
  Phalanx EDR 프로젝트 현재 상태 요약, 핵심 아키텍처 불변식, 컴포넌트 맵, 기술 스택,
  운영 가이드, 완료된 마일스톤 요약(Phase 1~5.5), 차기 활성 백로그(MITRE ATT&CK 내비게이터),
  및 후속 연구 과제.
related:
  - ../README.md
  - ./00_project_overview.md
  - ./01_system_architecture.md
  - ./02_ai_agent_investigation_design.md
  - ./04_performance_benchmarks.md
  - ../../troubleshooting/phalanx.md
---
# Phalanx Implementation Roadmap & Handover Specification

## 1. 프로젝트 현재 상태 요약

* **메인 리포지토리**: `../Phalanx`
  * 활성 작업 브랜치: `feature/phase3-ai-hunter`
  * 원격 저장소: `https://github.com/jin20203458/phalanx`
* **지식베이스 리포지토리**: `../Obsidian.Agent`
  * 공식 스펙: `Phalanx/docs/`
  * 트러블슈팅 런북: `troubleshooting/phalanx.md`
* **솔루션 파일**: `Phalanx.sln` (Visual Studio 2026 / Dev18 및 VS 2022 v17.x 호환)
* **현재 활성 마일스톤**: Phase 6 (차기 과제) - 1순위 과제: MITRE ATT&CK 내비게이터 뷰 (12대 전술 매트릭스 시각화)
* **단위 테스트**: 85개 전원 통과 (Category=Unit, 2026-10-01 기준)

---

## 2. 핵심 아키텍처 불변식 및 런타임 수명주기

에이전트 개발 및 런타임 최상위 행동 규약(AI Decision SSOT, UI 스레드 마샬링, Headless Null-Safety, 이모지 배제, 시크릿 격리)은 [`.agents/AGENTS.md`](../../../Phalanx/.agents/AGENTS.md)에 단일 진실 공급원(SSOT)으로 정의되어 있으므로 이를 엄격히 준수합니다.

본 문서에서는 시스템 런타임 통합 시 준수해야 하는 핵심 기술 불변식만을 유지합니다:

1. **세이프티 워치독 SLA 계약 (10초 기본 / 50초 연장 티켓)**:
   * C++ 센서는 기본 10초(10,000ms) 안전 타임아웃을 적용하며 ([`SafetyWatchdog.h:55`](../../../Phalanx/src/Phalanx.Sensor/Actuator/SafetyWatchdog.h#L55)), AI 심층 수사 진입 시 `ACTION_EXTEND_TIMEOUT` 티켓을 통해 1회 한정 +50초 연장(총 60초 예산)을 집행합니다.
   * C# 최상위 타임아웃 CTS는 **50초(50,000ms)**로 설정하여 워치독 만료 10초 전 안전 마진을 보장합니다.
2. **무결성 레벨 분리 및 수명주기 정리 (Orderly Teardown)**:
   * Cockpit과 Sensor 종료 시 역전송 및 동기화 순서를 엄격히 준수합니다:
     1. gRPC `PHALANX_SENSOR_SHUTDOWN` 역전송
     2. Win32 `Local\PhalanxSensorShutdownEvent` 시그널링
     3. `sensorProcess.WaitForExit(3000)` 대기 후 안전 종료.

---

## 3. 컴포넌트 맵 및 핵심 파일 색인

```
Phalanx Root
├── proto/phalanx.proto                      # gRPC 양방향 스트리밍 프로토콜
├── src/
│   ├── Phalanx.Sensor/ (C++20 Native)       # 커널 텔레메트리 센서 & 고속 액추에이터
│   │   ├── main.cpp                         # 센서 진입점, SeDebugPrivilege
│   │   ├── Collector/EtwKernelCollector.cpp # Windows 커널 ETW 수집기
│   │   ├── Actuator/ProcessActuator.cpp     # 24μs NtSuspendProcess / 0.1ms TerminateProcess
│   │   ├── Actuator/SafetyWatchdog.cpp      # 데드라인 기반 동결 해제 워치독
│   │   ├── Rules/LocalRuleEngine.cpp        # 100μs 오프라인 로컬 규칙 엔진
│   │   └── Ipc/GrpcStreamClient.cpp         # asio-grpc C++20 코루틴 클라이언트
│   └── Phalanx.Cockpit/ (C# .NET 9/10 WPF)  # 엔터프라이즈 관제 허브 & 자율 AI 헌터
│       ├── Program.cs                       # STA 진입점, Kestrel gRPC, --headless 지원
│       ├── Themes/EnterpriseTheme.xaml      # Obsidian 다크 토큰, 벡터 지오메트리
│       ├── Themes/Palettes/                 # DarkPalette.xaml, LightPalette.xaml
│       ├── Services/ThemeManager.cs         # 실시간 동적 테마 전환 엔진
│       ├── Views/                           # 4-View 모듈식 관제 뷰
│       ├── ViewModels/MainViewModel.cs      # 카운터, 필터, 사건 뷰모델 관리
│       ├── Services/CockpitUiBridge.cs      # UI 스레드 디스패처 마샬링 싱글톤
│       ├── Agent/AutonomousHunterAgent.cs   # Gemini ReAct 자율 수사관 & FSM 엔진
│       ├── CQRS/ProcessTreeProjectionManager.cs # 인메모리 프로세스 트리 투영
│       ├── Scenarios/AttackScenarioRegistry.cs  # 10대 실무 시나리오 레지스트리
│       ├── Services/AttackLabScenarioRunner.cs  # 어택랩 시나리오 실행 엔진
│       ├── Storage/ForensicArchiveManager.cs    # LiteDB 포렌식 아카이브
│       └── Tools/ (7대 OS 심층 포렌식 도구)
│           ├── DecodePayloadTool.cs         # Base64/Gzip/Hex 해독
│           ├── ProcessMemoryScanTool.cs     # VirtualQueryEx VAD 스캔 및 모듈 검증
│           ├── ThreatReputationTool.cs      # IoC 평가
│           ├── MitreClassifierTool.cs       # 26종 MITRE ATT&CK 매핑
│           ├── SystemFirewallTool.cs        # Windows 방화벽 C2 차단
│           ├── FileInspectionTool.cs        # Authenticode 서명/위장/엔트로피/사이드로딩 분석
│           └── RegistryInspectionTool.cs    # 64비트 레지스트리/간접 실행/COM 하이재킹 검증
├── scripts/
│   ├── run_attack_simulator.ps1             # 모의 공격 시뮬레이터 실행
│   ├── run_fullchain_test.ps1               # 5대 풀체인 E2E 통합 검증
│   └── build.ps1                            # C++ 센서 빌드
└── tests/
    ├── Phalanx.Agent.Tests/                 # C# 85개 단위 테스트 및 Live 풀체인 테스트
    └── FullChainCrossE2ETest/               # C++ ↔ C# 크로스 랭귀지 E2E 테스트
```

---

## 4. 기술 스택 및 참조 원칙

### 4.1 기술 스택

| 구분 | 기술 스택 및 라이브러리 | 용도 및 비고 |
| :--- | :--- | :--- |
| **IDE / 컴파일러** | Visual Studio 2026 (MSVC v14.51, C++20) / VS 2022 | 윈도우 네이티브 개발 표준 |
| **C++ 라이브러리** | `Microsoft.krabs-etw`, `asio-grpc`, `Boost.Asio` | 커널 수집, 인메모리 트리, 비동기 gRPC |
| **C# 런타임** | .NET 10.0 SDK (.NET 9.0 / 10.0 호환) | 관제 콘솔 및 AI 에이전트 스튜디오 |
| **C# 패키지** | `Grpc.Net.Client`, `LiteDB 5.0.21`, `QuestPDF` | 통신, 포렌식 아카이브, 리포팅 |
| **WPF UI** | `CommunityToolkit.Mvvm`, `EnterpriseTheme` | MVVM 다크 테마 4-View 관제 인터페이스 |
| **AI LLM** | `Google.Apis.Auth` / Gemini 3.7 Flash / Ollama | 구조화 JSON 모드 및 Tool Calling |

### 4.2 참조 로컬 코드 자산 및 차용 원칙 (Clean-Room Principles)

> **참조 원칙**: 본 참조 자산은 **'아키텍처 패턴', '동시성 알고리즘 뼈대', 'UI 디자인 토큰'**만을 학습/차용합니다. 기존 프로젝트의 파일 통째 복사, 비즈니스 도메인 모델/고유 네임스페이스를 복제하는 행위는 엄격히 금지됩니다.

1. **C++ 락-스왑 큐 & 비동기 gRPC**: `../MundusVivens.GameServer.Cpp` (패턴 구조만 참조)
2. **C# Gemini API & gRPC 수신**: `../MundusVivens` (통신 패턴만 참조)
3. **AI 사고 스트리밍 타이포그래피**: `../GRC` (텍스트 스타일 토큰만 참조)
4. **엔터프라이즈 대시보드 & 캡슐 버튼**: `ArqaStatic/Themes/DarkTheme.xaml` (브러시 키값만 참조)

---

## 5. 운영 가이드 및 시나리오 레지스트리

빌드 및 테스트 명령어(`dotnet build`, `dotnet test --filter "Category=Unit"`, `run_fullchain_test.ps1`), 시크릿 파일 격리(`.gitignore`) 규약은 [`.agents/AGENTS.md`](../../../Phalanx/.agents/AGENTS.md)의 `<critical_rules>`를 단일 진실 공급원(SSOT)으로 준수합니다.

### 5.1 LLM 인증 정보 로드 우선순위

1. **환경 변수**: `GOOGLE_APPLICATION_CREDENTIALS` 환경 변수 지정 시 최우선 로드.
2. **로컬 파일**: `src/Phalanx.Cockpit/Config/google-credentials.json` 및 `AppSettings.json`에서 자동 탐색.

### 5.2 10대 실무 시나리오 레지스트리 ([`AttackScenarioRegistry.cs`](../../../Phalanx/src/Phalanx.Cockpit/Scenarios/AttackScenarioRegistry.cs))

| ID | 시나리오 명칭 | 기대 처분 | 핵심 파이프라인 |
|---|---|---|---|
| **1** | Office LOLBAS C2 Dropper | `ACTION_KILL` | 24μs 동결 -> ReAct 3턴 사살 |
| **2** | Ransomware Shadow Copy Deletion | `ACTION_KILL` | C++ 커널 룰 0.08ms 즉각 사살 (Reflex Kill) |
| **3** | LOLBAS CertUtil Remote Payload | `ACTION_KILL` | 24μs 동결 -> 위협 평판 -> 사살 |
| **4** | Browser Drive-by HTA Attack | `ACTION_KILL` | 24μs 동결 -> T1218.005 분류 -> 사살 |
| **5** | Masquerading Dropper (T1036.005) | `ACTION_KILL` | 5턴 심층 수사 -> 사살 + 방화벽 차단 |
| **6** | Benign Admin Script (Known-Good) | `ACTION_RESUME` | 오탐 방지 가드 -> 동결 해제 |
| **7** | SCCM Maintenance Script (Tricky Benign) | `ACTION_RESUME` | 내부 도메인 확인 -> 1ms 조기 복구 |
| **8** | Dev Toolchain Loopback IPC (Tricky Benign) | `ACTION_RESUME` | 로컬 루프백 확인 -> 1ms 조기 복구 |
| **9** | LOLBAS Rundll32 Proxy (T1218.011) | `ACTION_KILL` | APT29 C2 적발 -> T1218.011 -> 사살 |
| **10** | Process Injection Unbacked Memory (T1055) | `ACTION_KILL` | VAD 스캔 -> LockBit C2 -> T1055 -> 사살 |
| **99** | Custom Dynamic Scenario Studio | 사용자 지정 | 동적 텔레메트리 주입 -> AI ReAct 검증 |

**3-모드 주입 체계**: CleanRoom (기본, 인프로세스) / OsHybrid (실제 PID 연동) / LiveExpert (실제 페이로드, Safe Weaponization 적용)

---

## 6. 완료된 마일스톤 요약

* **Phase 1** [완료]: ETW 커널 수집, 락-스왑 무손실 버퍼, gRPC 스트리밍
* **Phase 1.5** [완료]: NtSuspendProcess 24μs 원자적 동결, Toolhelp32 폴백, 10초 세이프티 워치독
* **Phase 2** [완료]: C++ 인메모리 프로세스 트리 DAG (O(1)), 100μs 로컬 룰 엔진, 0.1ms 현장 사살
* **Phase 2.5** [완료]: 스크립트/네이티브 방어 프로파일링 벤치마크, 카나리 누수 0 Bytes 실증
* **Phase 3** [완료]: Gemini ReAct 루프, 5대 OS 포렌식 도구, 10s/50s SLA 연장 안전망, LiteDB 아카이브
* **Phase 3.5** [완료]: C++ -> C# -> C++ 풀체인 E2E 실증 (555ms), 로컬 FSM 가중치 스코어링, 정상 족보 화이트리스트
* **Phase 4** [완료]: 엔터프라이즈 4-View 관제 아키텍처, Flat Virtualized Tree 60FPS, Obsidian 다크 테마
* **Phase 4.1** [완료]: 센서 UAC 자동 기동, 모의 침해 시뮬레이터 연동, Clean-Room 인증 분리
* **Phase 4.2** [완료]: Dark/Light/System 동적 테마, 3계층 실시간 수사 UX 파이프라인, Gauge 모델 통일, 빈 화면 방어
* **Phase 5.1** [완료]: `FileInspectionTool.cs` (WinVerifyTrust P/Invoke 서명 검증, T1036.005 시스템 경로 위장 적발, Shannon 엔트로피 연산, Clean-Room 모의 DB, 단위 테스트 전원 통과)
* **Phase 5.2** [완료]: 어택랩 10대 시나리오 체제 개편, 가상 VAD 스캔 어댑터, FSM 루프백 오탐 방지, 인젝션 독립 50점 가산
* **Phase 5.3** [완료]: `RegistryInspectionTool.cs` (64비트 레지스트리 뷰, CLSID/InprocServer32/ScriptletURL/Run 무결성 검증, COM 하이재킹 T1546.015 및 간접 실행 T1218.010 탐지, Clean-Room 모의 DB 연동, 10대 스트레스 벤치마크 100.0% All-Green 달성)
* **Phase 5.4** [완료]: 심층 수사 취소 및 동결 보존 파이프라인 (Fail-Safe Freeze Invariant: 관제사 수동 취소 시 커널 동결 보존 `ACTION_SUSPEND`/`SUSPENDED_MANUAL_HOLD`, 스피너 피드백, 전역 프로세스 트리 위치 확인 연동 및 Human-in-the-Loop 수동 사살/해제 지원)
* **Phase 5.5** [완료]: `ForensicPdfReportDocument.cs` 동적 적응형(Adaptive Dynamic Flow) 포렌식 A4 리포트 고도화 (단일 페이지 요약 ↔ 5턴 이상 다면 자동 확장, 자동 조치 vs 권고 조치 분리, MainViewModel 내보내기 연동)

> 각 Phase의 세부 구현 내역, DoD 및 벤치마크 데이터는 Git 히스토리 및 [`04_performance_benchmarks.md`](./04_performance_benchmarks.md)에서 확인할 수 있습니다.

---

## 7. 차기 활성 백로그 (Phase 6 Active Backlog)

### 7.1 차기 실체화 대기 항목

| 우선순위 | 컴포넌트 | 현재 상태 | 기대 동작 |
|---|---|---|---|
| **1순위** | MITRE ATT&CK 내비게이터 뷰 | 미구현 | 12대 공격 전술 매트릭스 미니맵 시각화 및 사건 연계 TTP 매핑 |
| **2순위** | 위협 인텔리전스 외부 연동 | 미구현 (설계 대기) | `ThreatReputationTool`에 VirusTotal / AlienVault OTX 외부 API 키 바인딩 옵션 추가 |
| **3순위** | WFP 네이티브 API 전환 | 기술 부채 | `SystemFirewallTool`의 `netsh` CLI 호출을 Windows Filtering Platform Win32 API로 전환하여 지연 최소화 |

### 7.2 유지 중인 설계 수준 목/스텁

| 컴포넌트 | 현재 상태 | 비고 |
|---|---|---|
| `ThreatReputationTool.cs` | 로컬 정적 딕셔너리 기반 (IoC 8건) | 로컬 전용 1차 구현체 유지 |
| `SystemFirewallTool.cs` | 비관리자 환경 시뮬레이션 분기 | 권한 격리 안전 분기 유지 |

---

## 8. 완료된 설계 아티팩트 이관 (Archived Specification SSOT)

Phase 5에서 실체화된 7대 포렌식 도구(`FileInspectionTool`, `RegistryInspectionTool`), 조사 취소 파이프라인 및 동적 적응형 리포트 엔진의 상세 규격과 실측 데이터는 지식베이스의 단일 진실 공급원(SSOT) 문서로 통합 관리됩니다:

* **7대 OS 심층 포렌식 도구 규격**: [`02_ai_agent_investigation_design.md#3-에이전트-전용-tool-calling-생태계`](./02_ai_agent_investigation_design.md)
* **도구 결핍 극복 및 10대 스트레스 벤치마크 실측치 (38ms All-Green)**: [`04_performance_benchmarks.md#10-phase-35-final-중립적-10대-엔터프라이즈-스트레스-벤치마크-1000-all-green-완전-정복`](./04_performance_benchmarks.md)
* **동적 적응형 A4 포렌식 리포트 레이아웃**: [`ForensicPdfReportDocument.cs`](../../../Phalanx/src/Phalanx.Cockpit/Reporting/ForensicPdfReportDocument.cs)

---

## 9. 후속 연구 과제 (Future Empirical Research)

1. **다중 프로세스 상속 체인 동시 동결/수사 확장성 검증**: 트리형 공격에서 동시 다중 프로세스 원자적 동결 및 상속 체인 일괄 처분 검증.
2. **초고부하 텔레메트리 gRPC 스트리밍 I/O 병목 실측**: 초당 10,000건 이상 이벤트 폭주 시 이벤트 드롭률 및 메모리 풋프린트 측정.
3. **로컬 경량 SLM(Ollama Qwen 2.5 / Llama 3) 오프라인 ReAct 실증**: 완전 폐쇄망 환경에서의 로컬 SLM 추론 지연시간 및 기계어 판정 정확도 평가.
