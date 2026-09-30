---
description: >-
  Phalanx EDR 프로젝트 현재 상태 요약, 핵심 아키텍처 불변식, 컴포넌트 맵, 기술 스택,
  운영 가이드, 완료된 마일스톤 요약, 활성 백로그(FileInspectionTool, QuestPDF, MITRE),
  복합 회피 실험 한계점 및 FileInspectionTool 상세 규격.
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
* **현재 활성 마일스톤**: Phase 5 (활성 백로그) - 차기 1순위 과제: `FileInspectionTool.cs` 구현
* **단위 테스트**: 49개 전원 통과 (2026-09-30 기준)

---

## 2. 핵심 아키텍처 불변식 (Architectural Invariants)

최상위 행동 수칙인 [`.agents/AGENTS.md`](../../../Phalanx/.agents/AGENTS.md)의 핵심 규칙들을 실제 코드베이스에서 안전하게 계승하기 위해, 후속 에이전트는 아래 5대 불변식을 준수해야 합니다:

1. **AI 수사관 단일 진실 공급원 (SSOT Decision Authority)**:
   * ReAct 루프가 정상 종결(`reachedFinal == true && hasValidAction`)된 경우, Gemini 모델의 `VerdictAction`(`ACTION_KILL` vs `ACTION_RESUME`)은 절대적 최상위 결정권을 가집니다.
   * `CommandLine.Contains("-enc")` 등 단순 정적 문자열 검사로 LLM의 정상 판결을 사살로 강제 오버라이드하는 하드코딩 if문을 절대 추가하지 마십시오.
   * 시스템 가드는 **최대 턴(5턴) 초과 타임아웃** 또는 **API 완전 단절/예외** 시의 Fail-Secure 방어에만 국한되어야 합니다.
2. **세이프티 워치독 SLA 계약 (10초 기본 / 50초 연장 티켓)**:
   * C++ 센서는 기본 10초(10,000ms) 안전 타임아웃 적용 ([`SafetyWatchdog`](../../../Phalanx/src/Phalanx.Sensor/Actuator/SafetyWatchdog.h)).
   * C# AI 헌터가 외부 LLM 심층 수사에 진입할 경우 `ACTION_EXTEND_TIMEOUT` 티켓으로 1회 한정 +50초 연장(총 60초 예산).
   * C# 최상위 타임아웃 CTS는 50초(50,000ms)로 엄격 제한.
3. **UI 스레드 안전 마샬링 및 Headless 호환성**:
   * 반드시 `CockpitUiBridge.Instance`를 거쳐 `Dispatcher.InvokeAsync`로 마샬링.
   * `Application.Current`가 null이어도 예외 없이 안전 통과하는 Null-Safety 방어 유지.
4. **무결성 레벨 분리 및 수명주기 정리 (Orderly Teardown)**:
   * 종료 시 (1) gRPC `PHALANX_SENSOR_SHUTDOWN` 역전송, (2) Win32 `Local\PhalanxSensorShutdownEvent` 시그널링, (3) `sensorProcess.WaitForExit(3000)` 순서 유지.
5. **모던 상용 EDR 룩앤필 (0 Emojis Policy)**:
   * XAML 및 뷰모델에 유니코드 이모티콘을 일절 사용하지 않으며, 순수 XAML 벡터 지오메트리와 팔레트 토큰만 사용.

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
│       └── Tools/ (5대 OS 포렌식 도구)
│           ├── DecodePayloadTool.cs         # Base64/Gzip/Hex 해독
│           ├── ProcessMemoryScanTool.cs     # VirtualQueryEx VAD 스캔
│           ├── ThreatReputationTool.cs      # IoC 평가
│           ├── MitreClassifierTool.cs       # 26종 MITRE ATT&CK 매핑
│           └── SystemFirewallTool.cs        # Windows 방화벽 C2 차단
├── scripts/
│   ├── run_attack_simulator.ps1             # 모의 공격 시뮬레이터 실행
│   ├── run_fullchain_test.ps1               # 5대 풀체인 E2E 통합 검증
│   └── build.ps1                            # C++ 센서 빌드
└── tests/
    ├── Phalanx.Agent.Tests/                 # C# 49개 단위 테스트 및 Live 풀체인 테스트
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

## 5. 운영 가이드 및 검증 체계

### 5.1 LLM 인증 정보 구성

1. **환경 변수 우선**: `GOOGLE_APPLICATION_CREDENTIALS` 환경 변수가 지정되어 있을 경우 최우선 로드.
2. **Phalanx 자체 로컬 Config**: `src/Phalanx.Cockpit/Config/google-credentials.json` 및 `AppSettings.json`에서 탐색.
3. **보안 규칙**: `google-credentials.json` 및 `AppSettings.json`은 `.gitignore`에 등록되어 엄격히 커밋에서 제외됨.

### 5.2 검증 체계

기본 빌드/테스트 명령어는 [`.agents/AGENTS.md`](../../../Phalanx/.agents/AGENTS.md)의 `<critical_rules>`에 SSOT로 정의되어 있습니다.

* **클라우드 Live AI Benchmark**: `dotnet test tests/Phalanx.Agent.Tests/ --filter "Category=Live"` (실행 전 유효한 인증 정보 필수, 약 3.5분 소요)

### 5.3 10대 실무 시나리오 레지스트리 ([`AttackScenarioRegistry.cs`](../../../Phalanx/src/Phalanx.Cockpit/Scenarios/AttackScenarioRegistry.cs))

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
* **Phase 5.2** [완료]: 어택랩 10대 시나리오 체제 개편, 가상 VAD 스캔 어댑터, FSM 루프백 오탐 방지, 인젝션 독립 50점 가산

> 각 Phase의 세부 구현 내역, DoD 및 벤치마크 데이터는 Git 히스토리 및 [`04_performance_benchmarks.md`](./04_performance_benchmarks.md)에서 확인할 수 있습니다.

---

## 7. 활성 백로그 (Phase 5 Active Backlog)

### 7.1 차기 실체화 대기 항목

| 우선순위 | 컴포넌트 | 현재 상태 | 기대 동작 |
|---|---|---|---|
| **1순위** | `FileInspectionTool.cs` | 파일 미생성 (미구현) | `WinVerifyTrust` P/Invoke, 시스템 경로 위장(T1036.005) 감별, Shannon 엔트로피 연산 |
| **2순위** | `InvestigationView.xaml` A4 리포트 | `Command` 미연동 (준비 중 툴팁) | QuestPDF 기반 A4 포렌식 PDF 렌더링/다운로드 |
| **3순위** | MITRE ATT&CK 내비게이터 뷰 | 미구현 | 12대 공격 전술 매트릭스 미니맵 시각화 |

### 7.2 현재 유지 중인 설계 수준 목/스텁

| 컴포넌트 | 현재 상태 | 비고 |
|---|---|---|
| `ThreatReputationTool.cs` | 로컬 정적 딕셔너리 기반 (IoC 8건) | 로컬 전용 1차 구현체 유지 |
| `SystemFirewallTool.cs` | 비관리자 환경 시뮬레이션 분기 | 권한 격리 안전 분기 유지 |

---

## 8. 복합 회피 공격 실험 결과 및 FileInspectionTool 필요성 (Phase 5.1 동기)

### 8.1 실험 개요

블랙리스트 미등록 외부 IP(`198.51.100.99`) + 시스템 파일명 위장(`C:\Windows\Temp\svchost.exe`)이 결합된 복합 회피 공격(T1036.005)을 모의하여 AI 수사관의 도구 연동 한계를 실측.

### 8.2 실험 결과 (Ground Truth)

* **총 소요**: 5턴, 36.5초 (최대 예산 완전 전소)
* **최종 판결**: `ACTION_KILL` (확신도 96%, 방화벽 차단 완료)
* **병목 원인**: 파일 검증 도구 부재로 `ProcessMemoryScanTool`로 우회 -> **Turn 3에서 21.7초 낭비** (워치독 SLA의 43.4% 잠식)

### 8.3 FileInspectionTool 상세 규격

#### A. 설계 목적

* 디스크 파일의 존재 유무, 정적 메타데이터(크기, 시간, 해시) 수집
* Win32 `WinVerifyTrust` 기반 Authenticode 체인 검증
* 시스템 핵심 실행 파일의 비인가 디렉터리 위장 배치(Masquerading) 적발
* PE 헤더 매직 바이트 검사 및 확장자 위장 감별

#### B. 디지털 서명 검증

* **구현**: `wintrust.dll` / `crypt32.dll`의 `WinVerifyTrust` API P/Invoke
* **검증 GUID**: `WINTRUST_ACTION_GENERIC_VERIFY_V2` (`{00AAC56B-CD44-11d0-8CC2-00C04FC295EE}`)
* **플래그**: `WTD_REVOCATION_CHECK_NONE` (오프라인 고속용) 또는 `WTD_REVOCATION_CHECK_CHAIN`
* **판정**: Microsoft 정규 서명이 없는 `svchost.exe`, `csrss.exe`, `lsass.exe` 등은 즉시 위험 점수 100점

#### C. 시스템 파일 경로 위장 탐지

* **화이트리스트**: `svchost.exe`, `csrss.exe`, `smss.exe`, `wininit.exe`, `winlogon.exe`, `services.exe`, `lsass.exe` -> 정규 경로: `C:\Windows\System32\`
* **탐지**: 파일명이 화이트리스트에 포함되나 실제 경로가 `\Temp\`, `\AppData\`, `\Users\Public\` 등에 위치할 경우 `IsPathMasqueraded = true`

#### D. 파일 엔트로피 및 확장자 위장

* **Shannon Entropy**: 파일 바이트 스트림(최대 1MB) 대상. 엔트로피 > 7.2 시 고밀도 암호화/패킹으로 분류.
* **매직 바이트**: `.dat`, `.jpg` 등 비실행형 확장자이나 `MZ` + `PE\0\0` 헤더 존재 시 `IsDisguisedExecutable = true`

#### E. 입출력 인터페이스 규격

```csharp
namespace Phalanx.Cockpit.Tools;

public sealed class FileInspectionTool : IInvestigationTool
{
    public string Name => "FileInspectionTool";
    public string Description => 
        "디스크 상의 파일 경로, 디지털 서명(Authenticode), 시스템 파일 위장(Masquerading), " +
        "PE 헤더 정합성, 엔트로피를 정밀 검증합니다. 인자: { \"filePath\": \"C:\\\\...\" }";

    public async Task<ToolResult> ExecuteAsync(Dictionary<string, object> parameters) { ... }

    public sealed class Input
    {
        [JsonPropertyName("filePath")]
        public string FilePath { get; set; } = string.Empty;
    }

    public sealed class Output
    {
        public bool Exists { get; set; }
        public string NormalizedPath { get; set; } = string.Empty;
        public long FileSizeBytes { get; set; }
        public string Sha256 { get; set; } = string.Empty;
        public double Entropy { get; set; }
        public bool IsSigned { get; set; }
        public string SignerSubject { get; set; } = string.Empty;
        public string SignatureStatus { get; set; } = string.Empty;
        public bool IsPathMasqueraded { get; set; }
        public bool IsDisguisedExecutable { get; set; }
        public int AnomalyScore { get; set; } // 0 ~ 100
        public string DiagnosticReason { get; set; } = string.Empty;
    }
}
```

#### F. DI 등록 및 프롬프트 통합

* **DI**: `Program.cs`에서 `builder.Services.AddSingleton<IInvestigationTool, FileInspectionTool>();`
* **프롬프트 수사 지침**: *"페이로드 해독이나 명령행에서 다운로드/생성되는 로컬 파일 경로가 포착되면, 메모리 스캔보다 먼저 FileInspectionTool을 호출하여 디지털 서명 및 시스템 경로 위장(Masquerading) 여부를 우선 확증하십시오."*

#### G. 기대 효과 (가설)

* 기존: 5턴, 36.5초 (Turn 3에서 메모리 스캔 21.7초 낭비)
* 개선: Turn 2에서 `FileInspectionTool`로 Temp 내 svchost 무서명/위장 확증(0.05초) -> **총 2~3턴, 10초 내외 종결 (72% 단축)**

---

## 9. 후속 실험 과제 (Future Empirical Research)

1. **FileInspectionTool 주입 후 복합 회피 공격 수사 압축률 검증**: 동일 T1036.005 시나리오에서 턴 수 및 레이턴시 단축 효과 실측.
2. **LOLBAS 듀얼 위장 분별 실험**: 정품 서명 바이너리 + 악성 인자 vs 가짜 바이너리 교체/위장의 이중 분별력 검증.
3. **다중 프로세스 상속 체인 동시 동결/수사 확장성 실험**: 트리형 공격에서 동시 다중 동결 및 상속 체인 일괄 처분 검증.
4. **고부하 텔레메트리 gRPC 스트리밍 I/O 병목 실측**: 초당 10,000건 이상 이벤트 폭주 시 이벤트 드롭률 및 메모리 풋프린트 측정.
