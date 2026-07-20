---
type: guideline
title: Agent Troubleshooting Guidelines
tags: [ai, agent, troubleshooting, circuit-breaker, handover]
related:
  - ../README.md
  - ./AI_Project_Integration_Guidelines.md
  - ./Agent_QA_Testing_Guidelines.md
status: stable
---
# Agent Troubleshooting Guidelines

> **부제**: 에이전트 무한 루프 방지 및 인간 이관(Hand-over) 지침

본 문서는 `Obsidian.Agent` 환경 내에서 코딩 및 문제 해결을 수행하는 자율형 AI 에이전트가 예기치 않은 오류나 컴파일 실패 상황에 직면했을 때 반드시 준수해야 하는 **위기 관리 표준**을 정의합니다. (기반 논문: *Hou et al., 2026 "When Agents Do Not Stop"*)

---

## 1. 3-Strike Out 서킷 브레이커 (Circuit Breaker)

에이전트는 "이번에는 될 것이다"라는 근거 없는 낙관으로 토큰(Context Window)을 탕진하는 **무한 에이전트 루프(Infinite Agentic Loop)**에 빠지는 것을 엄격히 경계해야 합니다.

### [원칙] 3회 이상 동일 실패 시 즉각 중지
* **판단 기준**: 동일한 파일에서 동일한 목적의 컴파일 에러나 테스트 실패를 고치기 위해 **3번의 도구 호출(Tool Call) 및 코드 수정**을 시도했음에도 문제가 해결되지 않았다면, 에이전트는 자신의 추론 한계에 도달했음을 자각해야 합니다.
* **행동 강령**: 3회째 실패가 확인되는 즉시, 에이전트는 어떠한 추가 수정 시도도 하지 않고 **코딩 작업을 강제 중단(서킷 브레이커 발동)**해야 합니다.

---

## 2. 훼손된 코드의 롤백 (Rollback)

### [원칙] 망가진 상태로 방치 금지
에이전트가 버그를 해결하려다 오히려 코드를 더 엉망으로 만들었거나, 원래 없던 의존성 에러까지 발생시켰다면, **작업을 중단하기 전에 반드시 코드를 원상 복구(Rollback)**해야 합니다.
* 깃(Git) 브랜치나 임시 저장 상태가 없다면, `multi_replace_file_content` 등의 에디팅 도구를 사용해 본인이 변경하기 직전의 코드로 되돌려 놓아야 합니다.
* 인간 개발자가 엉망이 된 코드를 직접 수습하는 비용을 최소화해야 합니다.

---

## 3. 구조화된 휴먼 인수인계 (Human-in-the-Loop Hand-over)

작업을 중단한 에이전트는 침묵하거나 단순히 "실패했습니다"라고 보고해서는 안 됩니다. 인간 개발자가 상황을 즉시 파악하고 개입할 수 있도록 다음의 프로세스를 거칩니다.

### [행동 1] 트러블슈팅 로그 작성
[AI_Project_Integration_Guidelines.md](./AI_Project_Integration_Guidelines.md)에 명시된 규칙에 따라, 해당 프로젝트의 트러블슈팅 폴더(`troubleshooting/<project_name>.md`)에 에러 상황을 박제합니다.
단, 해결하지 못했으므로 템플릿의 `3. 해결책` 항목을 **`3. 미해결 원인 분석 및 결정 요청`**으로 변경하여 작성합니다.

**기록 예시:**
```markdown
## 2026-07-15: [Unresolved] LLVM Build CMake Error
### 1. 현상 (Symptom)
- CMake Cache 충돌 에러가 발생하여 빌드가 진행되지 않음.
### 2. 시도 및 원인 (Root Cause Analysis)
- `CMakeCache.txt` 삭제 및 3회 재시도를 하였으나, 의존성 경로 문제로 인해 동일한 에러가 계속 발생함.
### 3. 미해결 원인 분석 및 결정 요청 (Hand-over)
- (1) 루트 CMakeLists.txt의 경로 매핑을 강제로 수정할지, (2) 임시 빌드 디렉토리를 완전히 날리고 클린 빌드를 진행할지 인간 개발자의 결정이 필요합니다.
```

### [행동 2] 명시적 지시 요청
로그 작성이 완료되면, 사용자 채팅창에 다음과 같이 명확히 보고하고 다음 지시를 대기합니다.
> "해당 에러를 3회 시도했으나 해결되지 않아 무한 루프 방지를 위해 서킷 브레이커를 발동하고 코드를 롤백했습니다. 상세한 실패 원인은 트러블슈팅 로그에 기록했습니다. A 방식과 B 방식 중 어떻게 진행할지 결정해 주시거나 다른 힌트를 주시겠습니까?"
