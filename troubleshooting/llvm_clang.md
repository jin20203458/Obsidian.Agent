# LLVM/Clang: 컴파일러 및 정적 분석기 트러블슈팅

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
* 템플릿 기반 C++ 코드(예: `OpenKAI-master` 프로젝트의 `_GeoFence.cpp` 및 `HttpClient.cpp`) 정적분석 진행 중, `clang-tidy` 프로세스가 아래와 같은 내부 Assertion 혹은 Unreachable 코드로 인해 비정상 종료(Crash)되는 문제 발생:
  1. `ast-representable-cast` 규칙 검사 중:
     `Assertion failed: !isValueDependent() && "Expression evaluator can't be called on a dependent expression."`
  2. `ast-switch-style` 규칙 검사 중:
     `Unknown builtin type! UNREACHABLE executed at ASTContext.cpp:2005!`

### 2. 원인 (Root Cause)
* **`ast-representable-cast`**: `RepresentableCastCheck.cpp`에서 변환 대상인 소스 표현식을 평가하기 위해 `EvaluateAsInt` 혹은 `EvaluateAsFloat`를 호출할 때, 해당 식에 템플릿 인자 등 컴파일 타임에 크기나 값이 결정되지 않은 템플릿 종속적 표현식(Value-dependent / Type-dependent expression)이 전달되어 AST 상수 평가기 내부에서 오류를 냄.
* **`ast-switch-style`**: `SwitchStyleCheck.cpp`에서 `switch` 조건문의 형식을 검증하기 위해 `type->getAs<BuiltinType>()`를 통해 내장 타입 판정 시, `Context->getTypeSize(type)`를 switch 조건식 가인식 영역 바로 앞에서 호출함. 이때 미확정 빌트인 타입(오버로드, 플레이스홀더, 종속형 템플릿 타입 등)이 유입되면 Clang AST 엔진 내부 크기 조회기에서 `UNREACHABLE`을 발생시켜 컴파일러가 크래시됨.

### 3. 해결책 (Resolution)
* **`ast-representable-cast`**: 상수 평가기를 호출하기 전, 해당 식의 종속 관계 여부를 체크하는 방어 조건문(`!SrcExpr->isValueDependent() && !SrcExpr->isTypeDependent()`)을 추가하여 템플릿 종속 식은 평가를 우회하게 조치함.
* **`ast-switch-style`**: `getTypeSize` 호출 위치를 `switch (BT->getKind())` 문 내부로 안전하게 이동시킴. 크기가 확실히 존재하는 실존 내장 정수 타입(`char`, `int`, `long` 등) 및 `bool` 케이스 분기 내에서만 크기를 구하게 하고, 크기가 없는 Dependent나 Placeholder 같은 미완성 타입은 크기 연산 없이 `default: break`로 건너뛰어 탈출하도록 구조 개선.
