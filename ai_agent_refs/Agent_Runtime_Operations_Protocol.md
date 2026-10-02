---
description: >-
  [Level 1: 헌법 / Hard Invariant] AI 에이전트 코드 수정 후 의무 QA 검증(Exit Code 0 Ground Truth),
  3-Strike 서킷 브레이커, 원자적 롤백 및 휴먼 인수인계 트러블슈팅을 정의한 런타임 신뢰성 최상위 운영 프로토콜 (SSOT).
related:
  - ../README.md
  - ./Knowledge_Base_Authoring_Guidelines.md
  - ./Agent_Collaboration_Workflow_Guidelines.md
  - ./Independent_Audit_Protocol_Guidelines.md
  - ./AI_Project_Integration_Guidelines.md
---
# Agent Runtime Operations Protocol (런타임 운영 및 신뢰성 프로토콜 - Level 1 SSOT)

본 문서는 글로벌 `<critical_rules>`(Mandatory QA, Circuit Breaker)의 **구현 매뉴얼**입니다. 원칙(What)은 글로벌 규칙에 정의되어 있으며, 본 문서는 각 원칙의 구체적 절차(How)를 규정합니다.

---

## 1. 런타임 폐루프 상태 흐름 (Runtime Lifecycle Flow)

코드 수정 → Mandatory QA(`Exit Code 0`) → **성공 시** 완료 보고 / **실패 시** 재시도(최대 2회) → **3회 연속 실패 시** 서킷 브레이커 발동 → 원자적 롤백(§2) → 트러블슈팅 로그 박제 + 휴먼 인수인계(§3, §4)

---

## 2. 의무 검증 상세 절차 (Mandatory QA Implementation)

### 2.1 언어별 빌드/테스트 명령어 (`run_command`)
* 코드 작성이 완료되면, 해당 프로젝트의 규격(`.agents/AGENTS.md`에 정의된 빌드/테스트 명령어)에 맞춰 터미널 명령어를 직접 실행합니다.
  * **C# / .NET**: `dotnet build`, `dotnet test`
  * **C++ / LLVM**: `cmake --build build --config Release`, `ctest`
  * **TypeScript / Node**: `npm run build`, `npm test`
  * **Rust**: `cargo check`, `cargo test`

### 2.2 사이드 이펙트(Side-Effect) 전체 빌드 검사
* 특정 파일 하나를 수정했다고 해서 전체 프로젝트가 안전한 것은 아닙니다. 특히 정적 타입 언어 환경에서는 단일 인터페이스 변경이 연쇄적인 컴파일 에러를 유발할 수 있습니다.
* **국지적 테스트와 전체 빌드 병행**: 수정한 모듈에 대한 단위 테스트를 통과했더라도, 반드시 **전체 프로젝트 빌드(Full Build)**를 수행하여 타 모듈에 미친 사이드 이펙트가 0건임을 검증해야 합니다.

---

## 3. 원자적 롤백 절차 (Atomic Rollback)

### 3.1 망가진 상태로 방치 금지 (Zero Broken State)
* 에이전트가 버그를 해결하려다 오히려 코드를 더 엉망으로 만들었거나, 원래 없던 연쇄 의존성 에러를 발생시켰다면, **작업을 중단하기 전에 반드시 코드를 원상 복구(Rollback)**해야 합니다.
* **복구 절차**:
  1. Git 버전 관리 환경인 경우 `git checkout -- <file>` 또는 직전 정상 커밋으로 롤백합니다.
  2. Git이 없는 임시 환경인 경우 `replace_file_content` / `write_to_file` 도구를 사용해 본인이 변경하기 직전의 클린 상태 코드로 되돌려 놓습니다.
* **목적**: 인간 개발자가 엉망이 된 에이전트의 파괴 코드를 수습하는 디버깅 부채(Cognitive Load)를 0으로 만듭니다.

---

## 4. 구조화된 휴먼 인수인계 (Human-in-the-Loop Hand-over)

작업을 중단한 에이전트는 침묵하거나 단순히 "실패했습니다"라는 한마디로 끝나서는 안 됩니다. 인간 개발자가 상황을 즉시 파악하고 개입(HITL)할 수 있도록 정형화된 인수인계 절차를 수행합니다.

### 4.1 트러블슈팅 로그 작성 표준
[Knowledge_Base_Authoring_Guidelines.md](./Knowledge_Base_Authoring_Guidelines.md) 및 [AI_Project_Integration_Guidelines.md](./AI_Project_Integration_Guidelines.md)에 따라, 해당 프로젝트의 트러블슈팅 문서(`troubleshooting/<project_name>.md`)에 에러 상황 또는 해결 내역을 기록합니다.

모든 트러블슈팅 문서는 탐색 일관성과 목차(TOC) 앵커 링크 보전을 위해 **H2(이슈 단위) + H3(속성 단위)**의 단일 계층 구조를 엄격히 준수합니다. 단일 일자에 복수의 이슈가 발생하더라도 H3로 중첩하지 않고 개별 H2 엔트리로 분리합니다. 단, 하나의 근본 원인(Root Cause)에서 파생된 복수 증상이 동일 세션에서 함께 해결된 경우(인과 체인)에 한해 단일 H2로 기록할 수 있습니다(판별 기준: `Root Cause` 섹션을 하나의 일관된 서술로 작성할 수 있는가).

#### [Resolved] 표준 트러블슈팅 마크다운 템플릿 (해결 완료 런북)
```markdown
## YYYY-MM-DD: [Resolved] <에러/이슈 명칭 요약>

### 1. 현상 (Symptom)
- 발생한 에러 메시지, 로그 내용 또는 시스템 오작동 상황 요약

### 2. 원인 (Root Cause)
- 코드, API, 아키텍처 또는 동시성 메커니즘 차원의 근본 원인 분석

### 3. 해결책 (Resolution)
- 적용된 코드 변경점, 설정 수정 및 검증 결과 (Exit Code 0 Ground Truth 확인)
```

#### [Unresolved] 표준 트러블슈팅 마크다운 템플릿 (서킷 브레이커 중단 및 인수인계)
```markdown
## YYYY-MM-DD: [Unresolved] <에러/이슈 명칭 요약>

### 1. 현상 (Symptom)
- 발생한 컴파일/테스트 에러 메시지 및 터미널 출력 내용 (핵심 2~3줄 요약)

### 2. 시도 및 원인 분석 (Attempts & Root Cause)
- 에이전트가 시도한 3회의 수정 내역과 실패 원인 분석

### 3. 미해결 원인 분석 및 결정 요청 (Hand-over Options)
- (1) **대안 A**: <방식 A 설명 및 장단점>
- (2) **대안 B**: <방식 B 설명 및 장단점>
- 인간 개발자의 방향성 결정 및 추가 힌트 개입 요청
```

### 4.2 사용자 대화창 표준 보고 양식
로그 작성을 마친 에이전트는 사용자에게 다음과 같이 구조화된 3단계 보고를 수행하고 추가 명령을 대기합니다:

> "해당 에러를 3회 시도했으나 해결되지 않아 무한 루프 방지 및 코드 보호를 위해 **3-Strike 서킷 브레이커를 발동하고 코드를 변경 전 클린 상태로 롤백**했습니다.
> 
> - **발생 오류**: `<핵심 에러 요약>`
> - **상세 로그**: [`troubleshooting/<project_name>.md`](file:///...)에 기록 완료
> 
> **선택 가능한 해결 옵션:**
> 1. **옵션 1**: `<대안 1 설명>`
> 2. **옵션 2**: `<대안 2 설명>`
> 
> 어느 방향으로 진행할지 결정해 주시거나 추가 힌트를 제공해 주시겠습니까?"
