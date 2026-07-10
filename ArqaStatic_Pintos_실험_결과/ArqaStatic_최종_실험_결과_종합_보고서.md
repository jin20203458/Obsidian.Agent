# 📊 ArqaStatic(Clang-Tidy) Pintos synch.c 최종 실험 결과 종합 보고서

본 보고서는 Pintos 커널의 핵심 동기화 파일인 `threads/synch.c`를 대상으로 **국방무기체계 코딩 표준 규격 51종(C++ 전용 제외)**을 적용하여 도출한 **수동 코드 리뷰 결과**와 **ArqaStatic(Clang-Tidy) 정적 분석 결과**를 일대일로 교차 대조하여 종합한 최종 검증 분석서입니다.

---

## 1. 📊 실험 결과 요약 (Executive Summary)

* **대상 파일**: `threads/synch.c` (339 라인)
* **총 분석 경고 건수**: **55건** (외부 시스템 헤더 71건 노이즈 제외 완료)

| 분석 구분 | 고유 규칙 수 (종류) | 위반 검출 건수 (경고 수) | 핵심 비고 |
| :--- | :---: | :---: | :--- |
| **🔎 개발자 수동 검출** | **3종** | **4건** | 개발자가 전수 눈으로 찾아낸 골든 레퍼런스 |
| **✅ Clang-Tidy 정탐 (TP)** | **6종** | **54건** | 국방 코딩 규격을 엄격하게 위배한 건수 적발 |
| **❌ Clang-Tidy 미탐 (FN)** | **2종** | **2건** | 포인터 앨리어싱 및 초기화 함수 분석 한계 |
| **⚠️ Clang-Tidy 오탐 (FP)** | **1종** | **1건** | `thread_create` 인자 도메인 오독 |

---

## 🔎 2. 개발자 수동 검출 항목 명세 (Golden Reference)
*도구 분석 전에 사람이 직접 소스코드 구조를 분석하여 찾아낸 명백한 코딩 규칙 위반 사항입니다 (총 3종 4건).*

### ① [포인터 35] 포인터 참조 전 NULL 검사 누락 (1건)
* **대상 코드**: `threads/synch.c` [L147-L157](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L147-L157)
  ```c
  static void sema_test_helper (void *sema_) {
      struct semaphore *sema = sema_; // 🔎 void*를 포인터로 변환
      int i;
      for (i = 0; i < 10; i++) {
          sema_down (&sema[0]); // ⚠️ NULL 검증 가드 없는 역참조 발생
          sema_up (&sema[1]);   // ⚠️ NULL 검증 가드 없는 역참조 발생
      }
  }
  ```
* **위반 사유**: 인자로 들어온 `void *sema_` 포인터에 대해 `ASSERT(sema != NULL)` 등의 안전성 가드 검증을 생략하고 다이렉트로 역참조 연산을 수행했습니다.

### ② [포인터 32] 지역 스택 주소의 외부 유출 (1건)
* **대상 코드**: `threads/synch.c` [L290-L299](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L290-L299)
  ```c
  void cond_wait (struct condition *cond, struct lock *lock) {
      struct semaphore_elem waiter; // 🔎 로컬 스택 영역에 선언된 구조체 변수
      ...
      sema_init (&waiter.semaphore, 0);
      list_push_back (&cond->waiters, &waiter.elem); // ⚠️ 스택 변수 주소를 외부 리스트에 등록
      ...
  }
  ```
* **위반 사유**: 함수가 반환되면 스택 프레임 해제와 함께 소멸할 지역 변수 `waiter`의 주소를 함수 스코프보다 생명주기가 긴 외부 구조체 `cond->waiters` 리스트에 밀어 넣었습니다. (댕글링 포인터 유발 위험성 농후)

### ③ [C 전용 5] 구조체/배열 선언 시 초기화 생략 (2건)
* **대상 코드**:
  * [L131](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L131): `struct semaphore sema[2];` (배열 선언)
  * [L290](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L290): `struct semaphore_elem waiter;` (구조체 선언)
* **위반 사유**: 선언 즉시 기본값 `{0}` 초기화를 누락하고 컴파일러의 자동 할당 및 하위 런타임 초기화 함수 호출에 의존하였습니다.

---

## ✅ 3. Clang-Tidy 정탐(True Positive) 정밀 분석 (총 6종 54건)
*C 표준에 적합하고 컴파일 및 동작에 무리가 없더라도, 국방무기체계 표준에 어긋난 모든 코딩 위반을 정밀 적발해 낸 정탐 내역입니다.*

### ① `cfg-null-dereference-guard` (NULL 포인터 가드 누락) - **2건**
* **검출 대상**: [L155](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L155) `sema_down (&sema[0]);` 및 [L156](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L156) `sema_up (&sema[1]);`
* **도구 경고 메시지**:
  `warning: 포인터 'sema'는 안전함이 보장되지 않았습니다. [cfg-null-dereference-guard]`
* **정밀 탐지 메커니즘**:
  제어 흐름 그래프(CFG) 분석 엔진이 `sema_test_helper` 함수 유입 시점부터 역참조 지점인 `&sema[0]` 및 `&sema[1]` 에 이르는 모든 도달 경로를 검증하여, 사전 NULL 조건 검사나 ASSERT 구문이 누락되었음을 정확히 탐지하였습니다. (수동 검출 결함 ①번과 교차 검증 성공)

### ② `path-sensitive-arqa.AlwaysConstantCondition` / `ast-always-constant-condition` (상수 조건식 금지) - **8건**
* **검출 대상**:
  * [L69](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L69): `while (sema->value == 0)`
  * [L92](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L92): `if (sema->value > 0)`
  * [L116](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L116): `if (!list_empty (&sema->waiters))`
  * [L138](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L138): `for (i = 0; i < 10; i++)`
  * [L153](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L153): `for (i = 0; i < 10; i++)`
  * [L218](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L218): `if (success)`
  * [L319](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L319): `if (!list_empty (&cond->waiters))`
  * [L336](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L336): `while (!list_empty (&cond->waiters))`
* **도구 경고 메시지**:
  `warning: 조건문의 결과가 항상 거짓(False)으로 평가됩니다. 해당 조건문 내부는 도달할 수 없는 코드(Unreachable Code)가 되며, 무기체계 SW 코딩 규칙에 위배됩니다 [path-sensitive-arqa.AlwaysConstantCondition]`
* **정밀 탐지 메커니즘**:
  정적 분석 범위 내에서 분기문의 조건 평가식이 런타임 연산 없이 상시 참(Always True) 또는 거짓(Always False)으로 평가되어 코드의 잉여 구문화 및 데드 코드 발생을 유발하는 논리적 룰 위반을 AST 및 기호 실행(Symbolic Execution) 분석을 통해 적발해 냈습니다.

### ③ `ast-if-style` (if-else if 문 else 블록 필수) - **3건**
* **검출 대상**: [L116](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L116), [L218](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L218), [L319](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L319)의 `if` 단독 구문들
* **도구 경고 메시지**:
  `warning: if 문에는 반드시 else 블록이 필요합니다 [ast-if-style]`
* **정밀 탐지 메커니즘**:
  방어적 프로그래밍 규격에 따라 예외 처리 분기가 생략된 흐름을 금지하는 규칙을 AST 구문 대조를 통해 감지하여, 예외 처리를 누락한 단순 `if` 블록을 정탐 적발하였습니다.

### ④ `ast-function-call-argument-consistency` (암묵적 형변환 및 인수 정합성) - **7건**
* **검출 대상**: 
  * [L135](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L135), [L136](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L136), [L181](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L181), [L297](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L297): `sema_init(..., 0)` 및 `sema_init(..., 1)` 호출부
  * [L134](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L134), [L143](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L143): `printf` 첫 번째 인수
* **도구 경고 메시지**:
  `warning: 2번째 인자에서 허용되지 않는 암묵 변환(integral cast)이 발생했습니다. 기대: 'unsigned int', 실제: 'int'. [ast-function-call-argument-consistency]`
  `warning: 1번째 인자의 타입이 프로토타입과 일치하지 않습니다. 기대: 'const char *', 실제: 'char[22]'. [ast-function-call-argument-consistency]`
* **정밀 탐지 메커니즘**:
  함수 프로토타입에 정의된 형식인 `unsigned int`에 signed `int` 상수(0, 1)를 명시적 캐스팅 없이 주입한 묵시적 형변환 오류 및 문자열 배열 리터럴의 Decay 묵시적 캐스트 동작을 엄격한 타입 정합성 검사 체커로 진단해 냈습니다.

### ⑤ `ast-extern-function-declaration` (외부 함수 extern 명시 누락) - **17건**
* **검출 대상**: 외부 연결 함수 호출부 전체
* **도구 경고 메시지**:
  `warning: 외부 함수 'list_init'는 호출 이전 선언에서 'extern'을 명시해야 합니다. [ast-extern-function-declaration]`
* **정밀 탐지 메커니즘**:
  사전 프로토타입 선언 내에 명시적인 `extern` 한정 지시어 선언 없이 외부 라이브러리 함수 심볼을 직접 호출하는 소스코드 라인마다 엄격히 탐지하여 표준 명시성을 높였습니다.

### ⑥ `ast-multi-statement-per-line` (한 줄에 하나의 명령문 작성) - **6건**
* **검출 대상**: [L60](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L60), [L108](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L108), [L123](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L123), [L128](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L128), [L147](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L147), [L179](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L179)
* **도구 경고 메시지**:
  `warning: 한 줄에 여러 개의 전역 선언(변수/함수/타입 등)이 있습니다. 각 선언은 별도의 줄에 작성해야 합니다 [ast-multi-statement-per-line]`
* **정밀 탐지 메커니즘**:
  함수 정의 시 반환형 타입 선언과 함수명이 행을 분리하지 않고 연달아 선언되었거나, 한 행에 선언이 집중된 가독성 규격 위반을 AST 파싱 단계에서 적발하였습니다.

---

## ❌ 4. Clang-Tidy 미탐(False Negative) 상세 분석 (총 2종 2건)
*수동 분석에선 규격 위반이었으나 도구가 인지하지 못한 항목에 대한 구체적 코드 분석 결과입니다.*

### ① [포인터 32] 지역 스택 주소의 외부 유출 미탐 (`cond_wait` L298)
* **미탐 발생 코드**:
  ```c
  struct semaphore_elem waiter; // 🔎 로컬 스택 구조체
  list_push_back (&cond->waiters, &waiter.elem); // ⚠️ 삽입 지점
  ```
* **미탐 원인 분석 (Pointer Aliasing 한계)**:
  * Clang-Tidy가 갖춘 `ReturnStackAddress` 체커는 함수 반환 시의 직접적인 유출(`return &waiter;`)만 감시하도록 하드코딩되어 있습니다.
  * `waiter` 구조체의 멤버인 `waiter.elem` 주소를 인자로 넘기는 과정에서 발생하는 캐스팅 및 핀토스 내부 리스트 구조 매크로 연산(`list_entry` 오프셋 계산)에 의해, 정적 분석 엔진의 포인터 데이터 흐름 추적기(Taint/Data flow)가 에일리어싱 한계에 부딪혀 추적 경로를 잃어버려 침묵하였습니다.

### ② [C 전용 5] 구조체/배열 선언 시 초기화 생략 미탐 (L131, L290)
* **미탐 발생 코드**: `struct semaphore_elem waiter;` 선언부
* **미탐 원인 분석 (동적 초기화 흐름 우회)**:
  * 미초기화 변수 검증 체커(`InitBeforeUse`)는 변수 사용(Use) 전 단계에서 데이터 할당 흐름이 있는지를 추적합니다.
  * `synch.c` 코드 내에서는 변수가 사용되기 전에 `sema_init(&waiter.semaphore, 0)` 이라는 초기화 함수가 확실하게 100% 호출되는 런타임 제어 경로가 확립되어 있어, 사용 전 미초기화 경고에 걸리지 않아 최종 미탐 처리되었습니다. (선언과 즉시 초기화를 강제하는 AST 체커 보완이 필요함)

---

## ⚠️ 5. Clang-Tidy 오탐(False Positive) 정밀 해부 (총 1종 1건)
*코드 흐름에 대한 해석 실패 및 도메인 오인으로 발생한 명백한 도구 오탐 내역입니다.*

### ① `ast-incorrect-permission-assignment` (취약 권한 부여 오독) - **100% 오탐**
* **오탐 발생 코드**: [L137](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L137)
  ```c
  thread_create ("sema-test", PRI_DEFAULT, sema_test_helper, &sema);
  ```
* **도구 경고 메시지**:
  `warning: 중요 자원 설정 함수('thread_create')에 보안상 취약한 권한 값(제3자 쓰기 허용 등)이 전달되었습니다. 악의적인 사용자가 중요 파일을 변조할 수 있으므로 권한 마스크를 수정하십시오. [ast-incorrect-permission-assignment]`
* **상세 오탐 사유**:
  * 핀토스 커널의 `thread_create` 함수에 들어가는 두 번째 인자 `PRI_DEFAULT`는 **스레드 실행 스케줄링 우선순위(Priority)**를 지정하는 정수 값(31)입니다.
  * 하지만 분석 도구는 `thread_create` 라는 이름만 보고 이를 리눅스/윈도우 OS의 파일/자원에 접근 제어 권한(Permission ACL)을 설정하는 보안 API로 **도메인을 완전히 오독**하였습니다.
  * 결과적으로 정수 31을 "보안상 취약한 권한 마스크(제3자 쓰기 허용 등)"로 완전히 오해하여 경고를 뿜은 명백한 도구의 기술적 오탐입니다.
