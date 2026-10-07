---
description: LLVM/Clang 커스텀 Tidy 체커 및 Static Analyzer 핵심 아키텍처, 기호 실행 엔진, CTU 및 Z3 솔버 통합 설계 명세서.
related:
  - ../README.md
  - ../../troubleshooting/llvm_clang.md
---
# LLVM Static Analyzer & Tidy Architecture

본 문서는 **LLVM/Clang** 정적 분석 엔진 인프라를 기반으로 구축되는 구문 린팅/변환기(Clang-Tidy)와 경로 민감 심볼릭 실행 엔진(Clang Static Analyzer, CSA)의 내부 아키텍처, 동작 원리, 확장 메커니즘 및 빌드/검증 규격을 정의하는 단일 진실 공급원(SSOT) 명세서입니다.

---

## 1. 정적 분석 양대 엔진 위상학 (Dual-Engine Topology)

LLVM 기반 정적 분석은 해결하고자 하는 결함의 수학적 본질에 따라 상호 보완적인 두 개의 독립 엔진으로 분기됩니다.

```
+-----------------------------------------------------------------------------------+
|                               Clang Compiler Driver                               |
|                     (Source Code -> Lexer -> Preprocessor -> AST)                 |
+-----------------------------------------------------------------------------------+
                                          |
                   +----------------------+----------------------+
                   |                                             |
                   v                                             v
+-------------------------------------+       +-------------------------------------+
|          Clang-Tidy Engine          |       |     Clang Static Analyzer (CSA)     |
+-------------------------------------+       +-------------------------------------+
| * 영역: 단일 컴파일 단위 (Intra-TU)    |       | * 영역: 경로 민감 / 인터프로시저럴    |
| * 기반: 구문 트리 (AST) & 전처리기   |       | * 기반: CFG 순회 및 ExplodedGraph   |
| * 탐색: 패턴 매칭 (AST Matchers)    |       | * 탐색: 기호 실행 (Symbolic Exec)   |
| * 초점: 코딩 표준, 안티패턴, 린팅   |       | * 초점: 메모리 누수, UAF, Null 역참조 |
| * 비용: O(N) 선형 소요 (초저지연)   |       | * 비용: 지수적 경로 분기 (고비용)   |
| * 수정: 자동 치유 (FixItHint)       |       | * 솔버: Z3 SMT 및 Range 제약 관리기  |
+-------------------------------------+       +-------------------------------------+
```

### 1.1 엔진 선택 기준 (Decision Matrix)
* **Clang-Tidy 적용 대상**:
  * 제어 흐름 추적 없이 국소적인 문법/타입/전처리기 패턴 매칭만으로 증명 가능한 규칙 (예: CWE 명명 규칙, C++ Core Guidelines, MISRA C++ 구문 규칙).
  * 코드 자동 치유(`FixItHint`)가 요구되는 변환 작업.
* **Clang Static Analyzer 적용 대상**:
  * 함수 호출 체인을 넘나들거나(`Interprocedural`), 특정 분기 조건의 참/거짓 조합에 따라 메모리 상태가 동적으로 변하는 런타임 결함 (예: 이중 해제(Double-Free), 해제 후 사용(Use-After-Free), 초기화되지 않은 메모리 읽기, 논리적 모순/도달 불능 코드).

---

## 2. Clang-Tidy 아키텍처 및 방어적 AST 설계

### 2.1 체커 등록 및 팩토리 수명주기
Clang-Tidy는 모듈 기반 플러그인 아키텍처를 따릅니다. 각 체커는 `ClangTidyCheck`를 상속하며, 모듈 클래스에 팩토리 함수를 등록함으로써 활성화됩니다.

```cpp
// 체커 기본 골격
class CustomStyleCheck : public ClangTidyCheck {
public:
  CustomStyleCheck(StringRef Name, ClangTidyContext *Context)
      : ClangTidyCheck(Name, Context) {}

  void registerMatchers(ast_matchers::MatchFinder *Finder) override;
  void registerPPCallbacks(const SourceManager &SM, Preprocessor *PP,
                           Preprocessor *ModuleExpanderPP) override;
  void check(const ast_matchers::MatchFinder::MatchResult &Result) override;
};
```

* **`registerMatchers()`**: 선언적 DSL 매처(`ast_matchers`)를 통해 분석 대상 AST 노드(함수, 클래스, 변수 선언, 호출 표현식 등)를 필터링하여 등록.
* **`registerPPCallbacks()`**: `#include`, `#define`, 조건부 컴파일 등 전처리기 토큰 레벨의 검사가 필요할 때 콜백 등록.
* **`check()`**: 매칭된 AST 노드를 검사하여 위반 사항 검출 시 `diag()` 방출 및 `FixItHint` 첨부.

### 2.2 방어적 AST 설계 불변식 (Defensive AST Traversal)
실무 대형 C++ 코드베이스 분석 시, 불완전한 템플릿 인스턴스화나 컴파일 에러 상태의 AST로 인해 체커 내부 크래시(Assertion Failure / UNREACHABLE)가 빈번히 발생합니다. 체커 작성 시 다음 4대 방어 불변식을 필수 준수해야 합니다.

1. **템플릿 종속식 상수 평가 가드 (`isValueDependent`)**:
   * 템플릿 매개변수에 의존하는 표현식은 인스턴스화 전까지 실제 값을 확정할 수 없습니다.
   * `Expr::EvaluateAsInt()` 또는 상수 평가기 호출 전, 반드시 `!E->isValueDependent() && !E->isTypeDependent()` 조건을 선행 검증해야 합니다. 미검증 시 LLVM 상수 평가기 내부 Assertion 크래시가 발생합니다.
2. **에러 복구 노드 격리 (`RecoveryExpr`)**:
   * 헤더 누락이나 오타 등으로 문법 분석에 실패한 코드는 Clang 내부에서 `RecoveryExpr` 노드로 치환됩니다.
   * 복구 노드에 대해 타입 크기(`Context->getTypeSize()`)를 조회하거나 세부 표현식을 순회하면 `UNREACHABLE executed at ASTContext.cpp` 크래시를 유발하므로, `isa<RecoveryExpr>(E)` 또는 유효하지 않은 타입(`QualType::isNull()`)을 최우선 필터링합니다.
3. **소스 위치 무결성 (`SpellingLoc` vs `ExpansionLoc`)**:
   * 매크로로 확장된 소스코드 위치를 진단할 때, 실제 텍스트가 작성된 위치는 `SM.getSpellingLoc()`, 매크로가 호출된 위치는 `SM.getExpansionLoc()`입니다.
   * Quick-Fix를 생성할 때는 실제 파일 수정 위치인 `SpellingLoc` 범위를 기반으로 치환 범위를 계산해야 파일 손상을 방지할 수 있습니다.
4. **널 포인터 및 상속 캐스팅 방어**:
   * AST 순회 시 부모/자식 노드 반환값(`dyn_cast`, `getAs<T>()`)은 항상 `nullptr` 가능성을 염두에 두고 즉시 Null 체크를 수행해야 합니다.

---

## 3. Clang Static Analyzer (CSA) 기호 실행 및 메모리 모델

Clang Static Analyzer는 제어 흐름 그래프(CFG)를 기반으로 프로그램의 가능한 모든 실행 경로를 시뮬레이션하는 기호 실행(Symbolic Execution) 엔진입니다.

```
                              [CFG Block Entry]
                                      |
                     +---------------------------------+
                     |   ExplodedNode N0               |
                     |   (ProgramPoint: Stmt A)        |
                     |   (ProgramState: { x -> $0 })   |
                     +---------------------------------+
                                      |
                            [Branch: if (x > 0)]
                                      |
                    +-----------------+-----------------+
                    |                                   |
         (Path True: $0 > 0)                  (Path False: $0 <= 0)
                    v                                   v
+---------------------------------------+ +---------------------------------------+
| ExplodedNode N1                       | | ExplodedNode N2                       |
| (ProgramPoint: Stmt B)                | | (ProgramPoint: Stmt C)                |
| (ProgramState: { x -> $0, $0 > 0 })   | | (ProgramState: { x -> $0, $0 <= 0 })  |
+---------------------------------------+ +---------------------------------------+
```

### 3.1 ExplodedGraph와 상태 불변식
* **`ExplodedNode`**: 프로그램의 실행 위치를 나타내는 `ProgramPoint`와, 해당 시점의 시스템 불변식 스냅샷인 `ProgramStateRef`의 결합 노드입니다.
* **상태 불변성 (Immutability)**: `ProgramState`는 함수형 영속 데이터 구조(`llvm::ImutAVLTree` 기반 `ImmutableMap`/`ImmutableSet`)로 관리됩니다. 새로운 상태가 생성되어도 이전 노드의 상태는 수정되지 않으며 포인터 참조만 분기됩니다.
* **분기점 조상 역추적 불변식**:
  * 조건문 분기 콜백(`checkBranchCondition`) 시점에서 `C.getState()`는 이미 해당 경로의 불리언 값(`1` 또는 `0`)으로 고정된 상태입니다.
  * 조건식의 원래 기호 값(`SymExpr`)을 평가하기 위해서는 조상 노드(`C.getPredecessor()`)를 역추적하여 상수가 아닌 최초의 `SVal`을 추출해야 중복/오탐 분기를 방지할 수 있습니다.

### 3.2 심볼릭 값 (`SVal`) 체계
CSA에서 모든 C/C++ 표현식의 값은 `SVal` 추상화로 표현됩니다:

* **`UndefinedVal`**: 쓰레기 값(미초기화 변수). 읽기 시도 시 결함 판정.
* **`UnknownVal`**: 분석기의 분석 범위를 벗어나 모델링할 수 없는 값.
* **`loc::ConcreteInt` / `nonloc::ConcreteInt`**: 컴파일 타임에 확정된 실제 정수/포인터 상수.
* **`loc::MemRegionVal`**: 물리적/논리적 메모리 위치를 가리키는 포인터.
* **`nonloc::SymbolVal`**: 런타임에 결정되는 미지의 대수적 기호(`SymbolRef`).

### 3.3 메모리 영역 모델 (`MemRegion` & `RegionStore`)
메모리는 타입 계층 구조를 갖는 `MemRegion`과, 영역과 값 간의 바인딩을 매핑하는 `RegionStore`로 모델링됩니다:

* **`StackLocalsSpaceRegion`**: 스코프 종료 시 자동으로 무효화되는 로컬 스택 프레임.
* **`HeapSpaceRegion`**: `malloc`/`new`로 할당되며 명시적 `free`/`delete` 추적이 필요한 힙 영역.
* **`SymbolicRegion`**: 포인터 역참조(`*p`)를 통해 접근하는 미지의 메모리 공간.

---

## 4. 제약 관리자 (ConstraintManager) 및 Z3 Solver 통합

기호 실행 중 분기문(`if`, `switch`)을 만날 때마다, 엔진은 해당 경로가 논리적으로 실현 가능한지(Path Feasibility) 판정해야 합니다.

### 4.1 2단계 제약 해결 토폴로지
1. **1차: `RangeConstraintManager` (경량 정수 구간 엔진)**:
   * 정수 변수의 상한/하한 구간(`[min, max]`)을 관리하여 빠른 O(1)~O(log N) 탐색 수행.
   * 단순 산술 부등식(`x > 5`, `y == 0`)을 초고속으로 평가.
2. **2차: `Z3ConstraintManager` (SMT Solver 폴백)**:
   * 비선형 연산, 복합 비트 연산, 다변수 연립 제약 조건 등 Range 매니저가 판정할 수 없는 난제 발생 시 Microsoft Z3 SMT Solver(`libz3.dll`)로 위임.
   * 만족 불가능(UNSAT) 판정 시 해당 경로는 유령 경로(Dead Path)로 간주하고 탐색 중단(Pruning).

### 4.2 Z3 활성화 빌드 규격
SMT 제약 검증 엔진을 활성화하기 위해 CMake 구성 시 Z3 라이브러리 연동 플래그를 필수로 주입해야 합니다:
* `-DLLVM_ENABLE_Z3_SOLVER=ON`
* `-DZ3_INCLUDE_DIR="<Path_To_Z3>/include"`
* `-DZ3_LIBRARIES="<Path_To_Z3>/bin/libz3.lib"`

---

## 5. 크로스 컴파일 단위 분석 (CTU: Cross-Translation-Unit)

전통적인 정적 분석은 단일 파일(`.c`/`.cpp`) 단위로 제한되어 타 소스 파일에 정의된 함수 호출 시 보수적으로 상태를 날려버리는(Invalidation) 한계가 있습니다. LLVM CSA의 CTU 파이프라인은 이를 전역으로 확장합니다.

```
+-----------------------------------------------------------------------------------+
| Phase 1: 외부 정의 매핑 인덱싱 (clang-extdef-mapping)                            |
| * 각 소스 파일의 최상위 함수 시그니처 USR -> 소스 AST 파일 위치 인덱스 파일 생성   |
+-----------------------------------------------------------------------------------+
                                          |
                                          v
+-----------------------------------------------------------------------------------+
| Phase 2: 온디맨드 AST 수입 (CrossTranslationUnitContext & ASTImporter)           |
| * CSA 분석 도중 타 파일에 정의된 함수 호출 발견 시, 해당 AST만 실시간 메모리로 임포트|
| * 컨텍스트 유지하며 Interprocedural 기호 실행 지속                                |
+-----------------------------------------------------------------------------------+
```

### 5.1 CTU 메모리 제어 불변식
* 대형 프로젝트 분석 시 모든 AST를 메모리에 올리면 OOM(Out of Memory)이 발생합니다.
* CTU 엔진은 파싱된 AST 수명을 제어하는 LRU 캐시 메커니즘을 내장하고 있으며, 분석 대상 프로젝트 빌드 데이터베이스(`compile_commands.json`)의 유효성을 전제로 작동합니다.

---

## 6. 체커 등록 및 확장 지점 (Extension Points SSOT)

LLVM 코드베이스 상에서 새로운 분석 규칙을 영구적으로 추가하는 정규 진입점입니다.

### 6.1 Clang Static Analyzer 체커 등록
1. **TableGen 선언 (`clang/include/clang/StaticAnalyzer/Checkers/Checkers.td`)**:
   * 체커 패키지(예: `alpha.core`, `security`), 의존성, 도움말 설명 등록.
   ```tablegen
   def CustomSecurityChecker : Checker<"CustomSecurity">,
     HelpText<"Detect dangerous pattern in custom security standard">,
     Documentation<HasDocumentation>;
   ```
2. **C++ 체커 구현 및 등록 매크로**:
   * `clang/lib/StaticAnalyzer/Checkers/` 하위에 소스 작성.
   * `registerCustomSecurityChecker(CheckerManager &mgr)` 및 `shouldRegisterCustomSecurityChecker(const CheckerManager &mgr)` 진입점 구현.

### 6.2 Clang-Tidy 체커 등록
1. **모듈 등록 (`clang-tools-extra/clang-tidy/<Module>/<Module>Module.cpp`)**:
   * `addCheckFactories()` 내부에 신규 체커 팩토리 바인딩.
   ```cpp
   CheckFactories.registerCheck<CustomStyleCheck>("custom-style-check");
   ```
2. **CMake 타겟 등록**:
   * `clang-tools-extra/clang-tidy/<Module>/CMakeLists.txt` 소스 파일 추가.

---

## 7. 빌드, 실행 및 검증 프로토콜 (Ground Truth)

`llvm-project` 환경에서 정적 분석 엔진을 컴파일하고 검증하기 위한 단일 표준 CLI 규격입니다.

### 7.1 CMake 구성 및 증분 빌드
```powershell
# 1. 빌드 환경 구성 (Z3 Solver 활성화)
cmake -S llvm -B build `
  -DLLVM_ENABLE_PROJECTS="clang;clang-tools-extra" `
  -DCMAKE_BUILD_TYPE=Release `
  -DLLVM_ENABLE_Z3_SOLVER=ON `
  -DZ3_INCLUDE_DIR="C:/Users/user/Documents/GitHub/llvm-project/z3/include" `
  -DZ3_LIBRARIES="C:/Users/user/Documents/GitHub/llvm-project/z3/bin/libz3.lib"

# 2. 증분 빌드 타겟 실행 (clang-tidy 또는 clang 단일 타겟)
cmake --build .\build --config Release --target clang-tidy
cmake --build .\build --config Release --target clang
```

### 7.2 체커 단독 실행 및 즉시 검증 (CLI)
* **Clang-Tidy 단독 실행**:
  ```powershell
  .\build\Release\bin\clang-tidy.exe --checks="<checker_name>" <target_file.cpp> -- -std=c++20
  ```
* **Clang Static Analyzer 단독 실행**:
  ```powershell
  .\build\Release\bin\clang.exe --analyze `
    -Xanalyzer -analyzer-checker=<checker_name> `
    -Xanalyzer -analyzer-output=text `
    <target_file.cpp>
  ```

---

