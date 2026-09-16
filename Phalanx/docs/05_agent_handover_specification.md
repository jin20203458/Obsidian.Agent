---
description: >-
  Phalanx EDR Phase 1~4.1 구현 완료 현황 및 후속 개발 에이전트를 위한 핵심 기술 인수인계 사양서.
  Phalanx 프로젝트의 기능 확장, 유지보수, 또는 신규 마일스톤 착수 시 최우선 참조.
related:
  - ../README.md
  - ./01_system_architecture.md
  - ./02_ai_agent_investigation_design.md
  - ./03_implementation_roadmap.md
  - ../../troubleshooting/phalanx.md
---
# Phalanx EDR Agent Handover Specification (Phase 1 ~ 4.1)

## 1. 프로젝트 현황 및 리포지토리 매핑

* **메인 리포지토리**: `../Phalanx` (로컬 워크스페이스: `C:\Users\adg01\Documents\GitHub\Phalanx`)
  * 활성 작업 브랜치: `feature/phase3-ai-hunter`
  * 원격 저장소: `https://github.com/jin20203458/phalanx` (최신 커밋 푸시 완료)
* **지식베이스 리포지토리**: `../Obsidian.Agent` (로컬 워크스페이스: `C:\Users\adg01\Documents\GitHub\Obsidian.Agent`)
  * 공식 스펙: `Phalanx/docs/`
  * 트러블슈팅 런북: `troubleshooting/phalanx.md` (12개 핵심 기술 문제 해결 내역 보존)
* **솔루션 파일**: `Phalanx.sln` (Visual Studio 2026 / Dev18 및 VS 2022 v17.x 호환 표준 솔루션)

---

## 2. 핵심 아키텍처 불변식 (Invariants - 위반 금지 규칙)

후속 작업을 수행하는 에이전트는 아래 5대 불변식을 반드시 준수해야 합니다:

1. **AI 수사관 단일 진실 공급원 (SSOT Decision Authority)**:
   * ReAct 루프가 정상 종결(`reachedFinal == true && hasValidAction`)된 경우, Gemini 모델의 `VerdictAction`(`ACTION_KILL` vs `ACTION_RESUME`)은 절대적 최상위 결정권을 가집니다.
   * `CommandLine.Contains("-enc")` 등 단순 정적 문자열 검사로 LLM의 정상 판결을 사살로 강제 오버라이드하거나 사내 IP를 임의 차단하는 하드코딩 if문을 절대 추가하지 마십시오.
   * 시스템 가드는 **최대 턴(5턴) 초과 타임아웃** 또는 **API 완전 단절/예외** 시의 Fail-Secure 방어에만 국한되어야 합니다.
2. **세이프티 워치독 SLA 계약 (10초 기본 / 50초 연장 티켓)**:
   * C++ 센서는 프로세스를 동결할 때 데드락 및 고아 동결(Orphan Freeze) 방지를 위해 기본 10초(10,000ms) 안전 타임아웃을 적용합니다 (`SafetyWatchdog.h:37`).
   * 오프라인 로컬 규칙 엔진(평균 23ms 완결)은 연장을 요청하지 않으므로, 비정상 크래시 시 10초 데드락 자가 회복(Auto-Resume)이 보장됩니다.
   * C# AI 헌터가 외부 LLM 심층 수사에 진입할 경우 즉시 `ACTION_EXTEND_TIMEOUT` 티켓을 선제 발송하여 마감 기한을 1회에 한해 50초 누적 연장(+50,000ms ➔ 총 60초 예산 확보)합니다 (`SafetyWatchdog.h:55`, `AutonomousHunterAgent.cs:120-125`).
   * C# 최상위 타임아웃 CTS는 네트워크 통신 레이스를 차단하고 워치독 만료 10초 전 안전 마진을 두기 위해 50초(50,000ms)로 엄격 제한합니다 (`AutonomousHunterAgent.cs:131`, `troubleshooting/phalanx.md:43`).
3. **UI 스레드 안전 마샬링 및 Headless 호환성**:
   * Kestrel gRPC 및 비동기 작업 스레드는 `ObservableCollection`을 직접 조작할 수 없습니다. 반드시 `CockpitUiBridge.Instance`를 거쳐 `Dispatcher.InvokeAsync`로 마샬링하십시오.
   * 비GUI 환경(단위 테스트 및 `--headless` CI 러너)을 위해 `Application.Current`가 null이어도 예외 없이 안전 통과하는 Null-Safety 방어를 유지해야 합니다.
4. **무결성 레벨 분리 및 수명주기 정리 (Orderly Teardown)**:
   * 관리자 권한(`High Integrity`)으로 기동된 C++ 센서는 일반 권한(`Medium Integrity`)의 C#에서 OS 레벨 강제 종료가 거부될 수 있습니다.
   * 종료 시에는 반드시 (1) gRPC `PHALANX_SENSOR_SHUTDOWN` 역전송, (2) Win32 `Local\PhalanxSensorShutdownEvent` 시그널링, (3) `sensorProcess.WaitForExit(3000)` 순서를 유지한 후 Kestrel gRPC 서버를 폐쇄하십시오.
5. **모던 상용 EDR 룩앤필 (0 Emojis Policy)**:
   * 관제 GUI 화면(XAML) 및 뷰모델에 이모티콘(🤖, 🔴, 🛡️ 등)을 사용하지 마십시오. 순수 XAML 벡터 지오메트리(`IconShield`, `IconTerminal`, `IconKill` 등)와 Obsidian 다크 팔레트 토큰만을 사용합니다.

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
│       ├── Views/MainWindow.xaml            # 상단 텔레메트리 바, 사건 목록, 우측 점진적 포렌식 인스펙터
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
└── tests/
    ├── Phalanx.Agent.Tests/                 # C# 25개 단위 테스트 및 Live 풀체인 테스트
    └── FullChainCrossE2ETest/               # C++ ➔ C# Cockpit ➔ C++ 크로스 랭귀지 E2E 테스트 바이너리
```

---

### 4.1 LLM 인증 정보 구성 (Clean-Room Credential Architecture)

Phalanx는 타 저장소(예: MundusVivens)에 대한 런타임 의존성 없이 자체 격리(Clean-Room) 환경에서 동작합니다:
1. **환경 변수 우선**: `GOOGLE_APPLICATION_CREDENTIALS` 환경 변수가 지정되어 있을 경우 최우선 로드.
2. **Phalanx 자체 로컬 Config**: 환경 변수 미지정 시 `src/Phalanx.Cockpit/Config/google-credentials.json` 및 `src/Phalanx.Cockpit/AppSettings.json`에서 자체 프로젝트/서비스 계정 정보 탐색.
3. **보안 규칙**: `google-credentials.json` 및 `AppSettings.json`은 `.gitignore`에 등록되어 엄격히 커밋에서 제외됨.

### 4.2 빌드 및 검증 명령어 (Mandatory Verification Suite)

모든 작업 완료 후 보고 전 반드시 아래 명령을 실행하여 **Exit Code 0**을 실사하십시오:

```powershell
# 1. C# 전체 프로젝트 빌드 (오류 0개, 경고 0개 확인)
dotnet build Phalanx.sln

# 2. C# 순수 단위 테스트 실행 (25개 전원 통과 확인, ~300ms)
dotnet test tests/Phalanx.Agent.Tests/ --filter "Category=Unit"

# 3. Google Cloud Vertex AI 실시간 Live 연동 테스트 (선택적: 인증 정보 세팅 시)
dotnet test tests/Phalanx.Agent.Tests/ --filter "Category=Live"

# 4. C++ 네이티브 프로젝트 빌드
powershell -ExecutionPolicy Bypass -File .\build.ps1

# 5. 5대 풀체인 E2E 통합 검증 스위트 실행 (전 단계 통과 확인)
powershell -ExecutionPolicy Bypass -File .\scripts\run_fullchain_test.ps1
```

---

## 5. 차기 백로그 및 확장 권장 사항 (Next Backlog)

다음 개발 주기에 착수 가능한 추천 작업 항목:
1. **QuestPDF 기반 포렌식 감사 리포트 (A4 PDF Export)**:
   * 사건 카드 선택 상태에서 `[EXPORT REPORT]` 버튼 클릭 시, LiteDB의 수사 기록(`ForensicIncidentRecord`)과 ReAct 추론 트레이스를 바탕으로 공공/금융 보안 표준 양식의 1장 요약 A4 PDF 문서 생성 기능 추가.
2. **YARA 룰셋 연동 고도화**:
   * `ProcessMemoryScanTool`에 컴파일된 YARA 룰셋 엔진을 결합하여, VAD Unbacked 메모리 영역에서 Cobalt Strike Beacon, Mimikatz 시그니처 핀포인트 탐지 지원.
3. **MITRE ATT&CK 내비게이터 매트릭스 시각화 뷰**:
   * 포렌식 인스펙터에 12대 공격 전술 매트릭스를 미니 맵 형태로 시각화하여 현재 탐지된 공격 단계(Tactic)를 하이라이트 표시.
