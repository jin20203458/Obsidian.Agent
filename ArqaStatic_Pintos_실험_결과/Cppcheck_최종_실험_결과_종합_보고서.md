# 📊 Cppcheck Pintos synch.c 최종 실험 결과 종합 보고서

본 보고서는 Pintos 커널의 핵심 동기화 파일인 `threads/synch.c`를 대상으로 **국방무기체계 코딩 표준 규격 51종(C++ 전용 제외)**을 적용하여 도출한 **수동 코드 리뷰 결과**와 **Cppcheck v2.21.0 정적 분석 결과**를 일대일로 교차 대조하여 종합한 최종 검증 분석서입니다.

---

## 1. 📊 실험 결과 요약 (Executive Summary)

* **대상 파일**: `threads/synch.c` (339 라인)
* **총 분석 경고 건수**: **32건** (Active checkers 정보 1건 제외)

| 분석 구분 | 고유 규칙 수 (종류) | 위반 검출 건수 (경고 수) | 핵심 비고 |
| :--- | :---: | :---: | :--- |
| **🔎 개발자 수동 검출** | **8종** | **57건** | 개발자가 규정 대입을 통해 규명한 골든 기준 데이터 |
| **✅ Cppcheck 정탐 (TP)** | **4종** | **30건** | 수동 검출 외 독자적 스타일/데드코드 30건 자동 포착 |
| **❌ Cppcheck 미탐 (FN)** | **8종** | **57건** | **수동 검출 위반 57건 전원 미탐 (NULL 가드, 스택 유출 등)** |
| **⚠️ Cppcheck 오탐 (FP)** | **1종** | **2건** | 리스트 팝 널 포인터 역참조 오판 (규격 외 오탐) |

* **💡 수동 검출 대비 도구 규칙 정탐률 (Recall):** **0.0%** (수동 위반 8종 중 Cppcheck 검출 성공 0종)
* **📉 실질 결함 개소 정탐율 (Warning Match):** **0.0%** (수동 위반 개소 57건 중 Cppcheck 적발 성공 0건)

---

## 🔎 2. 개발자 수동 검출 항목 명세 (Golden Reference)
*개발자가 직접 소스코드 구조를 분석하여 찾아낸 명백한 코딩 규칙 위반 사항입니다 (총 8종 57건).*

### ① [포인터 35] 포인터 참조 전 NULL 검사 누락 (2건)
* **대상 코드**: `threads/synch.c` [L147-L157](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L147-L157) (`sema_test_helper` 함수)
* **위반 사유**: 인자로 들어온 `void *sema_` 포인터에 대해 NULL 가드 검증을 생략하고 다이렉트로 역참조 연산(`&sema[0]`, `&sema[1]`)을 수행했습니다.

### ② [포인터 32] 지역 스택 주소의 외부 유출 (1건)
* **대상 코드**: `threads/synch.c` [L290-L299](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L290-L299) (`cond_wait` 함수)
* **위반 사유**: 함수가 반환되면 소멸할 지역 변수 `waiter`의 주소를 외부 구조체 `cond->waiters` 리스트에 밀어 넣었습니다.

### ③ [C-5] 구조체/배열 선언 시 초기화 생략 (2건)
* **대상 코드**: [L131](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L131) (`sema` 배열), [L290](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L290) (`waiter` 구조체)
* **위반 사유**: 선언 즉시 기본값 `{0}` 초기화를 누락하고 동적 초기화 함수 호출에 의존하였습니다.

### ④ [스타일 07, 08, 11] 및 [조건식 19], [변환 26] 표준 규격 위반 (총 52건)
* **스타일 07**: 외부 라이브러리 함수 호출 시 `extern` 명시 누락 (20건)
* **스타일 08**: 함수 반환형 개행 누락 및 한 줄 다중 선언 (5건)
* **스타일 11**: 방어적 예외 제어를 위한 `else` 블록 누락 (3건)
* **조건식 19**: 정적 흐름상 상수 조건식 방치 (16건 - 경로 분석 포함)
* **변환 26**: signed 상수를 unsigned 매개변수에 캐스트 없이 대입하거나 배열 Decay 암묵 변환 발생 (8건)

---

## ✅ 3. Cppcheck 정탐(True Positive) 정밀 분석 (총 4종 30건)
*수동 위반과 별개로, Cppcheck가 독자 탑재한 규칙 검출 체커를 통해 잡아낸 정탐 내역입니다.*

### ① `staticFunction` (static 링키지 권장) - **8건** (수동 규칙 식별자 16번 관련)
* **도구 경고**: `style: The function 'sema_init' should have static linkage since it is not used outside...`
* **정밀 분석**: 외부 파일에서 호출되지 않는 함수가 글로벌 네임스페이스를 오염시키는 것을 차단하기 위해 static 캡슐화를 명시 지적한 정탐입니다.

### ② `funcArgNamesDifferentUnnamed` (매개변수 선언-정의 불일치) - **15건** (수동 규칙 스타일 10번 관련)
* **도구 경고**: `style: inconclusive: Function 'sema_init' argument 1 names different: declaration '<unnamed>' definition 'sema'...`
* **정밀 분석**: 헤더의 프로토타입 선언 내 인자 형식(이름 없음)과 구현부 정의 인자 명칭(`sema` 등)의 스타일 정합성 불일치를 탐지한 정탐입니다.

### ③ `unusedFunction` (미사용 함수 / Dead Code) - **6건** (수동 규칙 조건식 21번 관련)
* **도구 경고**: `style: The function 'sema_self_test' is never used. [unusedFunction]`
* **정밀 분석**: 다른 링킹 대상이나 외부 호출 단위 없이 단독 선언되어 잉여 바이너리를 생성하는 도달 불가능 영역을 적발한 정탐입니다.

### ④ `constParameterPointer` (const 지향 표준 미준수) - **1건** (수동 규칙 초기화 관련)
* **도구 경고**: `style: Parameter 'lock' can be declared as pointer to const [constParameterPointer]`
* **정밀 분석**: 함수 내에서 값이 수정되지 않는 포인터 인자에 const 안전 한정자 명시를 강제한 정탐입니다.

---

## ❌ 4. Cppcheck 미탐(False Negative) 상세 분석 (총 8종 57건 전원 미탐)
*수동 분석에선 명백한 규격 위반이었으나 Cppcheck가 인지하지 못하고 놓친 57개 개소의 전원 미탐 상세 분석입니다.*

1. **[스타일 07] 외부 함수 extern 선언 누락 미탐 (20건 전체 미탐):**
   * 외부 함수가 사전 `extern` 명시 없이 다이렉트 호출되는 모든 지점에 대해 Cppcheck는 감지하지 못하고 통과시켰습니다.
2. **[스타일 11] else 블록 누락 미탐 (3건 전체 미탐):**
   * 방어적 프로그래밍을 위한 else 블록이 누락된 제어 구조를 단속하지 못했습니다.
3. **[포인터 35] NULL 가드 누락 미탐 (2건 전체 미탐):**
   * `sema_test_helper`에서 `void *sema_` 인자에 대해 NULL 가드 검증 생략 지점(L155, L156)을 정적으로 탐지하지 못했습니다.
4. **[포인터 32] 지역 스택 주소 유출 미탐 (1건 전체 미탐):**
   * 지역 변수 `waiter` 구조체의 스택 생명주기 탈출 경로(L298)를 Cppcheck 데이터 플로우가 추적해 내는 데 완전히 실패했습니다.
5. **[C-5] 구조체/배열 선언 시 초기화 생략 미탐 (2건 전체 미탐):**
   * 선언과 동시에 기본값 `{0}` 초기화를 누락한 규격 위반을 적발해 내지 못했습니다.
6. **기타 조건식 19 (16건), 변환 26 (8건), 스타일 08 (5건) ➡️ 전원 미탐 처리되었습니다.**

---

## ⚠️ 5. Cppcheck 오탐(False Positive) 정밀 해부 (총 1종 2건)
*코드 흐름에 대한 해석 실패 및 도메인 오인으로 발생한 명백한 도구 오탐 내역입니다.*

### ① `nullPointer` (리스트 팝 널 포인터 역참조 오판) - **2건**
* **대상 라인**: [L117](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L117), [L320](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L320)
* **오탐 사유**: 
  * Cppcheck는 리스트에서 맨 앞 노드를 꺼내는 `list_pop_front` 함수가 빈 리스트일 때 NULL을 반환할 수 있으므로 역참조 에러가 날 수 있다고 추론했습니다.
  * 그러나 `synch.c` 소스코드의 바로 윗 라인([L116](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L116), [L319](file:///C:/Users/user/Desktop/test_src/threads/synch.c#L319))을 보면 `if (!list_empty (...))` 로 **비어있지 않음을 100% 검증 가드한 후에만 진입하도록 제어가 확립**되어 있습니다.
  * 즉, 런타임에 절대 NULL이 반환될 수 없는 로직임에도 사전 가드 상태를 인지하지 못한 **분석기의 완전한 논리 오탐**입니다.
