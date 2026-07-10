# ArqaStatic 무기체계 코딩 규칙 검출 및 분석 보고서 (threads/synch.c)

본 문서는 ArqaStatic(Clang-Tidy) 정적 분석기가 Pintos 커널의 핵심 동기화 파일인 `threads/synch.c`를 대상으로 무기체계 소프트웨어 코딩 규칙 51종(C++ 전용 제외)을 분석한 **정량적 통계 및 상세 매핑 보고서**입니다.

---

## 1. 📊 종합 분석 결과 요약
* **분석 대상 파일**: `threads/synch.c` (339 라인)
* **총 유효 규칙**: 51종 (C/C++ 공통 및 C 전용 규칙)
* **✅ 실제 검출 규칙 수**: **7종** (도구 자동 검출 6종 + 수동 리뷰 분석 1종)
* **💡 총 검출 경고(Warnings)**: 112개

### 룰셋별 상세 검출 현황

| 룰셋 분류 (Category) | 대상 규칙 수 | 검출 성공 규칙 수 | 대표 매핑 체커 이름 |
| :--- | :---: | :---: | :--- |
| 공통 적용 (스타일) | 11 | 3 | `ast-extern-function-declaration`<br>`ast-multi-statement-per-line`<br>`ast-if-style` |
| 공통 적용 (초기화) | 3 | 0 | - |
| 공통 적용 (식별자) | 4 | 0 | - |
| 공통 적용 (조건식) | 5 | 1 | `ast-always-constant-condition` |
| 공통 적용 (변환) | 8 | 1 | `ast-function-call-argument-consistency` |
| 공통 적용 (포인터, 배열) | 5 | 1 | `cfg-null-dereference-guard` |
| 공통 적용 (연산자) | 9 | 0 | - |
| C 전용 규칙 | 5 | 1 | `ast-aggregate-init-style` *(수동 정탐)* |

---

## 2. 🔍 주요 정탐 항목 분석

### ① [포인터 규칙 가] 포인터 참조 전 NULL 검증
* **매핑 체커**: `cfg-null-dereference-guard`
* **위반 라인**: `synch.c` L155, L156
* **분석**: `sema_test_helper`에서 `void *`로 넘어온 포인터 인자를 NULL 검사 없이 안전하지 않게 연산하고 접근하는 결함을 제어 흐름 추적을 통해 정확히 진단했습니다.

### ② [조건식 규칙 나] 항상 True/False인 조건식 금지
* **매핑 체커**: `ast-always-constant-condition`
* **위반 라인**: `synch.c` L138, L153
* **분석**: `for (i = 0; i < 10; i++)` 루프의 첫 분기 진입 시 `i = 0`에 의해 조건식이 정적으로 항상 참이 됨을 AST 상수로 파악하여 검출하였습니다.

### ③ [스타일 규칙 타] if-else if 구문 else 블록 필수
* **매핑 체커**: `ast-if-style`
* **위반 라인**: `synch.c` L116, L218, L319
* **분석**: 방어적 예외 흐름 관리를 위해 `if` 블록 뒤에 `else` 처리가 누락되었음을 엄격하게 단속했습니다.
