---
description: AI 에이전트 필수 QA 및 검증(Ground Truth) 가이드라인. 코드 수정 후 빌드/테스트 검증 시 참조.
related:
  - ../README.md
  - ./Agent_Troubleshooting_Guidelines.md
---
# Agent QA Testing Guidelines

> **부제**: 에이전트 품질 검증(QA) 및 눈먼 코딩(Blind Coding) 방지 지침

본 문서는 `Obsidian.Agent` 환경의 AI 에이전트가 코드를 작성하거나 수정한 후, **결과물의 신뢰성을 자체적으로 검증**하기 위해 반드시 거쳐야 하는 **Evaluator-Optimizer(검증-최적화)** 행동 강령을 정의합니다. (기반 문서: *Anthropic Engineering Guidelines 2025*)

---

## 1. 눈먼 코딩 (Blind Coding) 엄격 금지

에이전트는 방대한 사전 지식을 가지고 있지만, 실제 개발 환경(의존성, 로컬 설정, 빌드 시스템)은 언제나 변수가 존재합니다.

### [원칙] 머릿속 추론만으로 성공을 단정 짓지 말 것
- 코드를 작성하거나 수정한 직후, **"코드를 수정했으니 완벽하게 작동할 것입니다"**라고 섣불리 사용자에게 확언해서는 안 됩니다.
- 코드의 문법이나 로직이 완벽해 보이더라도, 실제 샌드박스나 로컬 터미널에서 구동(Ground Truth)해보기 전까지는 완성된 것이 아닙니다.

---

## 2. 의무 검증 (Mandatory Validation) 프로세스

에이전트는 수정 사항을 인간에게 최종 보고하기 전에, 반드시 다음의 Evaluator(검증자) 단계를 백그라운드에서 수행해야 합니다.

### [행동 1] 실제 터미널(`run_command`) 구동
- 코드 작성이 완료되면, 해당 프로젝트의 규격(`.agents/AGENTS.md`에 정의된 빌드/테스트 명령어)에 맞춰 터미널 명령어를 직접 실행합니다.
- 예: `dotnet build`, `cmake --build .`, `npm run test` 등.

### [행동 2] Ground Truth(실제 로그) 기반 판단
- 백그라운드 터미널의 출력 결과(Exit Code가 0인지, 컴파일 경고나 에러가 없는지)를 직접 확인한 후, 비로소 작업이 성공했음을 선언해야 합니다.
- 만약 에러 로그가 발견된다면, 이는 실패한 것이며 Optimizer(최적화) 단계로 돌아가 코드를 다시 수정해야 합니다.

---

## 3. 사이드 이펙트(Side-Effect) 검사

특정 파일 하나를 완벽하게 수정했다고 해서 전체 프로젝트가 안전한 것은 아닙니다. 특히 C++, C#, Java와 같은 정적 타입 언어 환경에서는 하나의 인터페이스 변경이 연쇄적인 컴파일 에러를 유발할 수 있습니다.

### [원칙] 국지적 테스트와 전체 빌드 병행
- 수정한 부분에 대한 단위 테스트(Unit Test)를 실행하여 통과했더라도, **전체 프로젝트 빌드(Full Build)**를 한 번 더 실행하여 타 모듈에 끼친 사이드 이펙트가 없는지 확인하는 것을 권장합니다.
- 전체 빌드 과정에서 사이드 이펙트가 터졌고 이를 단기간에 수습하기 어렵다면, [Agent_Troubleshooting_Guidelines.md](./Agent_Troubleshooting_Guidelines.md)의 3-Strike 룰을 적용하여 즉시 롤백하고 인간에게 이관(Hand-over)해야 합니다.
