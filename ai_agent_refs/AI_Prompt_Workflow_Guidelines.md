---
type: guideline
title: AI Prompt Workflow Guidelines
tags: [ai, prompt, workflow, guidelines]
related:
  - ../README.md
  - ./AI_Prompt_Engineering_Guidelines.md
status: stable
---
# AI Prompt Workflow Guidelines
> **부제**: AI 프롬프트 유지보수 및 수명 주기 워크플로우 지침

본 문서는 프로젝트 내에서 AI 프롬프트를 신규로 작성하거나 기존 프롬프트를 유지보수할 때 적용해야 할 **공식 프롬프트 엔지니어링 워크플로우**를 정의합니다.

최고의 품질과 안정성을 지닌 프롬프트를 효율적으로 뽑아내기 위해, 기존에 작성된 두 가지 핵심 가이드라인 문서([AI_Prompt_Engineering_Guidelines.md](./AI_Prompt_Engineering_Guidelines.md) 및 [AI_Prompt_AntiPatterns_Guidelines.md](./AI_Prompt_AntiPatterns_Guidelines.md))를 결합한 **"3단계 프롬프트 조립법 (Build - Audit - Test)"**을 따를 것을 권장합니다.

---

##  1단계: 마인드셋 세팅 및 뼈대 잡기 (Engineering 주도)

처음 백지상태에서 프롬프트를 기획하고 작성할 때는 **[AI_Prompt_Engineering_Guidelines.md](./AI_Prompt_Engineering_Guidelines.md)**를 설계도처럼 활용합니다.

- **[행동 1] 구조화:** `<system_directive>`, `<role>`, `<task>`, `<rules>`, `<output_format>` 등의 명확한 XML 태그로 프롬프트의 뼈대를 만듭니다. 마크다운 헤더(`#`)는 인젝션에 취약하므로 시스템 경계선으로 사용하지 않습니다.
- **[행동 2] 코드 내 하드코딩 표준:** 프로그래밍 언어의 특성에 맞춰 이스케이프가 필요 없는 문자열 포맷(C#의 원시 문자열 `$$"""..."""`, C++의 raw string literal `R"(...)"` 등)을 적극 활용하여 가독성을 높이고 인코딩 실수를 예방합니다.
- **[행동 3] 스키마 정의 최적화:** 복잡한 구조를 출력해야 할 경우, 토큰 낭비가 심한 Raw JSON Schema 대신 **TypeScript 인터페이스**로 구조를 정의하고 1~2개의 간단한 예시를 추가합니다.

---

##  2단계: 함정 피하기 및 디버깅 (Anti-Patterns 주도)

초안이 작성되면 완성이라 생각하지 말고, **[AI_Prompt_AntiPatterns_Guidelines.md](./AI_Prompt_AntiPatterns_Guidelines.md)**를 안전 점검표 삼아 프롬프트를 깐깐하게 검수합니다.

- **[체크 1] 불필요한 지시 차단:** 무의식적으로 "단계별로 생각해(CoT)", "제발 실수하지 마" 같은 문구를 넣지 않았는지 확인하고 과감히 삭제합니다.
- **[체크 2] 부정 지시문 제거 (흰 곰 효과 방지):** "~하지 마라"라는 부정 지시문이 발견되면, 반드시 "오직 ~만 해라"라는 긍정 지시문으로 변환합니다.
- **[체크 3] 핵심 지시 위치 점검 (Lost in the middle 방지):** 절대 어겨선 안 되는 제약사항이나 최종 출력 형태 지시가 프롬프트 중간에 파묻혀 있지 않은지 점검합니다. 핵심 지시는 최상단이나 가장 하단의 `<final_instruction>` 태그 내로 이동시킵니다.

---

##  3단계: 테스트 후 교차 보완 (실전 단계)

프롬프트를 실제 LLM(Gemini 등)에 돌려본 후, 결과에 문제가 발생했다면 증상에 따라 두 문서를 교차하여 처방을 내립니다.

### 증상 A: 모델이 지시를 무시하거나 환각(Hallucination)을 보일 때
> **처방 가이드:** [AI_Prompt_AntiPatterns_Guidelines.md](./AI_Prompt_AntiPatterns_Guidelines.md) 가이드라인을 다시 확인합니다.
- "내가 혹시 부정 지시문(~하지 마)을 남겨 두었나?"
- "가장 중요한 지시가 긴 컨텍스트 윈도우 중간에 끼어 있지는 않은가?"

### 증상 B: 출력 포맷(JSON 등)이 깨지거나 일관성이 없을 때
> **처방 가이드:** [AI_Prompt_Engineering_Guidelines.md](./AI_Prompt_Engineering_Guidelines.md) 가이드라인을 다시 확인합니다.
- "내가 제공한 Few-shot(예시)의 질이 너무 떨어지거나 개수가 부족한가?"
- "유저 데이터와 시스템 규칙 격리가 허술해서 모델이 사용자의 말을 시스템 명령으로 착각했는가?"

---

**관련 문서 목록:**
- 설계 지침: [AI_Prompt_Engineering_Guidelines.md](./AI_Prompt_Engineering_Guidelines.md)
- 위험 회피: [AI_Prompt_AntiPatterns_Guidelines.md](./AI_Prompt_AntiPatterns_Guidelines.md)
