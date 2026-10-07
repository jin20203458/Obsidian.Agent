---
description: LLVM/Clang 커스텀 Tidy 체커 및 Static Analyzer 개발 트러블슈팅 런북.
related:
  - ../README.md
  - ../LLVM/README.md
  - ../LLVM/docs/01_static_analyzer_architecture.md
---
# LLVM/Clang Troubleshooting

본 문서는 LLVM/Clang 커스텀 Tidy 체커 및 Static Analyzer 개발 중 발생하는 버그와 오류 해결 방법을 기록하는 문서입니다.

---

**테스트 케이스 및 코드 품질 원칙**

* **인위적 편향 배제**: 특정 입력이나 이상적인 시나리오에만 통과하도록 테스트를 끼워 맞추지 않는다. 경계값, 비정상 입력, 극단적 예외 상황을 포함해 검증한다.
* **지속적 리팩터링 및 확장성 보장**: 결함 발견 시 임시 패치에 그치지 않고, 구조적 리팩터링을 통해 언제든 코드를 고도화할 수 있는 유연한 아키텍처를 유지한다.

---

## 2026-07-07: [Resolved] checkBranchCondition 콜백 내의 오탐지 (동일 조건식에 대한 참/거짓 경고 동시 발생)

### 1. 현상 (Symptom)
* 정상 조건문 `if (x == 5)`에 대해 "항상 참(True)"과 "항상 거짓(False)" 경고가 동일 위치에서 동시 검출되는 오탐 발생.

### 2. 원인 (Root Cause)
* Clang Static Analyzer는 조건문을 만나면 분석 상태(`State`)를 참/거짓 경로로 선행 분할함.
* `checkBranchCondition` 콜백의 `C.getState()`는 이미 해당 경로에 맞춰 1비트 상수로 고정된 상태(`1 U1b` 또는 `0 U1b`)이므로, 이를 그대로 `assume`하면 항상 단일 판정으로 평가됨.

### 3. 해결책 (Resolution)
* `C.getState()` 대신 조상 노드(`ExplodedNode *N = C.getPredecessor()`)를 역추적하여 1비트 상수가 아닌 최초의 Symbolic `SVal`(`CondVal`)과 해당 시점의 `AncestorState`를 추출한 후 `assume(CondVal)`을 수행.

```cpp
const ExplodedNode *N = C.getPredecessor();
while (N) {
  ProgramStateRef AncestorState = N->getState();
  SVal V = AncestorState->getSVal(Condition, C.getLocationContext());
  if (!V.isUnknownOrUndef() && !V.getAs<nonloc::ConcreteInt>()) {
    EvalState = AncestorState;
    CondVal = V;
    break;
  }
  N = N->getFirstPred();
}
std::tie(StateTrue, StateFalse) = EvalState->assume(CondVal);
```

---

## 2026-07-14: [Resolved] 커스텀 Tidy 체커 내 AST 상수 값 평가 중 크래시 (Expression evaluator can't be called on a dependent expression 및 Unknown builtin type)

### 1. 현상 (Symptom)
* 템플릿 C++ 코드 분석 중 내부 Assertion/Unreachable 크래시 발생:
  1. `Assertion failed: !isValueDependent() && "Expression evaluator can't be called on a dependent expression."`
  2. `Unknown builtin type! UNREACHABLE executed at ASTContext.cpp:2005!`

### 2. 원인 (Root Cause)
* `EvaluateAsInt` 등을 호출할 때 템플릿 종속식(Value/Type-dependent) 또는 헤더 누락으로 인한 복구 노드(`RecoveryExpr`)가 유입되어 상수 평가기 내부에서 오류 발생.
* `SwitchStyleCheck`에서 미확정 빌트인 타입(오버로드, 플레이스홀더 등)에 대해 `Context->getTypeSize(type)`를 호출하여 AST 크기 조회기 `UNREACHABLE` 유발.

### 3. 해결책 (Resolution)
* 상수 평가기 호출 전 종속 관계 방어 조건(`!Expr->isValueDependent() && !Expr->isTypeDependent()`) 추가.
* `getTypeSize` 호출을 `switch (BT->getKind())` 내부로 이동하여 실존 정수 타입 및 `bool` 분기 내에서만 크기를 구하고, 미완성 타입은 `default: break`로 안전 탈출.

---

## 2026-07-14: [Resolved] SingleExitAndReturnTypeCheck 내 템플릿 종속 타입(Dependent Type) 오탐지

### 1. 현상 (Symptom)
* 템플릿 C++ 코드 분석 시 리턴문이 존재함에도 `"함수 선언 반환형(_Bool)과 반환값 타입(<dependent type>)이 일치하지 않습니다"` 오탐 발생.

### 2. 원인 (Root Cause)
* Clang AST는 템플릿 인스턴스화 전까지 반환식 타입을 `<dependent type>`으로 간주함. 이를 `Ctx.hasSameType()`에 그대로 대조하여 불일치로 오판.

### 3. 해결책 (Resolution)
* `ReturnVisitor::VisitReturnStmt` 시작점에 템플릿 종속 타입 검출 조건(`isDependentType()`)을 추가하여, 반환형 또는 리턴식 타입 중 하나라도 종속 타입인 경우 매칭 성공(`TypesMatch = true`)으로 우회 처리.

---

## 2026-07-14: [Resolved] FunctionCallArgumentConsistencyCheck 내 참조형(&) 및 종속 타입(Dependent Type) 인자 오탐지

### 1. 현상 (Symptom)
* C++ 참조형 매개변수(`T &`) 함수 호출 시 `"1번째 인자의 타입이 프로토타입과 일치하지 않습니다. 기대: 'T &', 실제: 'T'"` 오탐 및 종속 타입 인자 오탐 발생.

### 2. 원인 (Root Cause)
* 호출 인자 표현식의 Clang AST 타입은 항상 넌-레퍼런스(`T`)로 평가되므로 `hasSameType()` 엄격 대조 시 불일치 발생.
* 템플릿 미확정 타입(`<dependent type>`) 우회 가드 부재.

### 3. 해결책 (Resolution)
* `EnforceExactParamType`에서 레퍼런스(`getNonReferenceType()`) 및 CVR 한정자(`getUnqualifiedType()`)를 벗긴 후 핵심 타입만 대조.
* 인자 루프 내 `ParamTy->isDependentType() || Arg->getType()->isDependentType()` 가드 추가.

---

## 2026-07-22: [Resolved] Clang RecoveryExpr 기반 AST 매칭 및 C/C++ 컴파일 플래그 다운그레이드

### 1. 현상 (Symptom)
* 미선언 함수 호출, 인자 개수 불일치 등 컴파일러 하드 에러 시 `clang-tidy` AST 체커들이 수집하지 못하고 미탐(0%) 발생.

### 2. 원인 (Root Cause)
* Clang 파서가 하드 에러 발생 시 정식 `callExpr` 대신 `RecoveryExpr` 노드로 포장하거나 AST 생성을 차단함.

### 3. 해결책 (Resolution)
1. **컴파일 플래그 다운그레이드**:
   * `-Wno-error=return-type`, `-Wno-error=implicit-function-declaration` 플래그로 경고 다운그레이드하여 정상 AST 생성 유도.
2. **`RecoveryExpr` AST 매처 보강**:
   * `isa<RecoveryExpr>` 및 첫 자식 노드가 `FunctionDecl` 참조인지 검증하는 매처 추가. 루프 내 `Arg->containsErrors()` 가드로 손상된 표현식 오탐 차단.

---

## 2026-07-22: [Resolved] AST Drop 구문에 대한 로케일 독립적 Clang Diagnostic ID 가로채기 및 한글 메시지 재정의

### 1. 현상 (Symptom)
* catch-all 위치 오류, virtual 키워드 누락 순수가상함수 등 파서 레벨에서 AST 노드가 100% Drop되는 구문은 AST 매치가 불가능하며, CLI 정규식 파싱은 다국어 환경에서 파손됨.

### 2. 원인 (Root Cause)
* Clang Sema가 문법 위반 시 노드를 트리에 포함하지 않고 삭제함. `DiagnosticSemaKinds.inc`는 TableGen 빌드 아티팩트이므로 직접 include 불가.

### 3. 해결책 (Resolution)
1. **정수 Diagnostic ID 기반 가로채기 (`DiagnosticSema.h`)**:
   * `#include "clang/Basic/DiagnosticSema.h"`를 사용하여 `ClangTidyDiagnosticConsumer::HandleDiagnostic` 내에서 정수 ID(`Info.getID()`)를 수집:
     * `diag::err_early_catch_all` $\rightarrow$ `"ast-unused-exception-handler"`
     * `diag::err_member_function_initialization` $\rightarrow$ `"ast-pure-virtual-init"`
     * `diag::err_non_virtual_pure` $\rightarrow$ `"ast-virtual-pure"`
     * `diag::err_static_downcast_via_virtual` $\rightarrow$ `"ast-virtual-base-cast"`
2. **한글 메시지 재정의**:
   * 최상위 스코프에 UTF-8 한글 문자열 리터럴(`u8"..."`)로 메시지를 정의하여 `ClangTidyDiagnosticRenderer`로 전달.

---

## 2026-09-01: [Resolved] NoAutoTypeCheck (`ast-no-auto-type`) 컴파일러 암시적 변수 오탐 및 복합 auto 타입 미탐 해결

### 1. 현상 (Symptom)
* C++11 범위 기반 `for` 루프에서 라인당 3건의 허위 오탐 발생 및 `auto*`, `const auto&` 등 복합 포인터/참조 `auto` 선언 미탐.

### 2. 원인 (Root Cause)
* Clang AST는 `for-range` 루프 시 `auto &&__range`, `auto __begin`, `auto __end` 컴파일러 암시적 `VarDecl`(`isImplicit() == true`)을 자동 생성함.
* `const auto&`는 `LValueReferenceType(AutoType)`으로 래핑되어 단순 `hasType(autoType())`으로 매칭되지 않음.

### 3. 해결책 (Resolution)
1. `isLanguageVersionSupported`: `LangOpts.CPlusPlus11` 가드 적용 (C++11 이상 한정).
2. `AutoTypeMatcher`: `qualType(anyOf(autoType(), pointsTo(qualType(autoType())), references(qualType(autoType()))))`로 재구성하고 `unless(isImplicit())` 가드 추가.

---

## 2026-09-04: [Resolved] PartialCopyAssignmentCheck (`ast-partial-copy-assignment`) C++ 클래스/구조체 멤버 대입 오탐지 해결

### 1. 현상 (Symptom)
* `operator=`에서 구조체 멤버(`Coordinate pos;`)를 정상 복사(`pos = rhs.pos;`)했음에도 "복사 대입 연산자에서 대입되지 않았습니다" 오탐 발생.

### 2. 원인 (Root Cause)
* 구조체/클래스 타입 객체의 `=` 대입은 `BinaryOperator(BO_Assign)`가 아닌 오버로딩된 멤버 함수 호출인 `CXXOperatorCallExpr` 노드로 생성됨.

### 3. 해결책 (Resolution)
1. `extractRootFieldDecl`: `LHS` 중첩 멤버 접근(`this->pos.lat`)에서 `MemberExpr::getBase()` 체인을 역추적하여 최상위 `FieldDecl` 추출.
2. `VisitCXXOperatorCallExpr`: `OCE->getOperator() == OO_Equal`인 경우 첫 번째 인자에서 루트 `FieldDecl`을 추출하여 `AssignedFields`에 등록.

---

## 2026-09-07: [Resolved] ThreadLockChecker (`path-sensitive-arqa.ThreadLock`) 함수 조기 반환 락 누수 미탐 및 NewDeleteLeaks 연동 해결

### 1. 현상 (Symptom)
* `pthread_mutex_lock` 또는 `EnterCriticalSection` 획득 후 조기 에러 반환(`return -1;`) 시 `unlock` 미호출 교착 결함에 대해 0건 미탐(Silent Failure) 발생.

### 2. 원인 (Root Cause)
* 전역/포인터 동기화 객체는 함수 종료 후에도 유효하여 `checkDeadSymbols`로는 함수 종료 시점 락 누수를 감지 불가.
* `check::EndFunction` 훅 부재 및 락 획득 인라인 래퍼 함수에 대한 가드 부재.

### 3. 해결책 (Resolution)
1. `REGISTER_MAP_WITH_PROGRAMSTATE(LockMap, const MemRegion *, LockEntry)` 도입.
2. `check::EndFunction` 구현: `if (!C.inTopFrame()) return;` 가드로 인라인 래퍼 조기 오탐을 차단하고, 최상위 프레임 종료 시점 잔여 `Locked` 객체 색출.
3. `generateNonFatalErrorNode`로 에러 노드 체이닝 보장.

---

## 2026-09-16: [Resolved] MultiStatementPerLineCheck (`ast-multi-statement-per-line`) 매크로 전개 누수, typedef struct 및 파일 간 라인 충돌 오탐 474건 전수 해결

### 1. 현상 (Symptom)
* 단일 라인 매크로 호출(414건), `typedef struct { ... } name_t;` 선언(59건), 헤더-메인 소스 간 라인 번호 충돌로 대규모 오탐 발생.

### 2. 원인 (Root Cause)
* 매크로 인자 확장 위치(`SM.getExpansionLoc`)와 본문 시작 위치의 좌표 불일치로 중복 필터 우회.
* Clang AST는 `typedef struct`에 대해 `RecordDecl`과 `TypedefDecl` 2개 노드를 생성.
* 맵 키가 `unsigned line` 단일 값이라 파일 간 동일 라인 번호 충돌 발생.

### 3. 해결책 (Resolution)
1. `getTopMacroInvocationLoc`: `SM.getTopMacroCallerLoc(Loc)` 기반 호출부 단일 좌표로 정규화하여 후속 문장 필터링.
2. `checkCompoundStmt` 시작부에 `if (CS->getBeginLoc().isMacroID()) return;` 가드 추가.
3. `TagDecl::getTypedefNameForAnonDecl()` 매칭 노드는 독립 선언 카운트에서 제외.
4. 맵 키를 `std::pair<FileID, unsigned>`로 교체하여 파일 간 라인 충돌 차단.

---

## 2026-09-16: [Resolved] NoMeaninglessExprCheck (`ast-no-meaningless-expr`) switch-case 라벨 상수 평가식 매칭 결함 오탐 247건 전수 해결

### 1. 현상 (Symptom)
* `case -(MBEDTLS_ERR_...):` 등 단항 음수 연산이나 비트 연산이 포함된 case 라벨 식 247건이 부작용 없는 의미 없는 구문으로 오탐.

### 2. 원인 (Root Cause)
* Clang AST 상에서 `CaseStmt`는 하위 실행 명령문(`getSubStmt()`)뿐만 아니라 라벨 상수식인 `getLHS()`와 `getRHS()`를 직계 자식 노드로 소유함. `hasParent(caseStmt())` 매처가 라벨 상수식을 실행문으로 오인.

### 3. 해결책 (Resolution)
1. `hasCaseSubStmt`, `hasLabelSubStmt` 커스텀 매처 도입: `Node.getSubStmt()`만을 대상으로 타겟 표현식을 검사하여 `getLHS()`와 `getRHS()`를 원천 배제.
2. AST 조상 탐색 루프(`Result.Context->getParents()`)를 추가하여 타겟 식이 `getLHS()` 또는 `getRHS()`에 속하면 경고 즉시 차단.

---

## 2026-09-17: [Resolved] NarrowingConversionChecker 심볼릭 오탐 116건 제거 (정수 승격 가짜 음수, 시프트 사전 절삭, 버퍼 길이 유계성 소실)

### 1. 현상 (Symptom)
* 바이트 XOR/OR 연산 후 무부호 변환 시 가짜 음수 경고(58건), 우측 시프트 절삭 미인식(17건), 직렬화 반환식 버퍼 길이 범위 초과 오탐(41건) 발생.

### 2. 원인 (Root Cause)
* `RangeConstraintManager`가 기호 변수 간 비트 연산(`SymSymExpr`)의 $[0, 255]$ 범위를 추론하지 못해 음수 분기를 가상 생성.
* 시프트 RHS 상수에 의한 유효 비트 폭 축소 미인식.
* 무부호 `SrcTy`에 대해 음수 최솟값과의 대소 비교 수행 및 파서 버퍼 크기 불변식 미인식.

### 3. 해결책 (Resolution)
1. `isEffectivelyNonNegative`: `BO_And`, `BO_Or`, `BO_Xor`, `BO_Shr` 및 8비트 이하 무부호 로컬 변수 대입식에 대한 비-음수성 판정.
2. `getEffectiveBitWidth` 및 Signed MSB 가드: `AllowedBits = DstSigned ? (DstBits - 1) : DstBits;`를 적용하여 안전한 절삭은 상한 검사 스킵.
3. 버퍼 길이 유계성 가드: `if (DstSigned && SrcSigned)` 하한 가드 및 직렬화 반환문 `size_t len` 상한 검사 스킵(`isBufferLengthReturnCast`).

---

## 2026-09-17: [Resolved] UnreachableCodeCheck (`cfg-unreachable-code`) 단축평가 조건식 서브 수식, 방어적 default 및 sizeof switch 오탐 157건 전수 해결

### 1. 현상 (Symptom)
* 루프 전개 매크로의 정적 최적화 분기(138건), `switch (sizeof(T))` 비활성 아키텍처 분기(2건), 완전 열거형 switch의 방어적 `default:`(1건) 등 157건 오탐 발생.

### 2. 원인 (Root Cause)
* Clang CFG가 논리 연산자(`&&`, `||`) 단축평가를 위해 생성한 조건식 내부 서브 수식을 독립 미도달 실행문으로 오인.
* 완전 열거형 switch에서 컴파일러가 가지치기한 `default:` 블록 및 `sizeof` 멀티 아키텍처 비활성 분기 미인식.

### 3. 해결책 (Resolution)
1. `isPrecededByTerminator`: 직전 형제 구문이 `return`, `break`, `continue`, `goto`, `throw`인 경우 즉시 진성 정탐(TP)으로 보고.
2. `isPartOfCondition`: AST 상향 추적으로 제어문 조건식 내부 단축평가 서브 표현식의 진단 배제.
3. `isUnderDefaultStmt`: `CFGBlock` 라벨이 `DefaultStmt`인 경우 진단 제외.
4. `isUnderInactiveCaseInSizeofSwitch`, `isFallbackAfterSizeofSwitch`: `sizeof` 조건 switch의 비활성 case 및 직후 폴백 return 보호.
5. 매크로 전개 필터링(`IgnoreMacroExpansions`) 적용.

---

## 2026-09-17: [Resolved] PointerCvQualifierDropCheck (`ast-pointer-cv-qualifier-drop`) 포인터 비교문 내 암묵적 형변환 오탐 68건 전수 해결

### 1. 현상 (Symptom)
* 포인터 버퍼 경계 비교문(`if (*p != end)`, `if (*p < end)`)에서 const 탈락 오탐 68건 발생.

### 2. 원인 (Root Cause)
* C99 규격에 따라 피연산자 중 한쪽만 `const`인 포인터 비교 연산 시 Clang 컴파일러가 자동 생성하는 `ImplicitCastExpr <BitCast>`를 한정자 탈락 위반으로 오진단.

### 3. 해결책 (Resolution)
1. `isInComparisonContext`: AST 상향 순회를 통해 상위 노드가 비교 연산자(`BO->isComparisonOp()`)인지 판별.
2. 진단 진입부 가드: `if (isa<ImplicitCastExpr>(CE) && isInComparisonContext(CE, *R.Context)) return;` 적용.
3. 명시적 `CStyleCastExpr` 및 비-비교 컨텍스트의 암묵적 캐스트는 100% 정상 정탐 보존.

---

## 2026-09-17: [Resolved] NoOutOfRangeAssignmentCheck (`ast-no-out-of-range-assignment`) 무부호 정수 리터럴 음수 오인 및 이항 연산 부호 오염 오탐 25건 전수 해결

### 1. 현상 (Symptom)
* 32/64비트 무부호 16진수 리터럴(`0xBB67AE85`) 및 상수식(`delta * 32`)에 대해 음수 초과값 대입 허위 경고 25건 발생.

### 2. 원인 (Root Cause)
* `IntegerLiteral` 평가 시 `llvm::APSInt(IL->getValue(), false)`로 `isUnsigned = false`가 하드코딩되어 MSB=1인 리터럴이 음수로 왜곡됨.
* 이항 연산자 평가 시 `L.isSigned() || R.isSigned()`로 인해 피연산자 하나만 signed여도 전체 식을 signed로 강제 변환.

### 3. 해결책 (Resolution)
1. AST 무부호 정수 타입 반영: `llvm::APSInt(IL->getValue(), IL->getType()->isUnsignedIntegerType())` 적용.
2. 이항 연산자 부호성 준수: `bool Signed = !BO->getType()->isUnsignedIntegerType();` 적용.
3. 단항 마이너스 부정 시 `std::max(64U, Sub.getBitWidth() + 1)` 상향 확장으로 2의 보수 오버플로우 방어.
4. 렉서 폴백 64비트 정밀도 확장 적용.

---

## 2026-09-17: [Resolved] cfg-null-dereference-guard 순수 방어 가드 래티스 개량 및 CSA NullDereference 중복 경고 차단

### 1. 현상 (Symptom)
* `if (p == NULL) { *p; }` 및 상위 함수에서 NULL을 넘긴 인라인 호출(`helper(NULL)`)에서 `cfg-` 체커와 CSA 체커가 동일 라인에 중복 경고를 방출하는 문제.

### 2. 원인 (Root Cause)
* 2-State Boolean 방식(`Safe`/`Unsafe`)으로 인해 명시적 NULL 분기 내부를 가드 누락으로 처리.
* 단일 함수 관점(CFG)과 함수 간 심볼릭 분석(CSA)의 관점 차이.

### 3. 해결책 (Resolution)
1. **3-상태 가드 래티스 도입 (`NullDereferenceGuardCheck.cpp`)**:
   * `enum class GuardState { Unchecked = 0, GuardedNonNull = 1, GuardedNull = 2 };`
   * 진입 시점은 `Unchecked`로 시작하고, 오직 역참조 시점 상태가 `Unchecked`인 경우에만 가드 누락 경고 방출.
   * `GuardedNull` 상태(명시적 NULL 분기 내부)는 CSA `NullDereference`에 100% 위임하여 침묵.
2. **ArqaStatic 파서 레벨 디듀플리케이션**:
   * `MainViewModel.FinalizeAnalysisAsync`에서 동일 라인 중복 수집 시 실행 경로 노트를 보유한 CSA 경고를 단일 유지하고 `cfg-` 경고 제거.

---

## 2026-09-17: [Resolved] path-sensitive-core.StackAddressEscape 대입 위치 ExplodedGraph 역추적 고도화 및 DAPA Rule 34 진단 위치 정밀화

### 1. 현상 (Symptom)
* 전역 변수에 로컬 주소를 대입한 코드(`pi = &a;`) 분석 시, 실제 대입 라인이 아닌 함수의 닫는 중괄호(`}`) 위치에 경고가 발생하는 문제.

### 2. 원인 (Root Cause)
* `StackAddrEscapeChecker.cpp`의 에러 노드가 `checkEndFunction` 시점의 `FunctionExitPoint` 프로그램 포인트이므로, 위치 계산기가 함수의 맨 마지막 닫는 중괄호 위치를 메인 진단 위치로 설정함.

### 3. 해결책 (Resolution)
* `ErrorNode`로부터 조상 노드(`Curr->getFirstPred()`)를 역추적하여 실제 대입이 일어난 `PostStmt<BinaryOperator>` 노드를 탐색:
  * RHS: 기저 리전 또는 AST `DeclRefExpr`이 탈출된 스택 변수와 일치하는지 확인.
  * LHS: 기저 리전 또는 AST 멤버/배열 기저 변수가 전역/정적 수신체와 일치하는지 확인.
* 매칭된 조상 대입 노드를 `ReportNode`로 설정하여 대입 연산자(`=`)의 정확한 라인/컬럼을 지목하도록 개선 (실패 시 종료 노드로 자동 폴백).

---

## 2026-09-17: [Resolved] cfg-nonzero-divisor-guard 3-상태 래티스 개량 및 CSA DivideZero 중복 경고 차단 (DAPA Rule 39)

### 1. 현상 (Symptom)
* 가드 없는 나눗셈(`return x / n;`)에 대해 정적분석 경고 미발생 및 명시적 0 분기 내부에서 `cfg-`와 CSA 간 중복 경고 충돌 위험.

### 2. 원인 (Root Cause)
* CSA `DivideZero`는 0이 되는 실행 경로를 추적할 뿐 unconstrained 매개변수에 대한 가드 누락을 탐지하지 못함.
* `if (n == 0)` 분기 내부에서 두 체커가 동시 경고를 낼 위험 상존.

### 3. 해결책 (Resolution)
1. **3-상태 가드 래티스 도입 (`NonZeroDivisorGuardCheck.cpp`)**:
   * `enum class GuardState { Unchecked = 0, GuardedNonZero = 1, GuardedZero = 2 };`
   * `Unchecked` 상태에서만 Rule 39 경고 방출. `GuardedZero` 분기는 CSA `DivideZero`에 전담 위임.
2. **특수 가드 전담화 (`ParmVarDecl` 한정)**:
   * `if (!isa<ParmVarDecl>(VD)) return;` 가드를 적용하여 매개변수의 사전 0 가드 부재 검사만 전담하고, 로컬/전역 변수는 CSA에 100% 위임.
3. **ArqaStatic 파이프라인 디듀플리케이션 통합**:
   * `MainViewModel.cs`에 라인 단위 자동 디듀플리케이션(`RemoveRange`) 안전망 구축.

---

## 2026-09-18: [Resolved] path-sensitive-arqa.UninitializedAddressToConstParam MbedTLS 12건 전수 오탐 제거 및 GDM 경로 민감 상태 추적 구축

### 1. 현상 (Symptom)
* MbedTLS 96개 파일 분석 시, 선행 비-const 포인터로 정상 초기화된 버퍼를 미초기화로 오인하여 12건 전수 오탐(FP 100%) 발생.

### 2. 원인 (Root Cause)
* `check::Bind`, `check::PostCall`, `check::PointerEscape`가 구현되지 않아 메모리 쓰기 및 탈출 이력을 경로별로 추적하지 못함.
* 비-역참조 포인터 비교 헬퍼 함수 및 공용체(Union) 특성 미고려.

### 3. 해결책 (Resolution)
1. `REGISTER_SET_WITH_PROGRAMSTATE(InitializedOrEscapedVars, const MemRegion *)` 도입.
2. `check::Bind` / `check::PostCall` / `check::PointerEscape` 구현하여 쓰기 및 비-const 인자 전달 시 GDM에 등록.
3. 5단계 정밀 진입 가드 구축:
   * Guard 1: `hasLocalStorage()` 지역 자동 변수만 검사.
   * Guard 2: 선언 시 명시적 초기화(`getInit()`) 존재 시 스킵.
   * Guard 3: 선행 변경/탈출 이력(`InitializedOrEscapedVars`) 존재 시 스킵.
   * Guard 4: 인접 0길이 인자(`len == 0`) 스킵.
   * Guard 5: 피호출 함수 정의 내 역참조 부재(`ParamDereferenceVisitor`) 시 스킵.
4. `RecordDecl::isUnion()`의 경우 최소 1개 필드가 유효하면 미초기화 판정 제외.

---

## 2026-09-18: [Resolved] StackAddressEscape 체커 내 호출자 출력 매개변수/힙/this 미탐 및 단언문 크래시 해결

### 1. 현상 (Symptom)
* 호출자 이중 포인터(`*out = &local`), 구조체 멤버(`ctx->ptr = &local`), 힙 객체, C++ `this` 멤버로의 스택 주소 유출 미탐(FN 100%) 및 단언문 크래시 발생.

### 2. 원인 (Root Cause)
* Upstream `HandleBinding`이 `GlobalsSpaceRegion`만 검사하고 `UnknownSpaceRegion`(`SymbolicRegion`) 및 `HeapSpaceRegion`을 누락.
* 호출자 매개변수는 함수 종료 전 `SymbolReaper`에 의해 데드 바인딩으로 조기 소멸됨.
* `assert(isa<StackSpaceRegion>(Space))`로 인한 크래시.

### 3. 해결책 (Resolution)
1. **`check::Bind`와 `ProgramState` GDM 연동 (`EscapedStackMap`)**:
   * 대입 즉시 탈출 대상(`isEscapingStorage`) 여부를 판별하여 GDM에 실시간 기록. 데드 심볼 수거 영향 없이 보존. 비-스택 값 덮어쓰기 시 즉시 맵에서 제거.
2. **수명주기 경계 판별**:
   * `GlobalsSpaceRegion`, `HeapSpaceRegion`, 기저 영역이 `SymbolicRegion` 또는 `CXXThisRegion`인 경우 탈출 판정. 현재 프레임 로컬 포인터는 제외.
3. **단언문 크래시 원천 차단**:
   * `assert` 전면 제거 및 다형적 안전 추출 적용.

---

## 2026-09-18: [Resolved] path-sensitive-arqa.ArrayBound 구조체 배열 센티널 오탐 3건 제거 및 상위 버퍼 크기 불일치 트레이드오프 선보고

### 1. 현상 (Symptom)
* 전역 상수 구조체 배열 종료 센티널(`{ NULL, 0, NULL, NULL }`) 미인식으로 인한 루프 초과 오탐(3건) 및 상위 호출자 버퍼 크기 불일치에 따른 경고(2건) 검출.

### 2. 원인 (Root Cause)
* `RegionStoreManager::getBindingForField`가 `superR`이 `ElementRegion`(`arr[i].field`)인 경우 상수 조회를 수행하지 않고 `UnknownVal`을 반환.
* 간접 호출로 상위 호출자 컨텍스트(`is384 == 1`)가 전달되지 않는 튜링 결정 불완전성 직면.

### 3. 해결책 (Resolution)
1. **구조체 배열 상수 폴딩 구현 (`RegionStore.cpp`)**:
   * `superR` 체인을 역추적하여 기저 `VarRegion`과 인덱스 경로를 추출하는 `PathStep` 파이프라인 도입.
   * `getConstantValueFromInitializerPath`를 통해 센티널 필드를 `ConcreteInt(0)`으로 정확히 바인딩.
2. **트레이드오프 선보고 및 구조적 한계 공인**:
   * 체커 레벨 억제 시 진성 버퍼 오버플로우가 침묵(FN)되므로, 버퍼 검출력 보존을 위해 사양서에 '구조적 잔류 한계(Known Limitation)'로 공식 공인.

---

## 2026-09-18: [Resolved] path-sensitive-core.NullDereference 심층 분석(Deep Mode) 억제 해제 정책과 단일 TU 외부 함수 심볼릭 한계에 따른 구조적 오탐 규명

### 1. 현상 (Symptom)
* MbedTLS 벤치마크에서 단일 TU 외부 함수 호출 결과 및 방어적 검사 분기 뒤의 역참조에 대해 5건의 "널 포인터 역참조" 경고 발생.

### 2. 원인 (Root Cause)
* 심층 분석 파이프라인 구축 시 미탐(FN) 최소화를 위해 `suppress-inlined-defensive-checks=false` 및 `suppress-null-return-paths=false`를 의도적으로 주입함.
* 단일 TU 엔진이 크로스 TU extern 함수(`mbedtls_pk_get_type`)의 반환 심볼에 대해 널 경로를 탐색하고, 억제 정책이 비활성화되어 경고 방출.

### 3. 아키텍처 결정 (Architectural Decision)
1. **엔진 중립성 수호 (Zero Cheating)**:
   * 특정 함수명을 하드코딩하거나 널 경로를 무차별 예외 처리하면 실제 세그멘테이션 결함(TP)을 놓치는 치명적 미탐 구멍이 발생하므로 엔진 코드 무수정 원칙 고수.
   * 사양서에 '설계된 심층 모드 억제 해제 정책에 따른 공인 구조적 오탐(Known Limitation)'으로 완결.
2. **러너 고도화**:
   * `AnalysisProfile`(표준 모드 vs 심층 모드) 분리 권고.

---

## 2026-09-18: [Resolved] cfg-null-pointer-arithmetic 복합 논리곱(&&) Terminator 미인식 및 힙 구조체 역참조 오인 결함 해결

### 1. 현상 (Symptom)
* `memset(&mac[operation->mac_size], ...)` 및 힙 포인터 연산(`ssl->handshake->cur_msg_p += ...`)에 대해 3건의 널 포인터 산술 연산 오탐 발생.

### 2. 원인 (Root Cause)
* CFG에서 단락 평가 수식은 terminator가 `BinaryOperator(BO_LAnd)`인 독립 블록으로 분할되는데, `checkEdgeCondition`이 논리 연산자 terminator를 누락함.
* `getReferencedDecl`이 `MemberExpr`에 대해 `FieldDecl*`을 반환하여 동일 타입의 모든 인스턴스를 동일 상태로 오인하고, 힙 간접 참조의 별칭 분석 한계 발생.

### 3. 해결책 (Resolution)
1. `evaluateEdgeGuard`: 단항 논리 부정, 포인터-불리언 캐스트, 이항 비교 및 복합 논리 연산자(`BO_LAnd`, `BO_LOr`)를 불리언 대수에 맞게 재귀 평가.
2. CFG 논리 연산자 Terminator 지원: `Term->isLogicalOp()`를 처리하여 단락 평가 블록 가드 연동.
3. `DeclRefExpr`(`VarDecl`) 전용 정규화 및 `areSameVars` 헬퍼 도입으로 변수 식별 무결성 확립.

---

## 2026-09-21: [Resolved] ast-extern-function-declaration 호스트 MSVC 전처리기 가드 오용 및 컴파일러 내장 함수(__builtin_*) 오탐 해결

### 1. 현상 (Symptom)
* `return __builtin_bswap32(x);` 등 컴파일러 내장 함수 사용 시 "외부 함수가 선언 없이 사용되었습니다" 오탐 2건 검출.

### 2. 원인 (Root Cause)
* `isFromSystemOrBuiltin` 내의 `FD->getBuiltinID() != 0` 검사가 `#if defined(__clang__)`으로 감싸져 있어, Visual Studio MSVC(`cl.exe`)로 컴파일 시 해당 코드가 증발함.
* 컴파일러 내장 함수는 가상 위치를 가질 수 있어 `L.isInvalid()` 검사보다 앞서 Builtin 판별이 우선되어야 함에도 순서가 뒤바뀜.

### 3. 해결책 (Resolution)
1. 호스트 가드 `#if defined(__clang__)` 완전 제거: 호스트 컴파일러와 무관하게 무조건 `FD->getBuiltinID() != 0` 호출.
2. 위치 유효성 검사 앞서 `FD->getBuiltinID() != 0` 및 `starts_with("__builtin_")`를 최우선 평가하도록 재배치.
3. 엔진 중립성 수호: 특정 함수명 하드코딩 0건 달성.

---

## 2026-09-21: [Resolved] lex-macro-defined-before-use 전처리기 내장 연산자(__has_builtin 등) 인자 오인 오탐 및 소스 위치 정규화 해결

### 1. 현상 (Symptom)
* `#if __has_builtin(__builtin_bswap32)` 등 전처리기 내장 연산자 사용 시 매크로 미정의 허위 경고 검출.

### 2. 원인 (Root Cause)
* `DefinedTracker`가 `defined`만 감지하도록 하드코딩되어 함수형 전처리기 연산자(`__has_builtin`, `__has_include` 등)의 인자 스코프를 인식하지 못함.
* 조건식 시작 위치가 매크로 확장 위치를 포함할 때 `Lexer::getSourceText`가 빈 문자열을 반환하여 위치 왜곡 발생.

### 3. 해결책 (Resolution)
1. `isPreprocessorOperator`: `defined`, `__has_builtin`, `__has_include` 등 11종 전처리기 연산자 정확 식별.
2. `DefinedTracker`: `OpParenDepth`로 괄호 깊이를 카운팅하여 닫는 괄호 매칭 시까지 연산자 인자 스코프 격리.
3. `FileCondRange`: `SM.getFileLoc`로 소스 범위를 정규화하여 토큰 위치 왜곡 방지.

---

## 2026-09-21: [Certified] path-sensitive-core.UndefinedBinaryOperatorResult 인라인 복합 에러 반환 제약 소실에 따른 가상 경로 오탐 규명 및 구조적 한계 공인

### 1. 현상 (Symptom)
* `buf[0] >> (8 - siglen * 8 + msb)` 구문에서 "연산자의 왼쪽 피연산자가 쓰레기 값입니다" 오탐 검출.

### 2. 원인 (Root Cause)
* `ret = mbedtls_rsa_public(...)` 호출부 복귀 후 `if (ret != 0) return ret;` 방어 코드가 존재함에도, CSA 심볼릭 엔진이 복합 비트 연산 매크로(`MBEDTLS_ERROR_ADD`) 반환 심볼에 대해 non-zero 제약을 상실(Constraint Loss)하여 `ret == 0` 가상 경로를 분기함.

### 3. 아키텍처 결정 (Architectural Decision)
1. **엔진 중립성 수호 (Zero Cheating)**:
   * 특정 변수명(`buf`)을 하드코딩하거나 비트 시프트의 `isUndef()`를 완화하면 진성 보안 취약점(CWE-457)을 놓치는 심각한 미탐 구멍이 발생하므로 엔진 코드 무수정 원칙 고수.
2. **구조적 한계 공인 (Known Structural Limitation)**:
   * CSA 심볼릭 실행기의 고전적 인라인 대수적 에러 반환 제약 소실에 기인하므로 사양서에 '설계 및 심볼릭 엔진 한계에 따른 공인 구조적 오탐'으로 공식 완결.

---

## 2026-09-21: [Certified] path-sensitive-core.CallAndMessage 심층 분석 모드(방어적 검사 억제 해제) 및 인라인 널 분기에 따른 가상 경로 오탐 규명 및 구조적 한계 공인

### 1. 현상 (Symptom)
* `f_dbg(p_dbg, level, file, line, str)` 호출 시 "호출된 함수 포인터가 널입니다" 오탐 검출.

### 2. 원인 (Root Cause)
* 인라인 호출된 `mbedtls_debug_print_ecp` 내부의 방어적 널 가드 `if (NULL == ssl->conf->f_dbg) return;`에 대해, 심층 분석 옵션(`suppress-inlined-defensive-checks=false`)으로 인해 CSA가 가상 경로를 열고 후속 함수 포인터 역참조 경고를 방출함.

### 3. 아키텍처 결정 (Architectural Decision)
1. **엔진 중립성 수호 (Zero Cheating)**:
   * 특정 함수명을 화이트리스트로 하드코딩하거나 널 함수 포인터 판별을 완화하면 진성 CWE-476 결함을 놓치므로 엔진 소스코드 원형 보존.
2. **구조적 한계 공인 (Known Structural Limitation)**:
   * 심층 분석 모드 하에서 인라인 방어적 가드가 야기하는 구조적 현상이므로 사양서에 '공인 구조적 오탐'으로 완결.

---

## 2026-09-22: [Resolved] LDRA 벤치마크 3대 진성 미탐(FN) 전수 해결 (Unsigned 단항 음수 중첩 수식 미탐 및 UO_Not 범위 초과 상수 평가 결함)

### 1. 현상 (Symptom)
* LDRA 벤치마크 교차 실사 결과 3건의 진성 미탐 확인:
  1. `const size_t diff_msb = (diff | (size_t) -diff);`: unsigned 단항 음수 연산 미검출.
  2. `peer_pms[0] = peer_pms[1] = ~0;`: 8비트 `unsigned char`에 32비트 signed `-1` 대입 미검출.
  3. `size_t in_padding = ~0;`: 64비트 `size_t`에 32비트 signed `-1` 대입 미검출.

### 2. 원인 (Root Cause)
* `UnsignedMinusAssignmentCheck.cpp`가 최상위 식만 검사하여 이항 연산자 내부 중첩 `UnaryOperator`(`-diff`)를 재귀 순회하지 못함.
* `NoOutOfRangeAssignmentCheck.cpp`에 `UO_Not`(`~`) 평가가 누락되어 `~0`을 상수로 인식하지 못하고, `CVal.isAllOnes()` 화이트리스트가 비트폭과 무관하게 무조건 면제함.

### 3. 해결책 (Resolution)
1. **`UnsignedMinusAssignmentCheck` 재귀 탐색 구축**:
   * `findAndReportUnsignedMinus` 재귀 탐색 함수로 AST 수식 트리를 전수 순회하여 `UnaryOperator(UO_Minus)` 포착. `if (BO->isAssignmentOp()) return;` 가드로 연쇄 대입 중복 진단 방지.
2. **`NoOutOfRangeAssignmentCheck` UO_Not 평가 및 엄격한 비트폭 일치 가드 구축**:
   * `evalIntWithLocals`에 `if (UO->getOpcode() == UO_Not) { Out = ~Sub; return true; }` 추가.
   * 비트마스크 관용구 검사를 `CVal.isAllOnes() && (CVal.getBitWidth() == M.Width)`로 엄격화하여, 동일 비트폭 마스크는 허용하고 폭이 다른 축소/음수 대입은 정탐으로 방출.


---

> [!NOTE]
> **[정제 완료 기준선]** 2026-10-06 이전 상위 항목은 정제 완료됨. 신규 인시던트는 이 아래에 추가됩니다.

---
