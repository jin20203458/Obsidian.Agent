---
type: reference
title: LLVM Static Analyzer & Tidy Architecture
tags: [llvm, clang-tidy, static-analyzer, architecture, cpp]
related:
  - ../README.md
status: stable
---
# LLVM Static Analyzer & Tidy Architecture
> **부제**: LLVM/Clang 정적 분석 및 Clang-Tidy 커스텀 체커 아키텍처

본 문서는 **LLVM/Clang** 정적 분석 엔진 상에 구현된 커스텀 체커(Clang Static Analyzer & Clang-Tidy)들의 구조적 통합 방향과 개발 아키텍처를 안내하는 문서입니다.

---

##  주요 아키텍처 개요

### 1. Clang-Tidy (AST 기반 구문 분석)
* **목적**: C++ 소스코드의 AST(Abstract Syntax Tree)를 파싱하여 무기체계 SW 코딩 규칙(CWE/CERT 등) 위배 여부를 패턴 매칭 형태로 정밀 검사합니다.
* **주요 특징**:
  * AST Matcher를 활용하여 빠른 패턴 검출 및 자동 치유(Quick Fix) 대응.
  * `clang-tools-extra/clang-tidy/ARQA/` 내에 개별 Tidy 룰 정의.

### 2. Clang Static Analyzer (경로 민감 분석)
* **목적**: 프로그램의 실행 흐름을 심볼릭 실행(Symbolic Execution)하며 메모리 누수, Use-After-Free, 상수 조건식 결함 등 논리적인 결함을 수학적으로 증명 및 탐지합니다.
* **주요 특징**:
  * Control Flow Graph(CFG)를 따라 가상 실행 상태(ProgramState)를 점진적으로 갱신.
  * `clang/lib/StaticAnalyzer/Checkers/` 내에 핵심 분석 체커 등록.

---

##  개발 및 참조 템플릿
* **체커 상태 모델링 규칙**: 분기(Assume) 조건문 평가 시 상태 갈라짐(Post-Split) 문제를 방지하기 위해 반드시 `checkBranchCondition` 내에서 `C.getPredecessor()`를 통한 선행 상태 역추적 기법을 유지하십시오. (상세 내용은 [troubleshooting/llvm_clang.md](../troubleshooting/llvm_clang.md) 참조)
* **빌드 스크립트 규격**: 증분 컴파일러 빌드를 위해 `Release` 사양으로 `clang` 단일 타겟 빌드를 기본으로 수행합니다.
