# 3. ArqaStatic 규칙 검출 정량적 통계 데이터 (Pintos threads/synch.c)

본 문서는 ArqaStatic(Clang-Tidy) 정적 분석기가 Pintos 커널의 핵심 동기화 모듈인 `threads/synch.c` 본체 코드를 대상으로 수행한 무기체계 소프트웨어 코딩 규칙 진단 성능을 **수치화(Metrics)하여 정리한 통계 자료**입니다.

*본 통계는 외부 시스템 헤더(71건)를 제외하고 오직 `synch.c` 사용자 소스 내에서 발생한 순수 경고 55건을 바탕으로, 국방무기체계 코딩 표준 준수성 관점에서 엄격하게 재판정하여 산출하였습니다.*

## 3.1. 전역 지표 요약 (Global Metrics)

| 지표 (Metrics) | 수치 (Value) | 비고 |
| :--- | :---: | :--- |
| **총 대상 규칙 (Total Rules)** | 51 | C++ 전용 규칙(15종)을 제외한 C/C++ 공통 및 C 전용 규칙 |
| **실제 분석 대상 파일** | 1 | `threads/synch.c` (339 라인) |
| | | |
| **✅ 정탐 (True Positives)** | **54** | 국방무기체계 코딩 규정을 어겨 적발된 실제 개별 경고 수 (형식 변환, else 블록 누락 등 규격 위반 일체 포함) |
| **❌ 오탐 (False Positives)** | **1** | 함수 도메인 인자 인지 오류에 기인한 순수 정적 오탐 (`thread_create` 권한 오독) |
| **🔎 수동 검출 규칙 종류** | **3종** | 스택 주소 유출, NULL 가드 생략, 선언부 초기화 생략 |
| **📉 검출된 분석 경고 총합 (Warnings)** | **55** | 외부 시스템 헤더(71건)를 차단하고 **`synch.c` 내부에서만 검출된 실질 경고 총합** |

---

## 3.2. 룰셋 카테고리별 검출 정량 데이터 (Category Metrics)

| 카테고리 | 대상 규칙 수 | 검출(Tool) | 검출(Manual) | 대표 매핑 체커 이름 |
| :--- | :---: | :---: | :---: | :--- |
| Common_Style | 11 | 3 | 0 | `ast-extern-function-declaration`<br>`ast-multi-statement-per-line`<br>`ast-if-style` |
| Common_Init | 3 | 0 | 0 | - |
| Common_Ident | 4 | 0 | 0 | - |
| Common_Cond | 5 | 1 | 0 | `ast-always-constant-condition`<br>`path-sensitive-arqa.AlwaysConstantCondition` |
| Common_Conv | 8 | 1 | 0 | `ast-function-call-argument-consistency` |
| Common_PtrArray | 5 | 1 | 0 | `cfg-null-dereference-guard` |
| Common_Expr | 9 | 0 | 0 | - |
| C_Specific | 5 | 0 | 1 | `ast-aggregate-init-style` (수동 탐지) |

---

## 3.3. 결론적 정량 평가
* 본 실험을 통해 ArqaStatic(Clang-Tidy)은 시스템 헤더 경고를 배제한 `synch.c` 본체 소스 내에서 **총 7종의 고유 무기체계 소프트웨어 코딩 규칙 위반(54건의 개별 정탐 경고)을 규명**하는 데 성공하였습니다.
* C/C++ 표준을 충족하거나 런타임에 정상 컴파일되는 코드일지라도, 국방무기체계 코딩 표준의 규격(else 블록 강제, 묵시적 캐스트 금지, extern 명시 등)을 한 치의 융통성도 없이 **엄격하게 대입하여 정탐으로 식별**해 내는 정밀 검출력을 증명하였습니다.
* 유일한 오탐 1건은 `thread_create` 함수의 우선순위(Priority) 매개변수를 파일 접근 권한 ACL 할당으로 오독한 컴파일러의 해석적 한계에 기인합니다.
