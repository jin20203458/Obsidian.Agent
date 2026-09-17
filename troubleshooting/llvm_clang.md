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
핵심 원칙 및 방향성에도 나와있듯이 테스트케이스를 인위적으로 ARQA에 유리하게 작성하지말고 실무패턴에서 ARQA의 약점이 있으면 언제든 수정해서 업그레이드가 가능해야해.
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

---

## 2026-09-07: ThreadLockChecker (`path-sensitive-arqa.ThreadLock`) 함수 조기 반환 락 누수 미탐 및 NewDeleteLeaks 연동 해결

### 1. 현상 (Symptom)
* CWE-404(부적절한 자원 해제) 정적 검증 시, `pthread_mutex_lock` 또는 `EnterCriticalSection` 획득 후 조건부 에러 분기나 조기 반환(`return -1;`) 시 `unlock`을 호출하지 않는 심각한 교착 상태(데드락) 결함에 대해 `ThreadLockChecker`가 0건 미탐(Silent Failure)을 발생시킴.
* 또한 C++ `new`/`new[]`로 할당된 힙 메모리 누수 검출을 위한 `path-sensitive-cplusplus.NewDeleteLeaks` 체커가 UI/프로필에 등록되지 않아 C++ 힙 누수가 연동되지 못함.

### 2. 원인 (Root Cause)
1. **`checkDeadSymbols` 콜백의 구조적 한계**:
   * 전역 변수나 포인터로 전달된 동기화 객체(`pthread_mutex_t`)는 함수가 종료되거나 반환된 이후에도 메모리 상에 살아있어 `SymReaper.isLiveRegion(LockR)`이 항상 `true`를 반환함.
   * 따라서 심볼 소멸 기반 검사기(`checkDeadSymbols`)로는 함수 조기 탈출 시점의 락 누수를 결코 인지할 수 없음.
2. **함수 종료 검열 콜백(`check::EndFunction`) 부재**:
   * 함수 실행 흐름이 완료되는 시점(`ReturnStmt` 또는 함수 바디 끝)에서 잔여 잠금 상태를 평가하는 훅이 누락되어 있었음.
3. **인라인 래퍼 함수 오탐 방지 가드 부재**:
   * 락 획득 전용 래퍼 함수(예: `void acquire(pthread_mutex_t *m) { pthread_mutex_lock(m); }`)의 경우 상위 호출자에게 잠긴 상태로 반환하는 것이 정상이므로, 하위 인라인 프레임 종료 시점에 조기 진단하면 허위 오탐이 발생함.
4. **에러 노드 체이닝 오류**:
   * `reportBug()` 내부에서 `C.generateErrorNode()`를 무조건 재호출하여 선행 노드와의 연결이 끊기거나 싱크 노드 중복 생성으로 인해 분석이 중단되거나 리포트가 소실됨.

### 3. 해결책 (Resolution)
1. **`ThreadLockChecker.cpp` 아키텍처 개편**:
   * `LockEntry` 구조체 정의 (`LockState State; const StackFrameContext *AcquiredFrame;`) 및 `REGISTER_MAP_WITH_PROGRAMSTATE(LockMap, const MemRegion *, LockEntry)` 도입.
   * `check::EndFunction` 인터페이스 구현:
     - `if (!C.inTopFrame()) return;` 가드레일을 적용하여 인라인 락 래퍼의 조기 오탐 원천 차단.
     - 현재 최상위 스택 프레임(또는 그 인라인된 자식 프레임)에서 획득된 후 함수 종료 시점까지 `Locked` 상태인 락 객체를 전수 색출.
     - `generateNonFatalErrorNode`를 적용하여 에러 노드 체이닝 보장.
2. **체커 레지스트리 및 한글화 완비**:
   * `Checkers.json`, `ComplianceRuleProvider.cs`, `CheckerProfiles.json`에 `path-sensitive-cplusplus.NewDeleteLeaks` 및 `path-sensitive-arqa.ThreadLock` 정규 등록 및 한글 프로필 연동.
   * C# 솔루션 컴파일(`dotnet build`): 경고 0개, 오류 0개 (Exit Code 0).
3. **36종 초고강도 리그레션 테스트 검증**:
   * `test_cwe404_regression_36.cpp`: `NewDeleteLeaks`(12종), `ThreadLock`(12종), `Stream`(6종), `Malloc`(6종).
   * 취약 18건 전수 정탐 (100.0%), 준수 18건 오탐 0건 (100% Clean / 0.0% FP).
4. **공식 1:1 실측 완료**:
   * Cppcheck 3/4 (75.0%) vs ARQA 4/4 (100.0%) [ARQA 우세].

---

## 2026-09-16: MultiStatementPerLineCheck (`ast-multi-statement-per-line`) 매크로 전개 누수, typedef struct 및 파일 간 라인 충돌 오탐 474건 전수 해결

### 1. 현상 (Symptom)
* DAPA 스타일 규칙 Rule 11 (카. 한 줄에 하나의 명령문을 사용한다) 및 MISRA C:2012 Rule 5.9 정적 검증 시, `mbedtls` 프로젝트 96개 소스 파일 대상 전수 분석에서 총 1,016건 중 **474건(46.7%)의 대규모 엔진 오탐(False Positive)** 발생:
  1. `sha1.c`, `md5.c`, `ripemd160.c`, `psa_crypto.c` 등 단일 라인 매크로 호출(`P(...)`, `LOCAL_INPUT_FREE(...)`)에 대해 414건의 오탐 발생.
  2. `oid.c`, `asn1parse.c` 등 `typedef struct { ... } name_t;` 선언에 대해 59건의 다중 전역 선언 오탐 발생.
  3. `psa_crypto_aead.c:25`의 `union { ... } ctx;` 인라인 공용체 필드 선언에 대해 1건의 다중 멤버 선언 오탐 발생.
  4. 복수 헤더 파일과 메인 파일 간 동일 라인 번호 충돌로 인한 허위 다중 전역 선언 오탐 발생.

### 2. 원인 (Root Cause)
1. **매크로 인자 확장 위치(`SM.isMacroArgExpansion`)와 본문 위치 불일치**:
   * 블록 없는 다중 문장 매크로(예: `LOCAL_INPUT_FREE(input, copy)`)에서 첫 문장이 인자 토큰으로 시작하면 `SM.getExpansionLoc`가 호출부 인자 위치(Col 37)를, 본문 시작 문장은 매크로 이름 위치(Col 5)를 반환함.
   * `ExpLoc.getRawEncoding()`이 달라 `macroExpansionSeen` 중복 필터를 우회하여 동일 라인에 복수 문장으로 등록됨.
2. **매크로 정의부 `CompoundStmt` 진입**:
   * `compoundStmt(unless(hasParent(functionDecl())))` 매처가 `do { ... } while(0)` 매크로 내부의 중괄호 블록까지 매칭하여, 매크로 본문의 다중 문장을 호출부 단일 라인의 다중 문장으로 오인함.
3. **`typedef struct` AST 듀얼 노드 생성**:
   * Clang AST는 `typedef struct { ... } name_t;` 구문에 대해 `RecordDecl`과 `TypedefDecl` 2개 노드를 동일 시작 위치에 생성함. 쉼표가 없다는 이유로 2개 선언으로 오인함.
4. **인라인 태그 멤버 중복 등록**:
   * `union { ... } ctx;` 선언 시 공용체 타입 정의 `RecordDecl`과 멤버 `FieldDecl`이 동일 좌표에 생성되어 별개 멤버 2건으로 계산됨.
5. **파일 간 라인 번호 충돌 (`FileID` 누락)**:
   * `declLineMap`과 `memberLineMap`의 키가 `unsigned line` 단일 값으로 되어 있어, 헤더 파일(`rsa.h:240`)과 메인 소스(`oid.c:240`)의 서로 다른 파일 선언이 단일 엔트리로 병합되어 오탐을 유발함.

### 3. 해결책 (Resolution)
1. **최상위 매크로 호출 위치 정규화 헬퍼(`getTopMacroInvocationLoc`) 구현**:
   * `SM.getTopMacroCallerLoc(Loc)` 및 `SM.getExpansionRange(TopLoc).getBegin()`을 적용하여, 매크로 인자든 본문이든 항상 호출부 시작점(Col 5)의 단일 좌표로 정규화.
   * `macroExpansionSeen`에 의해 단일 매크로 호출에서 파생된 후속 문장은 100% 차단(`continue`).
2. **매크로 내부 `CompoundStmt` 조기 탈출 가드**:
   * `checkCompoundStmt` 시작부에 `if (CS->getBeginLoc().isMacroID()) return;` 가드를 추가하여 매크로 래퍼 루프 내부 블록의 진입을 원천 차단.
3. **`typedef struct/enum/union` 연계 선언 필터링**:
   * `TagDecl::getTypedefNameForAnonDecl()` 또는 `TypedefNameDecl::getUnderlyingType()->getAsTagDecl()`과 일치하는 `TagDecl`은 `TypedefDecl`에 종속된 부속 노드로 판정하여 선언 카운트에서 제외.
4. **인라인 태그 멤버 필터링**:
   * 동일 레코드 내 `FieldDecl->getType()->getAsTagDecl()`과 일치하는 `RecordDecl`은 독립 멤버 카운트에서 제외.
5. **맵 키 `std::pair<FileID, unsigned>` 정밀화**:
   * `declLineMap`, `lineStmtMap`, `memberLineMap`의 키를 `std::pair<FileID, unsigned>`로 전면 교체하여 서로 다른 파일 간 라인 번호 충돌을 원천 차단.

### 4. 검증 결과 (Ground Truth)
* **컴파일 빌드**: `cmake --build .\build --config Release --target clang-tidy` $\rightarrow$ **Exit Code 0** 무결점 성공.
* **오탐 파일 실사**:
  - `sha1.c`: 80건 오탐 $\rightarrow$ **0건 전수 소멸 (100% 해결)**
  - `md5.c`: 64건 오탐 $\rightarrow$ **0건 전수 소멸 (100% 해결)**
  - `ripemd160.c`: 80건 오탐 $\rightarrow$ **0건 전수 소멸 (100% 해결)**
  - `psa_crypto.c`: 82건 오탐 $\rightarrow$ **0건 전수 소멸 (100% 해결)**
  - `oid.c`: 43건 오탐 $\rightarrow$ **0건 전수 소멸 (100% 해결)**
  - `psa_crypto_aead.c`: 3건 오탐 $\rightarrow$ **0건 전수 소멸 (100% 해결)**
* **정탐 보존 실사**:
  - `aes.c`: 한 줄 다중 대입(`415`), `case break`(`599`) 등 **30건 진성 규격 정탐 100% 보존**.
  - `ecp_curves.c`: 루프 내 다중 연산(`ADD; NEXT;` 등) **진성 규격 정탐 100% 보존**.

---

## 2026-09-16: NoMeaninglessExprCheck (`ast-no-meaningless-expr`) switch-case 라벨 상수 평가식 매칭 결함 오탐 247건 전수 해결

### 1. 현상 (Symptom)
* DAPA 스타일 규칙 Rule 4 (라. 부작용 없는 의미 없는 구문 사용 금지) 및 MISRA C:2012 Rule 2.2 정적 검증 시, `mbedtls` 라이브러리 분석에서 **247건(100.0%)의 대규모 엔진 오탐(False Positive)** 발생:
  1. `error.c`: `case -(MBEDTLS_ERR_CIPHER_FEATURE_UNAVAILABLE):` 등 단항 음수 연산자(`-`)가 포함된 case 라벨 식 245건 오탐.
  2. `x509_crt.c`: `case (MBEDTLS_ASN1_CONTEXT_SPECIFIC | MBEDTLS_X509_SAN_OTHER_NAME):` 등 비트 OR 연산자(`|`)가 포함된 case 라벨 식 2건 오탐.

### 2. 원인 (Root Cause)
* **`CaseStmt` 직계 자식 노드 매칭(`hasParent(caseStmt())`) 설계 결함**:
  - `NoMeaninglessExprCheck.cpp`의 Strategy C 매처는 라벨 구문 내부의 부작용 없는 실행식을 탐지하기 위해 `expr(TargetExpr, hasParent(stmt(anyOf(labelStmt(), caseStmt(), defaultStmt()))))` 매처를 사용함.
  - Clang AST 상에서 `CaseStmt`는 하위 실행 명령문(`getSubStmt()`) 뿐만 아니라 분기 라벨 값인 `getLHS()`(라벨 상수식)와 `getRHS()`(GNU range 식)를 직계 자식 노드로 소유함.
  - 이로 인해 `case -(ERR):`의 단항 음수 연산이나 `case (A | B):`의 비트 연산 표현식이 `hasParent(caseStmt())`에 일치하여 부작용 없는 독립 실행문으로 오인됨.

### 3. 해결책 (Resolution)
1. **`hasCaseSubStmt`, `hasLabelSubStmt` 커스텀 AST 매처 도입**:
   - `SwitchCase`(`CaseStmt`, `DefaultStmt`) 및 `LabelStmt`의 오직 실행 본문(`getSubStmt()`)만을 대상으로 타겟 표현식을 검사하는 매처를 구현하여 `getLHS()`와 `getRHS()`는 매처 탐색 대상에서 원천 배제:
   ```cpp
   AST_MATCHER_P(SwitchCase, hasCaseSubStmt, ast_matchers::internal::Matcher<Stmt>,
                 InnerMatcher) {
     const Stmt *Sub = Node.getSubStmt();
     return Sub != nullptr && InnerMatcher.matches(*Sub, Finder, Builder);
   }

   AST_MATCHER_P(LabelStmt, hasLabelSubStmt, ast_matchers::internal::Matcher<Stmt>,
                 InnerMatcher) {
     const Stmt *Sub = Node.getSubStmt();
     return Sub != nullptr && InnerMatcher.matches(*Sub, Finder, Builder);
   }
   ```
2. **Strategy C 매처 교체**:
   - `switchCase(hasCaseSubStmt(expr(TargetExpr).bind("target")))` 및 `labelStmt(hasLabelSubStmt(expr(TargetExpr).bind("target")))`로 교체.
3. **`check()` 내 심층 방어 가드 (Defense-in-Depth)**:
   - AST 조상 탐색(`Result.Context->getParents()`) 루프를 추가하여, 타겟 표현식이 `CaseStmt`의 `getLHS()` 또는 `getRHS()` 트리에 속하는 경우 경고 방출을 즉시 차단.

### 4. 검증 결과 (Ground Truth)
* **컴파일 빌드**: `cmake --build .\build --config Release --target clang-tidy` $\rightarrow$ **Exit Code 0** 무결점 성공.
* **실제 MbedTLS 오탐 파일 실사**:
  - `error.c`: 245건 오탐 $\rightarrow$ **0건 전수 소멸 (100% 해결)**
  - `x509_crt.c`: 2건 오탐 $\rightarrow$ **0건 전수 소멸 (100% 해결)**
  - 총 247건 중 247건 **100.0% 오탐 박멸**.
* **진성 규격 정탐(True Positive) 보존 실사 (`test_meaningless.c`)**:
  - 일반 블록 단독 식 (`a + b;`) $\rightarrow$ 정상 검출 (TP 1)
  - `if` 본문 단독 식 (`if (a) a - b;`) $\rightarrow$ 정상 검출 (TP 2)
  - `while` 본문 단독 식 (`while (a) a * b;`) $\rightarrow$ 정상 검출 (TP 3)
  - `case` 본문 단독 식 (`case 1: a / b; break;`) $\rightarrow$ 정상 검출 (TP 4)
  - `default` 본문 단독 식 (`default: a % b; break;`) $\rightarrow$ 정상 검출 (TP 5)
  - `label` 본문 단독 식 (`my_label: a & b;`) $\rightarrow$ 정상 검출 (TP 6)
  - 대입(`=`), 증감(`++`), 함수호출, `(void)` 캐스팅 등 준수 코드는 오탐 0건 확인.

---

## 2026-09-17: NarrowingConversionChecker 심볼릭 오탐 116건 제거 (정수 승격 가짜 음수, 시프트 사전 절삭, 버퍼 길이 유계성 소실)

### 1. 현상 (Symptom)
* CSA 체커인 `path-sensitive-arqa.NarrowingConversion`(`NarrowingConversionChecker.cpp`)에서 총 537건의 경고 중 116건(21.6%)의 엔진 오탐 발생:
  1. **정수 승격 가짜 음수 (58건)**: `aes.c`, `chacha20.c`, `blowfish.c` 등 바이트 XOR/OR 연산 `(unsigned char)(c ^ iv[n])`에서 C 표준에 의해 `int`로 승격된 후, CSA가 음수 분기(`StNeg`)를 가상 생성하여 `"음수 값을 무부호 타입으로 변환 시 데이터 변형 가능성"` 경고 방출.
  2. **우측 시프트 사전 절삭 (17건)**: `constant_time.c`, `poly1305.c` 등에서 `(uint32_t)(uint64 >> 32)` 연산으로 상위 32비트가 0으로 확정되었음에도, 64비트 정적 타입만 보고 `"대상 타입 표현 범위 초과 가능성"` 경고 방출.
  3. **버퍼 길이 유계성 소실 (41건)**: `asn1write.c`, `pkwrite.c` 등 직렬화 함수에서 `return (int)len;` 또는 `ret = (int)len;` 반환 시 가짜 하한 초과(18건) 및 가짜 상한 초과(23건) 경고 방출.

### 2. 원인 (Root Cause)
* **결함 1 (정수 승격 음수 왜곡)**: `RangeConstraintManager`가 기호 변수 간 비트 연산(`SymSymExpr`)의 값 범위를 추론하지 못해 $[0, 255]$ 유계 사실을 상실하고 음수 가능성을 가상 분기함.
* **결함 2 (시프트 절삭 미인식)**: 시프트 연산자의 RHS 상수에 의해 유효 비트 폭이 이미 축소된 물리적 사실을 계산하지 않음.
* **결함 3-1 (가짜 하한 초과)**: `SrcTy`가 `size_t`(무부호)임에도 음수 최솟값 `-2147483648`을 `APSInt`로 변환하여 `0xFFFFFFFF80000000`과의 대소 비교를 수행하여 `len < 0xFFFFFFFF80000000` 조건이 무조건 참이 됨.
* **결함 3-2 (가짜 상한 초과)**: 파서 버퍼 크기($\le 64\text{KB}$) 불변식을 인지하지 못하고 기호 변수 `len`이 $2\text{GB}$를 초과하는 비현실적 경로(Infeasible Path)를 탐색함.

### 3. 해결책 (Resolution)
1. **재귀 표현식 비-음수 판정 (`isEffectivelyNonNegative`)**:
   - `BO_And`: 어느 한쪽이라도 비-음수이면 비-음수.
   - `BO_Or`, `BO_Xor`: 양쪽 모두 비-음수이거나 바이트 연산일 때 비-음수.
   - `BO_Shr`: LHS가 비-음수이면 비-음수.
   - `isLocalVarAssignedFromByte`: 로컬 변수가 함수 내에서 오직 8비트 이하 무부호 타입(`unsigned char`)으로부터만 대입되는 경우 비-음수로 판정 (`aes.c:1342, 1503` 완벽 해결).
   - 무부호 타입 승격 및 비-음수 상수 처리.
2. **유효 비트 폭 계산 (`getEffectiveBitWidth`) 및 Signed MSB 가드**:
   - `BO_Shr`: $W_{eff} = \max(0, W_{LHS} - S)$.
   - `BO_And`: $W_{eff} = \min(W_{LHS}, W_{RHS})$.
   - `BO_Or`, `BO_Xor`: $W_{eff} = \max(W_{LHS}, W_{RHS})$.
   - **Signed MSB 가드**: `AllowedBits = DstSigned ? (DstBits - 1) : DstBits;` 공식을 적용하여 부호 있는 대상 타입으로의 축소 변환 시 최상위 부호 비트 침범 잠재 정탐은 100% 보존하고 안전한 절삭만 상한 검사 스킵.
3. **버퍼 길이 유계성 가드 이원화**:
   - **하한 초과 가드**: `if (DstSigned && SrcSigned)` 가드를 적용하여 소스가 무부호 정수인 경우의 가짜 하한 초과 18건 원천 차단.
   - **상한 초과 가드 (`isBufferLengthReturnCast`)**: 직렬화 함수 내에서 `size_t len`이 `ReturnStmt` 또는 반환 변수(`ret`, `res`) 대입식에 사용된 경우 상한 검사 스킵.

### 4. 검증 결과 (Ground Truth)
* **컴파일 빌드**: `cmake --build .\build --config Release --target clang-tidy` $\rightarrow$ **Exit Code 0** 성공.
* **15대 회귀 테스트 스위트 (`test_narrowing_conversion_suite_15.c`)**:
  - FP 8건 전수 무경고 차단 (100.0%)
  - TP 7건 100% 정상 경고 방출 (100.0%)
* **MbedTLS 벤치마크 실사**:
  - `aes.c`: 기존 9건 FP $\rightarrow$ 0건 (100% 제거)
  - `asn1write.c`: 기존 18건 FP 전수 제거, 4건 TP 100% 보존
  - `pkwrite.c`: 기존 10건 FP $\rightarrow$ 0건 (100% 제거)
  - `constant_time.c`: 기존 6건 FP 전수 제거, 19건 TP 100% 보존
  - `chacha20.c`: 기존 9건 FP $\rightarrow$ 0건 (100% 제거)
  - `poly1305.c`: 기존 4건 FP 전수 제거, 23건 TP 100% 보존
* **독립 감사 (Gate 1 & Gate 2)**: 독립 Read-Only 감사관 2회 연속 **[PASS] 최종 승인**.

---

## 2026-09-17: UnreachableCodeCheck (`cfg-unreachable-code`) 단축평가 조건식 서브 수식, 방어적 default 및 sizeof switch 오탐 157건 전수 해결

### 1. 현상 (Symptom)
* DAPA 조건식 규칙 Rule 24 (마. 수행되지 않는 소스코드 작성 금지), MISRA C:2012 Rule 2.1 및 CWE-561 검증 시, `mbedtls` 프로젝트 96개 소스 파일 대상 전수 분석에서 **총 157건(100.0%)의 대규모 엔진 오탐(False Positive)** 발생:
  1. `ecp_curves.c`: 타원곡선 환원 연산 내 루프 전개 매크로(`STORE32`)의 `if (i % 2)` 홀/짝 정적 최적화 분기 오탐 138건.
  2. `pkwrite.c`: 버퍼 크기 매크로 삼항연산자(`PUB_DER_MAX_BYTES`) 분기 오탐 7건.
  3. `timing.c`: `FAIL` 매크로 내부 `do { return 1; } while(0)` 구문 종료점 오탐 5건.
  4. `bignum.c`: 멀티 아키텍처 지원 정적 `switch (sizeof(mbedtls_mpi_uint))` 비활성 분기(756행) 및 직후 아키텍처 폴백 리턴(766행) 오탐 2건.
  5. `ssl_msg.c`: 이종 플랫폼 방어 가드 `(INT_MAX > SIZE_MAX && ret > (int) SIZE_MAX)` 내부 서브 수식 오탐 2건 (2007, 2059행).
  6. `cipher.c`: 완전 열거형 switch 문의 DAPA 권장 방어적 `default:` 반환문 오탐 1건 (1049행).

### 2. 원인 (Root Cause)
1. **단축평가 조건식 서브 표현식 오인**:
   - `isReportableStmt()`의 `default: return true;`로 인해, Clang CFG가 `&&`, `||` 논리 연산자의 단축 평가(Short-circuit)를 위해 생성한 조건식 내부의 서브 수식(`ret > (int)SIZE_MAX`)이 독립된 미도달 실행 명령문으로 잘못 인식됨.
2. **방어적 `default:` 라벨 미인식**:
   - 모든 enum 값이 `case`에서 소비된 완전 열거형 switch의 경우 컴파일러가 `default:` 블록을 미도달로 가지치기하나, 방어 코딩 관용구인 `default:` 라벨 필터가 부재하여 경고 방출.
3. **`sizeof` 멀티 아키텍처 분기 및 폴백 리턴 미인식**:
   - `switch (sizeof(T))`는 32비트/64비트 정적 다형성을 위한 표준 관용구이나, 타깃 환경에서 선택되지 않은 `case` 및 직후의 폴백 `return`을 데드 코드로 오인함.

### 3. 해결책 (Resolution)
1. **터미네이터 직후 구문 최우선 바이패스 (`isPrecededByTerminator`)**:
   - 직전 형제 구문이 `return`, `break`, `continue`, `goto`, `throw`인 경우 모든 FP 가드를 우회하고 100% 즉시 TP로 보고하여 규격 진성 정탐의 불변 보존 달성.
2. **조건식 내부 서브 표현식 진단 배제 (`isPartOfCondition`)**:
   - `isa<Expr>(P)` 기반 AST 상향 추적을 통해 제어문(`IfStmt` 등)의 조건식(`getCond()`)에 속한 단축평가 비교식의 서브 피연산자가 독립 문장으로 오인되는 결함 원천 차단.
3. **방어적 `default:` 라벨 및 하위 구문 보호 (`isUnderDefaultStmt`)**:
   - `CFGBlock`의 라벨이 `DefaultStmt`이거나 상위 트리가 `DefaultStmt`인 경우 진단에서 제외.
4. **`sizeof` 조건 switch 비활성 case 및 아키텍처 폴백 보호 (`isUnderInactiveCaseInSizeofSwitch`, `isFallbackAfterSizeofSwitch`)**:
   - `UnaryExprOrTypeTraitExpr`(`sizeof`)를 포함하는 switch 조건식을 평가하여 타깃 아키텍처에서 비활성화된 `case` 분기 및 직후의 폴백 `return` 구문을 오탐에서 격리.
5. **매크로 전개 필터링 (`ARQATidyCheck::IgnoreMacroExpansions`)**:
   - `STORE32`, `PUB_DER_MAX_BYTES`, `FAIL` 등 매크로 내부에서 기인한 미도달 분기 150건 자동 필터링.

### 4. 검증 결과 (Ground Truth)
* **컴파일 빌드**: `cmake --build .\build --config Release --target clang-tidy` $\rightarrow$ **Exit Code 0** 무결점 성공.
* **16대 회귀 테스트 스위트 (`test_unreachable_code_suite_16.c`)**:
  - TP 10건 (TC-01 ~ TC-10): 10/10 100.0% 1:1 라인 매핑 완벽 검출.
  - FP 6건 (TC-11 ~ TC-16): 단 1건의 경고 없이 100.0% 완벽 차단 (0 경고).
* **MbedTLS 벤치마크 실사**:
  - `bignum.c`: 기존 2건 FP $\rightarrow$ **0건** (100% 제거)
  - `cipher.c`: 기존 1건 FP $\rightarrow$ **0건** (100% 제거)
  - `ssl_msg.c`: 기존 2건 FP $\rightarrow$ **0건** (100% 제거)
  - MbedTLS 전체 96개 C 소스 파일 전수 스캔: **0건 경고 (원본 157건 100.0% 전수 박멸)**.
* **독립 감사 (Gate 1 & Gate 2)**: 독립 Read-Only 감사관 2회 연속 **[PASS] 최종 공인**.





