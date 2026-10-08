---
description: WPF 현대적 MVVM 아키텍처, 수명주기 관리, 고성능 스트리밍 및 반응형 UI 설계 지침서.
related:
  - ../README.md
  - ./Modern_AI_Desktop_UI_Guidelines.md
  - ../Phalanx/docs/01_system_architecture.md
---
# WPF Architecture Guidelines

본 문서는 현대적인 WPF(.NET 8.0/9.0/10.0) 애플리케이션을 초기 구축하거나 대규모 리팩토링할 때 AI 에이전트와 엔지니어가 반드시 준수해야 하는 **엔터프라이즈 아키텍처 단일 진실 공급원(SSOT) 마스터 지침서**입니다.

과거 .NET Framework 시절의 레거시 관행(수동 INPC 구현, 무거운 외부 래퍼 라이브러리 남용, 갓 윈도우/뷰모델 결합)을 완전히 배제하고, 실전 대규모 프로덕션(Phalanx EDR, ARQA 정적 분석기, GRC AI 클라이언트)의 실증 검증을 거친 **초고속 스트리밍, 엄격한 수명주기 메모리 회수, 모듈형 셸, 그리고 제로 결함(Zero Defect) 코드 패턴**을 집대성하여 제공합니다.

---

## 1. 아키텍처 원형(Archetypes) 비교 및 선택 가이드

WPF 애플리케이션은 시스템의 목적과 데이터 처리량에 따라 구조적 접근이 달라져야 합니다. 아래의 세 가지 아키텍처 원형 중 프로젝트의 성격에 부합하는 모델을 우선 선정합니다.

| 분류 항목 | 원형 1: 순수 ViewModel-First 단일 셸 (GRC 패턴) | 원형 2: 모듈형 셸 + 레일 + CQRS 리드 모델 (Phalanx 패턴) | 원형 3: 멀티 도구 분석 워크벤치 (ARQA 패턴) |
| :--- | :--- | :--- | :--- |
| **적합한 앱 유형** | 대화형 AI 클라이언트, 설정 마법사, 단일 워크플로 생산성 도구 | 실시간 보안 관제(SOC), 텔레메트리 대시보드, 클라우드 플릿 모니터링 | 정적 코드 분석기, 진단 도구, 복합 데이터그리드 IDE |
| **창/네비게이션 구조** | 초경량 `MainWindow`(50줄 이하) + `<ContentControl Content="{Binding CurrentPage}" />` | 52px 슬림 좌측 레일 + 모듈별 `UserControl` 교체 | 다중 분할 도킹 패널(`GridSplitter`), 탭 기반 워크스페이스 |
| **뷰-뷰모델 결합** | 암시적 `DataTemplate` 자동 맵핑 (View 코드-비하인드 제로) | 명시적 레일 커맨드 기반 모듈 활성화 | 영역별 독립 서브 뷰모델 분할 및 중앙 오케스트레이터 |
| **수명주기 모델** | `Transient` 등록 + 화면 전환 시 `ICleanup`을 통한 100% 메모리 회수 | `Singleton` 코어 엔진 + 장기 실행 텔레메트리 수신 파이프라인 | 세션별 도구 워크스페이스 수명주기 |
| **데이터 처리 특성** | 실시간 LLM 토큰 증분 수신, 음성/오디오 스트리밍 파이프라인 | 초당 수천 건 이벤트 버퍼링, 30~60Hz 배치 플러시 프로젝션 | 대용량 구문 분석 트리, 수만 건 진단 로그 가상화 렌더링 |

---

## 2. 프로젝트 표준 디렉터리 레이아웃

역할과 관심사의 분리를 명확히 하기 위해 아래의 표준 디렉터리 배치를 준수합니다.

```text
ProjectRoot/
├── Config/                   # JSON 설정, AppSettings, 런타임 프로파일
├── Models/                   # 순수 데이터 구조체 (POCO), DTO, 프로토콜 엔티티
├── Services/                 # 비즈니스 로직, API/gRPC 통신, IPC 클라이언트
│   ├── Abstractions/         # 서비스 인터페이스 (IDataService, INavigationService, IModalService)
│   └── Implementations/      # 구체 클래스 구현체 및 배치 프로젝터
├── ViewModels/               # UI 프레젠테이션 로직 (CommunityToolkit.Mvvm)
│   ├── Common/               # ViewModelBase, ICleanup, UiDispatcherBridge
│   ├── Dialogs/              # 전역 모달 뷰모델 (ConfirmModalViewModel 등)
│   └── Pages/                # 각 화면별 전용 뷰모델
├── Views/                    # XAML 선언 및 순수 UI 비하인드 코드
│   ├── Controls/             # 재사용 가능한 커스텀 UserControl (ToolCallCard 등)
│   ├── Dialogs/              # 인앱 모달 및 오버레이 뷰
│   └── Pages/                # 화면 본체 뷰
├── Themes/                   # 디자인 시스템 및 XAML 리소스 사전
│   ├── Tokens.xaml           # 컬러 팔레트, 브러시, 폰트 규격
│   ├── ControlStyles.xaml    # 버튼, 체크박스, 콤보박스, 스크롤바 등 기본 컨트롤 스타일 재정의
│   └── DataTemplates.xaml    # ViewModel-to-View 암시적 맵핑 정의
├── App.xaml                  # 시작점 및 전역 MergedDictionaries 등록 (StartupUri 절대 금지)
├── App.xaml.cs               # Generic Host 및 DI 컨테이너 구성, OnStartup 창 인스턴스화
└── Project.csproj            # 최신 SDK 스타일 프로젝트 파일
```

---

## 3. 의존성 주입(DI) 및 수명주기 관리 표준

`Microsoft.Extensions.DependencyInjection` 및 `Microsoft.Extensions.Hosting`을 표준 인프라로 사용하며, 서비스와 뷰모델의 수명주기를 엄격히 구분합니다.

### A. 필수 및 권장 NuGet 패키지 의존성
* `Microsoft.Extensions.Hosting` (최신 LTS 버전: Generic Host 및 호스트 수명주기 관리 - 필수)
* `Microsoft.Extensions.DependencyInjection` (의존성 주입 컨테이너 - 필수)
* `Microsoft.Extensions.Http` (`AddHttpClient` 팩토리 패턴 지원 및 소켓 누수 방지 - 필수)
* `CommunityToolkit.Mvvm` (버전 8.3/8.4+ 소스 제너레이터 - 필수)
* `System.Reactive` (선택: Rx 기반 비동기 반응형 이벤트 스트리밍 파이프라인 구성 시 권장)

### B. 서비스 수명주기 원칙
* **Singleton**: 통신 클라이언트(`HttpClient`, `GrpcChannel`), 전역 상태 저장소, 이벤트 중계자(`IMessenger`), 전역 모달 서비스(`IModalService`), 설정 관리자.
* **Transient**: 화면 뷰모델(`PageViewModel`), 단발성 다이얼로그 뷰모델.
  * 이유: 페이지를 이동하거나 닫을 때 이전 화면의 상태를 완전 소멸시키고 메모리 누수를 원천 방지하기 위함.
* **Scoped**: 단일 작업 세션 또는 특정 단위 작업(Unit-of-Work)에 종속된 데이터 컨텍스트.

### C. [중대 규칙] `App.xaml`의 `StartupUri` 제거 강제
`App.xaml.cs`의 `OnStartup`에서 DI 컨테이너를 통해 `MainWindow`를 인스턴스화하여 표시할 때, **`App.xaml`에 `StartupUri="MainWindow.xaml"`이 선언되어 있으면 WPF 런타임이 파라미터 없는 기본 생성자로 윈도우를 중복 생성하여 2개의 창이 뜨고 DI가 누락되는 치명적 결함**이 발생합니다.
따라서 `App.xaml`에서 `StartupUri` 속성은 반드시 삭제해야 합니다.

```xml
<!-- 올바른 App.xaml: StartupUri 속성이 없음 -->
<Application x:Class="MyWpfApp.App"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
    <Application.Resources>
        <ResourceDictionary>
            <ResourceDictionary.MergedDictionaries>
                <ResourceDictionary Source="Themes/EnterpriseTheme.xaml" />
            </ResourceDictionary.MergedDictionaries>
        </ResourceDictionary>
    </Application.Resources>
</Application>
```

### D. `App.xaml.cs` 완결 구성 템플릿

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
            services.AddSingleton<IModalService, ModalService>();
            services.AddSingleton<ISettingsService, LocalSettingsService>();
            services.AddSingleton<IDataService, DataProcessingService>();

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
            services.AddTransient<AnalyticsViewModel>();
            services.AddTransient<SettingsViewModel>();
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

### E. `ICleanup`을 통한 페이지 이탈 시 메모리 회수 패턴 (GRC 검증)

뷰모델이 `Transient`로 생성되더라도 백그라운드 태스크나 이벤트 핸들러를 구독 중이면 GC가 수거하지 못합니다. 명시적인 `ICleanup` 계약을 통해 화면 전환 즉시 리소스를 해제해야 합니다.

```csharp
namespace MyWpfApp.ViewModels.Common;

public interface ICleanup
{
    void Cleanup();
}

// ViewModel 구현체
public partial class DashboardViewModel : ObservableObject, ICleanup
{
    private readonly IDataService _dataService;
    private readonly CancellationTokenSource _cts = new();

    public DashboardViewModel(IDataService dataService)
    {
        _dataService = dataService;
        _dataService.OnDataArrived += HandleDataArrived;
    }

    public void Cleanup()
    {
        // 1. 이벤트 핸들러 명시적 분리 (메모리 누수 원천 차단)
        _dataService.OnDataArrived -= HandleDataArrived;

        // 2. 백그라운드 비동기 스트리밍 중단
        _cts.Cancel();
        _cts.Dispose();

        // 3. 내부 타이머/프로젝터 정리
        _projector?.Dispose();

        // 4. 컬렉션 바인딩 해제
        Items.Clear();
    }
}
```

### F. 헤드리스(Headless) 환경 및 단위 테스트 안전성 가드 (Phalanx 검증)

WPF UI 스레드 마샬링 코드(`Application.Current.Dispatcher`)는 단위 테스트 러너나 CLI 배치 실행 시 `Application.Current`가 `null`이 되어 크래시를 유발합니다. 이를 추상화한 안전 브릿지를 사용해야 합니다.

```csharp
namespace MyWpfApp.Services.Abstractions;

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
   * 속성 생성 대상 필드는 반드시 private `_camelCase`로 선언합니다. 소스 제너레이터가 public `PascalCase` 프로퍼티를 자동 생성합니다.
   * PascalCase 필드 선언 절대 금지 (컴파일 에러 발생).
3. **파생 속성 알림 (`[NotifyPropertyChangedFor]`)**:
   * 계산된 읽기 전용 프로퍼티가 의존하는 필드에 직접 부착합니다.
4. **비동기 커맨드 (`[RelayCommand]`)**:
   * 커맨드 핸들러는 반드시 `Task`를 반환해야 합니다 (`async void` 절대 금지).
   * 실행 상태 연동은 `[NotifyCanExecuteChangedFor(nameof(SubmitCommand))]`를 필드에 명시합니다.

### B. [치명적 안티패턴 방어] `async void`의 전면 금지 및 안전한 백그라운드 트리거
AI 에이전트가 `[RelayCommand]`에는 `Task`를 잘 적용하면서도, **내부 헬퍼 메서드나 이벤트 콜백을 작성할 때 습관적으로 `private async void DoWork()`를 작성하는 함정**에 자주 빠집니다. `async void` 메서드 내부에서 발생하는 예외는 호출자에서 포착할 수 없어 즉시 프로세스 비정상 종료(Crash)를 유발합니다.

* **원칙 1**: 모든 내부 비동기 메서드는 반드시 `Task`를 반환하도록 작성합니다.
* **원칙 2**: 최상위 이벤트 핸들러처럼 `void` 시그니처가 불가피한 경우, 반드시 `try-catch` 블록으로 전체를 감싸거나 안전 확장 메서드(`SafeFireAndForget`)를 사용합니다.

```csharp
// 안전한 비동기 백그라운드 호출 패턴
public static class TaskExtensions
{
    public static void SafeFireAndForget(this Task task, Action<Exception>? onError = null)
    {
        _ = task.ContinueWith(t =>
        {
            if (t.IsFaulted && t.Exception != null)
            {
                onError?.Invoke(t.Exception.GetBaseException());
            }
        }, TaskScheduler.Default);
    }
}
```

### C. 완전한 뷰모델 표준 예시

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using MyWpfApp.Services.Abstractions;

namespace MyWpfApp.ViewModels.Pages;

public partial class DashboardViewModel : ObservableObject
{
    private readonly IDataService _dataService;

    public DashboardViewModel(IDataService dataService)
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

### D. `WeakReferenceMessenger`를 통한 컴포넌트 간 비결합 통신

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

보안 관제 텔레메트리, 금융 호가창, 클라우드 메트릭 등 초당 수백~수천 건의 이벤트가 유입될 때 `ObservableCollection`에 이벤트를 건별로 즉시 추가하면 UI 스레드가 락(Lock)에 걸려 애플리케이션이 멈춥니다.

### A. CQRS 인메모리 프로젝션 아키텍처

```
[Raw Event Producer (gRPC/Socket/Sensor)]
                 │  (Non-blocking Thread-safe Enqueue)
                 ▼
     [ConcurrentQueue<T> / Channel<T>]
                 │
                 ▼  (Periodic Batch Timer: 30~60Hz)
 [DispatcherPriority.Background Batch Flush]
                 │
                 ▼
    [UI ObservableCollection Projection] (Sliding Window Cap: 1,000 items)
```

### B. 배치 플러시 프로젝터 완결 구현체 (`ThrottledStreamProjector.cs`)

```csharp
using System;
using System.Collections.Concurrent;
using System.Collections.ObjectModel;
using System.Windows.Threading;

namespace MyWpfApp.Services.Implementations;

public sealed class ThrottledStreamProjector<T> : IDisposable
{
    private readonly ConcurrentQueue<T> _incomingQueue = new();
    private readonly ObservableCollection<T> _targetCollection;
    private readonly DispatcherTimer _flushTimer;
    private readonly int _maxBatchSizePerTick;
    private readonly int _maxCollectionCapacity;

    public ThrottledStreamProjector(ObservableCollection<T> targetCollection, int fps = 30, int maxBatchSize = 100, int maxCapacity = 1000)
    {
        _targetCollection = targetCollection;
        _maxBatchSizePerTick = maxBatchSize;
        _maxCollectionCapacity = maxCapacity;

        _flushTimer = new DispatcherTimer(DispatcherPriority.Background)
        {
            Interval = TimeSpan.FromMilliseconds(1000.0 / fps)
        };
        _flushTimer.Tick += OnFlushTick;
        _flushTimer.Start();
    }

    // 백그라운드 스레드에서 무제약 호출 (락 경합 없음)
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

        // 최대 메모리 보호: 슬라이딩 윈도우 트리밍
        while (_targetCollection.Count > _maxCollectionCapacity)
        {
            _targetCollection.RemoveAt(0);
        }
    }

    public void Dispose()
    {
        _flushTimer.Stop();
        _flushTimer.Tick -= OnFlushTick;
    }
}
```

---

## 6. UI 가상화 및 렌더링 성능 엔지니어링

수천 개의 항목을 렌더링하는 `ListView`, `ListBox`, `DataGrid`에서 프레임 드랍을 방지하려면 UI 가상화 조건을 완벽히 충족해야 합니다.

### A. 가상화 붕괴를 막고 60fps 부드러운 스크롤을 달성하는 5대 원칙
1. **무한 크기 컨테이너 내부 중첩 금지**:
   * `ScrollViewer` 내부에 `ListView`를 넣거나, `StackPanel` 내부에 `DataGrid`를 넣으면 자식 컨트롤의 높이가 무한대로 계산되어 가상화가 즉시 무효화됩니다.
   * 해결: 고정 높이 또는 `Grid`의 `RowDefinition Height="*"` 내부에 배치합니다.
2. **`ScrollViewer.CanContentScroll="True"` 절대 유지**:
   * 기본값이 `True`입니다. 픽셀 단위 부드러운 스크롤을 구현하겠다고 `CanContentScroll="False"`로 변경하는 순간, WPF 내부의 논리적 스크롤 엔진이 물리적 스크롤 모드로 전락하여 **컨테이너 가상화가 100% 해제**되고 수천 개의 항목이 메모리에 일괄 생성되어 UI 스레드가 프리징됩니다.
3. **컨테이너 재활용 모드 활성화**:
   * `VirtualizingPanel.VirtualizationMode="Recycling"`을 명시하여 스크롤 시 시각적 요소(Item Container)를 파괴/재생성하지 않고 재사용합니다.
4. **`DataTemplate` 내부 Visual Tree 경량화**:
   * 아이템 템플릿 내부에 무거운 `DropShadowEffect`나 다중 중첩 `Grid`/`Border`를 배제합니다.
5. **픽셀 단위 부드러운 가상화 스크롤 (`VirtualizingPanel.ScrollUnit="Pixel"`)**:
   * 기본 WPF 동작은 `ScrollUnit="Item"`으로 설정되어 있어 마우스 휠 스크롤 시 한 항목 단위로 툭툭 끊기며 이동하여 부자연스러운 UX를 유발합니다.
   * 최신 .NET 8/9/10 WPF 표준: `VirtualizingPanel.ScrollUnit="Pixel"` (또는 `VirtualizingStackPanel.ScrollUnit="Pixel"`)을 선언하면 `ScrollViewer.CanContentScroll="True"`의 **컨테이너 가상화를 100% 온전히 유지한 채로 60fps의 매끄러운 픽셀 단위 부드러운 스크롤(Pixel-based smooth scrolling)**을 실현할 수 있습니다. 상용 고밀도 데이터 그리드 및 대용량 텔레메트리 리스트 컨트롤에는 반드시 명시해야 합니다.

### B. 가상화 극대화 프로덕션 ListView XAML 표준

```xml
<ListView ItemsSource="{Binding TelemetryEvents}"
          VirtualizingPanel.IsVirtualizing="True"
          VirtualizingPanel.VirtualizationMode="Recycling"
          VirtualizingPanel.ScrollUnit="Pixel"
          VirtualizingPanel.CacheLength="20,20"
          VirtualizingPanel.CacheLengthUnit="Item"
          ScrollViewer.CanContentScroll="True"
          ScrollViewer.HorizontalScrollBarVisibility="Disabled"
          ScrollViewer.VerticalScrollBarVisibility="Auto">
    <ListView.ItemTemplate>
        <DataTemplate>
            <!-- 경량화된 아이템 Visual Tree (단일 경계선 및 텍스트) -->
            <Border Padding="8,6" BorderBrush="#1AFFFFFF" BorderThickness="0,0,0,1">
                <TextBlock Text="{Binding Summary}" Foreground="#F2F4F8" FontSize="12" />
            </Border>
        </DataTemplate>
    </ListView.ItemTemplate>
</ListView>
```

---

## 7. 전역 인앱 모달 서비스 아키텍처 (`IModalService`)

개별 뷰마다 모달 오버레이를 하드코딩하거나 OS 기본 `ShowDialog()` 팝업을 남발하는 안티패턴(Dialog Sprawl)을 방지하기 위해, **메인 윈도우 상단에 단일 모달 호스트를 두고 뷰모델에서 `Task<bool>`로 비동기 대기하는 전역 모달 아키텍처**를 표준화합니다.

### A. 모달 서비스 계약 및 구현체

```csharp
using System.Threading.Tasks;
using CommunityToolkit.Mvvm.ComponentModel;

namespace MyWpfApp.Services.Abstractions;

public interface IModalService
{
    bool IsOpen { get; }
    string Title { get; }
    string Message { get; }
    string TargetDetails { get; }
    Task<bool> ShowConfirmAsync(string title, string message, string targetDetails);
    void Approve();
    void Reject();
}

public partial class ModalService : ObservableObject, IModalService
{
    private TaskCompletionSource<bool>? _tcs;

    [ObservableProperty] private bool _isOpen;
    [ObservableProperty] private string _title = string.Empty;
    [ObservableProperty] private string _message = string.Empty;
    [ObservableProperty] private string _targetDetails = string.Empty;

    public Task<bool> ShowConfirmAsync(string title, string message, string targetDetails)
    {
        Title = title;
        Message = message;
        TargetDetails = targetDetails;
        IsOpen = true;

        _tcs = new TaskCompletionSource<bool>();
        return _tcs.Task;
    }

    public void Approve()
    {
        IsOpen = false;
        _tcs?.TrySetResult(true);
    }

    public void Reject()
    {
        IsOpen = false;
        _tcs?.TrySetResult(false);
    }
}
```

### B. 뷰모델에서의 사용 예시
```csharp
[RelayCommand]
private async Task ExecuteCriticalActionAsync()
{
    bool approved = await _modalService.ShowConfirmAsync(
        "위험 행위 승인 요청",
        "클러스터 노드의 비정상 리소스를 강제 격리하시겠습니까?",
        "Target: worker-node-04 (Action: Cordon & Drain)");

    if (!approved) return;

    // 사용자 승인 후 실제 작업 진행
    await _clusterService.DrainNodeAsync("worker-node-04");
}
```

---

## 8. 디자인 시스템 및 커스텀 컨트롤 템플릿 아키텍처 (Lessons from ARQA & Phalanx)

무거운 서드파티 라이브러리 없이, `OverridesDefaultStyle="True"`를 활용하여 시스템 기본 컨트롤을 100% 네이티브 XAML로 재정의합니다.

### A. 기본 컨트롤 전면 재정의 사전 (`Themes/ControlStyles.xaml`)

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

    <!-- 3. 호버 시 확장되는 반응형 스크롤바 -->
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
                                            <Border x:Name="thumbBorder"
                                                    Background="#33FFFFFF"
                                                    CornerRadius="4"
                                                    Margin="1,0" />
                                            <ControlTemplate.Triggers>
                                                <Trigger Property="IsMouseOver" Value="True">
                                                    <Setter TargetName="thumbBorder" Property="Background" Value="#66FFFFFF" />
                                                </Trigger>
                                                <Trigger Property="IsDragging" Value="True">
                                                    <Setter TargetName="thumbBorder" Property="Background" Value="#4F46E5" />
                                                </Trigger>
                                            </ControlTemplate.Triggers>
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

    <!-- 4. 프로 도구용 체크박스 재정의 -->
    <Style TargetType="CheckBox">
        <Setter Property="OverridesDefaultStyle" Value="True" />
        <Setter Property="Foreground" Value="#F2F4F8" />
        <Setter Property="FontSize" Value="13" />
        <Setter Property="Cursor" Value="Hand" />
        <Setter Property="Template">
            <Setter.Value>
                <ControlTemplate TargetType="CheckBox">
                    <StackPanel Orientation="Horizontal" VerticalAlignment="Center">
                        <Border x:Name="box"
                                Width="18" Height="18"
                                Background="#13141C"
                                BorderBrush="#22FFFFFF"
                                BorderThickness="1"
                                CornerRadius="4"
                                Margin="0,0,8,0">
                            <Path x:Name="checkMark"
                                  Data="M 3 9 L 7 13 L 15 4"
                                  Stroke="#FFFFFF"
                                  StrokeThickness="2"
                                  Visibility="Collapsed" />
                        </Border>
                        <ContentPresenter VerticalAlignment="Center" />
                    </StackPanel>
                    <ControlTemplate.Triggers>
                        <Trigger Property="IsChecked" Value="True">
                            <Setter TargetName="box" Property="Background" Value="#4F46E5" />
                            <Setter TargetName="box" Property="BorderBrush" Value="#6366F1" />
                            <Setter TargetName="checkMark" Property="Visibility" Value="Visible" />
                        </Trigger>
                        <Trigger Property="IsMouseOver" Value="True">
                            <Setter TargetName="box" Property="BorderBrush" Value="#4F46E5" />
                        </Trigger>
                    </ControlTemplate.Triggers>
                </ControlTemplate>
            </Setter.Value>
        </Setter>
    </Style>

</ResourceDictionary>
```

---

## 9. AI 에이전트 절대 금지 안티패턴 카탈로그 (Strict Anti-Patterns)

1. **[금지] 갓 윈도우(God Window) & 갓 뷰모델(God ViewModel)**:
   * 파일 하나에 1,000줄 이상의 XAML이나 C# 코드를 작성하는 행위 금지. 각 화면과 탭은 독립된 `UserControl`과 서브 뷰모델로 분리합니다.
2. **[금지] 다이얼로그 남발 (Dialog Sprawl)**:
   * `new SubWindow().ShowDialog()` 남발 금지. 메인 윈도우 단일 오버레이 레이어 및 `IModalService`를 활용합니다.
3. **[금지] `App.xaml` 내 `StartupUri` 선언**:
   * DI 컨테이너를 사용할 때 `StartupUri`를 남겨두어 윈도우가 중복 생성되는 참사를 원천 차단합니다.
4. **[금지] `async void` 메서드 작성**:
   * 예외 처리가 불가능한 `async void` 선언 절대 금지. 모든 비동기 메서드는 `Task`를 반환해야 합니다.
5. **[금지] 비동기 동기 블로킹**:
   * UI 스레드에서 `.Result`, `.Wait()`, `.GetAwaiter().GetResult()` 호출 금지 (영구 데드락 유발).
6. **[금지] 이벤트 구독 미해제 (메모리 누수)**:
   * 서비스 이벤트를 구독한 뷰모델은 반드시 `ICleanup`을 구현하고 소멸 시 `-=`로 구독을 해제해야 합니다.

---

## 10. 프로덕션 레디 보일러플레이트 (Production Blueprints)

### Blueprint 1: 메인 셸 (`MainWindow.xaml` - 전역 모달 호스트 일체형)

```xml
<Window x:Class="MyWpfApp.Views.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:shell="clr-namespace:System.Windows.Shell;assembly=PresentationFramework"
        Title="Enterprise Pro App"
        Height="800" Width="1300"
        MinHeight="600" MinWidth="900"
        Background="#0B0C10"
        WindowStartupLocation="CenterScreen">
        <!-- 주의: Windows 11 Mica 백드롭을 적용할 때는 Background="Transparent"로 지정하고 내부 패널에 반투명 브러시를 적용합니다 -->

    <!-- 윈도우 크롬 일체화 -->
    <shell:WindowChrome.WindowChrome>
        <shell:WindowChrome CaptionHeight="44"
                            CornerRadius="0"
                            GlassFrameThickness="0"
                            NonClientFrameEdges="None"
                            ResizeBorderThickness="6"
                            UseAeroCaptionButtons="False" />
    </shell:WindowChrome.WindowChrome>

    <Grid>
        <Grid.RowDefinitions>
            <RowDefinition Height="44" /> <!-- 커스텀 타이틀바 -->
            <RowDefinition Height="*" />  <!-- 본문 영역 -->
        </Grid.RowDefinitions>

        <!-- 1. 커스텀 타이틀바 -->
        <Border Grid.Row="0" Background="#13141C" BorderBrush="#1AFFFFFF" BorderThickness="0,0,0,1">
            <Grid Margin="16,0">
                <Grid.ColumnDefinitions>
                    <ColumnDefinition Width="Auto" />
                    <ColumnDefinition Width="*" />
                    <ColumnDefinition Width="Auto" />
                </Grid.ColumnDefinitions>

                <TextBlock Grid.Column="0" Text="ENTERPRISE PRO APPLICATION"
                           Foreground="#F2F4F8" FontWeight="SemiBold" FontSize="12"
                           VerticalAlignment="Center" />

                <!-- 시스템 창 제어 버튼 (클릭 관통 방지) -->
                <StackPanel Grid.Column="2" Orientation="Horizontal" shell:WindowChrome.IsHitTestVisibleInChrome="True">
                    <Button Width="36" Height="30" Content="—" Click="MinimizeButton_Click" Background="Transparent" BorderThickness="0" />
                    <Button x:Name="MaximizeButton" Width="36" Height="30" Content="☐" Click="MaximizeButton_Click" Background="Transparent" BorderThickness="0" />
                    <Button Width="36" Height="30" Content="✕" Click="CloseButton_Click" Background="Transparent" BorderThickness="0" />
                </StackPanel>
            </Grid>
        </Border>

        <!-- 2. 메인 작업 영역: ViewModel-First 암시적 DataTemplate 전환 -->
        <ContentControl Grid.Row="1" Content="{Binding CurrentPage}" />

        <!-- 3. 전역 인앱 모달 오버레이 호스트 (화면 전역 딤 처리) -->
        <Grid Grid.RowSpan="2" Background="#80000000"
              Visibility="{Binding Modal.IsOpen, Converter={StaticResource BoolToVis}}">
            <Border Background="#1A1D27" BorderBrush="#EF4444" BorderThickness="1"
                    CornerRadius="12" Padding="20" MaxWidth="480"
                    VerticalAlignment="Center" HorizontalAlignment="Center">
                <StackPanel>
                    <StackPanel Orientation="Horizontal" Margin="0,0,0,12">
                        <Border Width="8" Height="8" Background="#EF4444" CornerRadius="4" VerticalAlignment="Center" Margin="0,0,8,0" />
                        <TextBlock Text="{Binding Modal.Title}" FontWeight="Bold" Foreground="#F2F4F8" FontSize="15" />
                    </StackPanel>
                    <TextBlock Text="{Binding Modal.Message}" Foreground="#AAB2C0" TextWrapping="Wrap" Margin="0,0,0,12" />
                    <Border Background="#13141C" Padding="12" CornerRadius="6" Margin="0,0,0,16">
                        <TextBlock Text="{Binding Modal.TargetDetails}" FontFamily="Cascadia Code, Consolas" FontSize="12" Foreground="#EF4444" />
                    </Border>
                    <Grid>
                        <Grid.ColumnDefinitions>
                            <ColumnDefinition Width="*" />
                            <ColumnDefinition Width="12" />
                            <ColumnDefinition Width="*" />
                        </Grid.ColumnDefinitions>
                        <Button Grid.Column="0" Content="거부 (Skip)" Command="{Binding RejectModalCommand}" />
                        <Button Grid.Column="2" Content="실행 승인 (Execute)" Background="#EF4444" BorderBrush="#F87171" Command="{Binding ApproveModalCommand}" />
                    </Grid>
                </StackPanel>
            </Border>
        </Grid>
    </Grid>
</Window>
```

### Blueprint 2: `MainWindow.xaml.cs` (Windows 11 Snap Layouts & DWM / Mica 연동)

```csharp
using System;
using System.Runtime.InteropServices;
using System.Windows;
using System.Windows.Interop;

namespace MyWpfApp.Views;

public partial class MainWindow : Window
{
    private const int WM_NCHITTEST = 0x0084;
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
        DWMSBT_MAINWINDOW = 2,      // Mica (윈도우 11 기본 은은한 배경 투과)
        DWMSBT_TRANSIENTWINDOW = 3,  // Acrylic (블러 강조 반투명)
        DWMSBT_TABBEDWINDOW = 4      // Mica Alt (탭 윈도우용 고대비)
    }

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

        int darkMode = 1;
        DwmSetWindowAttribute(hwnd, DWMWA_USE_IMMERSIVE_DARK_MODE, ref darkMode, sizeof(int));

        int cornerPreference = DWMWCP_ROUND;
        DwmSetWindowAttribute(hwnd, DWMWA_WINDOW_CORNER_PREFERENCE, ref cornerPreference, sizeof(int));

        // Windows 11 Mica 백드롭 활성화 (Window.Background="Transparent" 필요)
        int backdropType = (int)DWM_SYSTEMBACKDROP_TYPE.DWMSBT_MAINWINDOW;
        DwmSetWindowAttribute(hwnd, DWMWA_SYSTEMBACKDROP_TYPE, ref backdropType, sizeof(int));

        var hwndSource = HwndSource.FromHwnd(hwnd);
        hwndSource?.AddHook(WndProc);
    }

    private IntPtr WndProc(IntPtr hwnd, int msg, IntPtr wParam, IntPtr lParam, ref bool handled)
    {
        if (msg == WM_NCHITTEST)
        {
            int x = lParam.ToInt32() & 0xffff;
            int y = lParam.ToInt32() >> 16;
            if (MaximizeButton.IsLoaded)
            {
                var buttonPos = MaximizeButton.PointToScreen(new Point(0, 0));
                var buttonRect = new Rect(buttonPos.X, buttonPos.Y, MaximizeButton.ActualWidth, MaximizeButton.ActualHeight);
                if (buttonRect.Contains(new Point(x, y)))
                {
                    handled = true;
                    return new IntPtr(HTMAXBUTTON); // Win11 Snap Layouts 플라이아웃 활성화
                }
            }
        }
        return IntPtr.Zero;
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
```

### Blueprint 3: `MainViewModel.cs` (수명주기 전환 및 모달 오케스트레이터)

```csharp
using System;
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using Microsoft.Extensions.DependencyInjection;
using MyWpfApp.Services.Abstractions;
using MyWpfApp.ViewModels.Common;
using MyWpfApp.ViewModels.Pages;

namespace MyWpfApp.ViewModels;

public partial class MainViewModel : ObservableObject
{
    private readonly IServiceProvider _serviceProvider;
    public IModalService Modal { get; }

    [ObservableProperty]
    private ObservableObject? _currentPage;

    public MainViewModel(IServiceProvider serviceProvider, IModalService modalService)
    {
        _serviceProvider = serviceProvider;
        Modal = modalService;
        NavigateTo<DashboardViewModel>();
    }

    [RelayCommand]
    public void NavigateTo(Type viewModelType)
    {
        // 1. 이전 화면의 ICleanup 명시적 호출
        if (CurrentPage is ICleanup cleanable)
        {
            cleanable.Cleanup();
        }

        // 2. 신규 화면 Transient Resolve
        CurrentPage = (ObservableObject)_serviceProvider.GetRequiredService(viewModelType);
    }

    public void NavigateTo<T>() where T : ObservableObject => NavigateTo(typeof(T));

    [RelayCommand]
    private void ApproveModal() => Modal.Approve();

    [RelayCommand]
    private void RejectModal() => Modal.Reject();
}
```
