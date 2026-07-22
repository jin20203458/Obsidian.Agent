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

## 2026-07-22: AST Drop 구문에 대한 로케일 독립적 Clang Diagnostic ID 가로채기 (i18n 지원)

### 1. 현상 (Symptom)
* catch-all 위치 오류, virtual 키워드 누락 순수가상함수, virtual base 캐스팅 등 Clang 파서 레벨에서 AST 노드가 100% Drop되는 구문의 경우 AST 매칭이 불가능함.
* 이를 해결하기 위해 CLI 텍스트 파싱 기반으로 컴파일 에러를 가로채는 방식은 다국어 환경(한국어, 영어, 일본어 등)이나 컴파일러 메시지 변경 시 국제화(i18n) 실패 및 파싱 오탐지 위험 존재.

### 2. 원인 (Root Cause)
* Clang 파서가 문법 규격 위반을 만나면 해당 표현식/문장 노드 전체를 AST 트리에 포함하지 않고 삭제함.
* CLI 텍스트 정규식 가로채기는 OS/컴파일러의 시스템 로케일(System Locale) 설정에 종속됨.

### 3. 해결책 (Resolution)
* AST 노드가 100% Drop되어 AST 매칭이 불가능한 C++ 문법 위반(Rule 56, 62, 63, 64)의 경우, 텍스트 매칭 대신 `ClangTidyDiagnosticConsumer::HandleDiagnostic` 내에서 Clang 정수 진단 ID(`Info.getID()`)인 `diag::err_early_catch_all`, `diag::err_member_function_initialization`, `diag::err_non_virtual_pure`, `diag::err_static_downcast_via_virtual`를 직접 매핑함.
* 텍스트 파싱이 아닌 정수 ID 매핑이므로 **언어/로케일에 100% 독립적인 정밀 가로채기** 체계를 구현함.


