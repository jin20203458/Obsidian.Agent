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

### 1. Clang-Tidy (AST 기반 구문 분석)
* **목적**: C/C++ 소스코드의 AST(Abstract Syntax Tree)를 파싱하여 안전 크리티컬(Safety-Critical) 코딩 표준(CWE, CERT, MISRA 등) 위배 여부를 패턴 매칭 형태로 정밀 검사합니다.
* **주요 특징**:
  * AST Matcher를 활용하여 빠른 구문 패턴 검출 및 자동 치유(Quick Fix) 대응.
  * Tidy 모듈 내에 룰 클래스를 정의하고 모듈 팩토리를 통해 체커 등록.

### 2. Clang Static Analyzer (경로 민감 분석)
* **목적**: 프로그램의 실행 흐름을 심볼릭 실행(Symbolic Execution)하며 메모리 누수, Use-After-Free, 상수 조건식 결함 등 논리적인 결함을 수학적으로 증명 및 탐지합니다.
* **주요 특징**:
  * Control Flow Graph(CFG)를 따라 가상 실행 상태(ProgramState)를 점진적으로 갱신.
  * Static Analyzer 체커 레지스트리(TableGen 등)를 통해 핵심 분석 체커 등록.

---

## 빌드 및 검증 규격
* **빌드 스크립트 규격**: 증분 컴파일러 빌드를 위해 `Release` 사양으로 `clang` 및 `clang-tidy` 단일 타겟 빌드를 기본으로 수행합니다.
* **트러블슈팅 런북**: 체커 개발 및 컴파일러 엔진 예외 조치에 관한 실전 디버깅 기록은 [troubleshooting/llvm_clang.md](../troubleshooting/llvm_clang.md)를 참조하십시오.
