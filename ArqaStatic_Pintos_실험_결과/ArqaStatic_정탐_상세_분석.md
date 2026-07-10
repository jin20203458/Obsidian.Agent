# 1. ArqaStatic 정탐 (True Positive) 상세 분석 문서 (threads/synch.c)

본 문서는 ArqaStatic(Clang-Tidy) 정적 분석기가 Pintos 커널의 `threads/synch.c` 소스코드에서 국방무기체계 소프트웨어 코딩 표준을 대입하여 **실제 검출에 성공한 고유 체커(Checker ID)들의 세부 정보 및 탐지 명세**를 정밀하게 해부한 보고서입니다.

---

## 1. 📊 검출 요약 (Detection Summary)
* **총 고유 검출 규칙:** **8가지 종류**
  * **도구 자동 검출:** 7가지 체커 (`cfg-`, `ast-`, `path-sensitive-` 계열)
  * **개발자 수동 검출:** 1가지 규칙 (선언부 초기화 생략)
* **총 실질 경고 발생 건수 (Warnings):** **55건** (외부 시스템 헤더 71건 노이즈 필터링 완료)

---

## 🔍 2. 체커별 상세 검출 프로파일 및 대상 라인

### ① `cfg-null-dereference-guard` (NULL 포인터 가드 누락)
* **매핑 규칙**: [포인터 35] 포인터는 참조전에 NULL 여부를 반드시 확인하여야 한다.
* **실질 위반 건수**: **2건**
* **검출 대상 라인**:
  * [L155](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L155): `sema_down (&sema[0]);`
  * [L156](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L156): `sema_up (&sema[1]);`
* **검출 메커니즘**:
  `sema_test_helper` 함수에 매개변수로 유입된 `void *sema_` 포인터가 역참조 연산(`&sema[0]`)되기 전까지, 제어 흐름 그래프(CFG) 상에서 NULL 여부를 확인하는 분기 조건이나 `ASSERT`문이 전혀 수행되지 않았음을 탐지해 냈습니다.

---

### ② `path-sensitive-arqa.AlwaysConstantCondition` / `ast-always-constant-condition` (상수 조건식 금지)
* **매핑 규칙**: [조건식 19] 조건문의 결과가 항상 True 혹은 False라면 이는 조건문으로 작성해서는 안된다.
* **실질 위반 건수**: **8건**
* **검출 대상 라인**:
  * [L69](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L69): `while (sema->value == 0)` (참/거짓 이중 경로 분석)
  * [L92](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L92): `if (sema->value > 0)` (참/거짓 이중 경로 분석)
  * [L116](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L116): `if (!list_empty (&sema->waiters))` (참/거짓 이중 경로 분석)
  * [L138](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L138): `for (i = 0; i < 10; i++)` (AST 상시 참 분석)
  * [L153](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L153): `for (i = 0; i < 10; i++)` (AST 상시 참 분석)
  * [L218](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L218): `if (success)` (경로 민감 분석)
  * [L319](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L319): `if (!list_empty (&cond->waiters))` (경로 민감 분석)
  * [L336](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L336): `while (!list_empty (&cond->waiters))` (경로 민감 분석)
* **검출 메커니즘**:
  기호 실행(Symbolic Execution) 분석 엔진이 스레드 동기화 상태 변화에 따른 실행 시나리오를 가상 실행(Path-sensitive) 추적하여, 특정 프로그램 상태에서 해당 조건문 결과가 항상 참 또는 거짓으로 편향되어 하위 블록이 도달 불가능(Dead Code)해지는 기하학적 논리 오류를 규명했습니다.

---

### ③ `ast-if-style` (if-else if 문 else 블록 필수)
* **매핑 규칙**: [스타일 11] if - else if 문은 else 문도 반드시 포함 시킨다 (방어적 프로그래밍).
* **실질 위반 건수**: **3건**
* **검출 대상 라인**:
  * [L116](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L116): `if (!list_empty (&sema->waiters))`
  * [L218](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L218): `if (success)`
  * [L319](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L319): `if (!list_empty (&cond->waiters))`
* **검출 메커니즘**:
  구문 트리(AST) 분석을 통해 `if` 블록 뒤에 예외 상황을 격리/로깅하기 위한 예외 분기 처리인 `else` 선언 구문이 완전히 누락된 구조를 즉각 단속했습니다.

---

### ④ `ast-function-call-argument-consistency` (형변환 및 인수 정합성 에러)
* **매핑 규칙**: [변환 26] 음수 값을 unsigned 형식으로 변환 금지 및 함수 타입 정합성 검사.
* **실질 위반 건수**: **7건**
* **검출 대상 라인**:
  * [L134](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L134): `printf ("Testing semaphores...");`
  * [L135](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L135): `sema_init (&sema[0], 0);`
  * [L136](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L136): `sema_init (&sema[1], 0);`
  * [L137](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L137): `thread_create ("sema-test", PRI_DEFAULT, ...);` (2건 복수 발생)
  * [L143](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L143): `printf ("done.\n");`
  * [L181](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L181): `sema_init (&lock->semaphore, 1);`
  * [L297](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L297): `sema_init (&waiter.semaphore, 0);`
* **검출 메커니즘**:
  `sema_init` 함수의 2번째 프로토타입 정의 형식(`unsigned value`)과 실제 주입된 signed `int` 상수(0, 1)의 불일치로 유도되는 묵시적 캐스트 에러 및 타입 충돌을 정확히 지적하였습니다.

---

### ⑤ `ast-extern-function-declaration` (외부 함수 extern 명시 누락)
* **매핑 규칙**: [스타일 07] 외부 함수 사용 시 이를 명시하고 사용해야 한다.
* **실질 위반 건수**: **17건** (코드 내 반복 호출에 비례)
* **검출 대상 라인**: L50, L68, L71, L72, L75, L91, L99, L115, L117, L120, L137, L200, L219, L246, L264, L298, L338 등
* **검출 메커니즘**:
  외부 파일에 정의되어 연결되는 라이브러리 함수 심볼을 사용할 때, 사전 프로토타입 선언 내에 명시적인 `extern` 한정 지시어가 수반되지 않은 채 다이렉트 링킹 호출을 수행하는 모든 분기를 검출했습니다.

---

### ⑥ `ast-multi-statement-per-line` (한 줄에 하나의 명령문 작성)
* **매핑 규칙**: [스타일 08] 한 줄에 하나의 명령문을 사용한다.
* **실질 위반 건수**: **6건**
* **검출 대상 라인**:
  * [L60](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L60), [L108](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L108), [L128](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L128), [L147](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L147): 함수 반환 타입(`void` 등)과 함수 본 명칭이 개행되지 않고 한 행에 서술된 코딩 스타일 위반.
  * [L123](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L123): `static void sema_test_helper (void *sema_);` 선언부 줄바꿈 미준수.
  * [L179](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L179): `void lock_init (struct lock *lock)` 선언부 줄바꿈 미준수.

---

### ⑦ `ast-incorrect-permission-assignment` (취약한 자원 권한 부여 금지)
* **매핑 규칙**: [보안] 중요 시스템 자원 접근 권한 부여 표준 준수.
* **실질 위반 건수**: **1건**
* **검출 대상 라인**:
  * [L137](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L137): `thread_create ("sema-test", PRI_DEFAULT, sema_test_helper, &sema);`
* **검출 메커니즘**:
  중요 시스템 스레드 자원을 동적 생성(thread_create)할 때, 외부 또는 상위로부터 과도하거나 취약한 우선순위 권한 마스크가 가용되어 다른 악의적 코드에 의해 권한 상승 및 가로채기가 발생할 가능성을 AST 단계에서 경고했습니다.

---

### 🔎 ⑧ 수동 검출: `ast-aggregate-init-style` (구조체 선언부 초기화 생략)
* **매핑 규칙**: [C 전용 5] 구조체/배열 선언 시 `{}` 기본 초기화 의무 준수.
* **실질 위반 건수**: **2건** (도구 미탐 / 개발자 수동 정탐)
* **검출 대상 라인**:
  * [L131](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L131): `struct semaphore sema[2];` (배열 선언)
  * [L290](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L290): `struct semaphore_elem waiter;` (구조체 선언)
* **내용**:
  해당 변수들은 선언 즉시 `{0}` 기본 초기화 구문이 누락되었으나, 사용 전에 런타임 초기화 함수를 100% 호출하므로 도구의 미초기화 경고에선 탈출했습니다. 그러나 **"선언부 즉시 초기화 규격"**을 위배한 엄연한 코딩 규칙 위반 사항입니다.
