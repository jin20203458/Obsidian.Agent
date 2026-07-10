# 2. ArqaStatic 미탐 및 오탐(False Positive) 상세 분석 보고서 (threads/synch.c)

본 문서는 ArqaStatic(Clang-Tidy) 정적 분석기가 `threads/synch.c` 분석 시 발생시킨 **미탐(False Negative) 사유**와 더불어, 도메인(OS 커널) 및 C 표준 문법 관점에서 잘못 진단한 **오탐(False Positive) 항목들의 명확한 기술적 근거**를 분석한 최종 보고서입니다.

---

## 1. ❌ 주요 미탐(False Negative) 사유 분석
*사람의 눈(수동 검토)으로는 명백한 규격 위반이었으나, 도구가 놓친 항목입니다.*

### ① [포인터 32] 지역 스택 주소의 외부 유출 (`cond_wait` L298)
* **현상**: `waiter` 스택 구조체의 주소가 `&cond->waiters` 전역 대기열에 삽입되었으나 경고 없음.
* **사유 (Pointer Aliasing 한계)**:
  * Clang의 `ReturnStackAddress` 체커는 함수 반환 시점의 탈출(`return &local_var;`)만 검사하도록 국한되어 있습니다.
  * 복잡한 매크로 구조(`list_entry` 등)와 `void *` 포인터 캐스팅을 거쳐 연결 리스트 내부로 주소가 은폐되어 전파될 경우, 분석 엔진이 데이터 흐름 추적을 잃어버리는 포인터 에일리어싱 한계로 인해 미탐되었습니다.

---

## 2. ⚠️ 주요 오탐(False Positive) 및 노이즈 경고 해부
*도구는 위반으로 진단(Warning)했으나, 소스코드 동작 및 표준 C 규격상 **실제 결함이 아닌 오탐**으로 판정된 항목들입니다.*

### ① `ast-incorrect-permission-assignment` (취약 권한 대입 오인) - **명백한 오탐**
* **검출 대상**: [L137](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L137) `thread_create ("sema-test", PRI_DEFAULT, sema_test_helper, &sema);`
* **오탐 근거**:
  * 분석기는 `thread_create` 함수명을 유닉스의 파일/프로세스 권한 설정 함수로 잘못 오인하여, 2번째 인자인 `PRI_DEFAULT` (우선순위 상수 값 31)를 보안 권한 취약 마스크 값(제3자 쓰기 권한 등)으로 잘못 해석했습니다.
  * 핀토스 커널의 스레드 우선순위 설정 구문이므로 **보안 권한 결함과는 완전히 무관한 100% 도구 오탐**입니다.

### ② `ast-function-call-argument-consistency` (배열-포인터 Decay 오인) - **표준 규격 오탐**
* **검출 대상**: [L134](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L134) `printf ("Testing semaphores...");` 등
* **오탐 근거**:
  * 문자열 리터럴 `char[22]`가 `const char *` 매개변수로 전달될 때 C 표준에 명시된 **배열 Decay(포인터 부식) 규칙**을 무시하고, "허용되지 않는 암묵 변환"이라며 경고를 내뿜었습니다. C 표준상 완벽하게 합법인 구문이므로 도구의 판단 오류입니다.

### ③ `path-sensitive-arqa.AlwaysConstantCondition` (멀티스레드 비동기 제어 무시) - **실무상 오탐**
* **검출 대상**: `sema_down` [L69](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L69) `while (sema->value == 0)`, `cond_wait` [L336](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L336) 등
* **오탐 근거**:
  * 정적 분석 엔진은 단일 실행 흐름(Single-thread)만을 가정하여 `sema->value` 상태를 기호 실행하므로, `sema_down` 루프 진입 시 "조건이 항상 참이 되어 무한루프에 빠지거나 항상 거짓"이라고 오판했습니다.
  * 멀티스레드 커널 환경에서는 **다른 스레드가 비동기적으로 `sema_up`을 호출해 값을 변경**하므로 무한루프가 아니며, 이는 동기화 메커니즘을 분석기가 인지하지 못해 발생한 **실무 도메인 관점의 대표적 오탐**입니다.
  * `for (i = 0; i < 10; i++)` 루프의 조건식 `i < 10`에 대해 "항상 참"이라 경고한 것 또한 루프 내부의 인크리먼트 흐름을 오독한 명백한 FP(False Positive)입니다.

### ④ `cfg-null-dereference-guard` (주소 연산에 대한 가드 요구) - **과탐/노이즈**
* **검출 대상**: [L155](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L155), [L156](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L156) `sema_down (&sema[0]);`
* **오탐 근거**:
  * `sema_test_helper`가 전달받은 인자는 메인 함수 영역에 선언된 정적 배열의 주소 `&sema`입니다. C언어 문법상 정적 스택 변수의 주소값은 물리적으로 절대 NULL이 될 수 없습니다.
  * 그럼에도 불구하고 단순 포인터 매개변수라는 이유만으로 무조건적인 NULL 가드 검사를 요구한 것은 **실무 유효성 관점에서 과탐이자 오탐**에 해당합니다.
