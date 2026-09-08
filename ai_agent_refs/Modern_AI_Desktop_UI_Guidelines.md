---
description: >-
  WPF 기반 차세대 데스크톱 AI 및 보안 관제(SOC) UI/UX 설계 지침서. 다크 테마 디자인 토큰, 캡슐형 컨트롤, 노드 그래프 및 실시간 AI 사고 스트리밍 XAML 구현 시 참조.
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

* **결정론적 관제 뼈대 (Deterministic Frame)**: `ArqaStatic`에서 검증된 1280px 이상의 대형 분할 레이아웃, 캡슐형 컨트롤, 다크 팔레트를 기반으로 엔터프라이즈 전문 도구의 안정감을 제공합니다.
* **지능형 추론 서사 (Agentic Narrative Soul)**: `GRC`에서 검증된 실시간 사고(Thought) 및 서사(Narrative) 스트리밍 타이포그래피를 이식하여, AI가 내린 판단의 설명 가능성(Explainability)과 생동감을 극대화합니다.
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

최신 추론 모델(o1, DeepSeek-R1, Claude 3.7)의 패러다임에 맞추어, **AI의 내부 추론 과정(`<think>`)과 최종 분석 서사(Narrative)를 시각적으로 엄격히 분리**합니다.

```xml
<!-- GRC 기반 스트리밍 타이포그래피 리소스 -->
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
