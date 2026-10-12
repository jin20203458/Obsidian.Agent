---
description: WPF 현대적 MVVM 아키텍처, Generic Host DI, 수명주기, 유효성 검사, 제네릭 모달, 고성능 스트리밍 및 Windows 11 DWM 통합 마스터 지침서.
related:
  - ../README.md
  - ./Modern_AI_Desktop_UI_Guidelines.md
---
# WPF Architecture Guidelines

본 문서는 현대적인 WPF(.NET 8.0/9.0/10.0) 애플리케이션을 신규 생성하거나 대규모 엔터프라이즈 코드베이스를 리팩토링할 때 AI 에이전트와 엔지니어가 반드시 준수해야 하는 **범용 엔터프라이즈 아키텍처 단일 진실 공급원(SSOT) 마스터 지침서**입니다.

과거 .NET Framework 시절의 레거시 관행(수동 INPC, 무거운 외부 래퍼 라이브러리 남용, 갓 윈도우/뷰모델 결합)과 특정 비즈니스 도메인에 대한 과적합을 배제하고, 실전 대규모 소프트웨어에서 검증된 **Generic Host DI, CommunityToolkit.Mvvm 소스 제너레이터, ObservableValidator 폼 검증, 비동기 네비게이션 파이프라인, 제네릭 In-App 모달, 3계층 디자인 토큰, 고성능 스트리밍, 그리고 Windows 11 DWM/Snap Layouts 네이티브 통합 패턴**을 제공합니다.

---

## 1. 아키텍처 원형(Archetypes) 비교 및 선택 가이드

WPF 애플리케이션은 시스템의 목적과 데이터 처리량에 따라 구조적 접근이 달라져야 합니다. 프로젝트의 성격에 부합하는 원형을 우선 선정합니다.

| 분류 항목 | 원형 1: 순수 ViewModel-First 단일 셸 | 원형 2: 모듈형 셸 + 레일 + CQRS 리드 모델 | 원형 3: 멀티 도구 분석 워크벤치 |
| :--- | :--- | :--- | :--- |
| **적합한 앱 유형** | 대화형 AI 클라이언트, 설정 마법사, 단일 워크플로 생산성 도구 | 실시간 보안 관제, 텔레메트리 대시보드, 클라우드 플릿 모니터링 | 정적 코드 분석기, 진단 도구, 복합 데이터그리드 IDE |
| **창/네비게이션 구조** | 초경량 `MainWindow` + `<ContentControl Content="{Binding CurrentPage}" />` | 52px 슬림 좌측 레일 + 모듈별 `UserControl` 교체 | 다중 분할 도킹 패널(`GridSplitter`), 탭 기반 워크스페이스 |
| **뷰-뷰모델 결합** | 암시적 `DataTemplate` 자동 맵핑 (View 코드-비하인드 제로) | 명시적 레일 커맨드 기반 모듈 활성화 | 영역별 독립 서브 뷰모델 분할 및 중앙 오케스트레이터 |
| **수명주기 모델** | `Transient` 등록 + 화면 전환 시 `ICleanup`을 통한 메모리 회수 | `Singleton` 코어 엔진 + 장기 실행 텔레메트리 수신 파이프라인 | 세션별 도구 워크스페이스 수명주기 |
| **데이터 처리 특성** | 실시간 LLM 토큰 증분 수신, 음성/오디오 스트리밍 파이프라인 | 초당 수천 건 이벤트 버퍼링, 30~60Hz 배치 플러시 프로젝션 | 대용량 구문 분석 트리, 수만 건 진단 로그 가상화 렌더링 |

---

## 2. 프로젝트 표준 디렉터리 레이아웃

역할과 관심사의 분리를 명확히 하기 위해 아래의 표준 디렉터리 배치를 준수합니다.

```text
ProjectRoot/
├── Config/                   # JSON 설정, AppSettings, 런타임 프로파일
├── Models/                   # 순수 데이터 구조체 (POCO), DTO, 프로토콜 엔티티
├── Services/                 # 비즈니스 로직, API/gRPC 통신, IPC 클라이언트
│   ├── Abstractions/         # 서비스 인터페이스 (INavigationService, IDialogService, IDataService)
│   └── Implementations/      # 구체 클래스 구현체 및 배치 프로젝터
├── ViewModels/               # UI 프레젠테이션 로직 (CommunityToolkit.Mvvm)
│   ├── Common/               # ViewModelBase, ICleanup, INavigationAware, UiDispatcherBridge
│   ├── Dialogs/              # 모달 및 다이얼로그 뷰모델 (ConfirmDialogViewModel 등)
│   └── Pages/                # 각 화면별 전용 뷰모델
├── Views/                    # XAML 선언 및 순수 UI 비하인드 코드
│   ├── Controls/             # 재사용 가능한 커스텀 UserControl
│   ├── Dialogs/              # 인앱 모달 및 오버레이 뷰
│   └── Pages/                # 화면 본체 뷰
├── Themes/                   # 디자인 시스템 및 XAML 리소스 사전 (3계층)
│   ├── PrimitiveTokens.xaml  # 정적 컬러(Color), 불변 수치, 폰트 규격
│   ├── DarkTheme.xaml        # 시맨틱 솔리드 브러시 (DynamicResource 대상)
│   ├── LightTheme.xaml       # 라이트 테마 솔리드 브러시
│   ├── ControlStyles.xaml    # 버튼, 텍스트박스, 스크롤바 등 기본 컨트롤 스타일 재정의
│   └── DataTemplates.xaml    # ViewModel-to-View 암시적 맵핑 정의
├── App.xaml                  # 시작점 및 전역 MergedDictionaries 등록 (StartupUri 절대 금지)
├── App.xaml.cs               # Generic Host, DI 컨테이너 구성, 전역 예외 트랩
└── Project.csproj            # 최신 SDK 스타일 프로젝트 파일 (.NET 8.0/9.0/10.0)
```

---

## 3. 의존성 주입(DI) 및 수명주기 관리 표준

`Microsoft.Extensions.DependencyInjection` 및 `Microsoft.Extensions.Hosting`을 표준 인프라로 사용하며, 서비스와 뷰모델의 수명주기를 엄격히 구분합니다.

### A. 필수 NuGet 패키지 의존성
* `Microsoft.Extensions.Hosting` (Generic Host 및 호스트 수명주기 관리 - 필수)
* `Microsoft.Extensions.DependencyInjection` (DI 컨테이너 - 필수)
* `Microsoft.Extensions.Http` (`AddHttpClient` 팩토리 패턴 지원 및 소켓 누수 방지 - 필수)
* `CommunityToolkit.Mvvm` (버전 8.3/8.4+ 소스 제너레이터 - 필수)

### B. 서비스 수명주기 원칙
* **Singleton**: 통신 클라이언트(`HttpClient`), 전역 상태 저장소, 이벤트 중계자(`IMessenger`), 전역 다이얼로그 호스트 서비스(`IDialogService`), 네비게이션 서비스(`INavigationService`), 설정 관리자.
* **Transient**: 화면 뷰모델(`PageViewModel`), 단발성 다이얼로그 뷰모델.
  * 이유: 페이지를 이동하거나 다이얼로그를 닫을 때 이전 화면의 상태를 완전 소멸시키고 메모리 누수를 원천 방지하기 위함.
* **Scoped**: 단일 작업 세션 또는 특정 단위 작업(Unit-of-Work)에 종속된 데이터 컨텍스트.

### C. [중대 규칙] `App.xaml`의 `StartupUri` 제거 강제
`App.xaml.cs`의 `OnStartup`에서 DI 컨테이너를 통해 `MainWindow`를 인스턴스화할 때, **`App.xaml`에 `StartupUri="MainWindow.xaml"`이 선언되어 있으면 WPF 런타임이 파라미터 없는 기본 생성자로 윈도우를 중복 생성하여 2개의 창이 뜨고 DI가 누락되는 치명적 결함**이 발생합니다.
따라서 `App.xaml`에서 `StartupUri` 속성은 반드시 삭제해야 합니다.

```xml
<!-- 올바른 App.xaml: StartupUri 속성이 없음 -->
<Application x:Class="MyWpfApp.App"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
    <Application.Resources>
        <ResourceDictionary>
            <ResourceDictionary.MergedDictionaries>
                <ResourceDictionary Source="Themes/PrimitiveTokens.xaml" />
                <ResourceDictionary Source="Themes/DarkTheme.xaml" />
                <ResourceDictionary Source="Themes/ControlStyles.xaml" />
                <ResourceDictionary Source="Themes/DataTemplates.xaml" />
            </ResourceDictionary.MergedDictionaries>
        </ResourceDictionary>
    </Application.Resources>
</Application>
```

### D. `App.xaml.cs` 및 3대 전역 예외 처리(Global Exception Traps)
UI 스레드뿐 아니라 비동기 태스크 풀과 AppDomain 수준에서 처리되지 않은 예외를 포착하여 프로세스 비정상 종료(Silent Crash)를 방지합니다.

```csharp
using System;
using System.IO;
using System.Threading.Tasks;
using System.Windows;
using System.Windows.Threading;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using MyWpfApp.Services.Abstractions;
using MyWpfApp.Services.Implementations;
using MyWpfApp.ViewModels;
using MyWpfApp.ViewModels.Pages;
using MyWpfApp.Views;

namespace MyWpfApp;

public partial class App : Application
{
    private readonly IHost _host;

    public static new App Current => (App)Application.Current;
    public IServiceProvider Services => _host.Services;

    public App()
    {
        // 1. 전역 예외 처리 트랩 3대 등록
        DispatcherUnhandledException += OnDispatcherUnhandledException;
        AppDomain.CurrentDomain.UnhandledException += OnUnhandledException;
        TaskScheduler.UnobservedTaskException += OnUnobservedTaskException;

        // 2. Generic Host 구성
        _host = Host.CreateDefaultBuilder()
            .ConfigureServices((context, services) =>
            {
                ConfigureServices(services);
            })
            .Build();
    }

    private static void ConfigureServices(IServiceCollection services)
    {
        // 인프라 및 서비스
        services.AddSingleton<INavigationService, NavigationService>();
        services.AddSingleton<IDialogService, DialogService>();
        services.AddSingleton<IDataService, DataService>();

        // HTTP Factory
        services.AddHttpClient("ApiClient", client =>
        {
            client.Timeout = TimeSpan.FromSeconds(30);
        });

        // 메인 셸
        services.AddSingleton<MainViewModel>();
        services.AddSingleton<MainWindow>();

        // 화면별 뷰모델 (Transient: 화면 전환 시 생성/해제)
        services.AddTransient<DashboardViewModel>();
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

    private void OnDispatcherUnhandledException(object sender, DispatcherUnhandledExceptionEventArgs e)
    {
        LogFatalException("Dispatcher", e.Exception);
        MessageBox.Show($"예기치 않은 UI 오류가 발생했습니다:\n{e.Exception.Message}", "오류", MessageBoxButton.OK, MessageBoxImage.Error);
        e.Handled = true; // 프로세스 즉시 종료 방지
    }

    private void OnUnhandledException(object sender, UnhandledExceptionEventArgs e)
    {
        if (e.ExceptionObject is Exception ex)
        {
            LogFatalException("AppDomain", ex);
        }
    }

    private void OnUnobservedTaskException(object? sender, UnobservedTaskExceptionEventArgs e)
    {
        LogFatalException("TaskScheduler", e.Exception);
        e.SetObserved(); // 태스크 비관찰 예외 프로세스 다운 방어
    }

    private static void LogFatalException(string source, Exception ex)
    {
        try
        {
            var logPath = Path.Combine(AppDomain.CurrentDomain.BaseDirectory, "crash.log");
            File.AppendAllText(logPath, $"[{DateTime.Now:O}] [{source}] {ex}\n\n");
        }
        catch { /* 로깅 실패 무시 */ }
    }
}
```

### E. `ICleanup`을 통한 페이지 이탈 시 메모리 회수 패턴
뷰모델이 `Transient`로 생성되더라도 백그라운드 태스크나 서비스 이벤트 핸들러를 구독 중이면 가비지 컬렉터가 수거하지 못합니다. 명시적인 `ICleanup` 계약을 통해 화면 전환 즉시 리소스를 해제해야 합니다.

```csharp
namespace MyWpfApp.ViewModels.Common;

public interface ICleanup
{
    void Cleanup();
}

public partial class DashboardViewModel : ObservableObject, ICleanup
{
    private readonly IDataService _dataService;
    private readonly CancellationTokenSource _cts = new();

    public DashboardViewModel(IDataService dataService)
    {
        _dataService = dataService;
        _dataService.DataArrived += OnDataArrived;
    }

    public void Cleanup()
    {
        // 1. 서비스 이벤트 구독 해제
        _dataService.DataArrived -= OnDataArrived;

        // 2. 백그라운드 비동기 루프 취소 및 해제
        _cts.Cancel();
        _cts.Dispose();

        // 3. 내부 컬렉션 정리
        Items.Clear();
    }

    private void OnDataArrived(object? sender, EventArgs e) { /* 처리 */ }
}
```

### F. 헤드리스(Headless) 환경 및 단위 테스트 안전성 가드
WPF UI 스레드 마샬링 코드(`Application.Current.Dispatcher`)는 단위 테스트 러너나 CLI 배치 실행 시 `Application.Current`가 `null`이 되어 크래시를 유발합니다. 이를 추상화한 안전 브릿지를 사용합니다.

```csharp
namespace MyWpfApp.ViewModels.Common;

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

## 4. CommunityToolkit.Mvvm 소스 제너레이터 및 Form Validation 표준

최신 C# 소스 제너레이터(CommunityToolkit.Mvvm 8.3/8.4+)를 통해 보일러플레이트를 제거하고, `ObservableValidator`를 표준으로 삼아 입력 유효성 검증을 통합합니다.

### A. 핵심 선언 규칙
1. **`partial` 클래스 및 기본 상속**:
   * 일반 뷰모델: `public partial class MyViewModel : ObservableObject`
   * 입력 폼/설정 뷰모델: `public partial class FormViewModel : ObservableValidator`
2. **필드 네이밍 규격**:
   * 속성 생성 대상 필드는 반드시 private `_camelCase`로 선언합니다. 소스 제너레이터가 public `PascalCase` 프로퍼티를 자동 생성합니다.
3. **파생 속성 알림 (`[NotifyPropertyChangedFor]`)**:
   * 계산된 읽기 전용 프로퍼티가 의존하는 필드에 직접 부착합니다.

### B. `ObservableValidator` 기반 폼 유효성 검증 표준
`System.ComponentModel.DataAnnotations`와 결합하여 프로퍼티 값 변경 시 즉각 검증(`INotifyDataErrorInfo`)을 수행합니다.

```csharp
using System.ComponentModel.DataAnnotations;
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;

namespace MyWpfApp.ViewModels.Pages;

public partial class UserRegistrationViewModel : ObservableValidator
{
    [ObservableProperty]
    [Required(ErrorMessage = "사용자 이름을 입력해야 합니다.")]
    [MinLength(3, ErrorMessage = "최소 3자 이상 입력해야 합니다.")]
    private string _userName = string.Empty;

    [ObservableProperty]
    [Required(ErrorMessage = "이메일을 입력해야 합니다.")]
    [EmailAddress(ErrorMessage = "올바른 이메일 형식이 아닙니다.")]
    [CustomValidation(typeof(UserRegistrationViewModel), nameof(ValidateDomain))]
    private string _email = string.Empty;

    public static ValidationResult? ValidateDomain(string email, ValidationContext context)
    {
        if (!string.IsNullOrEmpty(email) && !email.EndsWith("@company.com"))
        {
            return new ValidationResult("사내 이메일(@company.com)만 등록 가능합니다.");
        }
        return ValidationResult.Success;
    }

    [RelayCommand]
    private async Task SubmitAsync()
    {
        // 중요: 사용자가 건드리지 않은 초기 필드 검증을 위해 반드시 ValidateAllProperties 호출
        ValidateAllProperties();

        if (HasErrors)
        {
            return;
        }

        // 비즈니스 로직 실행
        await Task.Delay(500);
    }

    [RelayCommand]
    private void ResetForm()
    {
        UserName = string.Empty;
        Email = string.Empty;
        ClearErrors(); // 이전 에러 메시지 초기화
    }
}
```

### C. 모던 Fluent Validation.ErrorTemplate XAML 정의
WPF 기본 빨간 테두리는 AdornerLayer에 그려져 팝업에 가려지거나 주변 컨트롤을 밀어내지 못합니다. 에러 메시지를 컨트롤 하단에 깔끔하게 배치하는 표준 템플릿입니다.

```xml
<!-- Themes/ControlStyles.xaml 내 선언 -->
<ControlTemplate x:Key="ModernErrorTemplate">
    <StackPanel>
        <AdornedElementPlaceholder x:Name="placeholder" />
        <TextBlock Margin="0,4,0,0"
                   FontSize="11"
                   Foreground="{DynamicResource AccentDangerBrush}"
                   Text="{Binding [0].ErrorContent}" />
    </StackPanel>
</ControlTemplate>

<!-- 사용처 XAML -->
<TextBox Text="{Binding UserName, UpdateSourceTrigger=PropertyChanged}"
         Validation.ErrorTemplate="{StaticResource ModernErrorTemplate}" />
```

### D. [치명적 안티패턴 방어] `async void`의 전면 금지 및 SafeFireAndForget
모든 내부 비동기 메서드는 반드시 `Task`를 반환해야 합니다. 최상위 이벤트 핸들러 등 `void` 시그니처가 불가피한 경우 안전 확장 메서드를 사용합니다.

```csharp
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

### E. `WeakReferenceMessenger`를 통한 컴포넌트 간 비결합 통신
서로 다른 뷰모델 간의 통신은 직접 참조를 피하고 약한 참조 메시징을 사용합니다.

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Messaging;

// 1. 불변 메시지 레코드
public sealed record EntityUpdatedMessage(string EntityId, string Action);

// 2. 수신 뷰모델
public partial class LogViewerViewModel : ObservableRecipient, IRecipient<EntityUpdatedMessage>
{
    public LogViewerViewModel()
    {
        IsActive = true; // 메시지 자동 구독 활성화
    }

    public void Receive(EntityUpdatedMessage message)
    {
        // 메시지 수신 처리
    }
}

// 3. 송신 측 호출
WeakReferenceMessenger.Default.Send(new EntityUpdatedMessage("ITEM-100", "Updated"));
```

---

## 5. 범용 네비게이션 서비스 및 비동기 수명주기 훅

화면 전환 시 파라미터 전달과 비동기 데이터 로딩을 안전하게 처리하기 위한 표준 네비게이션 파이프라인입니다.

### A. 수명주기 인터페이스 계약
```csharp
namespace MyWpfApp.ViewModels.Common;

public interface INavigationAware
{
    Task OnNavigatedToAsync(object? parameter);
    Task OnNavigatedFromAsync();
}

public interface IConfirmNavigation
{
    Task<bool> CanNavigateFromAsync();
}
```

### B. 네비게이션 서비스 계약 및 구현체
```csharp
using System;
using System.Collections.Generic;
using System.Threading.Tasks;
using CommunityToolkit.Mvvm.ComponentModel;
using Microsoft.Extensions.DependencyInjection;
using MyWpfApp.ViewModels.Common;

namespace MyWpfApp.Services.Abstractions;

public interface INavigationService
{
    ObservableObject? CurrentPage { get; }
    bool CanGoBack { get; }
    Task<bool> NavigateToAsync<TViewModel>(object? parameter = null) where TViewModel : ObservableObject;
    Task<bool> GoBackAsync();
    event EventHandler? CurrentPageChanged;
}

public class NavigationService : INavigationService
{
    private readonly IServiceProvider _serviceProvider;
    private readonly Stack<(Type ViewModelType, object? Parameter)> _backStack = new();

    public ObservableObject? CurrentPage { get; private set; }
    public bool CanGoBack => _backStack.Count > 0;
    public event EventHandler? CurrentPageChanged;

    public NavigationService(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }

    public async Task<bool> NavigateToAsync<TViewModel>(object? parameter = null) where TViewModel : ObservableObject
    {
        // 1. 이전 화면의 화면 이탈 확인 (저장되지 않은 변경사항 등)
        if (CurrentPage is IConfirmNavigation confirmNav && !await confirmNav.CanNavigateFromAsync())
        {
            return false;
        }

        // 2. 이전 화면의 OnNavigatedFromAsync 및 ICleanup 호출
        if (CurrentPage is INavigationAware oldAware)
        {
            await oldAware.OnNavigatedFromAsync();
        }
        if (CurrentPage is ICleanup cleanup)
        {
            cleanup.Cleanup();
        }

        // 3. 백스택에 현재 페이지 저장 (동일 타입 재방문 제외)
        if (CurrentPage != null)
        {
            _backStack.Push((CurrentPage.GetType(), parameter));
        }

        // 4. 새 페이지 Transient 인스턴스화
        var newPage = _serviceProvider.GetRequiredService<TViewModel>();
        CurrentPage = newPage;

        // 5. 새 페이지의 OnNavigatedToAsync 비동기 초기화 호출
        if (newPage is INavigationAware newAware)
        {
            await newAware.OnNavigatedToAsync(parameter);
        }

        CurrentPageChanged?.Invoke(this, EventArgs.Empty);
        return true;
    }

    public async Task<bool> GoBackAsync()
    {
        if (!CanGoBack) return false;

        var (prevType, param) = _backStack.Pop();
        // 백스택에서 꺼낸 타입으로 네비게이션 복원 로직 실행
        return true;
    }
}
```

---

## 6. 범용 In-App 모달 및 ContentDialogHost 아키텍처

별도의 OS Win32 윈도우(`ShowDialog`)를 띄우지 않고, 메인 윈도우 시각 트리 최상단에 반투명 스모크 레이어를 오버레이하여 임의의 자식 ViewModel을 호스팅하는 표준 아키텍처입니다.

### A. 다이얼로그 서비스 계약 및 ViewModel 인터페이스
```csharp
using System;
using System.Threading.Tasks;
using CommunityToolkit.Mvvm.ComponentModel;

namespace MyWpfApp.Services.Abstractions;

public interface IDialogAware<TResult>
{
    event Action<TResult>? RequestClose;
    string DialogTitle { get; }
    bool CanClose();
    void OnDialogOpened(object? parameter);
}

public interface IDialogService
{
    ObservableObject? CurrentDialog { get; }
    bool IsOpen { get; }
    Task<TResult?> ShowDialogAsync<TViewModel, TResult>(object? parameter = null)
        where TViewModel : ObservableObject, IDialogAware<TResult>;
    Task<bool> ShowConfirmAsync(string title, string message, string? details = null);
}
```

### B. 범용 다이얼로그 서비스 구현체
```csharp
using System;
using System.Threading.Tasks;
using CommunityToolkit.Mvvm.ComponentModel;
using Microsoft.Extensions.DependencyInjection;
using MyWpfApp.Services.Abstractions;

namespace MyWpfApp.Services.Implementations;

public partial class DialogService : ObservableObject, IDialogService
{
    private readonly IServiceProvider _serviceProvider;

    [ObservableProperty] private ObservableObject? _currentDialog;
    [ObservableProperty] private bool _isOpen;

    public DialogService(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }

    public async Task<TResult?> ShowDialogAsync<TViewModel, TResult>(object? parameter = null)
        where TViewModel : ObservableObject, IDialogAware<TResult>
    {
        var vm = _serviceProvider.GetRequiredService<TViewModel>();
        var tcs = new TaskCompletionSource<TResult?>();

        vm.OnDialogOpened(parameter);

        Action<TResult> closeHandler = null!;
        closeHandler = result =>
        {
            if (!vm.CanClose()) return;
            vm.RequestClose -= closeHandler;
            CurrentDialog = null;
            IsOpen = false;
            tcs.TrySetResult(result);
        };

        vm.RequestClose += closeHandler;
        CurrentDialog = vm;
        IsOpen = true;

        return await tcs.Task;
    }

    public async Task<bool> ShowConfirmAsync(string title, string message, string? details = null)
    {
        var result = await ShowDialogAsync<ConfirmDialogViewModel, bool>(new ConfirmDialogParams(title, message, details));
        return result;
    }
}

public record ConfirmDialogParams(string Title, string Message, string? Details);

public partial class ConfirmDialogViewModel : ObservableObject, IDialogAware<bool>
{
    public event Action<bool>? RequestClose;
    [ObservableProperty] private string _dialogTitle = "확인";
    [ObservableProperty] private string _message = string.Empty;
    [ObservableProperty] private string? _details;

    public bool CanClose() => true;
    public void OnDialogOpened(object? parameter)
    {
        if (parameter is ConfirmDialogParams p)
        {
            DialogTitle = p.Title;
            Message = p.Message;
            Details = p.Details;
        }
    }

    [RelayCommand] private void Confirm() => RequestClose?.Invoke(true);
    [RelayCommand] private void Cancel() => RequestClose?.Invoke(false);
}
```

---

## 7. 3계층 디자인 토큰 및 컨트롤 스타일 아키텍처

무거운 외부 UI 라이브러리 없이 순수 XAML로 일관된 테마를 구축하기 위해, **Primitive Tokens -> Semantic Theme Brushes -> Control Styles**의 3계층 분리를 준수합니다.

### A. 3계층 분리 및 DynamicResource vs StaticResource 원칙
* **Primitive Tokens (`PrimitiveTokens.xaml`)**: 불변 Color 및 치수 정의. 자체적으로 완결되며 `{StaticResource}`로 참조됩니다.
* **Semantic Theme (`DarkTheme.xaml`, `LightTheme.xaml`)**: 테마별 시맨틱 `SolidColorBrush`를 정의하며 `po:Freeze="True"`로 동결합니다.
* **Control Styles (`ControlStyles.xaml`)**:
  * 테마 가변 속성(Background, Foreground, BorderBrush): 반드시 **`{DynamicResource BrushKey}`**로 바인딩하여 런타임 테마 전환을 지원합니다.
  * 레이아웃 불변 속성(Padding, BorderThickness, CornerRadius): 반드시 **`{StaticResource}`**로 바인딩하여 바인딩 오버헤드를 제거합니다.

### B. 테마 토큰 사전 예시 (`Themes/DarkTheme.xaml`)
```xml
<ResourceDictionary xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
                    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
                    xmlns:po="http://schemas.microsoft.com/winfx/2006/xaml/presentation/options">

    <!-- 시맨틱 브러시 (동결 필수) -->
    <SolidColorBrush x:Key="AppBackgroundBrush" Color="#0B0C10" po:Freeze="True" />
    <SolidColorBrush x:Key="SurfaceBackgroundBrush" Color="#13141C" po:Freeze="True" />
    <SolidColorBrush x:Key="CardBackgroundBrush" Color="#1A1D27" po:Freeze="True" />
    <SolidColorBrush x:Key="CardBackgroundHoverBrush" Color="#252A38" po:Freeze="True" />
    <SolidColorBrush x:Key="BorderSubtleBrush" Color="#22FFFFFF" po:Freeze="True" />
    
    <SolidColorBrush x:Key="TextPrimaryBrush" Color="#F2F4F8" po:Freeze="True" />
    <SolidColorBrush x:Key="TextSecondaryBrush" Color="#AAB2C0" po:Freeze="True" />
    <SolidColorBrush x:Key="AccentPrimaryBrush" Color="#4F46E5" po:Freeze="True" />
    <SolidColorBrush x:Key="AccentDangerBrush" Color="#EF4444" po:Freeze="True" />

    <!-- 불변 치수 -->
    <CornerRadius x:Key="DefaultCornerRadius">6</CornerRadius>
    <Thickness x:Key="ButtonPadding">14,8</Thickness>
</ResourceDictionary>
```

### C. 기본 컨트롤 전면 재정의 사전 (`Themes/ControlStyles.xaml`)
```xml
<ResourceDictionary xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
                    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">

    <!-- 1. 기본 버튼 재정의 -->
    <Style TargetType="Button">
        <Setter Property="OverridesDefaultStyle" Value="True" />
        <Setter Property="Background" Value="{DynamicResource CardBackgroundBrush}" />
        <Setter Property="Foreground" Value="{DynamicResource TextPrimaryBrush}" />
        <Setter Property="BorderBrush" Value="{DynamicResource BorderSubtleBrush}" />
        <Setter Property="BorderThickness" Value="1" />
        <Setter Property="Padding" Value="{StaticResource ButtonPadding}" />
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
                            CornerRadius="{StaticResource DefaultCornerRadius}"
                            Padding="{TemplateBinding Padding}">
                        <ContentPresenter HorizontalAlignment="Center"
                                          VerticalAlignment="Center"
                                          RecognizesAccessKey="True" />
                    </Border>
                    <ControlTemplate.Triggers>
                        <Trigger Property="IsMouseOver" Value="True">
                            <Setter TargetName="border" Property="Background" Value="{DynamicResource CardBackgroundHoverBrush}" />
                            <Setter TargetName="border" Property="BorderBrush" Value="{DynamicResource AccentPrimaryBrush}" />
                        </Trigger>
                        <Trigger Property="IsPressed" Value="True">
                            <Setter TargetName="border" Property="Background" Value="{DynamicResource SurfaceBackgroundBrush}" />
                        </Trigger>
                        <Trigger Property="IsEnabled" Value="False">
                            <Setter TargetName="border" Property="Opacity" Value="0.4" />
                        </Trigger>
                    </ControlTemplate.Triggers>
                </ControlTemplate>
            </Setter.Value>
        </Setter>
    </Style>

    <!-- 2. 반응형 스크롤바 재정의 -->
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
                                                    <Setter TargetName="thumbBorder" Property="Background" Value="{DynamicResource AccentPrimaryBrush}" />
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

</ResourceDictionary>
```

---

## 8. 초고속 스트리밍 데이터 및 CQRS 리드 모델

대량의 이벤트가 유입될 때 `ObservableCollection`에 이벤트를 건별로 즉시 추가하면 UI 스레드가 락에 걸려 애플리케이션이 멈춥니다. 인메모리 버퍼와 주기적 일괄 플러시를 적용합니다.

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

    public void Enqueue(T item) => _incomingQueue.Enqueue(item);

    private void OnFlushTick(object? sender, EventArgs e)
    {
        if (_incomingQueue.IsEmpty) return;

        int processed = 0;
        while (processed < _maxBatchSizePerTick && _incomingQueue.TryDequeue(out var item))
        {
            _targetCollection.Add(item);
            processed++;
        }

        // 슬라이딩 윈도우 트리밍
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

## 9. UI 가상화 및 렌더링 성능 엔지니어링

수천 개의 항목을 렌더링하는 `ListView`, `ListBox`, `DataGrid`에서 60fps 부드러운 스크롤을 달성하기 위한 원칙입니다.

1. **무한 크기 컨테이너 내부 중첩 금지**: `ScrollViewer`나 `StackPanel` 내부에 가상화 컨트롤을 넣지 말고 `Grid`의 `RowDefinition Height="*"` 내부에 배치합니다.
2. **`ScrollViewer.CanContentScroll="True"` 유지**: `False`로 바꾸면 컨테이너 가상화가 100% 해제되어 수천 개의 시각 트리가 일괄 생성됩니다.
3. **컨테이너 재활용 모드 활성화**: `VirtualizingPanel.VirtualizationMode="Recycling"`을 명시합니다.
4. **픽셀 단위 스크롤 (`VirtualizingPanel.ScrollUnit="Pixel"`) 적용 시 가변 높이 주의**:
   * 아이템 높이가 균일(Fixed Height: 예 32px)한 리스트에는 `ScrollUnit="Pixel"`을 적용하여 60fps 부드러운 스크롤을 얻습니다.
   * `TextWrapping="Wrap"` 등으로 아이템 높이가 제각각인 가변 높이 리스트에 `ScrollUnit="Pixel"`을 적용하면 스크롤바 Thumb가 튀는 현상(Jumping Thumb)이 발생하므로, 가변 높이 리스트에는 `ScrollUnit="Item"`을 유지해야 합니다.

---

## 10. WindowChrome, DWM Mica 백드롭 및 Windows 11 Snap Layouts 일체형 셸

### A. 창 프레임워크 표준
* `WindowStyle="None"` 대신 **`WindowStyle="SingleBorderWindow"`**를 유지해야 DWM이 네이티브 그림자, 둥근 모서리, 작업 표시줄 영역 감지를 온전히 수행합니다.
* `WindowChrome.GlassFrameThickness="-1"`을 지정하여 전체 클라이언트 영역으로 DWM 합성을 확장합니다.

### B. Per-Monitor DPI 변환 및 Snap Layouts 히트 테스트 표준
`WM_NCHITTEST`의 `lParam`은 화면 물리 픽셀이므로, `PointFromScreen`을 통해 WPF 로컬 DIP 좌표계로 변환해야 고DPI 화면에서 Snap Layouts가 정상 작동합니다. 또한 `HTMAXBUTTON`을 반환하면 비클라이언트 마우스 메시지(`WM_NCLBUTTONDOWN`/`WM_NCLBUTTONUP`)를 수동으로 중계해야 클릭이 동작합니다.

---

## 11. AI 에이전트 절대 금지 안티패턴 카탈로그

1. **[금지] 갓 윈도우(God Window) & 갓 뷰모델(God ViewModel)**: 파일 하나에 수천 줄의 코드를 작성하지 말고 서브 뷰모델과 분리 컴포넌트로 나눕니다.
2. **[금지] `App.xaml` 내 `StartupUri` 선언**: DI 컨테이너를 사용할 때 창이 중복 생성되는 참사를 원천 차단합니다.
3. **[금지] `async void` 메서드 작성**: 예외 포착이 불가능한 `async void` 선언 금지 (`Task` 반환 강제).
4. **[금지] 비동기 동기 블로킹**: UI 스레드에서 `.Result`, `.Wait()`, `.GetAwaiter().GetResult()` 호출 금지 (데드락 방지).
5. **[금지] 이벤트 구독 미해제**: 서비스 이벤트를 구독한 뷰모델은 반드시 `ICleanup`을 구현하여 메모리 누수를 방지합니다.
6. **[금지] `AllowsTransparency="True"`와 DWM 백드롭 동시 사용**: DWM 하드웨어 가속 합성이 파괴되고 소프트웨어 렌더링으로 전락합니다.

---

## 12. 프로덕션 레디 보일러플레이트 (Production Blueprints)

### Blueprint 1: `MainWindow.xaml` (ContentControl + DialogHost 일체형)
```xml
<Window x:Class="MyWpfApp.Views.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:shell="clr-namespace:System.Windows.Shell;assembly=PresentationFramework"
        Title="Enterprise Application"
        Height="800" Width="1300"
        MinHeight="600" MinWidth="900"
        Background="Transparent"
        WindowStartupLocation="CenterScreen">

    <shell:WindowChrome.WindowChrome>
        <shell:WindowChrome CaptionHeight="44"
                            CornerRadius="0"
                            GlassFrameThickness="-1"
                            NonClientFrameEdges="None"
                            ResizeBorderThickness="6"
                            UseAeroCaptionButtons="False" />
    </shell:WindowChrome.WindowChrome>

    <!-- 최상위 루트 컨테이너 (Mica 반투명 틴트) -->
    <Grid Background="#E60B0C10">
        <Grid.RowDefinitions>
            <RowDefinition Height="44" />
            <RowDefinition Height="*" />
        </Grid.RowDefinitions>

        <!-- 1. 커스텀 타이틀바 -->
        <Border Grid.Row="0" Background="#13141C" BorderBrush="{DynamicResource BorderSubtleBrush}" BorderThickness="0,0,0,1">
            <Grid Margin="16,0">
                <Grid.ColumnDefinitions>
                    <ColumnDefinition Width="Auto" />
                    <ColumnDefinition Width="*" />
                    <ColumnDefinition Width="Auto" />
                </Grid.ColumnDefinitions>

                <TextBlock Grid.Column="0" Text="ENTERPRISE APPLICATION"
                           Foreground="{DynamicResource TextPrimaryBrush}" FontWeight="SemiBold" FontSize="12"
                           VerticalAlignment="Center" />

                <!-- 시스템 창 제어 버튼 -->
                <StackPanel Grid.Column="2" Orientation="Horizontal" shell:WindowChrome.IsHitTestVisibleInChrome="True">
                    <Button Width="36" Height="30" Content="—" Click="MinimizeButton_Click" Background="Transparent" BorderThickness="0" />
                    <Button x:Name="MaximizeButton" Width="36" Height="30" Content="☐" Click="MaximizeButton_Click" Background="Transparent" BorderThickness="0" />
                    <Button Width="36" Height="30" Content="✕" Click="CloseButton_Click" Background="Transparent" BorderThickness="0" />
                </StackPanel>
            </Grid>
        </Border>

        <!-- 2. 메인 네비게이션 영역: ViewModel-First 암시적 DataTemplate 전환 -->
        <ContentControl Grid.Row="1" Content="{Binding Nav.CurrentPage}" />

        <!-- 3. 최상위 In-App 다이얼로그 호스트 (ZIndex 최고층, 스모크 배경) -->
        <Grid Grid.RowSpan="2" Panel.ZIndex="1000" Background="#80000000"
              Visibility="{Binding Dialog.IsOpen, Converter={StaticResource BoolToVis}}">
            <Border Background="{DynamicResource CardBackgroundBrush}"
                    BorderBrush="{DynamicResource BorderSubtleBrush}" BorderThickness="1"
                    CornerRadius="12" Padding="24" MaxWidth="520"
                    VerticalAlignment="Center" HorizontalAlignment="Center">
                <ContentControl Content="{Binding Dialog.CurrentDialog}" />
            </Border>
        </Grid>
    </Grid>
</Window>
```

### Blueprint 2: `MainWindow.xaml.cs` (DWM Mica, Snap Layouts & Per-Monitor DPI)
```csharp
using System;
using System.Runtime.InteropServices;
using System.Windows;
using System.Windows.Interop;
using System.Windows.Media;

namespace MyWpfApp.Views;

public partial class MainWindow : Window
{
    private const int WM_NCHITTEST = 0x0084;
    private const int WM_NCLBUTTONDOWN = 0x00A1;
    private const int WM_NCLBUTTONUP = 0x00A2;
    private const int HTMAXBUTTON = 9;

    private const int DWMWA_USE_IMMERSIVE_DARK_MODE = 20;
    private const int DWMWA_WINDOW_CORNER_PREFERENCE = 33;
    private const int DWMWCP_ROUND = 2;
    private const int DWMWA_SYSTEMBACKDROP_TYPE = 38;

    public enum DWM_SYSTEMBACKDROP_TYPE
    {
        DWMSBT_AUTO = 0,
        DWMSBT_NONE = 1,
        DWMSBT_MAINWINDOW = 2,      // Mica
        DWMSBT_TRANSIENTWINDOW = 3,  // Acrylic
        DWMSBT_TABBEDWINDOW = 4      // Mica Alt
    }

    [DllImport("dwmapi.dll")]
    private static extern int DwmSetWindowAttribute(IntPtr hwnd, int attr, ref int attrValue, int attrSize);

    private bool _isMaxButtonClicked;

    public MainWindow()
    {
        InitializeComponent();
        Loaded += MainWindow_Loaded;
    }

    private void MainWindow_Loaded(object sender, RoutedEventArgs e)
    {
        var hwnd = new WindowInteropHelper(this).Handle;

        // DWM 다크 모드 및 둥근 모서리
        int darkMode = 1;
        DwmSetWindowAttribute(hwnd, DWMWA_USE_IMMERSIVE_DARK_MODE, ref darkMode, sizeof(int));
        int cornerPreference = DWMWCP_ROUND;
        DwmSetWindowAttribute(hwnd, DWMWA_WINDOW_CORNER_PREFERENCE, ref cornerPreference, sizeof(int));

        // Windows 11 22H2+ Mica 활성화 및 버퍼 투명화
        if (Environment.OSVersion.Version.Build >= 22621)
        {
            int backdropType = (int)DWM_SYSTEMBACKDROP_TYPE.DWMSBT_MAINWINDOW;
            DwmSetWindowAttribute(hwnd, DWMWA_SYSTEMBACKDROP_TYPE, ref backdropType, sizeof(int));

            var source = (HwndSource)PresentationSource.FromVisual(this);
            if (source?.CompositionTarget != null)
            {
                source.CompositionTarget.BackgroundColor = Colors.Transparent;
            }
        }

        var hwndSource = HwndSource.FromHwnd(hwnd);
        hwndSource?.AddHook(WndProc);
    }

    private IntPtr WndProc(IntPtr hwnd, int msg, IntPtr wParam, IntPtr lParam, ref bool handled)
    {
        switch (msg)
        {
            case WM_NCHITTEST:
                if (MaximizeButton.IsLoaded)
                {
                    int x = unchecked((short)(lParam.ToInt32() & 0xFFFF));
                    int y = unchecked((short)((lParam.ToInt32() >> 16) & 0xFFFF));
                    Point localPoint = MaximizeButton.PointFromScreen(new Point(x, y));

                    if (new Rect(default, MaximizeButton.RenderSize).Contains(localPoint))
                    {
                        handled = true;
                        return new IntPtr(HTMAXBUTTON); // Windows 11 Snap Layouts 플라이아웃 활성화
                    }
                }
                break;

            case WM_NCLBUTTONDOWN:
                if (wParam.ToInt32() == HTMAXBUTTON)
                {
                    _isMaxButtonClicked = true;
                    handled = true;
                    return IntPtr.Zero;
                }
                break;

            case WM_NCLBUTTONUP:
                if (wParam.ToInt32() == HTMAXBUTTON && _isMaxButtonClicked)
                {
                    _isMaxButtonClicked = false;
                    ToggleMaximize();
                    handled = true;
                    return IntPtr.Zero;
                }
                break;
        }

        return IntPtr.Zero;
    }

    private void ToggleMaximize()
    {
        if (WindowState == WindowState.Maximized)
            SystemCommands.RestoreWindow(this);
        else
            SystemCommands.MaximizeWindow(this);
    }

    private void MinimizeButton_Click(object sender, RoutedEventArgs e) => SystemCommands.MinimizeWindow(this);
    private void MaximizeButton_Click(object sender, RoutedEventArgs e) => ToggleMaximize();
    private void CloseButton_Click(object sender, RoutedEventArgs e) => SystemCommands.CloseWindow(this);
}
```

### Blueprint 3: `MainViewModel.cs` (네비게이션 및 다이얼로그 오케스트레이터)
```csharp
using System.Threading.Tasks;
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using MyWpfApp.Services.Abstractions;
using MyWpfApp.ViewModels.Pages;

namespace MyWpfApp.ViewModels;

public partial class MainViewModel : ObservableObject
{
    public INavigationService Nav { get; }
    public IDialogService Dialog { get; }

    public MainViewModel(INavigationService navigationService, IDialogService dialogService)
    {
        Nav = navigationService;
        Dialog = dialogService;

        // 시작 화면 비동기 로드
        _ = Nav.NavigateToAsync<DashboardViewModel>();
    }

    [RelayCommand]
    private async Task OpenSettingsAsync()
    {
        await Nav.NavigateToAsync<SettingsViewModel>();
    }

    [RelayCommand]
    private async Task RequestExitConfirmationAsync()
    {
        bool confirmed = await Dialog.ShowConfirmAsync("종료 확인", "애플리케이션을 종료하시겠습니까?");
        if (confirmed)
        {
            App.Current.Shutdown();
        }
    }
}
```
