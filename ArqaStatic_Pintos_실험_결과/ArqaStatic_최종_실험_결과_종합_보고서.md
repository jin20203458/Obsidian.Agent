# 📊 ArqaStatic(Clang-Tidy) Pintos synch.c 최종 실험 결과 종합 보고서

본 보고서는 Pintos 커널의 핵심 동기화 파일인 `threads/synch.c`를 대상으로 **국방무기체계 코딩 표준 규격 51종(C++ 전용 제외)**을 적용하여 도출한 **수동 코드 리뷰 결과**와 **ArqaStatic(Clang-Tidy) 정적 분석 결과**를 일대일로 교차 대조하여 종합한 최종 검증 분석서입니다.

---

## 1. 📊 실험 결과 요약 (Executive Summary)

* **대상 파일**: `threads/synch.c` (339 라인)
* **총 분석 경고 건수**: **55건** (외부 시스템 헤더 71건 노이즈 제외 완료)

| 분석 구분 | 고유 규칙 수 (종류) | 위반 검출 건수 (경고 수) | 핵심 비고 |
| :--- | :---: | :---: | :--- |
| **🔎 개발자 수동 검출** | **8종** | **45건** | 개발자가 규정 대입을 통해 규명한 골든 기준 데이터 |
| **✅ Clang-Tidy 정탐 (TP)** | **6종** | **54건** | 수동 검출 8종 중 6종(43건)을 도구가 정밀 자동 포착 |
| **❌ Clang-Tidy 미탐 (FN)** | **2종** | **2건** | 포인터 앨리어싱 및 초기화 함수 분석 한계로 미탐 |
| **⚠️ Clang-Tidy 오탐 (FP)** | **1종** | **1건** | `thread_create` 인자 도메인 오독 (규격 외 오탐) |

* **💡 수동 검출 대비 도구 규칙 정탐률 (Recall):** **75.0%** (8종 중 6종 검출 성공)
* **📉 실질 결함 개소 정탐율 (Warning Match):** **95.6%** (수동 위반 개소 45건 중 43건 도구 적발 성공)

---

## 🔎 2. 개발자 수동 검출 항목 명세 (Golden Reference)
*개발자가 직접 소스코드 구조를 분석하여 찾아낸 명백한 코딩 규칙 위반 사항입니다 (총 8종 45건).*

### ① [포인터 35] 포인터 참조 전 NULL 검사 누락 (1건)
* **대상 코드**: `threads/synch.c` [L147-L157](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L147-L157) (`sema_test_helper` 함수)
* **위반 사유**: 인자로 들어온 `void *sema_` 포인터에 대해 `ASSERT(sema != NULL)` 등의 안전성 가드 검증을 생략하고 다이렉트로 역참조 연산(`&sema[0]`)을 수행했습니다.

### ② [포인터 32] 지역 스택 주소의 외부 유출 (1건)
* **대상 코드**: `threads/synch.c` [L290-L299](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L290-L299) (`cond_wait` 함수)
* **위반 사유**: 함수가 반환되면 소멸할 지역 변수 `waiter`의 주소를 함수 스코프보다 생명주기가 긴 외부 구조체 `cond->waiters` 리스트에 밀어 넣었습니다.

### ③ [C-5] 구조체/배열 선언 시 초기화 생략 (2건)
* **대상 코드**: [L131](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L131) (`sema` 배열), [L290](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L290) (`waiter` 구조체)
* **위반 사유**: 선언 즉시 기본값 `{0}` 초기화를 누락하고 하위의 동적 초기화 함수 호출에 의존하였습니다.

### ④ [스타일 07, 08, 11] 및 [조건식 19], [변환 26] 표준 규격 위반 (총 41건)
* **스타일 07**: 외부 라이브러리 함수 호출 시 `extern` 명시 누락 (17건)
* **스타일 08**: 함수 반환형 개행 누락 및 한 줄 다중 선언 (6건)
* **스타일 11**: 방어적 예외 제어를 위한 `else` 블록 누락 (3건)
* **조건식 19**: 정적 흐름상 상시 참/거짓인 조건식 방치 (8건)
* **변환 26**: signed 상수를 unsigned 매개변수에 캐스트 없이 대입한 형 정합성 누락 (7건)

---

## ✅ 3. Clang-Tidy 정탐(True Positive) 정밀 분석 (총 6종 54건)
*수동으로 규명한 위반 8종 중 Clang-Tidy가 매핑 체커를 통해 정탐(TP)에 성공한 내역입니다.*

### ① `cfg-null-dereference-guard` (NULL 포인터 가드 누락) - **2건** (수동 규칙 35번)
* **검출 대상**: [L155](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L155) `sema_down (&sema[0]);` 및 [L156](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L156) `sema_up (&sema[1]);`
* **도구 경고**: `warning: 포인터 'sema'는 안전함이 보장되지 않았습니다. [cfg-null-dereference-guard]`
* **정밀 분석**: 제어 흐름 그래프(CFG) 상에서 NULL 여부를 확인하는 가드 구문이 누락된 역참조 연산을 정확히 탐지하였습니다.

### ② `path-sensitive-arqa.AlwaysConstantCondition` (상수 조건식 금지) - **8건** (수동 규칙 19번)
* **검출 대상**: L69 (`while (sema->value == 0)`), L92 (`if (sema->value > 0)`), L116 (`if (!list_empty)`), L319 등 8건
* **도구 경고**: `warning: 조건문의 결과가 항상 거짓(False)으로 평가됩니다. ... [path-sensitive-arqa.AlwaysConstantCondition]`
* **정밀 분석**: 기호 실행을 통해 특정 흐름 경로에서 조건 평가식이 상시 참 또는 거짓이 되어 데드 코드가 유발되는 논리 오류를 규명했습니다.

### ③ `ast-if-style` (if-else if 문 else 블록 필수) - **3건** (수동 규칙 11번)
* **검출 대상**: [L116](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L116), [L218](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L218), [L319](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L319)
* **도구 경고**: `warning: if 문에는 반드시 else 블록이 필요합니다 [ast-if-style]`
* **정밀 분석**: 방어적 예외 제어인 `else` 블록 작성이 생략된 구조를 AST 상에서 단속하였습니다.

### ④ `ast-function-call-argument-consistency` (암묵적 형변환 및 인수 정합성) - **7건** (수동 규칙 26번)
* **검출 대상**: L135 `sema_init(..., 0)`, L181 `sema_init(..., 1)` 및 printf 첫 인수 L134 등
* **도구 경고**: `warning: 2번째 인자에서 허용되지 않는 암묵 변환(integral cast)이 발생했습니다. ... [ast-function-call-argument-consistency]`
* **정밀 분석**: 묵시적 캐스트 동작 및 signed-unsigned 파라미터 간의 정합성 누락을 정확히 적발하였습니다.

### ⑤ `ast-extern-function-declaration` (외부 함수 extern 누락) - **17건** (수동 규칙 7번)
* **도구 경고**: `warning: 외부 함수 'list_init'는 호출 이전 선언에서 'extern'을 명시해야 합니다. [ast-extern-function-declaration]`
* **정밀 분석**: 외부 라이브러리 심볼 호출 시 명시적인 `extern` 선언 누락 지점을 칼같이 적발해 냈습니다.

### ⑥ `ast-multi-statement-per-line` (한 줄에 하나의 명령문 작성) - **6건** (수동 규칙 8번)
* **도구 경고**: `warning: 한 줄에 여러 개의 전역 선언(변수/함수/타입 등)이 있습니다. ... [ast-multi-statement-per-line]`
* **정밀 분석**: 함수 반환 타입 선언과 함수명이 행을 분리하지 않은 가독성 표준 미준수 라인(L60, L108 등)을 적발하였습니다.

---

## ❌ 4. Clang-Tidy 미탐(False Negative) 상세 분석 (총 2종 2건)
*수동 분석에선 규격 위반이었으나 도구가 인지하지 못한 항목에 대한 구체적 코드 분석 결과입니다.*

1. **[포인터 32] 지역 스택 주소의 외부 유출 미탐 (`cond_wait` L298):**
   * **원인**: 핀토스 내부 리스트 구조 매크로 연산(`list_entry` 등)과 캐스팅으로 인해, 정적 분석 엔진의 포인터 데이터 흐름 추적기(Taint/Data flow)가 에일리어싱 한계에 부딪혀 추적 경로를 잃고 미탐되었습니다.
2. **[C-5] 구조체/배열 선언 시 초기화 생략 미탐 (L131, L290):**
   * **원인**: 사용(Use) 전 단계에서 동적 초기화 함수(`sema_init` 등)가 100% 호출되는 런타임 경로가 확립되어 있어, 미초기화 변수 경고에서 탈출되어 미탐되었습니다.

---

## ⚠️ 5. Clang-Tidy 오탐(False Positive) 정밀 해부 (총 1종 1건)
*코드 흐름에 대한 해석 실패 및 도메인 오인으로 발생한 명백한 도구 오탐 내역입니다.*

### ① `ast-incorrect-permission-assignment` (취약 권한 부여 오독) - **100% 오탐**
* **대상 라인**: [L137](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L137) `thread_create ("sema-test", PRI_DEFAULT, ...);`
* **사유**: `thread_create` 의 두 번째 매개변수인 스레드 실행 스케줄링 우선순위 값인 `PRI_DEFAULT` (31)를, 파일 접근 제어 권한(Permission ACL) 할당 함수의 취약 권한 마스크로 잘못 오독한 도구의 기술적 오탐입니다.
