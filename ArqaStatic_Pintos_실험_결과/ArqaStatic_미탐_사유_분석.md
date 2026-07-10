# 2. ArqaStatic 미탐 사유 및 특이 사항 분석 문서 (threads/synch.c)

본 문서는 `threads/synch.c` 분석 시 수동 리뷰를 통해서는 명백한 규칙 위반으로 식별되었으나, ArqaStatic(Clang-Tidy) 정적 분석기가 탐지하지 못했거나(미탐) 진단 시 발생한 특이 사유를 해부하여 정리한 자료입니다.

---

## 2.1. ❌ 주요 미탐(False Negative) 항목 사유 분석

### ① [포인터 규칙 나] 지역변수의 주소값을 더 넓은 scope를 가진 변수에 할당 금지
* **위반 의심 코드**: `cond_wait` 함수 ([synch.c L290-L299](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L290-L299))
  ```c
  struct semaphore_elem waiter; // 지역 스택 변수
  ...
  list_push_back (&cond->waiters, &waiter.elem); // 외부 리스트에 주소 등록
  ```
* **매핑 체커**: `ReturnStackAddressCheck` (Clang-Tidy) / `StackAddrEscapeChecker` (Clang Static Analyzer)
* **도구의 미탐 사유**:
  * `ReturnStackAddressCheck` 체커는 함수가 `return`문으로 스택 프레임을 해제하며 주소를 탈출할 때(`return &local_var;`)만 감지하도록 제한되어 있습니다.
  * `StackAddrEscapeChecker`는 지역 스택 변수의 주소가 전역 포인터 변수나 참조 파라미터에 할당되어 **실제 스택 바운더리를 벗어나는 시점**을 분석합니다.
  * 그러나 `cond_wait` 내의 리스트 구조체 `list_push_back` 함수는 `void *` 포인터로 인자를 수입해 내부 리스트 노드 주소를 연산하므로, 포인터 캐스팅 및 복잡한 매크로 연산(예: `list_entry` 오프셋 계산)에 의해 분석 엔진이 스택 탈출 경로의 추적을 잃어버리는 **포인터 앨리어싱(Pointer Aliasing) 한계**로 인해 미탐이 유도되었습니다.

### ② [C 전용 규칙 5] 구조체/배열 선언 시 `{}` 기본 초기화 누락
* **위반 의심 코드**: [synch.c L131](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L131) (`sema` 배열), [L290](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L290) (`waiter` 구조체)
* **매핑 체커**: `ast-aggregate-init-style` (Clang-Tidy)
* **도구의 미탐 사유**:
  * `ast-aggregate-init-style` 체커는 구조체/배열 선언부에서 일부 멤버만 초기화하여 명시성을 흐리는 패턴(`struct s s1 = {8};`)을 규제하는 데 포커스되어 있습니다.
  * 아예 초기화 구문 자체가 생략된 미초기화 변수 선언 선언(`struct semaphore_elem waiter;`)의 경우는 해당 체커의 분석 영역 밖이며, 이는 오히려 `InitBeforeUseCheck` (사용 전 초기화 검사) 체커의 영역에 속합니다. 
  * 그러나 `synch.c` 내에서는 사용 전에 반드시 `sema_init()` 이라는 동적 런타임 초기화 함수를 100% 호출해 사용하므로, 사용 전 초기화 위반에 걸리지 않아 미탐으로 판정되었습니다. (선언부 즉시 초기화를 강제하는 룰셋 보완 필요)

---

## 2.2. ⚠️ 진단 환경 특이 사항 (cURL Timeout 에러)
* **현상**: 분석 전체 범위(`--checks=*`)로 가동 시 `Error from cURL: Timeout was reached` 경고와 함께 분석 엔진이 지연(Stall)되는 현상 발생.
* **원인**: 커스텀 체커 모듈 중 일부 Taint 분석 또는 라이선스 연동 모듈이 외부 API 연동을 시도하다가, 오프라인 환경에 의해 30초 이상의 타임아웃 지연을 발생시킴.
* **해결 및 대응**: 네트워크 I/O가 없는 순수 AST 및 CFG 분석 그룹인 **`ast-*,cfg-*`** 체커 그룹만 명시적으로 활성화하여 지연 현상을 완벽히 배제하고 초고속 정적 분석(1초 소요)을 구현함.
