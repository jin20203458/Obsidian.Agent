---
description: WPF 기반 차세대 데스크톱 AI 및 엔터프라이즈 도구 UI/UX 설계 지침서. 다크 테마 디자인 토큰, 윈도우 크롬 일체화, AI 추론 서사 및 실시간 스트리밍.
related:
  - ../README.md
  - ./WPF_Architecture_Guidelines.md
  - ../Phalanx/docs/01_system_architecture.md
---
# Modern AI Desktop UI Guidelines

본 문서는 고성능 데스크톱 환경(WPF/.NET 8.0 이상)에서 **엔터프라이즈 프로 도구(Enterprise Pro-Tool)**와 **추론형 AI 에이전트의 실시간 서사(Agentic Narrative)**를 결합할 때 준수해야 하는 공식 UI/UX 디자인 시스템 및 XAML 구현 지침서입니다.

보안 관제 엔진(Phalanx EDR), 대규모 분석 워크벤치(ARQA), 멀티모달 대화형 AI 도구(GRC)에서 검증된 인터페이스 설계 패턴과 최신 Windows 11 Fluent 및 다크 테마 트렌드를 종합하여 완전한 프로덕션 표준을 정의합니다.

---

## 1. 디자인 철학 및 데스크톱 AI 상호작용 패러다임

현대 데스크톱 UI는 장식용 시각 요소(화려한 그라디언트, 무거운 블러 그림자, 이모지 남발)를 배제하고, **고밀도 정보 전달과 즉각적인 시스템 반응성을 보장하는 프로 도구 미학(Pro-Tool High-Performance Aesthetic)**을 지향합니다.

* **네이티브 하드웨어 가속 우위**: 웹 브라우저 엔진(Electron)의 300MB~1GB 메모리 풋프린트를 배제하고, DirectX 하드웨어 가속 기반의 가벼운 30~80MB 풋프린트와 60fps 무프리징 렌더링을 보장합니다.
* **설명 가능성과 서사의 조화**: AI가 내린 판단의 과정(Thought), 호출한 도구(Action), 수집한 관측 데이터(Observation), 최종 판단(Verdict)을 시각적으로 체계화하여 사용자에게 완전한 투명성을 제공합니다.

### 2대 핵심 상호작용 패러다임

| 구분 | 패러다임 A: 대화형 코파일럿 워크벤치 (GRC/ARQA 패턴) | 패러다임 B: 자율형 에이전트 관제 루프 (Phalanx 패턴) |
| :--- | :--- | :--- |
| **주요 목적** | 다회차 질의응답, 문서/코드 생성, 분석 보조 | 자율적 이상 탐지, 포렌식 조사 연쇄 호출, 프로세스 능동 제어 |
| **상호작용 주체** | 인간 중심 (Human-in-the-Loop 요청에 AI가 응답) | 에이전트 중심 (에이전트가 자율 조사 후 인간에게 승인 요청) |
| **핵심 UI 구성** | 멀티턴 대화 버블, 코드 블록, 프롬프트 입력창, 멀티모달 첨부 | ReAct 루프 타임라인, 도구 호출 피드, 터미널 스트림, 상태 뱃지 |
| **실시간성 요구** | LLM 토큰 단위 점진적 스트리밍 텍스트 렌더링 | 초당 수천 건 원시 텔레메트리 배치 버퍼링 및 프로젝션 |

---

## 2. 디자인 토큰 시스템 (Design Tokens Specification)

### A. Dark-First 컬러 팔레트
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

## 3. 실시간 AI 사고(Thinking) 및 스트리밍 서사(Narrative) UI

현대 추론형 AI 모델(DeepSeek-R1, OpenAI o1/o3, Gemini 2.0 Flash Thinking)의 확장 사고 패러다임에 맞추어, **내부 추론 과정(`<think>`)과 최종 분석 서사(Narrative)를 시각적으로 엄격히 분리**합니다.

```
┌──────────────────────────────────────────────────────────────┐
│ ▾  AI 사고 과정 (Thinking Process - 1.8s, 420 tokens)        │  <- Expander
│   ├─ 프로세스 CommandLine 문자열에서 Base64 패턴 감지...     │  <- TextThought (#A2B9D8)
│   ├─ NtSuspendProcess 안전성 Watchdog 규칙 검토...            │
│   └─ 신뢰 서명 불일치 확인. 위험도 85 산출.                   │
└──────────────────────────────────────────────────────────────┘
│
[최종 분석 서사]
대상 프로세스(PID 4920)는 정상 시스템 프로세스로 위장한 드로퍼입니다.
네트워크 C2 비콘 연결을 시도하기 직전 선제 동결되었습니다.
```

### A. 무프리징 토큰 증분 렌더링 파이프라인
LLM 토큰이 초당 50~100개씩 쏟아질 때 전체 문자열을 바인딩하면 렌더 트리 재구성으로 인해 UI 스레드가 마비됩니다. 채널 기반 버퍼링과 증분 렌더링을 적용해야 합니다.

```csharp
using System;
using System.Text;
using System.Threading.Channels;
using System.Threading.Tasks;
using System.Windows;
using System.Windows.Threading;

public class StreamingNarrativeBuffer
{
    private readonly Channel<string> _tokenChannel = Channel.CreateUnbounded<string>();
    private readonly Action<string> _appendAction;

    public StreamingNarrativeBuffer(Action<string> appendAction)
    {
        _appendAction = appendAction;
        _ = ProcessTokenQueueAsync();
    }

    public void PushToken(string token)
    {
        _tokenChannel.Writer.TryWrite(token);
    }

    private async Task ProcessTokenQueueAsync()
    {
        var reader = _tokenChannel.Reader;
        var sb = new StringBuilder();

        while (await reader.WaitToReadAsync())
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
}
```

---

## 4. 자율형 에이전트 ReAct 루프 및 포렌식 도구 피드 시각화

자율 에이전트가 판단(Thought)하고 도구를 호출(Action)하며 결과(Observation)를 확인하는 전 과정을 타임라인 카드 형태로 시각화합니다.

### A. ReAct 단계별 시각 컴포넌트 구조
1. **Thought Block**: AI의 가설 및 의도 설명 (슬레이트 블루 배경, 모노스페이스 이탤릭).
2. **Action Block (Tool Call)**:
   * 호출 도구 칩 (예: `TOOL: InspectProcessMemory`).
   * 전달 파라미터 JSON 접이식 아코디언.
   * 실행 상태 인디케이터 (Running: Amber 펄스, Success: Emerald 체크, Failed: Crimson 에러).
3. **Observation Block**:
   * 도구 실행 결과 및 터미널 출력(stdout/stderr).
   * 160px 제한 높이의 내부 스크롤뷰어 및 가로 스크롤 방지 래핑.
4. **Final Verdict**:
   * 최종 판정 결과 및 프로세스 제어 상태 표시.

### B. 휴먼 인 더 루프(HITL) 위험 액션 승인 모달
치명적인 시스템 변경(프로세스 강제 종료, 파일 격리, 방화벽 차단) 실행 전에는 자율 루프를 일시 중지하고 사용자 명시적 승인을 요청하는 모달을 띄웁니다.

```xml
<!-- 인앱 오버레이 안전 확인 카드 -->
<Border Background="#1A1D27"
        BorderBrush="#EF4444"
        BorderThickness="1"
        CornerRadius="12"
        Padding="20"
        MaxWidth="480">
    <StackPanel>
        <StackPanel Orientation="Horizontal" Margin="0,0,0,12">
            <Border Width="8" Height="8" Background="#EF4444" CornerRadius="4" VerticalAlignment="Center" Margin="0,0,8,0" />
            <TextBlock Text="위험 행위 승인 요청" FontWeight="Bold" Foreground="#F2F4F8" FontSize="15" />
        </StackPanel>

        <TextBlock Text="에이전트가 리소스 안정성 확보 및 제어를 위해 다음 대상의 실행/변경 승인을 요청했습니다:"
                   Foreground="#AAB2C0" TextWrapping="Wrap" Margin="0,0,0,12" />

        <Border Background="#13141C" Padding="12" CornerRadius="6" Margin="0,0,0,16">
            <TextBlock Text="Target: [Resource_or_Process_Name] (ID: 4920)&#x0a;Operation: Force Terminate / Critical State Modification"
                       FontFamily="Cascadia Code, Consolas" FontSize="12" Foreground="#EF4444" />
        </Border>

        <Grid>
            <Grid.ColumnDefinitions>
                <ColumnDefinition Width="*" />
                <ColumnDefinition Width="12" />
                <ColumnDefinition Width="*" />
            </Grid.ColumnDefinitions>
            <Button Grid.Column="0" Content="거부 (Skip)" Style="{StaticResource SecondaryButtonStyle}" Command="{Binding RejectActionCommand}" />
            <Button Grid.Column="2" Content="실행 승인 (Execute)" Style="{StaticResource DangerActionButtonStyle}" Command="{Binding ApproveActionCommand}" />
        </Grid>
    </StackPanel>
</Border>
```

---

## 5. 윈도우 크롬 일체화 및 Windows 11 Fluent 시스템 통합

앱 상단 바와 윈도우 타이틀바를 일체화(Seamless)하여 상단 세로 공간을 32px 절약하고, Windows 11 네이티브 스냅 레이아웃(Snap Assist)과 심층 다크 모드를 완벽히 지원합니다.

### A. WindowChrome 기본 선언 (`MainWindow.xaml`)

```xml
<Window x:Class="MyWpfApp.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:shell="clr-namespace:System.Windows.Shell;assembly=PresentationFramework"
        Title="Enterprise AI Workbench"
        Height="800" Width="1300"
        Background="#0B0C10"
        WindowStartupLocation="CenterScreen">

    <shell:WindowChrome.WindowChrome>
        <shell:WindowChrome CaptionHeight="44"
                            CornerRadius="0"
                            GlassFrameThickness="0"
                            NonClientFrameEdges="None"
                            ResizeBorderThickness="6"
                            UseAeroCaptionButtons="False" />
    </shell:WindowChrome.WindowChrome>

    <!-- 최대화 시 모니터 경계 7px 잘림(Overshoot) 방지 스타일 (단일/동일 DPI 환경 기본값) -->
    <!-- (참고: 서로 다른 DPI의 다중 모니터 환경에서 픽셀 단위 정밀 제어가 필요한 경우 WM_GETMINMAXINFO 윈도우 프로시저 훅을 통해 동적 마진을 산출할 수 있습니다.) -->
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

    <Grid Background="#0B0C10">
        <!-- 상단 44px 커스텀 타이틀바 -->
        <Grid.RowDefinitions>
            <RowDefinition Height="44" />
            <RowDefinition Height="*" />
        </Grid.RowDefinitions>

        <!-- 타이틀바 컨테이너 -->
        <Border Grid.Row="0" Background="#13141C" BorderBrush="#1AFFFFFF" BorderThickness="0,0,0,1">
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

### B. Windows 11 Snap Layouts 지원 및 DWM 다크 모드 연동 (`MainWindow.xaml.cs`)

사용자가 최대화 버튼 위에 마우스를 올렸을 때 Windows 11의 Snap Layout 그리드가 나타나게 하려면 `WM_NCHITTEST`를 가로채어 `HTMAXBUTTON (9)`를 반환해야 합니다.

```csharp
using System;
using System.Runtime.InteropServices;
using System.Windows;
using System.Windows.Interop;

namespace MyWpfApp
{
    public partial class MainWindow : Window
    {
        private const int WM_NCHITTEST = 0x0084;
        private const int HTMAXBUTTON = 9;

        private const int DWMWA_USE_IMMERSIVE_DARK_MODE = 20;
        private const int DWMWA_WINDOW_CORNER_PREFERENCE = 33;
        private const int DWMWCP_ROUND = 2;

        [DllImport("dwmapi.dll")]
        private static extern int DwmSetWindowAttribute(IntPtr hwnd, int attr, ref int attrValue, int attrSize);

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

            // 3. Snap Layout 지원을 위한 Win32 메시지 훅 추가
            var hwndSource = HwndSource.FromHwnd(hwnd);
            hwndSource?.AddHook(WndProc);
        }

        private IntPtr WndProc(IntPtr hwnd, int msg, IntPtr wParam, IntPtr lParam, ref bool handled)
        {
            if (msg == WM_NCHITTEST)
            {
                // 화면 좌표 추출
                int x = lParam.ToInt32() & 0xffff;
                int y = lParam.ToInt32() >> 16;

                // 최대화 버튼의 화면상 사각 영역 계산
                if (MaximizeButton.IsLoaded)
                {
                    var buttonPos = MaximizeButton.PointToScreen(new Point(0, 0));
                    var buttonRect = new Rect(buttonPos.X, buttonPos.Y, MaximizeButton.ActualWidth, MaximizeButton.ActualHeight);

                    if (buttonRect.Contains(new Point(x, y)))
                    {
                        handled = true;
                        return new IntPtr(HTMAXBUTTON); // Windows 11 Snap Flyout 활성화
                    }
                }
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
}
```

---

## 6. 멀티모달 및 오디오/음성 인터랙션 UX (Lessons from GRC)

현대 데스크톱 AI 애플리케이션은 텍스트 프롬프트에 국한되지 않고, 파일/이미지 첨부 및 실시간 음성(STT/TTS)을 매끄럽게 지원해야 합니다.

### A. 음성 인터랙션 피드백 상태 머신
* **Idle (대기)**: 비활성 마이크 아이콘.
* **Listening (청취 중)**: 인디고 컬러 방사형 펄스 애니메이션 적용.
* **Speaking (TTS 재생 중)**: 오디오 파형 바 애니메이션 및 사용자 입력 시 즉각 음성 중단(Playback Interruption) 발동.

### B. 멀티모달 파일/이미지 첨부 칩(Pill)
첨부된 파일은 28px 높이의 콤팩트 칩으로 렌더링하며, 호버 시 삭제 버튼(`✕`)을 노출합니다.

```xml
<ItemsControl ItemsSource="{Binding AttachedFiles}">
    <ItemsControl.ItemsPanel>
        <ItemsPanelTemplate>
            <WrapPanel Orientation="Horizontal" />
        </ItemsPanelTemplate>
    </ItemsControl.ItemsPanel>
    <ItemsControl.ItemTemplate>
        <DataTemplate>
            <Border Background="#1A1D27" BorderBrush="#22FFFFFF" BorderThickness="1"
                    CornerRadius="14" Height="28" Padding="10,0" Margin="0,0,8,8">
                <StackPanel Orientation="Horizontal" VerticalAlignment="Center">
                    <TextBlock Text="{Binding FileName}" Foreground="#F2F4F8" FontSize="12" Margin="0,0,6,0" />
                    <TextBlock Text="{Binding FileSizeText}" Foreground="#6B7280" FontSize="10" Margin="0,0,8,0" />
                    <Button Content="✕"
                            Command="{Binding DataContext.RemoveAttachmentCommand, RelativeSource={RelativeSource AncestorType=ItemsControl}}"
                            CommandParameter="{Binding}"
                            Background="Transparent" BorderThickness="0" Foreground="#AAB2C0" FontSize="10" Cursor="Hand" />
                </StackPanel>
            </Border>
        </DataTemplate>
    </ItemsControl.ItemTemplate>
</ItemsControl>
```

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

### B. 토큰 소진 및 레이트 리밋(Rate Limit) 방어 안내
할당량 초과 시 크래시가 아닌 명확한 대기 시간 카운트다운 뱃지를 렌더링합니다.

---

## 8. XAML 표준 스타일 템플릿 카탈로그

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

</ResourceDictionary>
```
