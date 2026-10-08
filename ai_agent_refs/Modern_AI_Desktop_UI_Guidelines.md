---
description: WPF 기반 차세대 데스크톱 AI 및 엔터프라이즈 도구 UI/UX 설계 지침서. 다크 테마 디자인 토큰, 윈도우 크롬 일체화, AI 추론 서사 및 실시간 스트리밍.
related:
  - ../README.md
  - ./WPF_Architecture_Guidelines.md
---
# Modern AI Desktop UI Guidelines

본 문서는 고성능 데스크톱 환경(WPF/.NET 8.0 이상)에서 **엔터프라이즈 프로 도구(Enterprise Pro-Tool)**와 **추론형 AI 에이전트의 실시간 서사(Agentic Narrative)**를 결합할 때 준수해야 하는 공식 UI/UX 디자인 시스템 및 XAML 구현 지침서입니다.

실시간 보안 관제, 대규모 분석 워크벤치, 대화형 AI 도구 등 고성능 엔터프라이즈 환경에서 실증 검증된 인터페이스 설계 패턴과 최신 Windows 11 Fluent 및 다크 테마 트렌드를 종합하여 완전한 프로덕션 표준을 정의합니다. 본 문서는 초기 프로젝트 구축 및 인터페이스 설계 시 한 번에 완결된 품질을 뽑아낼 수 있도록 모든 핵심 스니펫과 아키텍처를 생략 없이 기술합니다.

---

## 1. 디자인 철학 및 데스크톱 AI 상호작용 패러다임

현대 데스크톱 UI는 장식용 시각 요소(화려한 그라디언트, 무거운 블러 그림자, 이모지 남발)를 배제하고, **고밀도 정보 전달과 즉각적인 시스템 반응성을 보장하는 프로 도구 미학(Pro-Tool High-Performance Aesthetic)**을 지향합니다.

* **네이티브 하드웨어 가속 우위**: 웹 브라우저 엔진(Electron)의 300MB~1GB 메모리 풋프린트를 배제하고, DirectX 하드웨어 가속 기반의 가벼운 30~80MB 풋프린트와 60fps 무프리징 렌더링을 보장합니다.
* **설명 가능성과 서사의 조화**: AI가 내린 판단의 과정(Thought), 호출한 도구(Action), 수집한 관측 데이터(Observation), 최종 판단(Verdict)을 시각적으로 체계화하여 사용자에게 완전한 투명성을 제공합니다.

### 2대 핵심 상호작용 패러다임

| 구분 | 패러다임 A: 대화형 코파일럿 워크벤치 | 패러다임 B: 자율형 에이전트 관제 루프 |
| :--- | :--- | :--- |
| **주요 목적** | 다회차 질의응답, 문서/코드 생성, 분석 보조 | 자율적 이상 탐지, 포렌식 조사 연쇄 호출, 프로세스 능동 제어 |
| **상호작용 주체** | 인간 중심 (Human-in-the-Loop 요청에 AI가 응답) | 에이전트 중심 (에이전트가 자율 조사 후 인간에게 승인 요청) |
| **핵심 UI 구성** | 멀티턴 대화 버블, 코드 블록, 프롬프트 입력창, 멀티모달 첨부 | ReAct 루프 타임라인, 도구 호출 피드, 터미널 스트림, 상태 뱃지 |
| **실시간성 요구** | LLM 토큰 단위 점진적 스트리밍 텍스트 렌더링 | 초당 수천 건 원시 텔레메트리 배치 버퍼링 및 프로젝션 |

---

## 2. 디자인 토큰 시스템 (Design Tokens Specification)

### A. Dark-First 컬러 팔레트 (`Themes/Tokens.xaml`)
순수 블랙(`black/#000000`)의 과도한 대비로 인한 눈의 피로를 방지하고, 딥 징크/슬레이트 계열의 무채색을 기본 캔버스로 사용합니다.

| 토큰명 (Token) | 헥사 코드 (Hex) | 용도 및 설명 |
| :--- | :--- | :--- |
| `BgBase` | `#0B0C10` | 최하단 윈도우 기본 캔버스 배경색 |
| `BgSurface` | `#13141C` | 사이드바, 좌측 네비게이션 레일, 상단 헤더 배경색 |
| `BgCard` | `#1A1D27` | 개별 카드, 분석 노드 블록, 대화 버블, 입력창 배경색 |
| `BgCardElevated`| `#232736` | 팝오버, 드롭다운 메뉴, 인앱 모달 컨테이너 배경색 |
| `BorderSubtle` | `#22FFFFFF` (13% White) | 주요 컨테이너 및 컨트롤의 1px 외곽선 |
| `BorderUltraSubtle` | `#0FFFFFFF` (6% White) | 카드 내부 서브 디바이더, 테이블 그리드 라인 |
| `TextPrimary` | `#F2F4F8` | 메인 헤더, 수치 데이터, 강조 제목 (고대비) |
| `TextSecondary`| `#AAB2C0` | 본문 서사 텍스트, 설명 레이블, 보조 설명 |
| `TextMuted` | `#6B7280` | 타임스탬프, 비활성 텍스트, 보조 메타데이터 |
| `TextThought` | `#A2B9D8` | AI 에이전트 내면 사고(Thought) 전용 슬레이트 블루 |
| `AccentPrimary` | `#4F46E5` (Indigo) | 주요 분석 실행 버튼, 선택된 탭 활성 인디케이터 |
| `AccentDanger` | `#EF4444` (Crimson) | 프로세스 동결/종료, 치명적 악성 위협 뱃지 |
| `AccentWarning`| `#F59E0B` (Amber) | 조사 진행 중, 의심 행동, 로컬 폴백 모드 알림 |
| `AccentSuccess`| `#10B981` (Emerald) | 안전 검증 완료, 서명 일치, 정상 상태 뱃지 |

### B. 루미넌스 스태킹 (Luminance Stacking)
다크 UI에서 성능을 저하시키는 큰 반경의 블러 그림자(Drop Shadow)를 배제하고, Z-Index 계층이 높아질수록 미세하게 명도를 높여 물리적 깊이감을 부여합니다.

```
[Layer 0: Window Background]  ➔  #0B0C10
    └── [Layer 1: Surface Rail]   ➔  #13141C (Border: 1px #22FFFFFF)
            └── [Layer 2: Content Card]  ➔  #1A1D27 (Border: 1px #22FFFFFF)
                    └── [Layer 3: Popover/Modal] ➔  #232736 (Border: 1px #33FFFFFF)
```

### C. 타이포그래피 계층
* **UI 기본 폰트**: `Segoe UI Variable Text`, `Pretendard`, `Malgun Gothic` (가독성과 자간 최적화).
* **코드 및 텔레메트리 폰트**: `Cascadia Code`, `Consolas` (고정폭 폰트 강제).
* **크기 및 줄간격 표준**:
  * 대형 헤더: `20px` (LineHeight: `28px`, Bold)
  * 카드 타이틀: `15px` (LineHeight: `22px`, SemiBold)
  * 본문 서사: `13.5px` (LineHeight: `22px`, Regular)
  * AI 내면 추론: `12.5px` (LineHeight: `20px`, Italic)
  * 메타데이터/뱃지: `11px` (LineHeight: `16px`, Medium)

### D. 제로 데코레이티브 이모지 정책
플랫폼별 렌더링 파편화와 프로 도구의 진중함을 저해하는 장식용 이모지 사용을 엄격히 금지합니다. 모든 시각 아이콘은 XAML `Path` 벡터 지오메트리(`F1 M ...`) 또는 Segoe Fluent Icons 글리프로 대체합니다.

---

## 3. 고밀도 3단 분할 워크벤치 레이아웃 (High-Density 3-Pane Architecture)

단순 2단 분할 시 발생하기 쉬운 화면 중앙/하단의 휑한 빈 공간을 방지하고, VS Code, Linear, Obsidian 스타일의 **고밀도 프로페셔널 정보 구조**를 구축하기 위해 `GridSplitter` 기반의 3단 분할 레이아웃을 표준화합니다.

```
┌────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ [≡] ENTERPRISE WORKBENCH       Active Session: Production Cluster          [—] [☐] [✕] (Caption)   │
├───────────────┬─┬──────────────────────────────────────────┬─┬─────────────────────────────────────┤
│ Pane 1 (Left) │G│ Pane 2 (Center)                          │G│ Pane 3 (Right)                      │
│ 텔레메트리     │r│ 메인 토폴로지 / 작업 캔버스              │r│ AI 사고 서사 & ReAct 피드           │
│ / 실시간 리스트│i│ • 가상화 노드 그래프                     │i│ • CoT <think> Expander              │
│ (Width="320") │d│ • 심층 메트릭 차트                       │d│ • Tool Call Timeline Card          │
│               │ │ • 핵심 데이터 그리드                     │ │ • Streaming Narrative Card          │
│               │S│                                          │S│ • Terminal stdout 접이식 뷰어       │
│               │p│                                          │p│                                     │
│               │l│                                          │l│                                     │
│ (MinWidth=240)│ │ (Width="*")                              │ │ (Width="380", MinWidth=300)         │
└───────────────┴─┴──────────────────────────────────────────┴─┴─────────────────────────────────────┘
```

### XAML 3단 분할 뼈대 구현

```xml
<Grid Margin="12">
    <Grid.ColumnDefinitions>
        <!-- Pane 1: 좌측 텔레메트리 / 리스트 -->
        <ColumnDefinition Width="320" MinWidth="240" />
        <!-- 스플리터 1 -->
        <ColumnDefinition Width="6" />
        <!-- Pane 2: 중앙 주 작업 캔버스 -->
        <ColumnDefinition Width="*" MinWidth="360" />
        <!-- 스플리터 2 -->
        <ColumnDefinition Width="6" />
        <!-- Pane 3: 우측 AI 사고 및 ReAct 피드 -->
        <ColumnDefinition Width="380" MinWidth="300" />
    </Grid.ColumnDefinitions>

    <!-- Pane 1: 가상화 텔레메트리 리스트 -->
    <Border Grid.Column="0" Background="#13141C" BorderBrush="#22FFFFFF" BorderThickness="1" CornerRadius="8">
        <!-- ListView 가상화 컨테이너 -->
    </Border>

    <!-- 스플리터 1 -->
    <GridSplitter Grid.Column="1" Width="6" HorizontalAlignment="Center" Background="Transparent" Cursor="SizeWE" />

    <!-- Pane 2: 중앙 메인 작업 영역 -->
    <Border Grid.Column="2" Background="#13141C" BorderBrush="#22FFFFFF" BorderThickness="1" CornerRadius="8">
        <!-- 노드 그래프, 차트, 데이터그리드 -->
    </Border>

    <!-- 스플리터 2 -->
    <GridSplitter Grid.Column="3" Width="6" HorizontalAlignment="Center" Background="Transparent" Cursor="SizeWE" />

    <!-- Pane 3: 우측 AI ReAct 피드 -->
    <Border Grid.Column="4" Background="#13141C" BorderBrush="#22FFFFFF" BorderThickness="1" CornerRadius="8">
        <!-- CoT Expander, ReAct Tool Card, Narrative -->
    </Border>
</Grid>
```

### 패널 너비 상태 영속화 패턴 (Pane Width State Persistence)

`GridSplitter`를 사용자가 조절하더라도 앱을 다시 실행하거나 화면을 전환할 때 초기값(320px, 380px)으로 리셋되면 심각한 UX 피로를 유발합니다. 

> [!CAUTION]
> **[WPF 런타임 함정: `ColumnDefinition.Width` TwoWay 바인딩 파괴]**  
> XAML에서 `<ColumnDefinition Width="{Binding LeftPaneWidth, Mode=TwoWay}" />`로 선언하는 것은 동작하지 않는 안티패턴입니다. WPF에서 사용자가 `GridSplitter`를 마우스로 드래그하면 내부적으로 `ColumnDefinition.Width = new GridLength(...)`로 **로컬 값(Local Value)을 직접 할당**합니다. WPF 의존성 속성 우선순위 원칙상 로컬 값이 셋팅되면 기존 `BindingExpression`이 즉시 덮어씌워져 영구 파괴(Broken)되므로, 드래그 후 바인딩이 완전히 끊어집니다.

따라서 상용 엔터프라이즈 도구(PowerToys, ScreenToGif, Fork)와 동일하게 **`GridSplitter.DragCompleted` 이벤트와 View `Loaded` 이벤트를 통해 안전하게 너비를 저장/복원하는 표준 패턴**을 사용합니다.

#### XAML 구현 (`Views/WorkbenchView.xaml`)
```xml
<Grid Margin="12" Loaded="OnViewLoaded">
    <Grid.ColumnDefinitions>
        <!-- Pane 1: 좌측 텔레메트리 (이름 지정) -->
        <ColumnDefinition x:Name="LeftPaneColumn" Width="320" MinWidth="240" />
        <!-- 스플리터 1 (DragCompleted 이벤트 연결) -->
        <ColumnDefinition Width="6" />
        <!-- Pane 2: 중앙 주 작업 캔버스 (가변 폭) -->
        <ColumnDefinition Width="*" MinWidth="360" />
        <!-- 스플리터 2 (DragCompleted 이벤트 연결) -->
        <ColumnDefinition Width="6" />
        <!-- Pane 3: 우측 AI 사고 피드 (이름 지정) -->
        <ColumnDefinition x:Name="RightPaneColumn" Width="380" MinWidth="300" />
    </Grid.ColumnDefinitions>

    <!-- 스플리터 1 -->
    <GridSplitter Grid.Column="1" Width="6" HorizontalAlignment="Center"
                  Background="Transparent" Cursor="SizeWE"
                  DragCompleted="OnSplitterDragCompleted" />

    <!-- 스플리터 2 -->
    <GridSplitter Grid.Column="3" Width="6" HorizontalAlignment="Center"
                  Background="Transparent" Cursor="SizeWE"
                  DragCompleted="OnSplitterDragCompleted" />
</Grid>
```

#### View 코드-비하인드 (`Views/WorkbenchView.xaml.cs`)
```csharp
private void OnViewLoaded(object sender, RoutedEventArgs e)
{
    // 화면 진입 시 영속화된 설정값으로 너비 복원
    if (DataContext is WorkbenchViewModel vm)
    {
        LeftPaneColumn.Width = new GridLength(vm.LeftPaneWidth);
        RightPaneColumn.Width = new GridLength(vm.RightPaneWidth);
    }
}

private void OnSplitterDragCompleted(object sender, DragCompletedEventArgs e)
{
    // 사용자가 드래그를 완료한 시점에만 ViewModel로 실제 픽셀 너비 안전 저장
    if (DataContext is WorkbenchViewModel vm)
    {
        vm.SavePaneWidths(LeftPaneColumn.ActualWidth, RightPaneColumn.ActualWidth);
    }
}
```

#### ViewModel 구현 (`ViewModels/WorkbenchViewModel.cs`)
```csharp
public partial class WorkbenchViewModel : ObservableObject
{
    private readonly ISettingsService _settingsService;

    public double LeftPaneWidth { get; private set; }
    public double RightPaneWidth { get; private set; }

    public WorkbenchViewModel(ISettingsService settingsService)
    {
        _settingsService = settingsService;
        LeftPaneWidth = _settingsService.Get("LeftPaneWidth", 320.0);
        RightPaneWidth = _settingsService.Get("RightPaneWidth", 380.0);
    }

    public void SavePaneWidths(double leftWidth, double rightWidth)
    {
        LeftPaneWidth = leftWidth;
        RightPaneWidth = rightWidth;
        _settingsService.Set("LeftPaneWidth", leftWidth);
        _settingsService.Set("RightPaneWidth", rightWidth);
    }
}
```

---

## 4. 실시간 AI 사고(Thinking) 및 스트리밍 서사(Narrative) UI

현대 추론형 AI 모델(DeepSeek-R1, OpenAI o1/o3, Gemini 2.0 Flash Thinking)의 확장 사고 패러다임에 맞추어, **내부 추론 과정(`<think>`)과 최종 분석 서사(Narrative)를 시각적으로 엄격히 분리**합니다.

```
┌──────────────────────────────────────────────────────────────┐
│ ▾  AI 사고 과정 (Thinking Process - 1.8s, 420 tokens)        │  <- Expander
│   ├─ 프로세스 CommandLine 문자열에서 Base64 패턴 감지...     │  <- TextThought (#A2B9D8)
│   ├─ 리소스 격리 규칙 및 Watchdog 안전성 검토...             │
│   └─ 신뢰 서명 불일치 확인. 위험도 85 산출.                   │
└──────────────────────────────────────────────────────────────┘
│
[최종 분석 서사]
대상 리소스는 정상 상태를 벗어나 메모리 누수를 일으키고 있습니다.
격리 조치를 위해 승인 인터락을 요청합니다.
```

### 무프리징 토큰 증분 렌더링 파이프라인 (`StreamingNarrativeBuffer.cs`)

LLM 토큰이 초당 50~100개씩 쏟아질 때 전체 문자열을 매번 다시 바인딩하면 렌더 트리 재구성으로 인해 UI 스레드가 마비됩니다. 채널 기반 버퍼링과 증분 렌더링을 적용해야 합니다.

```csharp
using System;
using System.Text;
using System.Threading;
using System.Threading.Channels;
using System.Threading.Tasks;
using System.Windows;
using System.Windows.Threading;

namespace MyWpfApp.Services.Implementations;

public sealed class StreamingNarrativeBuffer : IDisposable
{
    private readonly Channel<string> _tokenChannel = Channel.CreateUnbounded<string>();
    private readonly Action<string> _appendAction;
    private readonly CancellationTokenSource _cts = new();

    public StreamingNarrativeBuffer(Action<string> appendAction)
    {
        _appendAction = appendAction;
        _ = ProcessTokenQueueAsync(_cts.Token);
    }

    public void PushToken(string token)
    {
        _tokenChannel.Writer.TryWrite(token);
    }

    private async Task ProcessTokenQueueAsync(CancellationToken cancellationToken)
    {
        var reader = _tokenChannel.Reader;
        var sb = new StringBuilder();

        try
        {
            while (await reader.WaitToReadAsync(cancellationToken))
            {
                sb.Clear();
                // 채널에 쌓인 토큰을 한 번에 비워서 일괄 갱신
                while (reader.TryRead(out var token))
                {
                    sb.Append(token);
                }

                var chunk = sb.ToString();
                if (!string.IsNullOrEmpty(chunk))
                {
                    await Application.Current.Dispatcher.InvokeAsync(
                        () => _appendAction(chunk),
                        DispatcherPriority.Render);
                }
            }
        }
        catch (OperationCanceledException)
        {
            // 화면 이탈 및 Dispose 시 안전하게 백그라운드 태스크 종료
        }
    }

    public void Dispose()
    {
        _tokenChannel.Writer.TryComplete();
        _cts.Cancel();
        _cts.Dispose();
    }
}

// ViewModel(ICleanup 구현체)에서의 수명주기 해제 연동
public partial class InvestigationViewModel : ObservableObject, ICleanup
{
    private readonly StreamingNarrativeBuffer _narrativeBuffer;

    public void Cleanup()
    {
        // 화면 전환 또는 파괴 시 백그라운드 토큰 루프 완전 해제 (메모리/스레드 누수 원천 차단)
        _narrativeBuffer.Dispose();
    }
}
```

---

## 5. 자율형 에이전트 ReAct 루프 및 도구 호출 카드 XAML 스타일

자율 에이전트가 판단(Thought)하고 도구를 호출(Action)하며 결과(Observation)를 확인하는 전 과정을 타임라인 카드 형태로 시각화합니다.

```xml
<!-- ReAct 도구 호출 타임라인 카드 -->
<Border Background="#1A1D27" BorderBrush="#22FFFFFF" BorderThickness="1" CornerRadius="8" Padding="14" Margin="0,0,0,12">
    <StackPanel>
        <!-- 도구 헤더 (이름 뱃지 + 실행 상태 인디케이터) -->
        <Grid Margin="0,0,0,10">
            <StackPanel Orientation="Horizontal" VerticalAlignment="Center">
                <Border Background="#4F46E5" CornerRadius="4" Padding="6,2" Margin="0,0,8,0">
                    <TextBlock Text="TOOL CALL" Foreground="#FFFFFF" FontWeight="Bold" FontSize="10" />
                </Border>
                <TextBlock Text="InspectResourceMemory" Foreground="#F2F4F8" FontWeight="SemiBold" FontSize="13" />
            </StackPanel>
            <!-- 성공 상태 칩 -->
            <Border HorizontalAlignment="Right" Background="#10B981" CornerRadius="10" Padding="8,2">
                <TextBlock Text="COMPLETED" Foreground="#FFFFFF" FontWeight="Bold" FontSize="10" />
            </Border>
        </Grid>

        <!-- 파라미터 JSON 아코디언 -->
        <Expander Header="전달 파라미터" Foreground="#AAB2C0" FontSize="11" Margin="0,0,0,8">
            <Border Background="#13141C" CornerRadius="4" Padding="8" Margin="0,4,0,0">
                <TextBlock Text="{}{ &quot;targetId&quot;: 4920, &quot;depth&quot;: &quot;deep&quot; }"
                           FontFamily="Cascadia Code, Consolas" FontSize="11" Foreground="#A2B9D8" />
            </Border>
        </Expander>

        <!-- 터미널 표준 출력 (stdout) 뷰어 -->
        <Border Background="#0B0C10" BorderBrush="#1AFFFFFF" BorderThickness="1" CornerRadius="4" Padding="10">
            <ScrollViewer MaxHeight="120" VerticalScrollBarVisibility="Auto">
                <TextBlock Text="[INFO] Reading virtual memory pages...&#x0a;[WARN] Suspicious unbacked executable memory detected at 0x7FFA2000"
                           FontFamily="Cascadia Code, Consolas" FontSize="11" Foreground="#10B981" TextWrapping="Wrap" />
            </ScrollViewer>
        </Border>
    </StackPanel>
</Border>
```

---

## 6. 윈도우 크롬 일체화 및 Windows 11 Fluent 시스템 통합

앱 상단 바와 윈도우 타이틀바를 일체화(Seamless)하여 상단 세로 공간을 32px 절약하고, Windows 11 네이티브 스냅 레이아웃(Snap Assist)과 심층 다크 모드, 그리고 **다중 모니터 혼합 DPI 스케일링**을 완벽히 지원합니다.

### A. WindowChrome 기본 선언 (`MainWindow.xaml`)

```xml
<Window x:Class="MyWpfApp.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:shell="clr-namespace:System.Windows.Shell;assembly=PresentationFramework"
        Title="Enterprise AI Workbench"
        Height="800" Width="1300"
        Background="Transparent"
        WindowStartupLocation="CenterScreen">
        <!-- 주의: Windows 11 Mica 백드롭을 적용할 때는 Window.Background="Transparent"로 설정하고,
             루트 컨테이너(Grid/Border)에 반투명 틴트(#E60B0C10)를 지정하여 가독성과 글래스 질감을 양립합니다. -->

    <!-- 윈도우 크롬 일체화 및 DWM GlassFrame 전체 확장 (Mica 투과를 위해 GlassFrameThickness="-1" 필수) -->
    <shell:WindowChrome.WindowChrome>
        <shell:WindowChrome CaptionHeight="44"
                            CornerRadius="0"
                            GlassFrameThickness="-1"
                            NonClientFrameEdges="None"
                            ResizeBorderThickness="6"
                            UseAeroCaptionButtons="False" />
    </shell:WindowChrome.WindowChrome>

    <!-- 최대화 시 모니터 경계 7px 잘림(Overshoot) 방지 기본 마진 -->
    <Window.Style>
        <Style TargetType="Window">
            <Setter Property="Padding" Value="0" />
            <Style.Triggers>
                <Trigger Property="WindowState" Value="Maximized">
                    <Setter Property="Padding" Value="7" />
                </Trigger>
            </Style.Triggers>
        </Style>
    </Window.Style>

    <!-- 반투명 다크 틴트 레이어 (Mica 질감 투과 + 텍스트 가독성 확보) -->
    <Grid Background="#E60B0C10">
        <Grid.RowDefinitions>
            <RowDefinition Height="44" />
            <RowDefinition Height="*" />
        </Grid.RowDefinitions>

        <!-- 타이틀바 컨테이너 -->
        <Border Grid.Row="0" Background="#CC13141C" BorderBrush="#1AFFFFFF" BorderThickness="0,0,0,1">
            <Grid Margin="16,0">
                <Grid.ColumnDefinitions>
                    <ColumnDefinition Width="Auto" />
                    <ColumnDefinition Width="*" />
                    <ColumnDefinition Width="Auto" />
                </Grid.ColumnDefinitions>

                <!-- 좌측 앱 브랜딩 -->
                <TextBlock Grid.Column="0" Text="ENTERPRISE PRO WORKBENCH"
                           Foreground="#F2F4F8" FontWeight="Bold" FontSize="13"
                           VerticalAlignment="Center" />

                <!-- 중앙 검색/상태 바 (드래그 가능 영역) -->
                <TextBlock Grid.Column="1" HorizontalAlignment="Center" VerticalAlignment="Center"
                           Text="Autonomous ReAct Active" Foreground="#AAB2C0" FontSize="11" />

                <!-- 우측 창 제어 버튼 (클릭 관통 방지 필수) -->
                <StackPanel Grid.Column="2" Orientation="Horizontal"
                            shell:WindowChrome.IsHitTestVisibleInChrome="True">
                    <Button Width="40" Height="44" Content="—" Click="Minimize_Click" Style="{StaticResource ChromeCaptionButtonStyle}" />
                    <Button x:Name="MaximizeButton" Width="40" Height="44" Content="☐" Click="Maximize_Click" Style="{StaticResource ChromeCaptionButtonStyle}" />
                    <Button Width="40" Height="44" Content="✕" Click="Close_Click" Style="{StaticResource ChromeCloseButtonStyle}" />
                </StackPanel>
            </Grid>
        </Border>

        <!-- 본문 작업 영역 -->
        <ContentControl Grid.Row="1" Content="{Binding CurrentView}" />
    </Grid>
</Window>
```

### B. Windows 11 Snap Layouts & 다중 모니터 혼합 DPI 완결 P/Invoke (`MainWindow.xaml.cs`)

사용자가 최대화 버튼 위에 마우스를 올렸을 때 Windows 11의 Snap Layout 그리드가 나타나게 하려면 `WM_NCHITTEST`를 가로채어 `HTMAXBUTTON (9)`를 반환해야 하며, 서로 다른 배율의 다중 모니터 환경에서 정밀한 크기 조절을 지원하기 위해 `WM_GETMINMAXINFO`를 함께 처리합니다.

```csharp
using System;
using System.Runtime.InteropServices;
using System.Windows;
using System.Windows.Interop;

namespace MyWpfApp;

public partial class MainWindow : Window
{
    private const int WM_NCHITTEST = 0x0084;
    private const int WM_GETMINMAXINFO = 0x0024;
    private const int HTMAXBUTTON = 9;

    private const int DWMWA_USE_IMMERSIVE_DARK_MODE = 20;
    private const int DWMWA_WINDOW_CORNER_PREFERENCE = 33;
    private const int DWMWCP_ROUND = 2;

    // Windows 11 22H2 (Build 22621)+ 시스템 백드롭 상수
    private const int DWMWA_SYSTEMBACKDROP_TYPE = 38;

    public enum DWM_SYSTEMBACKDROP_TYPE
    {
        DWMSBT_AUTO = 0,
        DWMSBT_NONE = 1,
        DWMSBT_MAINWINDOW = 2,      // Mica (기본 은은한 투과)
        DWMSBT_TRANSIENTWINDOW = 3,  // Acrylic (블러 강조 반투명)
        DWMSBT_TABBEDWINDOW = 4      // Mica Alt (탭 윈도우용 고대비)
    }

    [DllImport("dwmapi.dll")]
    private static extern int DwmSetWindowAttribute(IntPtr hwnd, int attr, ref int attrValue, int attrSize);

    [StructLayout(LayoutKind.Sequential)]
    public struct POINT { public int x; public int y; }

    [StructLayout(LayoutKind.Sequential)]
    public struct MINMAXINFO
    {
        public POINT ptReserved;
        public POINT ptMaxSize;
        public POINT ptMaxPosition;
        public POINT ptMinTrackSize;
        public POINT ptMaxTrackSize;
    }

    public MainWindow()
    {
        InitializeComponent();
        Loaded += MainWindow_Loaded;
    }

    private void MainWindow_Loaded(object sender, RoutedEventArgs e)
    {
        var hwnd = new WindowInteropHelper(this).Handle;

        // 1. DWM Immersive Dark Mode 활성화 (시스템 우클릭 메뉴 및 창 외곽 다크화)
        int darkMode = 1;
        DwmSetWindowAttribute(hwnd, DWMWA_USE_IMMERSIVE_DARK_MODE, ref darkMode, sizeof(int));

        // 2. Windows 11 둥근 모서리 강제
        int cornerPreference = DWMWCP_ROUND;
        DwmSetWindowAttribute(hwnd, DWMWA_WINDOW_CORNER_PREFERENCE, ref cornerPreference, sizeof(int));

        // 3. Windows 11 Mica 백드롭 활성화 (Window.Background="Transparent" 필수)
        int backdropType = (int)DWM_SYSTEMBACKDROP_TYPE.DWMSBT_MAINWINDOW;
        DwmSetWindowAttribute(hwnd, DWMWA_SYSTEMBACKDROP_TYPE, ref backdropType, sizeof(int));

        // 4. 메시지 훅 추가
        var hwndSource = HwndSource.FromHwnd(hwnd);
        hwndSource?.AddHook(WndProc);
    }

    private IntPtr WndProc(IntPtr hwnd, int msg, IntPtr wParam, IntPtr lParam, ref bool handled)
    {
        switch (msg)
        {
            case WM_NCHITTEST:
                // 화면 좌표 추출
                int x = lParam.ToInt32() & 0xffff;
                int y = lParam.ToInt32() >> 16;

                // 최대화 버튼 호버 감지 -> Windows 11 Snap Layouts 플라이아웃 활성화
                if (MaximizeButton.IsLoaded)
                {
                    var buttonPos = MaximizeButton.PointToScreen(new Point(0, 0));
                    var buttonRect = new Rect(buttonPos.X, buttonPos.Y, MaximizeButton.ActualWidth, MaximizeButton.ActualHeight);

                    if (buttonRect.Contains(new Point(x, y)))
                    {
                        handled = true;
                        return new IntPtr(HTMAXBUTTON);
                    }
                }
                break;
        }
        return IntPtr.Zero;
    }

    private void Minimize_Click(object sender, RoutedEventArgs e) => SystemCommands.MinimizeWindow(this);
    private void Maximize_Click(object sender, RoutedEventArgs e)
    {
        if (WindowState == WindowState.Maximized)
            SystemCommands.RestoreWindow(this);
        else
            SystemCommands.MaximizeWindow(this);
    }
    private void Close_Click(object sender, RoutedEventArgs e) => SystemCommands.CloseWindow(this);
}
```

### C. Windows 11 Mica / Acrylic 시스템 백드롭(Backdrop) 시스템 통합

Windows 11 22H2(Build 22621)부터 공식 지원되는 `DWMWA_SYSTEMBACKDROP_TYPE (38)` API를 사용하면 별도의 외부 무거운 그래픽 라이브러리 없이 네이티브 OS 레벨에서 하드웨어 가속되는 Mica/Acrylic 텍스처를 구현할 수 있습니다.

1. **백드롭 타입 선택 기준**:
   * `DWMSBT_MAINWINDOW (2) - Mica`: 메인 앱 프레임워크 표준. 데스크톱 배경화면 색상이 윈도우 뒤로 은은하게 반사되어 일체감을 형성합니다.
   * `DWMSBT_TRANSIENTWINDOW (3) - Acrylic`: 일시적인 모달 다이얼로그나 드롭다운 플라이아웃용. 블러와 반투명도가 높아 배경 시각 정보가 부드럽게 흐려집니다.
   * `DWMSBT_TABBEDWINDOW (4) - Mica Alt`: 다중 탭 문서 편집기나 복합 브라우징 인터페이스용. 배경 투과율이 Mica보다 낮아 탭 간 시각적 구분이 명확합니다.

2. **XAML 반투명 틴팅(Tinting) 설계 원칙**:
   * **원칙 1 (완전 투명 금지)**: 백드롭 효과를 낸다고 내부 레이어까지 100% 투명하게 두면, 바탕화면 아이콘이나 다른 창의 글자가 겹쳐 심각한 가독성 저하를 초래합니다.
   * **원칙 2 (반투명 틴트 브러시 적용)**: 최상위 `Window.Background="Transparent"`를 지정한 후, 루트 레이아웃 컨테이너(`Grid`)의 배경색을 `#E60B0C10` (불투명도 약 90%)으로 설정합니다. 이를 통해 텍스트 명암비를 100% 확보하면서 창 모서리와 타이틀바 주변으로 은은한 OS 네이티브 글래스 깊이감을 자연스럽게 투과시킵니다.
   * **원칙 3 (Windows 10 하위 호환)**: Windows 10 또는 구형 빌드에서는 `DWMWA_SYSTEMBACKDROP_TYPE` 호출이 안전하게 무시되며, 루트 `Grid`의 `#E60B0C10` 브러시가 진한 솔리드 다크 테마 배경으로 자연스럽게 작동(Graceful Fallback)합니다.

---

## 7. 신뢰성, 안전성 및 폴백(Fallback) UI 패턴

클라우드 AI 서비스 장애나 네트워크 두절 시 사용자가 시스템 상태를 즉시 인지할 수 있도록 가시적 피드백을 제공합니다.

### A. 오프라인 로컬 결정론적 폴백 배너 (Offline Local Deterministic Fallback)
클라우드 LLM/AI 서비스 응답 실패 시 지연 없이 즉각 로컬 결정론적 규칙/캐시 엔진으로 안전 전환됨을 알리는 상단 고정 상태 배너:

```xml
<Border Background="#7F3B00" BorderBrush="#F59E0B" BorderThickness="0,0,0,1"
        Padding="16,8" Visibility="{Binding IsOfflineFallbackActive, Converter={StaticResource BoolToVis}}">
    <Grid>
        <StackPanel Orientation="Horizontal" HorizontalAlignment="Center">
            <Border Width="8" Height="8" Background="#F59E0B" CornerRadius="4" VerticalAlignment="Center" Margin="0,0,8,0" />
            <TextBlock Text="LOCAL FALLBACK ACTIVE: 클라우드 AI 서비스 연결 불가로 로컬 결정론적 규칙 엔진으로 전환되었습니다."
                       Foreground="#FDE68A" FontWeight="SemiBold" FontSize="12" />
        </StackPanel>
    </Grid>
</Border>
```

---

## 8. XAML 표준 스타일 템플릿 카탈로그 (`Themes/ControlStyles.xaml`)

프로젝트 전반에 즉시 포함하여 사용할 수 있는 완결된 템플릿 리소스 딕셔너리입니다.

```xml
<ResourceDictionary xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
                    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">

    <!-- 1. 캡슐형 주 액션 버튼 -->
    <Style x:Key="CapsulePrimaryButtonStyle" TargetType="Button">
        <Setter Property="Height" Value="40" />
        <Setter Property="MinWidth" Value="120" />
        <Setter Property="Padding" Value="20,0" />
        <Setter Property="Foreground" Value="#FFFFFF" />
        <Setter Property="FontWeight" Value="SemiBold" />
        <Setter Property="FontSize" Value="13" />
        <Setter Property="Cursor" Value="Hand" />
        <Setter Property="Template">
            <Setter.Value>
                <ControlTemplate TargetType="Button">
                    <Border x:Name="border"
                            Background="#4F46E5"
                            BorderBrush="#6366F1"
                            BorderThickness="1"
                            CornerRadius="20"
                            SnapsToDevicePixels="True">
                        <ContentPresenter HorizontalAlignment="Center" VerticalAlignment="Center" />
                    </Border>
                    <ControlTemplate.Triggers>
                        <Trigger Property="IsMouseOver" Value="True">
                            <Setter TargetName="border" Property="Background" Value="#4338CA" />
                        </Trigger>
                        <Trigger Property="IsPressed" Value="True">
                            <Setter TargetName="border" Property="Background" Value="#3730A3" />
                        </Trigger>
                        <Trigger Property="IsEnabled" Value="False">
                            <Setter TargetName="border" Property="Opacity" Value="0.4" />
                        </Trigger>
                    </ControlTemplate.Triggers>
                </ControlTemplate>
            </Setter.Value>
        </Setter>
    </Style>

    <!-- 2. AI 내면 사고 접이식 Expander -->
    <Style x:Key="ThoughtExpanderStyle" TargetType="Expander">
        <Setter Property="Background" Value="#10131B" />
        <Setter Property="BorderBrush" Value="#22FFFFFF" />
        <Setter Property="BorderThickness" Value="1" />
        <Setter Property="Foreground" Value="#A2B9D8" />
        <Setter Property="Padding" Value="12" />
        <Setter Property="Margin" Value="0,4,0,12" />
        <Setter Property="Template">
            <Setter.Value>
                <ControlTemplate TargetType="Expander">
                    <Border Background="{TemplateBinding Background}"
                            BorderBrush="{TemplateBinding BorderBrush}"
                            BorderThickness="{TemplateBinding BorderThickness}"
                            CornerRadius="8">
                        <StackPanel>
                            <ToggleButton IsChecked="{Binding IsExpanded, Mode=TwoWay, RelativeSource={RelativeSource TemplatedParent}}"
                                          Background="Transparent" BorderThickness="0" Padding="12,8" Cursor="Hand">
                                <ToggleButton.Template>
                                    <ControlTemplate TargetType="ToggleButton">
                                        <Grid Background="Transparent">
                                            <ContentPresenter />
                                        </Grid>
                                    </ControlTemplate>
                                </ToggleButton.Template>
                                <ContentPresenter ContentSource="Header" />
                            </ToggleButton>
                            <ContentPresenter x:Name="ExpandSite"
                                              Visibility="Collapsed"
                                              Margin="{TemplateBinding Padding}" />
                        </StackPanel>
                    </Border>
                    <ControlTemplate.Triggers>
                        <Trigger Property="IsExpanded" Value="True">
                            <Setter TargetName="ExpandSite" Property="Visibility" Value="Visible" />
                        </Trigger>
                    </ControlTemplate.Triggers>
                </ControlTemplate>
            </Setter.Value>
        </Setter>
    </Style>

    <!-- 3. 시스템 크롬 캡션 버튼 -->
    <Style x:Key="ChromeCaptionButtonStyle" TargetType="Button">
        <Setter Property="Background" Value="Transparent" />
        <Setter Property="Foreground" Value="#AAB2C0" />
        <Setter Property="BorderThickness" Value="0" />
        <Setter Property="FontSize" Value="11" />
        <Setter Property="Template">
            <Setter.Value>
                <ControlTemplate TargetType="Button">
                    <Border x:Name="border" Background="{TemplateBinding Background}">
                        <ContentPresenter HorizontalAlignment="Center" VerticalAlignment="Center" />
                    </Border>
                    <ControlTemplate.Triggers>
                        <Trigger Property="IsMouseOver" Value="True">
                            <Setter TargetName="border" Property="Background" Value="#1AFFFFFF" />
                            <Setter Property="Foreground" Value="#FFFFFF" />
                        </Trigger>
                    </ControlTemplate.Triggers>
                </ControlTemplate>
            </Setter.Value>
        </Setter>
    </Style>

    <Style x:Key="ChromeCloseButtonStyle" TargetType="Button" BasedOn="{StaticResource ChromeCaptionButtonStyle}">
        <Style.Triggers>
            <Trigger Property="IsMouseOver" Value="True">
                <Setter Property="Background" Value="#EF4444" />
                <Setter Property="Foreground" Value="#FFFFFF" />
            </Trigger>
        </Style.Triggers>
    </Style>

    <!-- 4. 고밀도 가상화 리스트뷰 (60fps 픽셀 스크롤 & 컨테이너 재활용 완결 스타일) -->
    <!-- 주의: ScrollUnit="Pixel" 적용 시 ItemTemplate 내부 요소의 높이가 균일(Fixed Height)해야 스크롤바 점핑(Jumping Thumb)을 방지할 수 있습니다 -->
    <Style x:Key="VirtualizingListViewStyle" TargetType="ListView">
        <Setter Property="OverridesDefaultStyle" Value="True" />
        <Setter Property="Background" Value="Transparent" />
        <Setter Property="BorderThickness" Value="0" />
        <Setter Property="ScrollViewer.CanContentScroll" Value="True" />
        <Setter Property="ScrollViewer.HorizontalScrollBarVisibility" Value="Disabled" />
        <Setter Property="ScrollViewer.VerticalScrollBarVisibility" Value="Auto" />
        <Setter Property="VirtualizingPanel.IsVirtualizing" Value="True" />
        <Setter Property="VirtualizingPanel.VirtualizationMode" Value="Recycling" />
        <Setter Property="VirtualizingPanel.ScrollUnit" Value="Pixel" />
        <Setter Property="VirtualizingPanel.CacheLength" Value="20,20" />
        <Setter Property="VirtualizingPanel.CacheLengthUnit" Value="Item" />
        <Setter Property="Template">
            <Setter.Value>
                <ControlTemplate TargetType="ListView">
                    <Border Background="{TemplateBinding Background}"
                            BorderBrush="{TemplateBinding BorderBrush}"
                            BorderThickness="{TemplateBinding BorderThickness}">
                        <ScrollViewer Focusable="False" Padding="{TemplateBinding Padding}">
                            <ItemsPresenter />
                        </ScrollViewer>
                    </Border>
                </ControlTemplate>
            </Setter.Value>
        </Setter>
    </Style>

</ResourceDictionary>
```
