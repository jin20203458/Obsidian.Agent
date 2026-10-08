---
description: WPF 현대적 MVVM 아키텍처, 수명주기 관리, 고성능 스트리밍 및 반응형 UI 설계 지침서.
related:
  - ../README.md
  - ./Modern_AI_Desktop_UI_Guidelines.md
  - ../Phalanx/docs/01_system_architecture.md
---
# WPF Architecture Guidelines

본 문서는 현대적인 WPF(.NET 8.0/9.0/10.0) 애플리케이션을 구축할 때 AI 에이전트와 휴먼 엔지니어가 반드시 준수해야 하는 엔터프라이즈 아키텍처 단일 진실 공급원(SSOT) 지침서입니다.

과거 레거시 WPF(.NET Framework 시절의 관습, 무거운 서드파티 라이브러리 남용, 수동 INPC 구현)를 완전히 배제하고, 실제 상용급 대형 프로젝트(보안 관제 엔진 Phalanx, 대규모 정적 분석기 ARQA, 멀티모달 대화형 AI 도구 GRC)에서 검증된 고성능 아키텍처 패턴을 집대성하여 제공합니다.

---

## 1. 아키텍처 원형(Archetypes) 비교 및 선택 가이드

WPF 애플리케이션은 목적과 데이터 처리량에 따라 구조적 접근이 달라져야 합니다. 아래의 세 가지 아키텍처 원형 중 프로젝트의 성격에 부합하는 모델을 우선 선정합니다.

| 분류 항목 | 원형 1: 순수 ViewModel-First 단일 셸 (GRC 패턴) | 원형 2: 모듈형 셸 + 레일 + CQRS 리드 모델 (Phalanx 패턴) | 원형 3: 멀티 도구 분석 워크벤치 (ARQA 패턴) |
| :--- | :--- | :--- | :--- |
| **적합한 앱 유형** | 대화형 AI 클라이언트, 설정 마법사, 단일 워크플로 생산성 도구 | 실시간 보안 관제(SOC), 텔레메트리 대시보드, 고처리량 스트리밍 콘솔 | 정적 코드 분석기, 진단 도구, 복합 데이터그리드 워크벤치 |
| **창/네비게이션 구조** | 초경량 `MainWindow`(30줄) + `<ContentControl Content="{Binding CurrentPage}" />` | 52px 슬림 좌측 레일 + 모듈별 `UserControl` 교체 | 다중 도킹 패널, 탭 기반 워크스페이스, 다차원 분석 뷰 |
| **뷰-뷰모델 결합** | 암시적 `DataTemplate` 자동 맵핑 (View 코드-비하인드 제로) | 명시적 레일 커맨드 기반 모듈 활성화 | 영역별 독립 뷰모델 분할 및 중앙 오케스트레이터 |
| **수명주기 모델** | `Transient` 등록 + 화면 전환 시 `ICleanup`을 통한 명시적 메모리 회수 | `Singleton` 코어 엔진 + 장기 실행 텔레메트리 수신 파이프라인 | 도구별 독립 워크스페이스 수명주기 |
| **데이터 처리 특성** | 실시간 LLM 토큰 증분 수신, 음성/오디오 스트리밍 파이프라인 | 초당 수천 건 ETW/gRPC 이벤트 버퍼링, 배치 플러시 프로젝션 | 대용량 소스코드 파싱 트리, 수만 건 진단 로그 가상화 렌더링 |

### A. 원형 1: 순수 ViewModel-First 네비게이션 (Recommended for Modern AI Apps)
* **핵심 철학**: 뷰는 뷰모델의 상태 표현체일 뿐이며, 화면 전환은 `CurrentPage` 프로퍼티의 참조 교체로만 완결됩니다.
* **구현 방식**:
  1. `MainWindow.xaml`에는 전체 크롬과 상단 바, 그리고 페이지를 담을 `<ContentControl Content="{Binding CurrentPage}" />`만 배치합니다.
  2. `ResourceDictionary` 내에 `<DataTemplate DataType="{x:Type vm:ChatViewModel}"><views:ChatView /></DataTemplate>`를 선언하여 WPF 런타임이 타입에 맞춰 뷰를 자동 인스턴스화하도록 위임합니다.
  3. 페이지 전환 시 이전 뷰모델의 이벤트 구독 해제와 백그라운드 태스크 취소를 보장하는 `ICleanup` 패턴을 결합합니다.

### B. 원형 2: 엔터프라이즈 모듈형 셸 + 네비게이션 레일 + CQRS
* **핵심 철학**: 무중단 데이터 유입(gRPC, 소켓, ETW) 환경에서 렌더링 부하를 비동기 CQRS 리드 모델로 격리합니다.
* **구현 방식**:
  1. 52px 너비의 아이콘 기반 슬림 네비게이션 레일을 좌측에 고정합니다.
  2. 고빈도 원시 텔레메트리는 백그라운드 스레드의 채널/락-스왑 큐에 축적하고, UI 렌더링은 30~60Hz 타이머를 통해 인메모리 프로젝션 스냅샷 형태로 일괄 반영(Batch Flush)합니다.
  3. 단위 테스트 및 CLI/헤드리스 환경에서 WPF 런타임 없이도 동작할 수 있도록 디스패처 호출을 브릿지(`CockpitUiBridge`)로 추상화합니다.

### C. 원형 3: 멀티 도구 분석 워크벤치
* **핵심 철학**: 복잡한 계측/진단 화면을 다루되, 거대 윈도우(God Window) 및 거대 뷰모델(God ViewModel) 안티패턴을 철저히 방지합니다.
* **구현 방식**:
  1. 각 진단 영역을 독립적인 서브 뷰모델과 `UserControl`로 격리하고, 부모 뷰모델은 이들의 조합과 라이프사이클만 제어합니다.
  2. 수만 행의 분석 데이터를 렌더링할 때는 UI 가상화와 컨테이너 재활용(Recycling)을 강제합니다.

---

## 2. 프로젝트 표준 디렉터리 레이아웃

역할과 관심사의 분리를 명확히 하기 위해 아래의 표준 디렉터리 배치를 준수합니다.

```text
ProjectRoot/
├── Config/               # JSON 설정, AppSettings, 런타임 프로파일
├── Models/               # 순수 데이터 구조체 (POCO), DTO, 프로토콜 엔티티
├── Services/             # 비즈니스 로직, API/gRPC 통신, IPC 클라이언트
│   ├── Abstractions/     # 서비스 인터페이스 (IApiService, INavigationService)
│   └── Implementations/  # 구체 클래스 구현체
├── ViewModels/           # UI 프레젠테이션 로직 (CommunityToolkit.Mvvm)
│   ├── Common/           # ViewModelBase, ICleanup, PageViewModelBase
│   └── Pages/            # 각 화면별 전용 뷰모델
├── Views/                # XAML 선언 및 순수 UI 비하인드 코드
│   ├── Controls/         # 재사용 가능한 커스텀 UserControl
│   ├── Dialogs/          # 인앱 모달 및 오버레이 뷰
│   └── Pages/            # 화면 본체 뷰
├── Themes/               # 디자인 시스템 및 XAML 리소스 사전
│   ├── Tokens.xaml       # 컬러 팔레트, 브러시, 폰트 규격
│   ├── ControlStyles.xaml# 버튼, 체크박스, 콤보박스 등 기본 컨트롤 스타일 재정의
│   └── DataTemplates.xaml# ViewModel-to-View 암시적 맵핑 정의
├── App.xaml              # 시작점 및 전역 MergedDictionaries 등록
├── App.xaml.cs           # Generic Host 및 DI 컨테이너 구성
└── Project.csproj        # 최신 SDK 스타일 프로젝트 파일
```

---

## 3. 의존성 주입(DI) 및 수명주기 관리 표준

`Microsoft.Extensions.DependencyInjection`을 표준 컨테이너로 사용하며, 서비스와 뷰모델의 수명주기를 엄격히 구분합니다.

* **필수 및 권장 NuGet 패키지 의존성**:
  - `Microsoft.Extensions.Hosting` (Generic Host 및 호스트 수명주기 관리 - 필수)
  - `Microsoft.Extensions.DependencyInjection` (의존성 주입 컨테이너 - 필수)
  - `Microsoft.Extensions.Http` (`AddHttpClient` 팩토리 패턴 지원 - 필수)
  - `CommunityToolkit.Mvvm` (버전 8.3/8.4+ 소스 제너레이터 - 필수)
  - `System.Reactive` (선택: Rx 기반 비동기 반응형 이벤트 스트리밍 파이프라인 구성 시 권장)

### A. 서비스 수명주기 원칙
* **Singleton**: 통신 클라이언트(`HttpClient`, `GrpcChannel`), 전역 상태 저장소, 이벤트 중계자(`IMessenger`), 설정 관리자.
* **Transient**: 화면 뷰모델(`PageViewModel`), 단발성 다이얼로그 뷰모델.
  * 이유: 페이지를 닫거나 이동할 때 이전 화면의 상태를 완전 소멸시키고 메모리 누수를 원천 방지하기 위함.
* **Scoped**: 단일 세션 또는 특정 작업 단위(Unit-of-Work)에 종속된 데이터 컨텍스트.

### B. `App.xaml.cs` 구성 표준

```csharp
using System;
using System.Windows;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using MyWpfApp.Services.Abstractions;
using MyWpfApp.Services.Implementations;
using MyWpfApp.ViewModels;
using MyWpfApp.ViewModels.Pages;
using MyWpfApp.Views;

namespace MyWpfApp
{
    public partial class App : Application
    {
        private readonly IHost _host;

        public static new App Current => (App)Application.Current;
        public IServiceProvider Services => _host.Services;

        public App()
        {
            _host = Host.CreateDefaultBuilder()
                .ConfigureServices((context, services) =>
                {
                    ConfigureServices(services);
                })
                .Build();
        }

        private static void ConfigureServices(IServiceCollection services)
        {
            // 1. 단일 인스턴스 전역 인프라 및 서비스
            services.AddSingleton<INavigationService, NavigationService>();
            services.AddSingleton<ISettingsService, LocalSettingsService>();

            // 2. HTTP Factory 구성 (소켓 누수 방지)
            services.AddHttpClient<IApiClient, ApiClient>(client =>
            {
                client.BaseAddress = new Uri("https://api.example.com/");
                client.Timeout = TimeSpan.FromSeconds(30);
            });

            // 3. 메인 셸 (Singleton)
            services.AddSingleton<MainViewModel>();
            services.AddSingleton<MainWindow>();

            // 4. 화면별 페이지 뷰모델 (Transient: 전환 시마다 새로 생성 및 소멸)
            services.AddTransient<DashboardViewModel>();
            services.AddTransient<ChatViewModel>();
            services.AddTransient<DiagnosticsViewModel>();
        }

        protected override async void OnStartup(StartupEventArgs e)
        {
            base.OnStartup(e);
            await _host.StartAsync();

            var mainWindow = Services.GetRequiredService<MainWindow>();
            mainWindow.DataContext = Services.GetRequiredService<MainViewModel>();
            mainWindow.Show();
        }

        protected override async void OnExit(ExitEventArgs e)
        {
            using (_host)
            {
                await _host.StopAsync(TimeSpan.FromSeconds(5));
            }
            base.OnExit(e);
        }
    }
}
```

### C. `ICleanup`을 통한 페이지 이탈 시 자원 해제 패턴 (GRC 검증)

뷰모델이 `Transient`로 생성되더라도, 내부에서 백그라운드 스트리밍을 수행하거나 이벤트 핸들러를 구독 중이면 GC가 수거하지 못합니다. 명시적인 `ICleanup` 계약을 통해 화면 전환 즉시 리소스를 해제해야 합니다.

```csharp
public interface ICleanup
{
    void Cleanup();
}

// ViewModel 구현체
public partial class ChatViewModel : ObservableObject, ICleanup
{
    private readonly CancellationTokenSource _cts = new();

    public void Cleanup()
    {
        // 1. 백그라운드 비동기 스트리밍 중단
        _cts.Cancel();
        _cts.Dispose();

        // 2. 오디오/미디어 플레이어 정리
        _audioPlayer?.Dispose();

        // 3. 컬렉션 바인딩 해제
        Messages.Clear();
    }
}
```

### D. 헤드리스(Headless) 환경 및 테스트 안전성 가드 (Phalanx 검증)

WPF UI 스레드 마샬링 코드(`Application.Current.Dispatcher`)는 단위 테스트 러너나 CLI 배치 실행 시 `Application.Current`가 `null`이 되어 NullReferenceException을 발생시킵니다. 이를 추상화한 안전 브릿지를 사용해야 합니다.

```csharp
public sealed class UiDispatcherBridge
{
    public static UiDispatcherBridge Instance { get; } = new();

    public void Invoke(Action action)
    {
        if (Application.Current?.Dispatcher != null && !Application.Current.Dispatcher.CheckAccess())
        {
            Application.Current.Dispatcher.Invoke(action);
        }
        else
        {
            // 헤드리스 테스트 환경이거나 이미 UI 스레드인 경우 직접 실행
            action();
        }
    }

    public async Task InvokeAsync(Action action)
    {
        if (Application.Current?.Dispatcher != null)
        {
            await Application.Current.Dispatcher.InvokeAsync(action);
        }
        else
        {
            action();
        }
    }
}
```

---

## 4. CommunityToolkit.Mvvm 소스 제너레이터 구현 표준

최신 C# 소스 제너레이터(CommunityToolkit.Mvvm 8.3/8.4+)를 통해 보일러플레이트를 완전히 제거하고 컴파일 타임 최적화를 실현합니다.

### A. 핵심 선언 규칙
1. **`partial` 클래스 및 `ObservableObject` 상속 강제**:
   ```csharp
   public partial class DashboardViewModel : ObservableObject
   ```
2. **필드 네이밍 규격**:
   * 속성 생성 대상 필드는 반드시 private `_camelCase`로 선언합니다. 소스 제너레이터가 public `CamelCase` 프로퍼티를 자동 생성합니다.
   * PascalCase 필드 선언 절대 금지 (컴파일 에러 발생).
3. **파생 속성 알림 (`[NotifyPropertyChangedFor]`)**:
   * 계산된 읽기 전용 프로퍼티가 의존하는 필드에 직접 부착합니다.
4. **비동기 커맨드 (`[RelayCommand]`)**:
   * 커맨드 핸들러는 반드시 `Task`를 반환해야 합니다 (`async void` 절대 금지).
   * 실행 상태 연동은 `[NotifyCanExecuteChangedFor(nameof(SubmitCommand))]`를 필드에 명시합니다.

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;

public partial class DashboardViewModel : ObservableObject
{
    private readonly IDataProcessingService _dataService;

    public DashboardViewModel(IDataProcessingService dataService)
    {
        _dataService = dataService;
    }

    [ObservableProperty]
    [NotifyPropertyChangedFor(nameof(StatusDescription))]
    [NotifyCanExecuteChangedFor(nameof(ProcessDataCommand))]
    private bool _isProcessing;

    [ObservableProperty]
    private int _processedItemCount;

    public string StatusDescription => IsProcessing ? "데이터 분석 및 처리 진행 중..." : $"대기 중 (처리 완료: {ProcessedItemCount}건)";

    private bool CanProcessData() => !IsProcessing;

    [RelayCommand(CanExecute = nameof(CanProcessData))]
    private async Task ProcessDataAsync(CancellationToken cancellationToken)
    {
        IsProcessing = true;
        try
        {
            ProcessedItemCount = await _dataService.ExecuteProcessingAsync(cancellationToken);
        }
        catch (OperationCanceledException)
        {
            // 작업 취소 정상 수렴
        }
        finally
        {
            IsProcessing = false;
        }
    }
}
```

### B. `WeakReferenceMessenger`를 통한 컴포넌트 간 비결합 통신

서로 다른 뷰모델 간의 상태 공유는 직접 참조를 피하고 약한 참조 메시징을 사용합니다. 람다 식 내부에서 `this` 인스턴스를 캡처하면 메모리 누수가 발생하므로 `IRecipient<T>` 인터페이스를 사용합니다.

```csharp
using System;
using System.Collections.ObjectModel;
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Messaging;

// 1. 불변 메시지 레코드 정의 (도메인 중립적 이벤트 알림)
public sealed record BatchCompletedMessage(string BatchId, int ItemCount, DateTime Timestamp);

// 2. 수신 뷰모델 구현
public partial class NotificationPanelViewModel : ObservableRecipient, IRecipient<BatchCompletedMessage>
{
    public ObservableCollection<string> NotificationLogs { get; } = new();

    public NotificationPanelViewModel()
    {
        // 메시지 자동 등록 활성화 (IsActive가 true로 변경될 때 수신 등록)
        IsActive = true;
    }

    public void Receive(BatchCompletedMessage message)
    {
        // 수신된 비동기 메시지 UI 컬렉션에 추가
        NotificationLogs.Insert(0, $"[{message.Timestamp:HH:mm:ss}] 배치 완료 {message.BatchId}: {message.ItemCount}건");
    }
}

// 3. 송신 측 호출 (어디서나 타입 기반 브로드캐스트)
WeakReferenceMessenger.Default.Send(new BatchCompletedMessage("BATCH-2026-001", 1500, DateTime.UtcNow));
```

---

## 5. 초고속 스트리밍 데이터 및 CQRS 리드 모델 (High-Throughput Concurrency)

보안 관제 텔레메트리나 AI 토큰 스트리밍과 같이 초당 수백~수천 건의 이벤트가 유입될 때 `ObservableCollection`에 이벤트를 건별로 즉시 추가하면 UI 스레드가 락(Lock)에 걸려 애플리케이션이 멈춥니다.

### A. CQRS 인메모리 프로젝션 아키텍처

```
[Raw Event Producer (gRPC/ETW/LLM)]
                 │  (Non-blocking Write)
                 ▼
     [System.Threading.Channels / Lock-Swap Queue]
                 │
                 ▼  (Background Processing Task)
       [In-Memory Projection State]
                 │
                 ▼  (Periodic Batch Timer: 30~60Hz)
 [Dispatcher.InvokeAsync(DispatcherPriority.Background)]
                 │
                 ▼
    [UI ObservableCollection Projection]
```

### B. 배치 플러시 프로젝터 구현체 (Phalanx 검증)

```csharp
using System;
using System.Collections.Concurrent;
using System.Collections.ObjectModel;
using System.Threading;
using System.Threading.Tasks;
using System.Windows;
using System.Windows.Threading;

public sealed class ThrottledStreamProjector<T> : IDisposable
{
    private readonly ConcurrentQueue<T> _incomingQueue = new();
    private readonly ObservableCollection<T> _targetCollection;
    private readonly DispatcherTimer _flushTimer;
    private readonly int _maxBatchSizePerTick;

    public ThrottledStreamProjector(ObservableCollection<T> targetCollection, int fps = 30, int maxBatchSize = 100)
    {
        _targetCollection = targetCollection;
        _maxBatchSizePerTick = maxBatchSize;

        _flushTimer = new DispatcherTimer(DispatcherPriority.Background)
        {
            Interval = TimeSpan.FromMilliseconds(1000.0 / fps)
        };
        _flushTimer.Tick += OnFlushTick;
        _flushTimer.Start();
    }

    // 백그라운드 스레드에서 자유롭게 호출 (락 경합 없음)
    public void Enqueue(T item)
    {
        _incomingQueue.Enqueue(item);
    }

    private void OnFlushTick(object? sender, EventArgs e)
    {
        if (_incomingQueue.IsEmpty) return;

        int processed = 0;
        while (processed < _maxBatchSizePerTick && _incomingQueue.TryDequeue(out var item))
        {
            _targetCollection.Add(item);
            processed++;
        }

        // 최대 노출 항목 수 초과 시 오래된 항목 트리밍
        while (_targetCollection.Count > 1000)
        {
            _targetCollection.RemoveAt(0);
        }
    }

    public void Dispose()
    {
        _flushTimer.Stop();
    }
}
```

---

## 6. UI 가상화 및 렌더링 성능 엔지니어링

수천 개의 항목을 렌더링하는 `ListView`, `ListBox`, `DataGrid`에서 프레임 드랍을 방지하려면 UI 가상화 조건을 완벽히 충족해야 합니다.

### A. 가상화 붕괴를 막는 4대 원칙
1. **무한 크기 컨테이너 내부 중첩 금지**:
   * `ScrollViewer` 내부에 `ListView`를 넣거나, `StackPanel` 내부에 `DataGrid`를 넣으면 자식 컨트롤의 높이가 무한대로 계산되어 가상화가 즉시 무효화됩니다.
   * 해결: 고정 높이 또는 `Grid`의 `RowDefinition Height="*"` 내부에 배치합니다.
2. **`ScrollViewer.CanContentScroll="True"` 유지**:
   * 기본값이 `True`이나, 픽셀 단위 부드러운 스크롤을 구현하겠다고 `False`로 변경하면 컨테이너 가상화가 해제됩니다.
3. **컨테이너 재활용 모드 활성화**:
   * `VirtualizingPanel.VirtualizationMode="Recycling"`을 명시하여 스크롤 시 시각적 요소를 파괴/재생성하지 않고 재사용합니다.
4. **`DataTemplate` 내부 Visual Tree 경량화**:
   * 아이템 템플릿 내부에 무거운 `DropShadowEffect`나 깊은 그리드 중첩을 배제합니다.

### B. 표준 가상화 XAML 구성

```xml
<ListView ItemsSource="{Binding TelemetryEvents}"
          VirtualizingStackPanel.IsVirtualizing="True"
          VirtualizingPanel.VirtualizationMode="Recycling"
          ScrollViewer.CanContentScroll="True"
          ScrollViewer.VerticalScrollBarVisibility="Auto"
          ScrollViewer.HorizontalScrollBarVisibility="Disabled"
          BorderThickness="0"
          Background="Transparent">
    <ListView.ItemsPanel>
        <ItemsPanelTemplate>
            <VirtualizingStackPanel IsVirtualizing="True"
                                   VirtualizationMode="Recycling" />
        </ItemsPanelTemplate>
    </ListView.ItemsPanel>
    <ListView.ItemTemplate>
        <DataTemplate>
            <Border Height="32"
                    BorderBrush="#1AFFFFFF"
                    BorderThickness="0,0,0,1"
                    Padding="12,0">
                <Grid>
                    <Grid.ColumnDefinitions>
                        <ColumnDefinition Width="80" />
                        <ColumnDefinition Width="120" />
                        <ColumnDefinition Width="*" />
                    </Grid.ColumnDefinitions>
                    <TextBlock Grid.Column="0" Text="{Binding Timestamp, StringFormat='{}{0:HH:mm:ss.fff}'}" Foreground="#AAB2C0" VerticalAlignment="Center" />
                    <TextBlock Grid.Column="1" Text="{Binding EventType}" FontWeight="SemiBold" Foreground="#F2F4F8" VerticalAlignment="Center" />
                    <TextBlock Grid.Column="2" Text="{Binding Summary}" TextTrimming="CharacterEllipsis" Foreground="#F2F4F8" VerticalAlignment="Center" />
                </Grid>
            </Border>
        </DataTemplate>
    </ListView.ItemTemplate>
</ListView>
```

---

## 7. 디자인 시스템 및 커스텀 컨트롤 템플릿 아키텍처 (Lessons from ARQA & Phalanx)

상용 무거운 UI 프레임워크(예: 수백 KB의 종속성을 끌어오는 복잡한 라이브러리)에 의존하는 대신, 순수 네이티브 WPF XAML 템플릿을 통해 초경량 프로페셔널 도구 미학을 구현합니다.

### A. 리소스 사전 계층 구조
1. **`Tokens.xaml`**: 원시 컬러(`Color`), SolidColorBrush, 폰트 패밀리, 코너 반경 정의.
2. **`ControlStyles.xaml`**: `OverridesDefaultStyle="True"`를 활용하여 시스템 기본 컨트롤(회색 윈도우 95 스타일)을 전면 교체.
3. **`DataTemplates.xaml`**: 뷰모델과 뷰의 암시적 결합.
4. **`EnterpriseTheme.xaml`**: 상기 리소스들을 병합(Merge)하여 앱 전역에 주입.

### B. ARQA/Phalanx 검증: 기본 컨트롤 완전 재정의 템플릿 (`ControlStyles.xaml`)

```xml
<ResourceDictionary xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
                    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">

    <!-- 1. 기본 버튼 재정의 -->
    <Style TargetType="Button">
        <Setter Property="OverridesDefaultStyle" Value="True" />
        <Setter Property="Background" Value="#1A1D27" />
        <Setter Property="Foreground" Value="#F2F4F8" />
        <Setter Property="BorderBrush" Value="#22FFFFFF" />
        <Setter Property="BorderThickness" Value="1" />
        <Setter Property="Padding" Value="14,8" />
        <Setter Property="FontFamily" Value="Segoe UI Variable Text, Malgun Gothic" />
        <Setter Property="FontSize" Value="13" />
        <Setter Property="Cursor" Value="Hand" />
        <Setter Property="SnapsToDevicePixels" Value="True" />
        <Setter Property="Template">
            <Setter.Value>
                <ControlTemplate TargetType="Button">
                    <Border x:Name="border"
                            Background="{TemplateBinding Background}"
                            BorderBrush="{TemplateBinding BorderBrush}"
                            BorderThickness="{TemplateBinding BorderThickness}"
                            CornerRadius="6"
                            Padding="{TemplateBinding Padding}">
                        <ContentPresenter HorizontalAlignment="Center"
                                          VerticalAlignment="Center"
                                          RecognizesAccessKey="True" />
                    </Border>
                    <ControlTemplate.Triggers>
                        <Trigger Property="IsMouseOver" Value="True">
                            <Setter TargetName="border" Property="Background" Value="#252A38" />
                            <Setter TargetName="border" Property="BorderBrush" Value="#4F46E5" />
                        </Trigger>
                        <Trigger Property="IsPressed" Value="True">
                            <Setter TargetName="border" Property="Background" Value="#151822" />
                        </Trigger>
                        <Trigger Property="IsEnabled" Value="False">
                            <Setter TargetName="border" Property="Opacity" Value="0.4" />
                        </Trigger>
                    </ControlTemplate.Triggers>
                </ControlTemplate>
            </Setter.Value>
        </Setter>
    </Style>

    <!-- 2. 입력 텍스트박스 재정의 -->
    <Style TargetType="TextBox">
        <Setter Property="OverridesDefaultStyle" Value="True" />
        <Setter Property="Background" Value="#13141C" />
        <Setter Property="Foreground" Value="#F2F4F8" />
        <Setter Property="CaretBrush" Value="#4F46E5" />
        <Setter Property="BorderBrush" Value="#22FFFFFF" />
        <Setter Property="BorderThickness" Value="1" />
        <Setter Property="Padding" Value="10,7" />
        <Setter Property="FontSize" Value="13" />
        <Setter Property="SnapsToDevicePixels" Value="True" />
        <Setter Property="Template">
            <Setter.Value>
                <ControlTemplate TargetType="TextBox">
                    <Border x:Name="border"
                            Background="{TemplateBinding Background}"
                            BorderBrush="{TemplateBinding BorderBrush}"
                            BorderThickness="{TemplateBinding BorderThickness}"
                            CornerRadius="6">
                        <ScrollViewer x:Name="PART_ContentHost"
                                      Focusable="False"
                                      HorizontalScrollBarVisibility="Hidden"
                                      VerticalScrollBarVisibility="Hidden"
                                      Padding="{TemplateBinding Padding}" />
                    </Border>
                    <ControlTemplate.Triggers>
                        <Trigger Property="IsFocused" Value="True">
                            <Setter TargetName="border" Property="BorderBrush" Value="#4F46E5" />
                            <Setter TargetName="border" Property="Background" Value="#171A24" />
                        </Trigger>
                        <Trigger Property="IsEnabled" Value="False">
                            <Setter TargetName="border" Property="Opacity" Value="0.4" />
                        </Trigger>
                    </ControlTemplate.Triggers>
                </ControlTemplate>
            </Setter.Value>
        </Setter>
    </Style>

    <!-- 3. 고밀도 스크롤바 미니멀화 -->
    <Style TargetType="ScrollBar">
        <Setter Property="OverridesDefaultStyle" Value="True" />
        <Setter Property="Width" Value="8" />
        <Setter Property="Template">
            <Setter.Value>
                <ControlTemplate TargetType="ScrollBar">
                    <Grid Background="Transparent">
                        <Track x:Name="PART_Track" IsDirectionReversed="True">
                            <Track.Thumb>
                                <Thumb>
                                    <Thumb.Template>
                                        <ControlTemplate TargetType="Thumb">
                                            <Border Background="#33FFFFFF"
                                                    CornerRadius="4"
                                                    Margin="1,0" />
                                        </ControlTemplate>
                                    </Thumb.Template>
                                </Thumb>
                            </Track.Thumb>
                        </Track>
                    </Grid>
                </ControlTemplate>
            </Setter.Value>
        </Setter>
    </Style>

</ResourceDictionary>
```

---

## 8. AI 에이전트 절대 금지 안티패턴 카탈로그 (Strict Anti-Patterns)

AI 에이전트가 코드를 생성하거나 리팩토링할 때 다음의 안티패턴을 발생시키면 컴파일러 경고나 런타임 결함이 유발됩니다.

###  [안티패턴 1] 갓 윈도우(God Window) & 갓 뷰모델(God ViewModel) (ARQA 교훈)
* **결함 증상**: 단일 `MainWindow.xaml`이 2,000줄을 초과하고, `MainViewModel.cs`가 모든 탭과 기능의 상태 프로퍼티(수십 개)를 한곳에 선언함.
* **해결책**:
  * 각 탭/모듈은 반드시 독립적인 `UserControl`과 서브 뷰모델로 분리합니다.
  * XAML 파일 하나당 400줄, 뷰모델 파일 하나당 300줄을 초과하지 않도록 컴포넌트화를 강제합니다.

###  [안티패턴 2] 다이얼로그 팝업 남발 (Dialog Sprawl)
* **결함 증상**: 사소한 사용자 입력이나 알림마다 OS 기본 다이얼로그(`new SubWindow().ShowDialog()`)를 띄워 멀티 윈도우 관리가 꼬이고 창 포커스가 손실됨.
* **해결책**:
  * 메인 윈도우 최상단 레이어에 `<Grid x:Name="ModalHost" Visibility="{Binding IsModalOpen, Converter={StaticResource BoolToVis}}" />` 형태의 인앱 오버레이 레이어를 구성합니다.

###  [안티패턴 3] 서비스 로케이터(Service Locator) 남용
* **결함 증상**: 뷰모델 생성자 내부나 메서드에서 `App.Current.Services.GetRequiredService<T>()` 또는 `Ioc.Default.Get<T>()`를 직접 호출함.
* **해결책**:
  * 모든 의존성은 반드시 생성자 매개변수를 통해서만 주입받습니다 (Constructor Injection).

###  [안티패턴 4] 비동기 블로킹 및 `async void`
* **결함 증상**:
  * UI 스레드에서 `task.Result`, `task.Wait()`, `task.GetAwaiter().GetResult()` 호출 -> 영구 데드락(Deadlock) 유발.
  * 이벤트나 커맨드에 `async void` 사용 -> 내부 예외 포착 불가로 애플리케이션 강제 종료.
* **해결책**:
  * 끝까지 `await`를 전파하고, 모든 커맨드는 `Task`를 반환하는 `[RelayCommand]`를 사용합니다.

###  [안티패턴 5] 뷰 비하인드 코드(`.xaml.cs`)에 비즈니스/통신 로직 작성
* **결함 증상**: 버튼 클릭 이벤트 핸들러(`Button_Click`) 내에서 API 호출, 데이터 변환, 로컬 파일 I/O를 직접 수행함.
* **해결책**:
  * 비하인드 코드는 순수한 UI 시각적 상호작용(예: 특정 애니메이션 트리거, WindowChrome 핸들러) 외에는 비워두고, 모든 로직은 뷰모델의 Command로 이관합니다.

---

## 9. 프로덕션 레디 보일러플레이트 (Production Blueprints)

### Blueprint 1: 순수 ViewModel-First 셸 (`MainWindow.xaml`)

```xml
<Window x:Class="MyWpfApp.Views.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:shell="clr-namespace:System.Windows.Shell;assembly=PresentationFramework"
        Title="Enterprise Pro App"
        Height="780" Width="1240"
        MinHeight="600" MinWidth="900"
        Background="#0B0C10"
        WindowStartupLocation="CenterScreen">

    <!-- 네이티브 WindowChrome 일체화 -->
    <shell:WindowChrome.WindowChrome>
        <shell:WindowChrome CaptionHeight="44"
                            CornerRadius="0"
                            GlassFrameThickness="0"
                            NonClientFrameEdges="None"
                            ResizeBorderThickness="5"
                            UseAeroCaptionButtons="False" />
    </shell:WindowChrome.WindowChrome>

    <Grid>
        <Grid.RowDefinitions>
            <RowDefinition Height="44" /> <!-- 커스텀 타이틀바 -->
            <RowDefinition Height="*" />  <!-- 메인 작업 영역 -->
        </Grid.RowDefinitions>

        <!-- 타이틀바 -->
        <Border Grid.Row="0" Background="#13141C" BorderBrush="#1AFFFFFF" BorderThickness="0,0,0,1">
            <Grid Margin="16,0">
                <TextBlock Text="ENTERPRISE PRO APPLICATION"
                           Foreground="#AAB2C0"
                           FontWeight="SemiBold"
                           FontSize="12"
                           VerticalAlignment="Center" />

                <!-- 시스템 창 제어 버튼 -->
                <StackPanel Orientation="Horizontal" HorizontalAlignment="Right" shell:WindowChrome.IsHitTestVisibleInChrome="True">
                    <Button Width="36" Height="30" Content="—" Click="MinimizeButton_Click" Background="Transparent" BorderThickness="0" />
                    <Button Width="36" Height="30" Content="☐" Click="MaximizeButton_Click" Background="Transparent" BorderThickness="0" />
                    <Button Width="36" Height="30" Content="✕" Click="CloseButton_Click" Background="Transparent" BorderThickness="0" />
                </StackPanel>
            </Grid>
        </Border>

        <!-- 본문: ViewModel-First 바인딩을 통해 CurrentPage에 따라 뷰 자동 렌더링 -->
        <ContentControl Grid.Row="1" Content="{Binding CurrentPage}" />
    </Grid>
</Window>
```

### Blueprint 2: `MainWindow.xaml.cs` (시스템 명령 연동)

```csharp
using System.Windows;

namespace MyWpfApp.Views
{
    public partial class MainWindow : Window
    {
        public MainWindow()
        {
            InitializeComponent();
        }

        private void MinimizeButton_Click(object sender, RoutedEventArgs e) => SystemCommands.MinimizeWindow(this);
        private void MaximizeButton_Click(object sender, RoutedEventArgs e)
        {
            if (WindowState == WindowState.Maximized)
                SystemCommands.RestoreWindow(this);
            else
                SystemCommands.MaximizeWindow(this);
        }
        private void CloseButton_Click(object sender, RoutedEventArgs e) => SystemCommands.CloseWindow(this);
    }
}
```

### Blueprint 3: `MainViewModel.cs` (수명주기 전환 오케스트레이터)

```csharp
using System;
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using Microsoft.Extensions.DependencyInjection;
using MyWpfApp.ViewModels.Common;
using MyWpfApp.ViewModels.Pages;

namespace MyWpfApp.ViewModels
{
    public partial class MainViewModel : ObservableObject
    {
        private readonly IServiceProvider _serviceProvider;

        [ObservableProperty]
        private ObservableObject? _currentPage;

        public MainViewModel(IServiceProvider serviceProvider)
        {
            _serviceProvider = serviceProvider;
            // 기본 시작 페이지로 이동
            NavigateTo<DashboardViewModel>();
        }

        [RelayCommand]
        public void NavigateTo(Type viewModelType)
        {
            // 이전 화면이 ICleanup을 구현했다면 명시적 정리 수행
            if (CurrentPage is ICleanup cleanable)
            {
                cleanable.Cleanup();
            }

            // DI 컨테이너에서 새 인스턴스 Resolve
            CurrentPage = (ObservableObject)_serviceProvider.GetRequiredService(viewModelType);
        }

        public void NavigateTo<T>() where T : ObservableObject => NavigateTo(typeof(T));
    }
}
```
