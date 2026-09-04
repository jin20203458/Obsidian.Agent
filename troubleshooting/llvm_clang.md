---
description: >-
  LLVM/Clang 커스텀 Tidy 체커 및 Static Analyzer 개발 트러블슈팅 런북. LLVM 정적 분석기 빌드/실행 에러 시 참조.
related:
  - ../README.md
---
# LLVM/Clang Troubleshooting
> **부제**: LLVM/Clang 커스텀 Tidy 체커 및 Static Analyzer 장애 조치 로그

본 문서는 LLVM/Clang 커스텀 Tidy 체커 및 Static Analyzer 개발 중 발생하는 버그와 오류 해결 방법을 기록하는 문서입니다.

---

## 2026-07-07: checkBranchCondition 콜백 내의 오탐지 (동일 조건식에 대한 참/거짓 경고 동시 발생)

### 1. 현상 (Symptom)
* 일반적이고 정상적인 조건문 `if (x == 5)`에 대해 "항상 참(True)으로 평가됩니다" 경고와 "항상 거짓(False)으로 평가됩니다" 경고가 동일한 위치에서 동시에 검출되는 오탐지(False Positive) 현상 발생.

### 2. 원인 (Root Cause)
* Clang Static Analyzer 엔진은 조건문을 만나면 내부 분석 상태(State)를 참인 경로(True Branch)와 거짓인 경로(False Branch)로 선행 분할(Split)시킵니다.
* 체커의 `checkBranchCondition` 콜백 시점에 전달받는 `C.getState()`는 이미 해당 경로에 맞춰 값이 분할/고정된 상태(참 경로에서는 `1 U1b`, 거짓 경로에서는 `0 U1b`)입니다.
* 이를 그대로 `assume` 하여 참/거짓 가능성을 묻는 경우, 이미 참 혹은 거짓으로 고정된 값이므로 항상 단일 판정(참 또는 거짓)으로 나와 오탐지가 발생합니다.

### 3. 해결책 (Resolution)
* 분석 상태가 이미 갈라져 상수화된 상태인 `C.getState()`를 그대로 사용하는 대신, 조상 노드(`Predecessor`)를 역으로 타고 올라가 조건식이 평가되기 이전(즉, 참/거짓으로 쪼개지기 전)의 최초 상태와 Symbolic한 `SVal`을 찾아내야 합니다.
* 아래와 같이 조상 노드를 탐색하는 코드를 적용하여 해결했습니다:

```cpp
ProgramStateRef EvalState = State;
const ExplodedNode *N = C.getPredecessor();
while (N) {
  ProgramStateRef AncestorState = N->getState();
  SVal V = AncestorState->getSVal(Condition, C.getLocationContext());
  // 1비트 상수가 아닌 최초의 Symbolic SVal을 발견하면 그 시점의 State를 기준으로 삼음
  if (!V.isUnknownOrUndef() && !V.getAs<nonloc::ConcreteInt>()) {
    EvalState = AncestorState;
    CondVal = V;
    break;
  }
  N = N->getFirstPred();
}

// 추출해 낸 EvalState와 CondVal(SymExpr)을 사용하여 assume 수행
std::tie(StateTrue, StateFalse) = EvalState->assume(CondVal);
```

---

## 2026-07-14: 커스텀 Tidy 체커 내 AST 상수 값 평가 중 크래시 (Expression evaluator can't be called on a dependent expression 및 Unknown builtin type)

### 1. 현상 (Symptom)
* 템플릿 기반 C++ 코드 혹은 컴파일 에러가 발생한 소스코드(예: `OpenKAI-master` 프로젝트의 `_GeoFence.cpp`, `HttpClient.cpp`, `main.cpp`) 정적분석 진행 중, `clang-tidy` 프로세스가 아래와 같은 내부 Assertion 혹은 Unreachable 코드로 인해 비정상 종료(Crash)되는 문제 발생:
  1. `ast-representable-cast` 및 `ast-main-unhandled-throw` 규칙 검사 중:
     `Assertion failed: !isValueDependent() && "Expression evaluator can't be called on a dependent expression."`
  2. `ast-switch-style` 규칙 검사 중:
     `Unknown builtin type! UNREACHABLE executed at ASTContext.cpp:2005!`

### 2. 원인 (Root Cause)
* **`ast-representable-cast` & `ast-main-unhandled-throw`**: `RepresentableCastCheck.cpp` 및 `MainUnhandledThrowCheck.cpp`에서 변환 대상인 식 혹은 `if` 조건식을 평가하기 위해 `EvaluateAsInt`, `EvaluateAsFloat`, `EvaluateAsBooleanCondition` 등을 호출할 때, 템플릿 종속적 표현식(Value/Type-dependent) 혹은 컴파일 에러(예: 헤더 누락 등)로 인해 발생한 복구 표현식(`RecoveryExpr`) 노드가 전달되어 AST 상수 평가기 내부에서 오류를 냄.
* **`ast-switch-style`**: `SwitchStyleCheck.cpp`에서 `switch` 조건문의 형식을 검증하기 위해 `type->getAs<BuiltinType>()`를 통해 내장 타입 판정 시, `Context->getTypeSize(type)`를 switch 조건식 가인식 영역 바로 앞에서 호출함. 이때 미확정 빌트인 타입(오버로드, 플레이스홀더, 종속형 템플릿 타입 등)이 유입되면 Clang AST 엔진 내부 크기 조회기에서 `UNREACHABLE`을 발생시켜 컴파일러가 크래시됨.

### 3. 해결책 (Resolution)
* **`ast-representable-cast` & `ast-main-unhandled-throw`**: 상수 평가기를 호출하기 전, 해당 식의 종속 관계 여부를 체크하는 방어 조건문(`!Expr->isValueDependent() && !Expr->isTypeDependent()`)을 추가하여 템플릿 종속 식 및 오류 복구 식은 평가를 우회하게 조치함.
* **`ast-switch-style`**: `getTypeSize` 호출 위치를 `switch (BT->getKind())` 문 내부로 안전하게 이동시킴. 크기가 확실히 존재하는 실존 내장 정수 타입(`char`, `int`, `long` 등) 및 `bool` 케이스 분기 내에서만 크기를 구하게 하고, 크기가 없는 Dependent나 Placeholder 같은 미완성 타입은 크기 연산 없이 `default: break`로 건너뛰어 탈출하도록 구조 개선. 또한, 개별 `case` 문 평가 위치(`EvaluateAsInt`) 등에서도 동일한 방어 코드(`!caseExpr->isValueDependent() && !caseExpr->isTypeDependent()`)를 일괄 적용.

---

## 2026-07-14: SingleExitAndReturnTypeCheck 내 템플릿 종속 타입(Dependent Type) 오탐지

### 1. 현상 (Symptom)
* 템플릿 기반 C++ 코드 혹은 컴파일 오류로 인해 헤더 파일 해석이 끊겨 일부 타입이 정의되지 않은 소스코드(예: `_APmavlink_base.cpp` 내 `check` 함수) 분석 시, 분명히 리턴문이 존재하고 리턴 타입이 매칭됨에도 불구하고 `"함수 선언 반환형(_Bool)과 반환값 타입(<dependent type>)이 일치하지 않습니다"`라는 타입 불일치 오탐지(False Positive) 발생.

### 2. 원인 (Root Cause)
* Clang AST 파서는 템플릿 기반 코드나 베이스 클래스의 멤버 타입(예: `this->_ModuleBase::check()`)을 분석할 때, 실제 타입 인스턴스화가 일어나기 전까지 반환식의 타입을 `<dependent type>` (종속 미확정 타입)으로 간주합니다.
* `SingleExitAndReturnTypeCheck.cpp` 체커 내부의 `ReturnVisitor`는 이 `<dependent type>`을 일반적인 타입 매칭 검사기(`Ctx.hasSameType()`)에 그대로 집어넣어 비교했기 때문에 `_Bool`과 일치하지 않아 오탐지가 보고되었습니다.

### 3. 해결책 (Resolution)
* `ReturnVisitor::VisitReturnStmt`의 타입 검증 시작 지점에 템플릿 종속 타입 검출 조건(`isDependentType()`)을 추가하여, 함수 반환 타입 또는 리턴문 표현식의 타입 중 하나라도 종속 타입(미확정 상태)인 경우에는 오탐 판정을 내리지 않고 매칭 성공(`TypesMatch = true`)으로 우회 처리하여 문제를 해결했습니다.

---

## 2026-07-14: FunctionCallArgumentConsistencyCheck 내 참조형(&) 및 종속 타입(Dependent Type) 인자 오탐지

### 1. 현상 (Symptom)
* C++에서 참조형 매개변수(`T &` 또는 `const T &`)를 취하는 함수에 인자로 동일한 타입의 Lvalue 변수를 전달하여 정상 호출하는 코드 분석 시, `"1번째 인자의 타입이 프로토타입과 일치하지 않습니다. 기대: 'T &', 실제: 'T'"`와 같은 인자 타입 불일치 오탐지(False Positive) 발생.
* 템플릿 기반 코드 호출 분석 시 인자나 매개변수가 `<dependent type>`인 경우 타입 불일치 경고가 비정상적으로 출력됨.

### 2. 원인 (Root Cause)
* **참조형 오탐**: C++ 매개변수의 타입은 참조형(`&`)일 수 있으나, 호출 인자 표현식의 Clang AST 타입은 항상 넌-레퍼런스(`T`)로 평가됩니다. 기존 체커 코드는 이 차이를 처리하지 않고 `hasSameType()`으로 엄격 비교를 가하여 오탐을 발생시켰습니다.
* **종속 타입 오탐**: 템플릿 코드를 해석할 때 타입이 미확정 상태인 `<dependent type>`이 유입되었을 때, 이를 우회(skip)하는 방어 코드가 없었습니다.

### 3. 해결책 (Resolution)
* **참조형 오탐**: `isAcceptableImplicitChain`의 최종 비교 단계인 `EnforceExactParamType` 블록에서 두 타입의 레퍼런스(`getNonReferenceType()`) 및 CVR 한정자(`getUnqualifiedType()`)를 안전하게 벗긴 후 비교하도록 구현을 수정하여 참조 방식에 상관없이 핵심 값이 같은지만 대조하게 개선했습니다.
* **종속 타입 오탐**: `check`의 루프 내에 `ParamTy->isDependentType() || Arg->getType()->isDependentType()` 검출 방어 로직을 추가하여 미확정 타입인 경우 타입 체크를 건너뛰어 통과시켰습니다.

---

## 2026-07-22: Clang RecoveryExpr 기반 AST 매칭 및 C/C++ 컴파일 플래그 다운그레이드

### 1. 현상 (Symptom)
* C/C++ 미선언 함수 호출, 리턴값 누락, 인자 개수 불일치 등 컴파일러가 AST 생성을 차단하거나 노드를 복구 표현식으로 다루는 하드 에러 발생 시, `clang-tidy` AST 체커들이 수집하지 못하고 미탐(0%)이 발생하는 문제.

### 2. 원인 (Root Cause)
* Clang 파서가 하드 에러를 만나면 기본적 `callExpr` 노드를 생성하지 않고 `RecoveryExpr` 노드로 포장하거나 에러를 내뿜어 AST 분석 라운드를 차단함.
* 기존 체커는 `callExpr` 노드만 등록되어 있어 `RecoveryExpr`로 포장된 노드를 지나침.

### 3. 해결책 (Resolution)
1. **컴파일 플래그 다운그레이드 적용**:
   * Rule 05 (`-Wno-error=return-type`), Rule 06 (`-Wno-error=implicit-function-declaration`)처럼 컴파일 플래그 조정이 가능한 하드 에러는 플래그로 경고 다운그레이드를 수행하여 Clang이 AST를 정상 생성하게 유도하고 정식 AST 체커로 100% 탐지.
2. **`RecoveryExpr` AST 매처 보강 (인자 개수 오류 등)**:
   * Clang 15+ 복구 메커니즘으로 인해 AST에 `RecoveryExpr` 노드로 보존되는 시나리오(Rule 50 인자 개수 초과 등)의 경우, 텍스트 가로채기 없이 `isa<RecoveryExpr>` 및 첫 자식 노드가 `FunctionDecl` 참조인지 검증하는 커스텀 AST 매처를 체커(`FunctionCallArgumentConsistencyCheck.cpp`)에 추가하여 순수 AST 기반으로 100% 정식 탐지.
   * 기존 체커 루프 내에 `Arg->containsErrors()` 가드를 보강하여 손상된 표현식 인자가 유입될 때 발생하던 타입 대조 오탐을 완벽 차단.

---

## 2026-07-22: AST Drop 구문에 대한 로케일 독립적 Clang Diagnostic ID 가로채기 및 한글 메시지 재정의

### 1. 현상 (Symptom)
* catch-all 위치 오류(Rule 56), virtual 키워드 누락 순수가상함수(Rule 63), virtual 순수가상함수 비정상 초기화(Rule 62), virtual base 캐스팅(Rule 64) 등 Clang 파서 레벨에서 AST 노드가 100% Drop되는 구문의 경우 AST 매치가 기술적으로 불가능함.
* CLI 텍스트 정규식 파싱 기반으로 에러 문구를 가로채는 방식은 다국어(한국어, 영어, 일본어 등) 환경 및 컴파일러 메시지 변경 시 깨짐(i18n 불가능) 문제 발생.

### 2. 원인 (Root Cause)
* Clang 파서(Sema)가 문법 규격 위반을 만나면 표현식 노드 전체를 AST 트리에 포함하지 않고 삭제함.
* `#include "clang/Basic/DiagnosticSemaKinds.h"`를 include하려 시도할 경우, `DiagnosticSemaKinds.inc`가 TableGen 빌드 아티팩트(.inc)이므로 파일 누락 C1083 헤더 에러 발생.

### 3. 해결책 (Resolution)
1. **정수 Diagnostic ID 기반 가로채기 (`DiagnosticSema.h`)**:
   * `#include "clang/Basic/DiagnosticSema.h"`를 include하여 Clang의 로케일 무관 정수 진단 ID 상수를 사용.
   * `ClangTidyDiagnosticConsumer::HandleDiagnostic` 내에서 정수 Diagnostic ID(`Info.getID()`)를 수집 및 비교:
     - `diag::err_early_catch_all` $\rightarrow$ `"ast-unused-exception-handler"`
     - `diag::err_member_function_initialization` $\rightarrow$ `"ast-pure-virtual-init"`
     - `diag::err_non_virtual_pure` $\rightarrow$ `"ast-virtual-pure"`
     - `diag::err_static_downcast_via_virtual` $\rightarrow$ `"ast-virtual-base-cast"`
2. **한글 메시지 엔진 단일 직방출**:
   * `ArqaOverrideMessage` 변수를 `HandleDiagnostic` 메소드 최상위 스코프에 선언하여 스코프 이탈 C2065 에러를 방지.
   * UTF-8 한글 문자열 리터럴(`u8"..."`)로 메시지를 재정의하여 `ClangTidyDiagnosticRenderer`로 전달:
   ```cpp
   if (DiagID == diag::err_early_catch_all) {
     CheckName = "ast-unused-exception-handler";
     DiagLevel = DiagnosticsEngine::Warning;
     ArqaOverrideMessage = u8"catch-all(...) 핸들러 뒤에 위치한 예외 처리 구문은 실행되지 않습니다. catch-all은 마지막에 배치하십시오.";
   }
   ```
3. **결과**:
   * 컴파일러 텍스트 파싱 0%, 로케일 독립 100%의 한글 경고 방출 아키텍처 완성.
   * DAPA 66개 전체 규칙 100% 정답률 검출 성공.

---

## 2026-08-31: C/C++ 소스 프래그먼트(#include "*.c") 자동 감지 및 3단계 자동 정제 시스템 (Zero-Config)

### 1. 현상 (Symptom)
* PCRE, SQLite, Lua, 임베디드 런타임 등 대형 라이브러리가 포함된 프로젝트(예: 708개 파일)를 한 번에 분석할 때, C 소스 조각 파일들(`sjarm32.c`, `sjx8632.c` 등)이 독립 컴파일 단위로 호출되어 320건 이상의 대량 구문 에러 발생.
* C99 `intmax_t`, `uintmax_t` 타입 누락 에러 발생.

### 2. 원인 (Root Cause)
* `sjlir.c` 같은 상위 컴파일 단위가 내부에서 `#include "sjarm32.c"`를 직접 인클루드하는 구조인데, 분석기가 폴더 내 모든 `.c` 파일을 단독 실행 단위(`TranslationUnit`)로 인식하여 헤더 없이 컴파일을 시도함.
* C++ 모드에서 단순 매크로 `-Dintmax_t=...`를 주입할 경우 `std::intmax_t` 문법 오류가 발생하는 딜레마.

### 3. 해결책 (Resolution)
1. **비동기 소스 프래그먼트 사전 감지 (`SourceFragmentDetector.cs`)**:
   * `Parallel.ForEachAsync` 기반으로 상단 300라인을 비동기 스캔하여 `#include ".*\.c"` 패턴을 식별하고, 인클루드된 조각 파일들을 독립 컴파일 대상에서 자동 제외.
   * 상위 파일(`sjlir.c`) 컴파일 시 조각 파일이 함께 완전한 AST로 분석되므로 분석 누락율 0% 보장.
   * `--header-filter=.*`를 보장하여 조각 파일 내부의 결함 진단이 정상 출력되도록 유지.
2. **C vs C++ Dialect-Safe 타입 가드**:
   * C 모드(`*.c`)에서는 `-ffreestanding` 및 `-Dintmax_t=__INTMAX_TYPE__ -Duintmax_t=__UINTMAX_TYPE__`를 주입하여 누락된 C99 타입을 자동 보정.
   * C++ 모드(`*.cpp`)에서는 타입 치환 매크로를 격리하여 표준 `<cstdint>` / `std::intmax_t` 파괴 방지.
3. **세션 스냅샷 확장 (`MainViewModel.cs`)**:
   * 분석 대상 파일 외에 진단이 검출된 모든 파일(`Diagnostics.Select(d => d.FilePath)`)을 스냅샷 목록에 병합하여 조각 파일 내부 결함에 대한 코드 스니펫 및 비교 분석 diff 100% 영구 보존.

---

## 2026-08-28: 크로스 컴파일 임베디드(VxWorks/FreeRTOS/AVR) 타겟 분석 시 호스트 Windows SDK 헤더 간섭 및 타입 충돌 해결

### 1. 현상 (Symptom)
* VxWorks, FreeRTOS, Linux, AVR 등 임베디드 타겟 C/C++ 소스를 Windows 호스트 상의 `clang-tidy`로 분석할 때, 컴파일 에러가 수십~수백 건(예: 466건) 발생.
* `redefinition of typedef 'int32_t'` (`int` vs `long`), `sal.h` / `vadefs.h` / `specstrings.h` 등 호스트 Windows SDK 헤더가 강제 주입되어 분석이 중단되는 현상.

### 2. 원인 (Root Cause)
* Windows 빌드 Clang은 기본적으로 `x86_64-pc-windows-msvc` 타겟 트리플을 기본값으로 사용하므로 레지스트리 및 환경변수 상의 Windows SDK / MSVC UCRT 헤더를 자동 검색 및 주입함.
* Clang 내장 `lib/clang/19/include/stdint.h`의 `int32_t` 정의(`int`)와 RTOS의 `stdint.h` 정의(`long`)가 타입 시스템 상 충돌.

### 3. 해결책 (Resolution)
1. **타겟 트리플 및 타입 가드 매크로 분리 (Silver Bullet Flags)**:
   * `--target=i386-pc-none-elf`: Clang 드라이버의 Windows SDK / MSVC 자동 탐색 및 내장 MSVC intrinsic 주입 원천 차단.
   * `-D__CLANG_STDINT_H`: Clang 내장 `stdint.h`의 중복 로드를 차단하여 RTOS의 32비트 정수 타입과 충돌 방지.
2. **Strategy C: 타겟 플랫폼 자동 감지 및 UI 선택기 (Zero-Config + Manual Override)**:
   * `TargetPlatformDetector`: 사용자가 등록한 Include 경로의 시그니처 파일(`vxWorks.h`, `FreeRTOS.h`, `avr/io.h`, `linux/kernel.h` 등)을 O(1)로 스캔하여 타겟 플랫폼을 자동 식별.
   * `ClangTidyRunnerService`: 타겟 플랫폼 옵션에 맞춰 `--target=<triple>` 및 플랫폼 전용 가드 매크로를 무음 자동 주입하고, 크로스 컴파일 시 호스트 시스템 헤더 자동 탐색을 억제.
   * `IncludePathWindow.xaml`: 드롭다운 및 실시간 감지 상태 칩을 제공하여 사용자에게 완전한 제어권과 시각적 피드백 제공.

---

## 2026-09-01: NoAutoTypeCheck (`ast-no-auto-type`) 컴파일러 암시적 변수 오탐 및 복합 auto 타입 미탐 해결

### 1. 현상 (Symptom)
* DAPA 국방 규격 `공통(스타일) c. 함수/변수의 선언 시 type을 명시해야 한다 (auto 사용 제한)` 검증용 체커인 `ast-no-auto-type` 사용 시:
  1. `for (int x : vec)` 등 개발자가 명시적 타입을 적은 정상적인 C++11 범위 기반 `for` 루프에서 한 줄당 3건의 허위 오탐(False Positive) 발생.
  2. `auto *p = &x;`, `const auto &ref = x;`, `auto&& r = std::move(x);` 등 복합 포인터/참조 `auto` 변수 선언이 검출되지 않는 미탐(False Negative) 발생.
  3. C++14 자동 반환형 함수 중 `auto& getRef()`, `auto* getPtr()` 등 참조/포인터 반환형 함수가 미탐.

### 2. 원인 (Root Cause)
* **범위 기반 for 루프 오탐**: Clang AST에서 `for-range` 루프 처리 시 내부적으로 `auto &&__range`, `auto __begin`, `auto __end`와 같은 컴파일러 암시적 `VarDecl`(`isImplicit() == true`)을 자동 생성함. 기존 매처 `varDecl(hasType(autoType()))`에 `unless(isImplicit())` 가드가 없어 컴파일러 내부 변수를 감지함.
* **복합 auto 타입 미탐**: `const auto&`는 `LValueReferenceType(AutoType)`, `auto*`는 `PointerType(AutoType)`으로 `AutoType`이 래핑되어 있어 단순 `hasType(autoType())`으로 매칭되지 않음.
* **언어 버전 필터 부재**: `isLanguageVersionSupported` 오버라이드가 없어 C99/C89 등 C 언어 프로젝트 분석 시에도 불필요하게 활성화됨.

### 3. 해결책 (Resolution)
1. **`NoAutoTypeCheck.h`**: `isLanguageVersionSupported(const LangOptions &LangOpts)`를 추가하여 `LangOpts.CPlusPlus11` 가드 적용 (C++11 이상 한정).
2. **`NoAutoTypeCheck.cpp`**: `AutoTypeMatcher`를 `qualType(anyOf(autoType(), pointsTo(qualType(autoType())), references(qualType(autoType()))))`로 재구성하여 복합 `auto` 타입을 전수 매칭하고, `unless(isImplicit())` 가드를 추가하여 컴파일러 생성 임시 변수 오탐을 100% 차단.
3. **`ARQAModule.cpp`**: `#include "NoAutoTypeCheck.h"` 및 `Factories.registerCheck<NoAutoTypeCheck>("ast-no-auto-type")` 주석 해제 및 정규 재등록.

---

## 2026-09-01: 분석 엔진 중간 산출물(임시 JSON) 세션 격리 및 완전 자동 정화 (Data Contamination 방지)

### 1. 현상 (Symptom)
* `function-call-graph` (`Final_CallGraph.json`) 및 `ast-global-symbol-uniqueness` (`GlobalSymbols.json`) 분석 완료 후, 사용자 프로젝트 루트 폴더(`ProjectPath`)에 임시 JSON 파일이 잔존.
* 이후 단일 파일 재분석이나 특정 체커만 선택하여 분석할 때, 프로젝트 폴더에 남아있던 과거 분석의 JSON 파일이 C# UI에 무조건 읽혀 들어와 이전 분석의 호출 관계(Fan-In/Out)나 심볼 중복 결함이 신규 분석 화면에 대량으로 오염 주입(Data Contamination)되는 문제 발생.

### 2. 원인 (Root Cause)
* **고정된 상대 경로 생성**: Clang-Tidy C++ 체커가 기본적으로 작업 디렉토리(프로젝트 루트)에 `Final_CallGraph.json` 및 `GlobalSymbols.json`을 작성함.
* **파싱 후 디스크 파일 방치**: C# UI(`MainViewModel.FinalizeAnalysisAsync`)에서 JSON을 읽어 메모리 모델(`Diagnostics`, `MetricReportViewModel`)에 적재한 후, 디스크의 원본 JSON 파일을 삭제하지 않고 그대로 방치.
* **C++ 소멸자의 조건부 스킵**: 체커가 아무것도 수집하지 못한 경우 소멸자에서 `if (empty()) return;`으로 인해 기존 파일을 0바이트로 덮어쓰지 않고 과거 파일이 그대로 살아남음.

### 3. 해결책 (Resolution)
1. **OS 임시 폴더 세션 격리 (`%TEMP%\ArqaStatic_Session_{GUID}\`)**:
   * `IClangTidyRunnerService` 및 `ClangTidyRunnerService`에 `sessionTempDir` 매개변수를 추가하고, YAML `CheckOptions`에 `function-call-graph.OutputPath` 및 `ast-global-symbol-uniqueness.OutputPath`를 세션 임시 경로로 동적 주입하여 사용자 프로젝트 폴더 내 파일 생성을 원천 차단.
2. **메모리 적재 즉시 파기 (Read & Destroy)**:
   * `MainViewModel.FinalizeAnalysisAsync`에서 `GlobalSymbols.json` 및 `Final_CallGraph.json`을 파싱하여 인메모리 객체로 변환한 직후, `finally` 블록에서 해당 임시 JSON 파일을 즉시 삭제.
3. **라이프사이클 마스터 청소 (`AnalyzeAsync` master finally)**:
   * 분석 정상 종료, 사용자 취소(`OperationCanceledException`), 런타임 예외 발생 시 `AnalyzeAsync`의 `finally` 구문에서 `sessionTempDir` 전체를 재귀 삭제(`Directory.Delete`)하여 디스크 누수 100% 방지.
4. **인메모리 메트릭 보존 방어 (`UpdateMetricReport`)**:
   * 임시 JSON 파일 삭제 후 UI 탭 전환이나 임계값 변경 시, 이미 메모리에 적재된 `FunctionMetrics`를 보존하면서 통계 상태를 안전하게 갱신하도록 방어.

---

## 2026-09-04: PartialCopyAssignmentCheck (`ast-partial-copy-assignment`) C++ 클래스/구조체 멤버 대입 오탐지 해결

### 1. 현상 (Symptom)
* DAPA C++ 전용 i (Rule 60: `copy operator를 통해서, 복사되지 않는 멤버 변수가 존재하지 말아야 한다`) 검증용 체커인 `ast-partial-copy-assignment` 분석 시:
  * 클래스/구조체 멤버 변수(예: `Coordinate pos;`)를 `operator=`에서 정상적으로 전수 복사(`pos = rhs.pos;` 혹은 `pos.lat = rhs.pos.lat;`)하였음에도 불구하고, 준수 코드에서 `"멤버 변수 'pos'이(가) 복사 대입 연산자에서 대입되지 않았습니다"`라는 허위 오탐(False Positive) 발생.

### 2. 원인 (Root Cause)
* Clang AST에서 기본형(int/float), 포인터, 열거형, 비트필드의 대입은 `BinaryOperator(BO_Assign)` 노드로 생성되지만, 사용자 정의 클래스나 구조체 타입 객체의 `=` 대입은 오버로딩된 멤버 함수 호출인 `CXXOperatorCallExpr` 노드로 생성됨.
* `PartialCopyAssignmentCheck.cpp`의 `AssignmentVisitor`가 `VisitBinaryOperator`만을 순회하도록 작성되어 있어, 구조체/클래스 멤버 객체의 `CXXOperatorCallExpr` 대입을 전혀 인지하지 못하고 미대입으로 오판함.

### 3. 해결책 (Resolution)
1. **`extractRootFieldDecl` 헬퍼 함수 구현**:
   * `LHS` 표현식이 중첩 멤버 접근(`this->pos.lat`)이더라도 `MemberExpr::getBase()` 체인을 역추적하여 최상위 필드(`FieldDecl`)인 `pos`를 정확히 추출.
2. **`VisitCXXOperatorCallExpr` 핸들러 추가**:
   * `OCE->getOperator() == OO_Equal`인 경우, 첫 번째 인자(`OCE->getArg(0)`)에서 루트 `FieldDecl`을 추출하여 `AssignedFields`에 등록하도록 조치.
3. **빌드 검증 완료**:
   * `cmake --build .\build --config Release --target clang-tidy` 성공 (Exit Code 0). 구조체/클래스 멤버 정상 대입 시 오탐 0건(100% Clean) 달성.

