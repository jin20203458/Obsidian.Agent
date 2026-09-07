---
description: >-
  LLVM/Clang 커스텀 Tidy 체커 및 Static Analyzer 아키텍처 명세서. LLVM 정적 분석 체커 작성 시 참조.
related:
  - ../README.md
  - ../troubleshooting/llvm_clang.md
---
# LLVM Static Analyzer & Tidy Architecture
> **부제**: LLVM/Clang 정적 분석 및 Clang-Tidy 커스텀 체커 아키텍처

본 문서는 **LLVM/Clang** 정적 분석 엔진 상에 구현된 커스텀 체커(Clang Static Analyzer & Clang-Tidy)들의 구조적 통합 방향과 개발 아키텍처를 안내하는 문서입니다.

---

## 주요 아키텍처 개요

### 1. Clang-Tidy (AST & Preprocessor 기반 구문 린팅)
* **목적**: C/C++ 소스코드의 AST(Abstract Syntax Tree)와 전처리기 이벤트를 분석하여 안전 크리티컬 코딩 표준(CWE, CERT, MISRA 등) 위반을 패턴 매칭하고 자동 치유(Quick Fix)합니다.
* **핵심 메커니즘**:
  * `ClangTidyCheck`를 상속하여 `registerMatchers()`(AST Matcher 등록)와 `check()`(진단 방출 및 `FixItHint` 적용) 구현.
  * 전처리기 지시자 및 헤더 검사가 필요한 경우 `registerPPCallbacks()` 콜백 등록.
  * `ClangTidyModule::addCheckFactories()`를 통해 모듈 단위 체커 팩토리 등록.

### 2. Clang Static Analyzer (경로 민감 심볼릭 실행 엔진)
* **목적**: 프로그램의 실행 흐름을 경로 민감(Path-sensitive) 및 인터프로시저럴(Interprocedural)하게 기호 실행(Symbolic Execution)하여 런타임 결함(메모리 누수, Null 역참조, Use-After-Free, 논리 모순)을 수학적으로 증명 및 탐지합니다.
* **핵심 메커니즘**:
  * `ExprEngine`이 CFG를 순회하며 실행 지점과 불변 상태(`ProgramState`)를 결합한 **`ExplodedGraph`**를 점진적으로 구축.
  * `ConstraintManager`(Range/Z3)를 통해 심볼릭 값(`SVal`, `SymExpr`)의 경로별 조건 제약 전파 및 분기 가능성(Branch Feasibility) 판정.
  * `Checker<check::PreStmt, check::PostStmt, check::BranchCondition>` 등 이벤트 콜백 템플릿을 상속하고 TableGen(`Checkers.td`)에 메타데이터 등록.

---

## 빌드 및 검증 규격
* **빌드 스크립트 규격**: 증분 컴파일러 빌드를 위해 `Release` 사양으로 `clang` 및 `clang-tidy` 단일 타겟 빌드를 기본으로 수행합니다.
* **트러블슈팅 런북**: 체커 개발 및 컴파일러 엔진 예외 조치에 관한 실전 디버깅 기록은 [troubleshooting/llvm_clang.md](../troubleshooting/llvm_clang.md)를 참조하십시오.
