---
description: >-
  WPF 기반 차세대 데스크톱 AI 및 보안 관제(SOC) UI/UX 설계 지침서. 다크 테마 디자인 토큰, 캡슐형 컨트롤, 윈도우 타이틀바 일체화 방법론, 노드 그래프 및 실시간 AI 사고 스트리밍 XAML 구현 시 참조.
related:
  - ../README.md
  - ./WPF_Architecture_Guidelines.md
  - ../Phalanx/docs/01_system_architecture.md
---
# Modern AI Desktop UI Guidelines

본 문서는 고성능 데스크톱 환경(WPF/.NET)에서 **엔터프라이즈 관제 대시보드(SOC Cockpit)**와 **추론형 AI 에이전트의 실시간 사고 서사(Agentic Narrative)**를 결합할 때 준수해야 하는 공식 디자인 시스템 및 XAML 구현 표준을 정의합니다.

---

## 1. 디자인 철학 (Design Philosophy)

현대 데스크톱 UI는 과거의 과도한 장식(스큐어모피즘, 번지는 드롭 섀도우)이나 극단적인 플랫 디자인을 지양하고, **고성능 프로 도구(Pro-Tool High-Performance Aesthetic)**를 지향합니다.

* **결정론적 관제 뼈대 (Deterministic Frame)**: 상용 엔터프라이즈 분석 및 보안 관제 콘솔에서 검증된 1280px 이상의 대형 분할 레이아웃, 캡슐형 컨트롤, 다크 팔레트를 기반으로 엔터프라이즈 전문 도구의 안정감을 제공합니다.
* **지능형 추론 서사 (Agentic Narrative Soul)**: 고도화된 데스크톱 AI 클라이언트에서 검증된 실시간 사고(Thought) 및 서사(Narrative) 스트리밍 타이포그래피를 채택하여, AI가 내린 판단의 설명 가능성(Explainability)과 생동감을 극대화합니다.
* **Electron 대비 고성능 네이티브 우위**: 웹 브라우저 엔진의 300MB~1GB 풋프린트를 배제하고, DirectX 하드웨어 가속 기반의 가벼운 수십 MB 풋프린트와 무프리징 렌더링을 보장합니다.

---

## 2. 디자인 토큰 규격 (Design Tokens Specification)

### A. Dark-First 컬러 팔레트
순수 블랙(`black/#000000`)을 배제하고 눈의 피로를 최소화하는 딥 징크/슬레이트 계열의 다크 테마를 강제합니다.

| 토큰명 (Token) | 헥사 코드 (Hex) | 용도 및 설명 |
| :--- | :--- | :--- |
| `BgBase` | `#0B0C10` | 윈도우 최하단 기본 캔버스 배경색 |
| `BgSurface` | `#13141C` | 사이드바, 패널, 모듈 컨테이너 배경색 |
| `BgCard` | `#1A1D27` | 개별 카드, 노드 블록, 입력 폼 배경색 |
| `BorderSubtle` | `#22FFFFFF` (13% White) | 컨테이너 및 컨트롤 1px 외곽선 |
| `BorderUltraSubtle` | `#0FFFFFFF` (6% White) | 내부 서브 디바이더 및 그리드 라인 |
| `TextPrimary` | `#F2F4F8` | 메인 헤더, 수치 데이터, 강조 텍스트 |
| `TextSecondary` | `#AAB2C0` | 본문 설명, 서사 텍스트, 비활성 레이블 |
| `TextThought` | `#A2B9D8` | AI 에이전트 내면 사고(Thought) 전용 슬레이트 블루 |
| `AccentPrimary` | `#4F46E5` (Indigo) | 주요 분석 실행 버튼, 선택된 활성 탭 |
| `AccentDanger` | `#EF4444` (Crimson) | 프로세스 동결/강제종료, 치명적 위협 뱃지 |

### B. 루미넌스 스태킹 (Luminance Stacking)
다크 모드에서 칙칙하게 번지는 블러 그림자(Drop Shadow) 사용을 금지하며, 계층(Z-Index)이 높아질수록 미세하게 명도를 단계별로 높여 물리적 깊이감을 구현합니다.

```
[Layer 0: Window Background]  ➔  #0B0C10
    └── [Layer 1: Surface Panel]   ➔  #13141C (1px Border: #22FFFFFF)
            └── [Layer 2: Content Card]  ➔  #1A1D27 (1px Border: #22FFFFFF)
                    └── [Layer 3: Popover/Modal] ➔  #232736
```

---

## 3. 캡슐형(Pill) 컨트롤 XAML 스타일 템플릿

고밀도 대시보드 속에서 시각적 완충과 조작 가독성을 제공하기 위해 주요 액션 버튼과 상태 칩은 **완전 라운딩 캡슐 형태(`CornerRadius="24"` 또는 `9999`)**로 구현합니다.

```xml
<ResourceDictionary xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
                    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">

    <!-- 캡슐형 주 액션 버튼 스타일 -->
    <Style x:Key="CapsulePrimaryButtonStyle" TargetType="Button">
        <Setter Property="Height" Value="48" />
        <Setter Property="MinWidth" Value="140" />
        <Setter Property="Padding" Value="24,0" />
        <Setter Property="Foreground" Value="#FFFFFF" />
        <Setter Property="FontWeight" Value="SemiBold" />
        <Setter Property="FontSize" Value="14" />
        <Setter Property="Cursor" Value="Hand" />
        <Setter Property="Template">
            <Setter.Value>
                <ControlTemplate TargetType="Button">
                    <Border x:Name="border"
                            Background="#4F46E5"
                            BorderBrush="#6366F1"
                            BorderThickness="1"
                            CornerRadius="24"
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
                            <Setter TargetName="border" Property="Opacity" Value="0.5" />
                        </Trigger>
                    </ControlTemplate.Triggers>
                </ControlTemplate>
            </Setter.Value>
        </Setter>
    </Style>

    <!-- 캡슐형 상태 표시 뱃지 (Badge/Chip) -->
    <Style x:Key="StatusPillBadgeStyle" TargetType="Border">
        <Setter Property="Height" Value="24" />
        <Setter Property="Padding" Value="10,0" />
        <Setter Property="CornerRadius" Value="12" />
        <Setter Property="BorderThickness" Value="1" />
        <Setter Property="HorizontalAlignment" Value="Center" />
        <Setter Property="VerticalAlignment" Value="Center" />
    </Style>

</ResourceDictionary>
```

---

## 4. 실시간 AI 사고(Thinking) 및 서사(Narrative) 스트리밍 UI

현대 추론형 AI 모델의 **확장 사고(Extended CoT, Chain-of-Thought / Internal Reasoning) 패러다임**에 맞추어, **AI의 내부 추론/조사 과정(`<think>`)과 최종 분석 서사(Narrative)를 시각적으로 엄격히 분리**합니다.

```xml
<!-- 실시간 AI 추론 및 서사 스트리밍 타이포그래피 리소스 -->
<ResourceDictionary xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
                    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">

    <!-- AI 내면 추론 텍스트 스타일: 슬레이트 블루 + 이탤릭 -->
    <Style x:Key="StreamingThoughtTextStyle" TargetType="TextBlock">
        <Setter Property="FontSize" Value="13.5" />
        <Setter Property="LineHeight" Value="22" />
        <Setter Property="Foreground" Value="#A2B9D8" />
        <Setter Property="FontStyle" Value="Italic" />
        <Setter Property="TextWrapping" Value="Wrap" />
        <Setter Property="Padding" Value="16,12" />
    </Style>

    <!-- AI 최종 서사 리포트 텍스트 스타일: 차분한 화이트/그레이 -->
    <Style x:Key="StreamingNarrativeTextStyle" TargetType="TextBlock">
        <Setter Property="FontSize" Value="14.5" />
        <Setter Property="LineHeight" Value="24" />
        <Setter Property="Foreground" Value="#F2F4F8" />
        <Setter Property="TextWrapping" Value="Wrap" />
        <Setter Property="Padding" Value="16,12" />
    </Style>

    <!-- 접이식 추론 컨테이너 (Collapsible Thought Expander) -->
    <Style x:Key="ThoughtExpanderStyle" TargetType="Expander">
        <Setter Property="Background" Value="#10131B" />
        <Setter Property="BorderBrush" Value="#22FFFFFF" />
        <Setter Property="BorderThickness" Value="1" />
        <Setter Property="Foreground" Value="#A2B9D8" />
        <Setter Property="Padding" Value="8" />
        <Setter Property="Margin" Value="0,4,0,12" />
    </Style>

</ResourceDictionary>
```

### 무프리징 증분 렌더링 규칙
* LLM 토큰이 초당 수십 개 쏟아질 때 전체 문자열을 매번 다시 바인딩하면 UI 스레드 렌더 트리가 재구성되어 버벅임이 발생합니다.
* 반드시 `System.Threading.Channels`를 통해 백그라운드 큐에서 토큰을 청크(Chunk) 단위로 버퍼링한 후, `Dispatcher.InvokeAsync(..., DispatcherPriority.Render)`로 텍스트 끝에 증분 덧붙이기(Append)를 수행해야 합니다.

---

## 5. 인터랙티브 노드 그래프(Graph Canvas) 시각화 규칙

* **DirectX 하드웨어 가속 최적화**: 수백 개 이상의 프로세스 노드를 그릴 때 무거운 개별 WPF `UIElement` 대신, `DrawingVisual` 기반의 경량 렌더링 컨테이너를 채택하여 60fps 유지를 보장합니다.
* **노드 상태 비주얼 정의**:
  * **안전 노드 (WhiteList)**: `#10B981` (Emerald), 불투명 외곽선.
  * **조사 중 노드 (Active Investigating)**: `#F59E0B` (Amber), 방사형 펄스 애니메이션 적용.
  * **차단 노드 (Terminated)**: `#EF4444` (Crimson), 중앙 해골 아이콘 및 취소선 표시.
* **줌/팬 인터랙션**: 마우스 휠 기반 줌인/줌아웃 및 캔버스 드래그 패닝(Panning)을 기본 지원하여 대규모 트리 탐색을 지원합니다.

---

## 6. AI 에이전트 준수 XAML 안티패턴 (Strict Rules)

AI 에이전트가 WPF XAML 코드를 작성할 때 다음 안티패턴을 절대 발생시켜서는 안 됩니다:

1. **[금지] 기본 Windows 시스템 컨트롤 날것(Raw) 사용**:
   * 기본 회색 사각 버튼, 기본 시스템 콤보박스, 흰색 기본 창 크롬 사용 절대 금지. 모든 컨트롤은 정의된 리소스 딕셔너리 스타일을 상속받아야 합니다.
2. **[금지] 인라인 하드코딩 브러시 남발**:
   * `<Border Background="#123456">`처럼 인라인에 직접 컬러 코드를 박는 행위를 금지하며, `DynamicResource` 또는 사전 정의된 팔레트 키를 바인딩해야 합니다.
3. **[금지] UI 스레드 블로킹**:
   * gRPC 수신 이벤트나 AI 스트리밍 파이프라인에서 동기 대기(`.Wait()`, `.Result`)를 절대 호출하지 않습니다.
4. **[금지] 과도한 중첩 드롭 섀도우**:
   * 성능을 갉아먹는 블러 반경 20px 이상의 `DropShadowEffect` 사용을 금지하며, 루미넌스 스태킹과 미세 1px Border로 대체합니다.

---

## 7. 데스크톱 윈도우 프레임 통합 방법론 (Unified Window Title Bar Methodology)

본 절은 강제 규격이 아니며, VS Code, Obsidian, Linear와 같이 앱의 상단 바와 윈도우 타이틀바를 일체화(Seamless)하여 상단 세로 공간(약 30~32px)을 확보하고 프로 도구의 일체감을 부여하고자 할 때 활용할 수 있는 **선택적 고급 아키텍처 방법론**입니다.

### A. 배경 및 문제 의식
* **2단 헤더 중복**: 전통적인 데스크톱 창은 OS 비클라이언트 영역(Non-Client Area)의 타이틀바와 내부 UI 헤더가 나란히 적층되어, 프로그램 타이틀이 중복 노출되고 유효 작업 영역이 낭비되는 문제가 발생합니다.
* **WindowStyle="None"의 한계**: 단순히 OS 기본 프레임을 숨기면 윈도우 외곽 크기 조절(Resize), Windows 11 스냅 레이아웃(Snap Assist), 창 그림자(Drop Shadow), 최소화/최대화 네이티브 애니메이션이 손상됩니다.

### B. 핵심 구현 아키텍처 (`WindowChrome` 기반)

#### 1. 비클라이언트 억제 및 클라이언트 확장
`System.Windows.Shell.WindowChrome`을 적용하여 OS 기본 캡션 렌더링을 억제하고, XAML 클라이언트 영역을 윈도우의 최상단(0px)까지 확장합니다:

```xml
<WindowChrome.WindowChrome>
    <WindowChrome
        CaptionHeight="48"
        CornerRadius="0"
        GlassFrameThickness="0"
        NonClientFrameEdges="None"
        ResizeBorderThickness="5"
        UseAeroCaptionButtons="False" />
</WindowChrome.WindowChrome>
```
* `CaptionHeight`: 타이틀바 역할을 수행할 상단 바의 높이(예: 48px). 이 영역 내의 빈 공간을 클릭·드래그하면 창 이동이 발동하고, 더블 클릭 시 최대화/복원이 동작합니다.
* `ResizeBorderThickness`: 창 테두리 리사이징 감도를 유지합니다(통상 4~6px).
* `GlassFrameThickness="0"` / `UseAeroCaptionButtons="False"`: OS 기본 Aero 버튼과 프레임을 완전히 비활성화합니다.

#### 2. 상단 바 컨트롤의 클릭 관통 방지 (`IsHitTestVisibleInChrome`)
타이틀바 영역 내부의 인터랙티브 요소(로고, 검색창, 새로고침 버튼, 시스템 창 제어 버튼 등)는 창 드래그 이벤트에 가로채이지 않도록 명시적으로 히트 테스트를 활성화해야 합니다:

```xml
<Button Command="{Binding RefreshCommand}"
        WindowChrome.IsHitTestVisibleInChrome="True" />
```

#### 3. 윈도우 시스템 캡션 컨트롤 및 네이티브 명령 연동
WPF의 `SystemCommands`를 활용하여 Windows 고유의 부드러운 전환 애니메이션을 보장합니다:

```csharp
private void MinimizeButton_Click(object sender, RoutedEventArgs e) => SystemCommands.MinimizeWindow(this);
private void MaximizeButton_Click(object sender, RoutedEventArgs e)
{
    if (WindowState == WindowState.Maximized)
        SystemCommands.RestoreWindow(this);
    else
        SystemCommands.MaximizeWindow(this);
}
private void CloseButton_Click(object sender, RoutedEventArgs e) => SystemCommands.CloseWindow(this);
```

#### 4. 창 최대화 시 모니터 가장자리 잘림(Overshoot) 보정
`WindowChrome`이 적용된 창이 최대화될 때, Windows OS의 비클라이언트 메트릭스로 인해 창 경계가 모니터 바깥으로 7~8px 확장되는 현상이 발생합니다. 루트 그리드 컨테이너에 상태 트리거 마진을 적용하여 작업 표시줄 오버레이 및 경계 잘림을 보정합니다:

```xml
<Grid>
    <Grid.Style>
        <Style TargetType="Grid">
            <Setter Property="Margin" Value="0" />
            <Style.Triggers>
                <DataTrigger Binding="{Binding RelativeSource={RelativeSource AncestorType=Window}, Path=WindowState}" Value="Maximized">
                    <Setter Property="Margin" Value="7" />
                </DataTrigger>
            </Style.Triggers>
        </Style>
    </Grid.Style>
    <!-- 내부 레이아웃 -->
</Grid>
```

#### 5. DWM 심층 다크 모드 속성과의 하이브리드 결합
상단 바를 직접 그려도, 키보드 `Alt + Space` 입력 시 나타나는 OS 창 제어 시스템 팝업, 창 외곽 DWM 섀도우, Windows 11 스냅 가이드라인이 어색한 흰색으로 뜨지 않도록 `DwmSetWindowAttribute`의 `DWMWA_USE_IMMERSIVE_DARK_MODE` 속성을 함께 활성화하는 것을 권장합니다:

```csharp
[DllImport("dwmapi.dll", PreserveSig = true)]
private static extern int DwmSetWindowAttribute(IntPtr hwnd, int attr, ref int attrValue, int attrSize);

private const int DWMWA_USE_IMMERSIVE_DARK_MODE = 20;     // Win 10 20H1+ 및 Win 11
private const int DWMWA_USE_IMMERSIVE_DARK_MODE_OLD = 19; // Win 10 1809 - 1909

int useImmersiveDarkMode = 1;
if (DwmSetWindowAttribute(hwnd, DWMWA_USE_IMMERSIVE_DARK_MODE, ref useImmersiveDarkMode, sizeof(int)) != 0)
{
    DwmSetWindowAttribute(hwnd, DWMWA_USE_IMMERSIVE_DARK_MODE_OLD, ref useImmersiveDarkMode, sizeof(int));
}
```
*(참고: WindowChrome으로 자체 타이틀바를 렌더링하는 경우, `DWMWA_CAPTION_COLOR` 및 `DWMWA_TEXT_COLOR`는 렌더 트리에 반영되지 않으므로 불필요한 P/Invoke를 생략할 수 있습니다.)*

