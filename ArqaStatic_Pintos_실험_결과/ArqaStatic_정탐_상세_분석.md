# 1. ArqaStatic 정탐 (True Positive) 상세 분석 문서 (threads/synch.c)

본 문서는 ArqaStatic(Clang-Tidy) 정적 분석기가 Pintos 커널의 `threads/synch.c` 소스코드에서 무기체계 소프트웨어 코딩 규칙의 결함을 **정확하게 진단(정탐)**해낸 항목들을 정리한 자료입니다.

기대했던 경고(Expected)와 실제 출력된 핵심 경고(Actual) 및 매핑 체커 이름을 테이블 형태로 대조하여 검증 신뢰도를 확보했습니다.

| 규칙 번호 | 카테고리 | 규칙 핵심 요약 | 매핑 체커 이름 (Clang-Tidy) | 실제 검출 경고 메시지 | 판정 |
| :---: | :--- | :--- | :--- | :--- | :---: |
| **07** | Common_Style | 외부 함수 사용 시 이를 명시하고 사용해야 한다. | `ast-extern-function-declaration` | ⚠️ 외부 함수 'list_init'는 호출 이전 선언에서 'extern'을 명시해야 합니다. | ✅ 정탐 |
| **11** | Common_Style | 한 줄에 하나의 명령문을 사용한다. | `ast-multi-statement-per-line` | ⚠️ 한 줄에 여러 개의 전역 선언(변수/함수/타입 등)이 있습니다. | ✅ 정탐 |
| **12** | Common_Style | if - else if 문은 else 문도 반드시 포함 시킨다. | `ast-if-style` | ⚠️ if 문에는 반드시 else 블록이 필요합니다 | ✅ 정탐 |
| **21** | Common_Cond | 조건문의 결과가 항상 True 혹은 False라면 이는 조건문으로 작성해서는 안된다. | `ast-always-constant-condition` | ⚠️ 상위 조건으로 인해 이 조건은 항상 참입니다. | ✅ 정탐 |
| **21** | Common_Cond | 조건문의 결과가 항상 True 혹은 False라면 이는 조건문으로 작성해서는 안된다. (경로 민감) | `path-sensitive-arqa.AlwaysConstantCondition` | ⚠️ 조건문의 결과가 항상 거짓(False)으로 평가됩니다. 해당 조건문 내부는 도달할 수 없는 코드(Unreachable Code)가 됩니다. | ✅ 정탐 |
| **29** | Common_Conv | 음수 값을 unsigned type으로 변환해서는 안된다. (함수 파라미터 캐스트 포함) | `ast-function-call-argument-consistency` | ⚠️ 2번째 인자에서 허용되지 않는 암묵 변환(integral cast)이 발생했습니다. | ✅ 정탐 |
| **33** | Common_PtrArray | 포인터는 참조전에 NULL 여부를 반드시 확인하여야 한다. | `cfg-null-dereference-guard` | ⚠️ 포인터 'sema'는 안전함이 보장되지 않았습니다. | ✅ 정탐 |
| **C-5** | C_Specific | 구조체/배열의 초기화 시 default 초기화 값(0)을 제외하고, 구조에 맞게 ‘{}’를 사용하여 초기화 해야 한다. | `ast-aggregate-init-style` | *(수동 정탐)* 구조체/배열 선언 즉시 `{0}` 초기화 누락 확인. | 🔎 수동정탐 |
