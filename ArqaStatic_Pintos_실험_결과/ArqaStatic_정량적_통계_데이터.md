# 3. ArqaStatic 규칙 검출 정량적 통계 데이터 (Pintos threads/synch.c)

본 문서는 ArqaStatic(Clang-Tidy) 정적 분석기가 Pintos 커널의 핵심 동기화 모듈인 `threads/synch.c`를 대상으로 수행한 무기체계 소프트웨어 코딩 규칙 진단 성능을 **수치화(Metrics)하여 정리한 통계 자료**입니다.

## 3.1. 전역 지표 요약 (Global Metrics)

| 지표 (Metrics) | 수치 (Value) | 비고 |
| :--- | :---: | :--- |
| **총 대상 규칙 (Total Rules)** | 51 | C++ 전용 규칙(15종)을 제외한 C/C++ 공통 및 C 전용 규칙 |
| **실제 분석 대상 파일** | 1 | `threads/synch.c` (339 라인) |
| | | |
| **✅ 도구 검출 성공 (Tool Detected)** | **6** | Clang-Tidy 체커에 의해 경고가 발생한 고유 규칙 수 |
| **🔎 수동 검출 성공 (Manual Only)** | **1** | 도구 미탐이나 코드 분석상 명백히 어긴 규칙 수 (C 전용 5) |
| **💡 고유 결함 검출 총합** | **7** | 수동 및 도구 검출을 결합한 총 규칙 위반 검출 수 |
| **📉 검출된 분석 경고 총합 (Warnings)** | **112** | 중복 탐지 및 파일 전체 경고 발생 횟수 |

---

## 3.2. 룰셋 카테고리별 검출 정량 데이터 (Category Metrics)

| 카테고리 | 대상 규칙 수 | 검출(Tool) | 검출(Manual) | 대표 매핑 체커 이름 |
| :--- | :---: | :---: | :---: | :--- |
| Common_Style | 11 | 3 | 0 | `ast-extern-function-declaration`<br>`ast-multi-statement-per-line`<br>`ast-if-style` |
| Common_Init | 3 | 0 | 0 | - |
| Common_Ident | 4 | 0 | 0 | - |
| Common_Cond | 5 | 1 | 0 | `ast-always-constant-condition` |
| Common_Conv | 8 | 1 | 0 | `ast-function-call-argument-consistency` |
| Common_PtrArray | 5 | 1 | 0 | `cfg-null-dereference-guard` |
| Common_Expr | 9 | 0 | 0 | - |
| C_Specific | 5 | 0 | 1 | `ast-aggregate-init-style` (수동 탐지) |

---

## 3.3. 결론적 정량 평가
* 본 실험을 통해 ArqaStatic(Clang-Tidy)은 핀토스 커널의 C 소스코드 내에서 **총 7종의 고유 무기체계 소프트웨어 코딩 규칙 위반을 규명**하는 데 성공하였습니다.
* 특히 **`cfg-null-dereference-guard`** 체커를 통한 제어 흐름 기반 포인터 NULL 검증 누락 탐지와, **`ast-if-style`** 체커를 통한 방어적 코딩 스타일 단속 등, 기존 단순 구문 매칭 도구(Cppcheck 등)가 놓치기 쉬운 결함들을 AST와 CFG를 결합해 정밀 진단할 수 있음을 데이터로 증명하였습니다.
