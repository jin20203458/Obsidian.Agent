# 📊 Cppcheck Pintos synch.c 최종 실험 결과 종합 보고서

본 보고서는 Pintos 커널의 핵심 동기화 파일인 `threads/synch.c`를 대상으로 **국방무기체계 코딩 표준 규격 51종(C++ 전용 제외)**을 적용하여 도출한 **수동 코드 리뷰 결과**와 **Cppcheck v2.21.0 정적 분석 결과**를 일대일로 교차 대조하여 종합한 최종 검증 분석서입니다.

---

## 1. 📊 실험 결과 요약 (Executive Summary)

* **대상 파일**: `threads/synch.c` (339 라인)
* **총 분석 경고 건수**: **32건** (Active checkers 정보 1건 제외)

| 분석 구분 | 고유 규칙 수 (종류) | 위반 검출 건수 (경고 수) | 핵심 비고 |
| :--- | :---: | :---: | :--- |
| **🔎 개발자 수동 검출** | **3종** | **4건** | 개발자가 전수 눈으로 찾아낸 골든 레퍼런스 |
| **✅ Cppcheck 정탐 (TP)** | **4종** | **30건** | 매개변수 일치, static 링크 권장, 데드 코드 등 검출 |
| **❌ Cppcheck 미탐 (FN)** | **3종** | **4건** | **수동 검출 결함 전원 미탐 (NULL 가드, 스택 유출 등)** |
| **⚠️ Cppcheck 오탐 (FP)** | **1종** | **2건** | 리스트 팝 널 포인터 역참조 오판 |

---

## 🔎 2. 개발자 수동 결함에 대한 Cppcheck의 탐지 실패 (전원 미탐)
*개발자가 눈으로 규명한 실제 synch.c의 보안/메모리 규격 결함 4건을 Cppcheck는 단 한 건도 검출해 내지 못했습니다 (미탐률 100%).*

1. **[포인터 35] 포인터 참조 전 NULL 검사 누락 (L155-L156 `sema_test_helper`) ➡️ ❌ 미탐 (Silent)**
   * `void *sema_` 인자가 널 검사 없이 즉시 역참조 연산에 가용되는 패턴을 인지하지 못하고 아무런 경고도 띄우지 못했습니다.
2. **[포인터 32] 지역 스택 주소의 외부 유출 (`cond_wait` L298) ➡️ ❌ 미탐 (Silent)**
   * 지역 변수 `waiter` 구조체의 스택 생명주기 탈출 경로를 Cppcheck 데이터 플로우가 추적해 내는 데 완전히 실패했습니다.
3. **[C 전용 5] 구조체/배열 선언 시 초기화 생략 (L131, L290) ➡️ ❌ 미탐 (Silent)**
   * 미초기화 변수 선언 표준 위반에 대해 경고(`uninitvar`)를 발생시키지 않았습니다.

---

## ✅ 3. Cppcheck 정탐(True Positive) 정밀 분석 (총 4종 30건)
*일반 표준 적합 여부와 무관하게, 국방무기체계 코딩 표준 규격 관점에서 적발한 Cppcheck의 정탐 내역입니다.*

### ① `staticFunction` (static 링키지 권장) - **8건**
* **대상 라인**: `synch.c` L45 (`sema_init`), L61 (`sema_down`), L84 (`sema_try_down`), L109 (`sema_up`) 등 8개 함수
* **도구 경고**: `style: The function 'sema_init' should have static linkage since it is not used outside...`
* **규격 매핑**: [식별자 16] 유일한 글로벌 식별자명 사용 권장.
* **사유**: 외부 파일에서 호출되지 않는 함수가 글로벌 네임스페이스를 오염시키는 것을 차단하기 위해 `static` 캡슐화를 규격대로 명시 지적한 정탐입니다.

### ② `funcArgNamesDifferentUnnamed` (매개변수 선언-정의 불일치) - **15건**
* **대상 라인**: `synch.c` L45, L61, L84, L109, L176, L193, L210, L229, L242, L260, L288, L312, L331 등 15건
* **도구 경고**: `style: inconclusive: Function 'sema_init' argument 1 names different: declaration '<unnamed>' definition 'sema'...`
* **규격 매핑**: [스타일 10] 선언과 구현의 구조적 명시성 및 정합성 일치.
* **사유**: 헤더의 프로토타입 선언 내 인자 형식(이름 없음)과 구현부 정의 인자 명칭(`sema` 등)의 스타일 정합성 불일치를 탐지한 정탐입니다.

### ③ `unusedFunction` (미사용 함수 / Dead Code) - **6건**
* **대상 라인**: `synch.c` L86 (`sema_self_test`), L90 (`lock_init`), L93 (`lock_try_acquire`) 등 6개 함수
* **도구 경고**: `style: The function 'sema_self_test' is never used. [unusedFunction]`
* **규격 매핑**: [조건식 21] 수행되지 않는 소스코드(Dead Code)를 작성하지 말아야 한다.
* **사유**: 다른 링킹 대상이나 외부 호출 단위 없이 단독 선언되어 잉여 바이너리를 생성하는 도달 불가능 영역을 정확히 적발한 정탐입니다.

---

## ⚠️ 4. Cppcheck 오탐(False Positive) 정밀 분석 (총 1종 2건)
*코드 실행 제어 논리를 분석기가 완전하게 독해하지 못하여 띄운 대표적인 도구 오탐 내역입니다.*

### ① `nullPointer` (리스트 팝 널 포인터 역참조 오판) - **2건**
* **대상 라인**:
  * [L117](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L117): `thread_unblock (list_entry (list_pop_front (&sema->waiters), ...))`
  * [L320](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L320): `sema_up (&list_entry (list_pop_front (&cond->waiters), ...))`
* **도구 경고**: `error: Null pointer dereference: (struct thread*)0 [nullPointer]`
* **오탐 사유**: 
  * Cppcheck는 리스트에서 맨 앞 노드를 꺼내는 `list_pop_front` 함수가 빈 리스트일 때 NULL을 반환하므로 역참조 에러가 날 수 있다고 추론했습니다.
  * 그러나 `synch.c` 소스코드의 바로 윗 라인([L116](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L116), [L319](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L319))을 보면 `if (!list_empty (...))` 로 **비어있지 않음을 100% 검증 가드한 후에만 진입하도록 제어가 확립**되어 있습니다.
  * 즉, 런타임에 절대 NULL이 반환될 수 없는 로직임에도 사전 가드 상태를 인지하지 못한 **분석기의 완벽한 논리 오탐**입니다.

---

## 💡 5. Clang-Tidy vs Cppcheck 비교 분석 요약

* **결함 탐지력 차이 (Clang-Tidy 압승):**
  * Cppcheck는 실무 개발자가 눈으로 발굴한 핵심 메모리 결함(NULL 가드 누락 L155, 스택 주소 유출 L298)을 완전히 미탐( FN 100% )하였습니다.
  * 반면 Clang-Tidy(ArqaStatic)는 `cfg-null-dereference-guard` 및 `path-sensitive-` 기호 실행 엔진을 기반으로 해당 핵심 결함들을 매우 명확하게 적발해 내어, 정적 분석 설계 신뢰성 면에서 Clang-Tidy가 압도적 우위에 있음을 증명했습니다.
