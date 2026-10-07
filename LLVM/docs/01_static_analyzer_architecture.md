---
description: LLVM/Clang 커스텀 Tidy 체커 및 Static Analyzer 아키텍처 명세서 (스켈레톤).
related:
  - ../README.md
  - ../../README.md
  - ../../troubleshooting/llvm_clang.md
---
# LLVM Static Analyzer & Tidy Architecture

본 문서는 **LLVM/Clang** 정적 분석 엔진 상에 구현된 커스텀 체커(Clang Static Analyzer & Clang-Tidy)들의 구조적 통합 방향과 개발 아키텍처를 안내하는 스켈레톤 문서입니다.

---

## 1. 주요 아키텍처 개요

### 1.1 Clang-Tidy (AST & Preprocessor 구문 분석 및 린팅)
* **목적**: C/C++ 소스코드의 AST(Abstract Syntax Tree)와 전처리기 이벤트를 분석하여 안전 코딩 표준 위반을 탐지하고 자동 치유(FixItHint)를 제공합니다.
* **핵심 메커니즘**:
  * `ClangTidyCheck` 상속: `registerMatchers()`(AST Matcher) 및 `check()`(진단 방출) 구현
  * 전처리기 지시자 검사: `registerPPCallbacks()` 등록
  * 모듈 팩토리 등록: `ClangTidyModule::addCheckFactories()`

### 1.2 Clang Static Analyzer (경로 민감 심볼릭 실행 엔진)
* **목적**: 프로그램의 실행 흐름을 경로 민감(Path-sensitive) 및 인터프로시저럴(Interprocedural)하게 기호 실행하여 런타임 메모리 결함 및 논리 모순을 수학적으로 증명 및 탐지합니다.
* **핵심 메커니즘**:
  * `ExprEngine`의 CFG 순회 및 `ExplodedGraph` / `ProgramState` 상태 불변식 유지
  * `ConstraintManager`를 통한 경로별 심볼릭 값(`SVal`, `SymExpr`) 제약 전파 및 분기 가능성 판정
  * 이벤트 콜백(`check::PreStmt`, `check::PostStmt` 등) 구현 및 TableGen(`Checkers.td`) 등록

---

## 2. 빌드, 검증 및 참조 경로
* **형제 작업공간**: [llvm-project](../../../llvm-project)
* **트러블슈팅 런북**: [troubleshooting/llvm_clang.md](../../troubleshooting/llvm_clang.md)
* **로컬 상위 인덱스**: [LLVM Index](../README.md)
* **루트 인덱스**: [Obsidian.Agent Index](../../README.md)
