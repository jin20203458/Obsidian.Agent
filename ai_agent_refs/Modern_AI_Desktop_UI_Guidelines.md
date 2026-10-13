---
description: WPF 기반 차세대 데스크톱 AI 및 엔터프라이즈 도구 UI/UX 확장 지침서. 다크 테마 디자인 토큰, 3단 워크벤치, AI 추론 서사, 토큰 스트리밍 버퍼 및 ReAct 루프.
related:
  - ../README.md
  - ./WPF_Architecture_Guidelines.md
---
# Modern AI Desktop UI Guidelines

본 문서는 고성능 데스크톱 환경(WPF/.NET 8.0/9.0/10.0)에서 **엔터프라이즈 프로 도구(Enterprise Pro-Tool)**와 **추론형 AI 에이전트의 실시간 서사(Agentic Narrative)**를 결합할 때 준수해야 하는 공식 UI/UX 디자인 시스템 및 특화 컴포넌트 구현 지침서입니다.

기본 애플리케이션 셸, Generic Host DI, 수명주기, 유효성 검사, In-App 모달 및 WindowChrome/DWM 시스템 통합에 관한 핵심 아키텍처는 단일 진실 공급원(SSOT)인 [WPF_Architecture_Guidelines.md](./WPF_Architecture_Guidelines.md)를 따르며, 본 문서는 **AI 사고 과정(Thinking), 무프리징 토큰 스트리밍, ReAct 도구 호출 시각화, 고밀도 3단 워크벤치 및 로컬 폴백 인터페이스**에 집중합니다.

---

## 1. 디자인 철학 및 데스크톱 AI 상호작용 패러다임

현대 데스크톱 UI는 장식용 시각 요소를 배제하고, **고밀도 정보 전달과 즉각적인 시스템 반응성을 보장하는 프로 도구 미학(Pro-Tool High-Performance Aesthetic)**을 지향합니다.

* **네이티브 하드웨어 가속 우위**: 웹 브라우저 엔진(Electron)의 300MB~1GB 메모리 풋프린트를 배제하고, DirectX 하드웨어 가속 기반의 가벼운 30~80MB 풋프린트와 60fps 무프리징 렌더링을 보장합니다.
* **설명 가능성과 서사의 조화**: AI가 내린 판단의 과정(Thought), 호출한 도구(Action), 수집한 관측 데이터(Observation), 최종 판단(Verdict)을 시각적으로 체계화하여 사용자에게 완전한 투명성을 제공합니다.

### 2대 핵심 상호작용 패러다임

| 구분 | 패러다임 A: 대화형 코파일럿 워크벤치 | 패러다임 B: 자율형 에이전트 관제 루프 |
| :--- | :--- | :--- |
| **주요 목적** | 다회차 질의응답, 문서/코드 생성, 분석 보조 | 자율적 이상 진단, 배치 분석 연쇄 호출, 시스템 능동 제어 |
| **상호작용 주체** | 인간 중심 (Human-in-the-Loop 요청에 AI가 응답) | 에이전트 중심 (에이전트가 자율 조사 후 인간에게 승인 요청) |
| **핵심 UI 구성** | 멀티턴 대화 버블, 코드 블록, 프롬프트 입력창, 멀티모달 첨부 | ReAct 루프 타임라인, 도구 호출 피드, 터미널 스트림, 상태 뱃지 |
| **실시간성 요구** | LLM 토큰 단위 점진적 스트리밍 텍스트 렌더링 | 초당 수천 건 원시 텔레메트리 배치 버퍼링 및 프로젝션 |

---

## 2. Dark-First AI 디자인 토큰 시스템

순수 블랙(`black/#000000`)의 과도한 대비로 인한 눈의 피로를 방지하고, 딥 징크/슬레이트 계열의 무채색을 기본 캔버스로 사용합니다.

### A. AI 시맨틱 컬러 토큰 규격 (`Themes/DarkTheme.xaml`)

| 토큰 키 (ResourceKey) | 헥사 코드 (Hex) | 용도 및 설명 |
| :--- | :--- | :--- |
| `AppBackgroundBrush` | `#0B0C10` | 최하단 윈도우 기본 캔버스 배경색 |
| `SurfaceBackgroundBrush` | `#13141C` | 사이드바, 좌측 네비게이션 레일, 상단 헤더 배경색 |
| `CardBackgroundBrush` | `#1A1D27` | 개별 카드, 분석 노드 블록, 대화 버블 배경색 |
| `CardBackgroundElevatedBrush`| `#232736` | 팝오버, 드롭다운 메뉴, 인앱 모달 컨테이너 배경색 |
| `BorderSubtleBrush` | `#22FFFFFF` (13% White) | 주요 컨테이너 및 컨트롤의 1px 외곽선 |
| `TextPrimaryBrush` | `#F2F4F8` | 메인 헤더, 수치 데이터, 강조 제목 (고대비) |
| `TextSecondaryBrush` | `#AAB2C0` | 본문 서사 텍스트, 설명 레이블 |
| `TextMutedBrush` | `#6B7280` | 타임스탬프, 비활성 텍스트, 보조 메타데이터 |
| `TextThoughtBrush` | `#A2B9D8` | AI 에이전트 내면 사고(Thought) 전용 슬레이트 블루 |
| `AccentPrimaryBrush` | `#4F46E5` (Indigo) | 주요 분석 실행 버튼, 활성 탭 인디케이터 |
| `AccentDangerBrush` | `#EF4444` (Crimson) | 치명적 오류, 삭제/파괴적 액션 뱃지 |
| `AccentWarningBrush` | `#F59E0B` (Amber) | 주의 필요, 진행 중인 작업, 오프라인 폴백 모드 알림 |
| `AccentSuccessBrush` | `#10B981` (Emerald) | 작업 성공 완료, 데이터 검증 완료, 정상 상태 뱃지 |

### B. 루미넌스 스태킹 (Luminance Stacking)
다크 UI에서 성능을 저하시키는 큰 반경의 블러 그림자(Drop Shadow)를 배제하고, 계층(Z-Index)이 높아질수록 미세하게 명도를 높여 물리적 깊이감을 부여합니다.

```text
[Layer 0: Window Background]  ➔  #0B0C10
    └── [Layer 1: Surface Rail]   ➔  #13141C (Border: 1px #22FFFFFF)
            └── [Layer 2: Content Card]  ➔  #1A1D27 (Border: 1px #22FFFFFF)
                    └── [Layer 3: Popover/Modal] ➔  #232736 (Border: 1px #33FFFFFF)
```

---

## 3. 고밀도 3단 분할 워크벤치 레이아웃

VS Code, Linear, Obsidian 스타일의 **고밀도 프로페셔널 정보 구조**를 구축하기 위해 `GridSplitter` 기반의 3단 분할 레이아웃을 표준화합니다.

```text
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
│ (MinWidth=240)│ │ (Width="*")                              │ │ (Width="380", MinWidth=300)         │
└───────────────┴─┴──────────────────────────────────────────┴─┴─────────────────────────────────────┘
```

### 패널 너비 상태 영속화 패턴 (Pane Width State Persistence)

> [!CAUTION]
> **[WPF 런타임 함정: `ColumnDefinition.Width` TwoWay 바인딩 파괴 방어]**  
> XAML에서 `<ColumnDefinition Width="{Binding LeftPaneWidth, Mode=TwoWay}" />`로 선언하는 것은 동작하지 않는 안티패턴입니다. WPF에서 사용자가 `GridSplitter`를 마우스로 드래그하면 내부적으로 `ColumnDefinition.Width = new GridLength(...)`로 **로컬 값(Local Value)을 직접 할당**합니다. WPF 의존성 속성 우선순위 원칙상 로컬 값이 셋팅되면 기존 `BindingExpression`이 즉시 덮어씌워져 영구 파괴되므로, 드래그 후 바인딩이 완전히 끊어집니다.

따라서 상용 엔터프라이즈 도구(PowerToys, ScreenToGif)와 동일하게 **`GridSplitter.DragCompleted` 이벤트와 View `Loaded` 이벤트를 통해 안전하게 너비를 저장/복원하는 표준 패턴**을 사용합니다.

#### View 구현 (`Views/WorkbenchView.xaml.cs`)
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
    // 드래그 완료 시점에만 실제 픽셀 너비를 ViewModel 설정으로 저장
    if (DataContext is WorkbenchViewModel vm)
    {
        vm.SavePaneWidths(LeftPaneColumn.ActualWidth, RightPaneColumn.ActualWidth);
    }
}
```

---

## 4. 실시간 AI 사고(Thinking) 및 스트리밍 서사(Narrative) UI

현대 추론형 AI 모델(DeepSeek-R1, OpenAI o1/o3, Gemini 2.0 Flash Thinking)의 확장 사고 패러다임에 맞추어, **내부 추론 과정(`<think>`)과 최종 분석 서사(Narrative)를 시각적으로 엄격히 분리**합니다.

```text
┌──────────────────────────────────────────────────────────────┐
│ ▾  AI 사고 과정 (Thinking Process - 1.8s, 420 tokens)        │  <- Expander
│   ├─ 요청 파라미터 유효성 및 원격 데이터 정합성 검증...      │  <- TextThoughtBrush
│   ├─ 이상 변동치 및 정책 임계값 비교 분석...                 │
│   └─ 기준치 초과 변동 확인. 위험도 지표 85 산출.              │
└──────────────────────────────────────────────────────────────┘
│
[최종 분석 서사]
대상 메트릭에서 기준치를 초과하는 변동 패턴이 감지되었습니다.
자동 보정 및 동기화 조치를 위해 승인 인터락을 요청합니다.
```

### 무프리징 토큰 증분 렌더링 파이프라인 (`StreamingNarrativeBuffer.cs`)
LLM 토큰이 초당 50~100개씩 유입될 때 전체 문자열을 매번 다시 바인딩하면 렌더 트리 재구성으로 인해 UI 스레드가 마비됩니다. 채널 기반 버퍼링과 증분 렌더링을 적용해야 합니다.

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

    /// <summary>
    /// 사용자 요청 또는 에러 발생 시 버퍼를 즉시 비우고 렌더 루프를 강제 중단합니다.
    /// </summary>
    public void Abort()
    {
        _cts.Cancel();
        while (_tokenChannel.Reader.TryRead(out _)) { /* 미출력 잔여 토큰 즉시 드레인 */ }
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
                // 채널에 대기 중인 모든 토큰을 일괄 비우기
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
            // 화면 이탈 또는 Abort 호출 시 안전하게 루프 종료
        }
    }

    public void Dispose()
    {
        _tokenChannel.Writer.TryComplete();
        _cts.Cancel();
        _cts.Dispose();
    }
}
```

### 생성 중단(Abort/Stop) 인터랙션 오케스트레이션 파이프라인
사용자가 UI의 "생성 중단(Stop)" 버튼을 클릭했을 때 백엔드 AI 통신 태스크와 프론트엔드 UI 버퍼가 원자적으로 동시에 중단되어야 토큰 비용 낭비와 UI 잔류 출력을 방지할 수 있습니다.

```csharp
public partial class AiWorkbenchViewModel : ObservableObject
{
    private CancellationTokenSource? _generationCts;
    private StreamingNarrativeBuffer? _narrativeBuffer;

    [ObservableProperty] private bool _isGenerating;

    [RelayCommand]
    private async Task StartGenerationAsync(string prompt)
    {
        IsGenerating = true;
        _generationCts = new CancellationTokenSource();
        _narrativeBuffer = new StreamingNarrativeBuffer(chunk => AppendNarrative(chunk));

        try
        {
            // 백엔드 AI 스트리밍 호출 (CancellationToken 전달)
            await foreach (var token in _aiService.StreamGenerateAsync(prompt, _generationCts.Token))
            {
                _narrativeBuffer.PushToken(token);
            }
        }
        catch (OperationCanceledException)
        {
            // 취소 정상 처리
        }
        finally
        {
            IsGenerating = false;
            _narrativeBuffer?.Dispose();
        }
    }

    [RelayCommand]
    private void StopGeneration()
    {
        // 1. 백엔드 HTTP/gRPC AI 스트리밍 즉각 취소 (토큰 과금 중단)
        _generationCts?.Cancel();

        // 2. UI 채널 버퍼 대기열 즉각 드레인 및 렌더 루프 중단
        _narrativeBuffer?.Abort();

        IsGenerating = false;
    }
}
```

### Markdown 증분 렌더링 전략 (Plain Text vs Rich Markdown)
단순 `TextBlock`은 코드 펜스(```` ```csharp ````), 굵은 글씨(`**`), 인라인 코드(`` ` ``) 등 LLM의 마크다운 서사를 원시 문자열로만 노출하는 한계가 있습니다. 프로덕션 환경에서는 다음 렌더링 전략을 채택합니다:

1. **토큰 단위 재파싱 금지**: 토큰 유입 시마다 전체 마크다운 문서를 통째로 AST 파싱하면 O(N^2) CPU 락이 발생합니다.
2. **블록 단위 증분 렌더링**: 개행 문자(`\n\n`) 단위로 완성된 문단 및 코드 블록 단위로 마크다운 AST를 구성하고, 작성 중인 최신 블록만 인라인 스트리밍 텍스트로 합성합니다.
3. **권장 오픈소스 바인딩**: 경량 WPF 마크다운 엔진인 `Markdig.Wpf` 또는 `MdXaml`을 사용하며, 렌더링 쓰로틀링(Throttling)을 결합하여 60fps 렌더 예산을 유지합니다.

### 스트리밍 스마트 오토스크롤 Attached Behavior (`SmartAutoScrollBehavior`)
텍스트가 늘어날 때 무조건 `ScrollToEnd()`를 호출하면 사용자가 이전 추론 과정을 읽으려고 위로 스크롤했을 때 강제로 바닥으로 끌어내려지는 **스크롤 하이재킹(Scroll Hijacking)**이 발생합니다. 사용자가 하단 부근에 머물러 있을 때만 자동 스크롤을 수행합니다.

```csharp
namespace MyWpfApp.Views.Controls;

public static class ScrollViewerExtensions
{
    public static readonly DependencyProperty AutoScrollToBottomProperty =
        DependencyProperty.RegisterAttached(
            "AutoScrollToBottom",
            typeof(bool),
            typeof(ScrollViewerExtensions),
            new PropertyMetadata(false, OnAutoScrollToBottomChanged));

    public static void SetAutoScrollToBottom(DependencyObject obj, bool value) => obj.SetValue(AutoScrollToBottomProperty, value);
    public static bool GetAutoScrollToBottom(DependencyObject obj) => (bool)obj.GetValue(AutoScrollToBottomProperty);

    private static void OnAutoScrollToBottomChanged(DependencyObject d, DependencyPropertyChangedEventArgs e)
    {
        if (d is ScrollViewer sv && (bool)e.NewValue)
        {
            sv.ScrollChanged += (s, args) =>
            {
                // 내용 크기(ExtentHeight)가 증가했을 때만 판단
                if (args.ExtentHeightChange > 0)
                {
                    // 사용자가 하단 허용 오차(24px) 이내에 머물러 있는 경우에만 최하단 스크롤 유지
                    // 위로 스크롤하여 이전 서사를 읽는 중이면 스크롤 하이재킹 방지
                    bool isAtBottom = sv.VerticalOffset >= (sv.ScrollableHeight - args.ExtentHeightChange - 24);
                    if (isAtBottom)
                    {
                        sv.ScrollToEnd();
                    }
                }
            };
        }
    }
}
```

```xml
<!-- XAML 사용처: ScrollViewer에 스마트 오토스크롤 활성화 -->
<ScrollViewer controls:ScrollViewerExtensions.AutoScrollToBottom="True"
              VerticalScrollBarVisibility="Auto">
    <!-- 스트리밍 서사 뷰어 컨텐츠 -->
</ScrollViewer>
```

---

## 5. 자율형 에이전트 ReAct 루프 및 도구 호출 카드 XAML 스타일

자율 에이전트가 판단(Thought)하고 도구를 호출(Action)하며 결과(Observation)를 확인하는 전 과정을 타임라인 카드 형태로 시각화합니다.

```xml
<!-- ReAct 도구 호출 타임라인 카드 -->
<Border Background="{DynamicResource CardBackgroundBrush}"
        BorderBrush="{DynamicResource BorderSubtleBrush}"
        BorderThickness="1" CornerRadius="8" Padding="14" Margin="0,0,0,12">
    <StackPanel>
        <!-- 도구 헤더 (이름 뱃지 + 실행 상태 인디케이터) -->
        <Grid Margin="0,0,0,10">
            <StackPanel Orientation="Horizontal" VerticalAlignment="Center">
                <Border Background="{DynamicResource AccentPrimaryBrush}" CornerRadius="4" Padding="6,2" Margin="0,0,8,0">
                    <TextBlock Text="TOOL CALL" Foreground="#FFFFFF" FontWeight="Bold" FontSize="10" />
                </Border>
                <TextBlock Text="ExecuteAnalysisTool" Foreground="{DynamicResource TextPrimaryBrush}" FontWeight="SemiBold" FontSize="13" />
            </StackPanel>
            
            <Border HorizontalAlignment="Right" Background="{DynamicResource AccentSuccessBrush}" CornerRadius="10" Padding="8,2">
                <TextBlock Text="COMPLETED" Foreground="#FFFFFF" FontWeight="Bold" FontSize="10" />
            </Border>
        </Grid>

        <!-- 파라미터 JSON 아코디언 -->
        <Expander Header="전달 파라미터" Foreground="{DynamicResource TextSecondaryBrush}" FontSize="11" Margin="0,0,0,8">
            <Border Background="{DynamicResource SurfaceBackgroundBrush}" CornerRadius="4" Padding="8" Margin="0,4,0,0">
                <TextBlock Text="{}{ &quot;targetId&quot;: 4920, &quot;scope&quot;: &quot;detailed&quot; }"
                           FontFamily="Cascadia Code, Consolas" FontSize="11" Foreground="{DynamicResource TextThoughtBrush}" />
            </Border>
        </Expander>

        <!-- 터미널 표준 출력 (stdout) 뷰어 -->
        <Border Background="{DynamicResource AppBackgroundBrush}" BorderBrush="{DynamicResource BorderSubtleBrush}" BorderThickness="1" CornerRadius="4" Padding="10">
            <ScrollViewer MaxHeight="120" VerticalScrollBarVisibility="Auto">
                <TextBlock Text="[INFO] Querying remote telemetry endpoint...&#x0a;[WARN] Latency spike detected on secondary channel"
                           FontFamily="Cascadia Code, Consolas" FontSize="11" Foreground="{DynamicResource AccentSuccessBrush}" TextWrapping="Wrap" />
            </ScrollViewer>
        </Border>
    </StackPanel>
</Border>
```

---

## 6. 신뢰성 및 오프라인 로컬 결정론적 폴백(Fallback) UI 패턴

클라우드 AI 서비스 장애나 네트워크 두절 시 사용자가 시스템 상태를 즉시 인지하고 안전하게 로컬 규칙 엔진으로 전환되도록 가시적 피드백을 제공합니다.

```xml
<!-- 상단 고정 오프라인 알림 배너 (BoolToVis는 WPF_Architecture_Guidelines.md 규격에 따라 App.xaml에 등록된 전역 컨버터) -->
<Border Background="#7F3B00" BorderBrush="{DynamicResource AccentWarningBrush}" BorderThickness="0,0,0,1"
        Padding="16,8" Visibility="{Binding IsOfflineFallbackActive, Converter={StaticResource BoolToVis}}">
    <Grid>
        <StackPanel Orientation="Horizontal" HorizontalAlignment="Center">
            <Border Width="8" Height="8" Background="{DynamicResource AccentWarningBrush}" CornerRadius="4" VerticalAlignment="Center" Margin="0,0,8,0" />
            <TextBlock Text="LOCAL FALLBACK ACTIVE: 클라우드 AI 연결 불가로 로컬 결정론적 규칙 엔진으로 전환되었습니다."
                       Foreground="#FDE68A" FontWeight="SemiBold" FontSize="12" />
        </StackPanel>
    </Grid>
</Border>
```

---

## 7. AI 특화 XAML 컨트롤 스타일 사전

AI 에이전트 인터페이스에서 사용되는 특화 컨트롤 템플릿입니다.

```xml
<ResourceDictionary xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
                    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">

    <!-- 1. AI 내면 사고 접이식 Expander -->
    <Style x:Key="ThoughtExpanderStyle" TargetType="Expander">
        <Setter Property="Background" Value="{DynamicResource SurfaceBackgroundBrush}" />
        <Setter Property="BorderBrush" Value="{DynamicResource BorderSubtleBrush}" />
        <Setter Property="BorderThickness" Value="1" />
        <Setter Property="Foreground" Value="{DynamicResource TextThoughtBrush}" />
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

    <!-- 2. 캡슐형 주 액션 버튼 -->
    <Style x:Key="CapsulePrimaryButtonStyle" TargetType="Button">
        <Setter Property="Height" Value="36" />
        <Setter Property="Padding" Value="18,0" />
        <Setter Property="Background" Value="{DynamicResource AccentPrimaryBrush}" />
        <Setter Property="Foreground" Value="#FFFFFF" />
        <Setter Property="FontWeight" Value="SemiBold" />
        <Setter Property="FontSize" Value="13" />
        <Setter Property="Cursor" Value="Hand" />
        <Setter Property="Template">
            <Setter.Value>
                <ControlTemplate TargetType="Button">
                    <Border x:Name="border"
                            Background="{TemplateBinding Background}"
                            CornerRadius="18"
                            SnapsToDevicePixels="True">
                        <ContentPresenter HorizontalAlignment="Center" VerticalAlignment="Center" />
                    </Border>
                    <ControlTemplate.Triggers>
                        <Trigger Property="IsMouseOver" Value="True">
                            <Setter TargetName="border" Property="Opacity" Value="0.9" />
                        </Trigger>
                        <Trigger Property="IsEnabled" Value="False">
                            <Setter TargetName="border" Property="Opacity" Value="0.4" />
                        </Trigger>
                    </ControlTemplate.Triggers>
                </ControlTemplate>
            </Setter.Value>
        </Setter>
    </Style>

</ResourceDictionary>
```
