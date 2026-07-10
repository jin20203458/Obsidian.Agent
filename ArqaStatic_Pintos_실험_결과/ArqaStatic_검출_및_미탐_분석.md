# ArqaStatic 무기체계 코딩 규칙 검출 및 분석 보고서 (threads/synch.c)

본 문서는 ArqaStatic(Clang-Tidy) 정적 분석기가 Pintos 커널의 핵심 동기화 파일인 `threads/synch.c`를 대상으로 무기체계 소프트웨어 코딩 규칙 51종(C++ 전용 제외)을 분석한 **정량적 통계 및 상세 매핑 보고서**입니다.

*본 보고서는 일반 C 표준 적합 여부가 아닌 '국방무기체계 코딩 표준 규격의 엄격한 준수' 관점으로 분석 결과를 재평가하였으며, 외부 시스템 헤더(71건)를 차단한 순수 사용자 코드 영역의 경고만을 분석 모수로 취급하였습니다.*

---

## 1. 📊 종합 분석 결과 요약
* **분석 대상 파일**: `threads/synch.c` (339 라인)
* **총 유효 규칙**: 51종 (C/C++ 공통 및 C 전용 규칙)
* **💡 총 검출 경고(Warnings)**: **55개** (외부 시스템 헤더 71건 제외 완료)
  * **✅ 정탐 (True Positives):** **54개** (국방무기체계 표준에 위배된 실질 코딩 규정 위반 검출)
  * **❌ 오탐 (False Positives):** **1개** (`thread_create` 우선순위 인자를 권한으로 오독한 건)
* **🔎 수동 검출 규칙 수**: **3종** (스택 유출, NULL 가드 생략, 선언부 초기화 생략)

### 룰셋별 상세 검출 현황

| 룰셋 분류 (Category) | 대상 규칙 수 | 검출 성공 규칙 수 | 대표 매핑 체커 이름 |
| :--- | :---: | :---: | :--- |
| 공통 적용 (스타일) | 11 | 3 | `ast-extern-function-declaration`<br>`ast-multi-statement-per-line`<br>`ast-if-style` |
| 공통 적용 (초기화) | 3 | 0 | - |
| 공통 적용 (식별자) | 4 | 0 | - |
| 공통 적용 (조건식) | 5 | 1 | `ast-always-constant-condition`<br>`path-sensitive-arqa.AlwaysConstantCondition` |
| 공통 적용 (변환) | 8 | 1 | `ast-function-call-argument-consistency` |
| 공통 적용 (포인터, 배열) | 5 | 1 | `cfg-null-dereference-guard` |
| 공통 적용 (연산자) | 9 | 0 | - |
| C 전용 규칙 | 5 | 1 | `ast-aggregate-init-style` *(수동 정탐)* |

---

## 2. 🔍 주요 정탐 항목 분석

### ① [포인터 규칙 가] 포인터 참조 전 NULL 검증
* **매핑 체커**: `cfg-null-dereference-guard`
* **위반 라인**: `synch.c` L155, L156
* **분석**: 배열 주소의 물리적 안전성과 관계없이 포인터 변수 참조 시 사전 검사 가드 코드가 부재한 **무기체계 표준 위반 사항**을 정직하게 검출해 낸 정탐입니다.

### ② [조건식 규칙 나] 항상 True/False인 조건식 금지 (AST & 경로 민감 더블 탐지)
* **매핑 체커**: `ast-always-constant-condition` / `path-sensitive-arqa.AlwaysConstantCondition`
* **위반 라인**: `synch.c` L69, L92, L116, L138, L153, L218, L319, L336
* **분석**: 비동기 멀티스레드 런타임 조건과 무관하게 정적 기호 실행(Symbolic Execution) 범위 내에서 조건식이 상시 참/거짓으로 굳어지는 코드의 정형 구조 및 데드 코드를 규격대로 검출한 정탐입니다.

### ③ [스타일 규칙 타] if-else if 구문 else 블록 필수
* **매핑 체커**: `ast-if-style`
* **위반 라인**: `synch.c` L116, L218, L319
* **분석**: 가독성이나 동작 여부와 무관하게 예외 분기 차단을 목적으로 else 명시를 강제하는 **방어적 프로그래밍 스타일 규격 위반**을 잡아낸 완벽한 정탐입니다.
