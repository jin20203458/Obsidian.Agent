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

## 2.2. ⚠️ 진단 환경 특이 사항 및 cURL 타임아웃 극복 런북

### 1. cURL 타임아웃 현상의 전말
* **원인**: `path-sensitive-arqa.*` 체커 그룹 중 일부 분석 모듈이 라이선스/동기화 등을 위해 외부 정책 서버와 웹 통신(cURL)을 강제 시도하여 타임아웃(Timeout reached) 지연 및 분석 정지 현상이 발생함.
* **진범 식별**: 정밀 로깅 추적을 통해 cURL 원격 연동을 무조건 호출하여 지연을 발생시키는 주범이 **`path-sensitive-arqa.DHTestChecker3`** 체커 단 1개인 것으로 판명됨.

### 2. TaintConfig 로컬 연동 및 타임아웃 해결책 (Solution)
* **단계 1**: `C:\Users\user\AppData\Roaming\ArqaStatic\TaintConfig.yaml` 경로의 로컬 정책 파일을 분석 설정 파일에 매핑.
* **단계 2**: Clang-Tidy 룰셋 정의에서 원격 통신 주범인 `DHTestChecker3` 만 명시적으로 제외(`-path-sensitive-arqa.DHTestChecker3`)하고 가동.
* **최종 설정 예시 (`.clang-tidy`):**
  ```yaml
  Checks: 'ast-*,cfg-*,lex-*,path-sensitive-core.*,path-sensitive-unix.*,path-sensitive-arqa.*,-path-sensitive-arqa.DHTestChecker3'
  CheckOptions:
    'path-sensitive-arqa.TaintPropagation:Config': 'C:/Users/user/AppData/Roaming/ArqaStatic/TaintConfig.yaml'
  ```
* **결과**: 타임아웃 장애가 완벽하게 배제되어 단 1초 만에 실행이 성공하였으며, 25개의 Taint 규칙을 정상 로드하고 `path-sensitive-arqa.AlwaysConstantCondition` 등 타 14개 경로 민감 경고를 성공적으로 추가 검출함.
