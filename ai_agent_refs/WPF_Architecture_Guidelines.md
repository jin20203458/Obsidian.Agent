# WPF 현대적인 MVVM 아키텍처 가이드 (AI 프롬프트/CursorRules 템플릿)

본 문서는 현대적인 WPF(.NET 8.0 이상) 애플리케이션을 빌드할 때 AI 코딩 에이전트(GitHub Copilot, Cursor, Claude 등)가 참조해야 할 완벽한 지침서입니다. AI가 자주 범하는 **잘못된 패턴(Anti-Patterns)을 엄격히 금지**하고, CommunityToolkit.Mvvm의 소스 제너레이터(Source Generators)를 100% 활용하도록 유도하기 위해 작성되었습니다.

---

## 1. 프로젝트 디렉터리 구조

역할 분담과 유지보수성을 극대화하기 위해 아래와 같은 표준 디렉터리 배치를 사용합니다.

```text
📁 MyWpfApp/
├── 📁 Config/           # 앱 설정 파일 (JSON, XML, 환경설정 등)
├── 📁 Models/           # 순수 데이터 구조체 (POCO), 데이터베이스/API 엔티티
├── 📁 Services/         # 비즈니스 로직, API 통신, 파일 I/O, DB 접근 레이어
│   ├── IApiService.cs
│   └── ApiService.cs
├── 📁 ViewModels/       # UI 프레젠테이션 로직 (CommunityToolkit.Mvvm 상속)
│   ├── MainViewModel.cs
│   └── SubViewModel.cs
├── 📁 Views/            # XAML UI 선언 및 최소한의 비하인드 코드
│   ├── MainWindow.xaml
│   └── MainWindow.xaml.cs
├── 📁 Resources/        # 정적 에셋 (아이콘, 이미지 등)
├── App.xaml             # 애플리케이션 시작점 및 전역 리소스 정의
├── App.xaml.cs          # DI(의존성 주입) 설정 및 초기 구동 로직
└── MyWpfApp.csproj      # 최신 SDK 스타일의 프로젝트 파일
```

---

## 2. 의존성 주입 (DI) 설정 (`App.xaml.cs`)

`Microsoft.Extensions.DependencyInjection`을 사용하여 전역 컨테이너를 구성합니다.

```csharp
using Microsoft.Extensions.DependencyInjection;
using System;
using System.Windows;
using MyWpfApp.Services;
using MyWpfApp.ViewModels;
using MyWpfApp.Views;

namespace MyWpfApp
{
    public partial class App : Application
    {
        // 뷰(View) 단에서 서비스가 필요할 때 전역에서 DI 컨테이너에 접근할 수 있게 제공
        public new static App Current => (App)Application.Current;
        public IServiceProvider Services { get; }

        public App()
        {
            Services = ConfigureServices();
        }

        private static IServiceProvider ConfigureServices()
        {
            var services = new ServiceCollection();

            // 1. 핵심 서비스 등록 (보통 싱글톤)
            services.AddSingleton<ISettingsService, JsonSettingsService>();
            services.AddHttpClient<IApiService, ApiService>(); // Factory 패턴

            // 2. ViewModels 등록
            services.AddSingleton<MainViewModel>(); // 앱 전역 뷰모델
            services.AddTransient<SettingsViewModel>(); // 열 때마다 새로 생성

            // 3. Views 등록
            services.AddTransient<MainWindow>();

            return services.BuildServiceProvider();
        }

        protected override void OnStartup(StartupEventArgs e)
        {
            base.OnStartup(e);

            var mainWindow = Services.GetRequiredService<MainWindow>();
            mainWindow.DataContext = Services.GetRequiredService<MainViewModel>();
            mainWindow.Show();
        }
    }
}
```

---

## 3. AI 에이전트를 위한 핵심 준수 규칙 및 안티패턴 (절대 금지 사항)

AI 에이전트가 코드를 작성할 때 다음의 **명시적 제한 사항과 규칙**을 절대적으로 준수해야 합니다. (이 항목들을 프롬프트나 `.cursorrules` 파일에 그대로 복사하여 사용하십시오.)

### 🚨 [규칙 1] 의존성 주입 (Dependency Injection) 안티패턴 금지
* **절대 금지 (Service Locator)**: 뷰모델(ViewModel) 내부에서 `Ioc.Default.GetService<T>()` 또는 `App.Current.Services.GetRequiredService<T>()`를 절대 호출하지 마십시오.
* **해결책**: 모든 외부 서비스, 리포지토리, 하위 뷰모델은 **반드시 생성자 주입(Constructor Injection)**을 통해서만 받아야 합니다.
* 단, XAML에 묶인 View의 비하인드 코드(`.xaml.cs`) 내부에서는 예외적으로 `App.Current.Services`를 통해 뷰모델을 Resolve할 수 있습니다.

### 🚨 [규칙 2] CommunityToolkit.Mvvm 소스 제너레이터 규칙
* **절대 금지**: `INotifyPropertyChanged` 인터페이스를 수동으로 구현하지 마십시오.
* **필수 사항**: 모든 뷰모델은 반드시 `ObservableObject`를 상속받아야 하며, 클래스 선언에 반드시 `partial` 키워드를 포함해야 합니다.
* **필드 네이밍**: 상태 관리가 필요한 필드는 반드시 `camelCase` 혹은 `_camelCase`로 선언하고 `[ObservableProperty]` 속성을 부여하십시오. AI가 실수로 필드명을 `PascalCase`로 짓게 되면 소스 제너레이터가 생성하는 프로퍼티명과 충돌하여 컴파일 에러가 발생합니다.
* **의존 속성**: 특정 프로퍼티가 변경될 때 다른 프로퍼티도 UI 업데이트가 필요하다면 getter 내부에서 이벤트를 수동으로 호출하지 말고, 원본 필드에 `[NotifyPropertyChangedFor(nameof(파생프로퍼티명))]` 속성을 사용하십시오.

### 🚨 [규칙 3] 비동기 커맨드(Commands)의 `async void` 절대 금지
* **절대 금지**: 커맨드 메소드를 정의할 때 `async void`를 절대 사용하지 마십시오. (예외 처리가 불가능하여 앱 크래시를 유발합니다.)
* **해결책**: 모든 비동기 커맨드 메소드는 반드시 `Task`를 반환해야 합니다. `[RelayCommand]` 속성을 붙이면 소스 제너레이터가 안전한 `IAsyncRelayCommand` 객체를 자동으로 생성해 줍니다.
* **상태 연동**: 커맨드 실행 가능 여부(`CanExecute`)가 특정 속성 값에 의존하는 경우, 해당 속성 필드에 `[NotifyCanExecuteChangedFor(nameof(커맨드이름Command))]`를 명시하십시오.

### 🚨 [규칙 4] 메신저(IMessenger) 메모리 누수 방지
* **절대 금지**: `WeakReferenceMessenger`를 등록할 때 람다식 내부에 `this`를 캡처하는 인스턴스 메소드를 직접 참조하지 마십시오. (메모리 누수 원인)
* **해결책**: 메시지 수신 로직을 작성할 때는 반드시 `static` 람다를 사용하십시오. 예: `(r, m) => r.OnMessage(m)`. 또는 `ObservableRecipient`를 상속받아 `IRecipient<TMessage>` 인터페이스를 구현하고 `Messenger.RegisterAll(this)`를 사용하십시오.

---

## 4. 완벽한 ViewModel 템플릿 예시

위 규칙들이 완벽하게 적용된 뷰모델 템플릿입니다. AI는 새로운 화면을 만들 때 이 형태를 그대로 모방해야 합니다.

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using System.Threading.Tasks;
using MyWpfApp.Services;

namespace MyWpfApp.ViewModels
{
    // [규칙 2] partial 클래스 및 ObservableObject 상속
    public partial class MainViewModel : ObservableObject
    {
        private readonly IApiService _apiService;

        // [규칙 1] 반드시 생성자를 통해 의존성을 주입받음 (Service Locator 패턴 금지)
        public MainViewModel(IApiService apiService)
        {
            _apiService = apiService;
            Title = "현대적인 WPF MVVM 템플릿";
        }

        // [규칙 2] 필드는 _camelCase, 속성은 [ObservableProperty]
        // 컴파일 시 public string Title 프로퍼티가 자동 생성됨
        [ObservableProperty]
        private string _title = string.Empty;

        [ObservableProperty]
        [NotifyCanExecuteChangedFor(nameof(SubmitCommand))] // [규칙 3] IsProcessing 변경 시 SubmitCommand의 CanExecute 재평가
        private bool _isProcessing;

        private bool CanSubmit() => !IsProcessing;

        // [규칙 3] async void 절대 금지. Task 반환형 사용
        // 컴파일 시 public IAsyncRelayCommand SubmitCommand 자동 생성
        [RelayCommand(CanExecute = nameof(CanSubmit))]
        private async Task SubmitAsync()
        {
            IsProcessing = true;
            try
            {
                await _apiService.SendDataAsync(Title);
            }
            finally
            {
                IsProcessing = false;
            }
        }
    }
}
```
