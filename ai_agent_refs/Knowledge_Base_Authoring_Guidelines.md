---
description: Obsidian 지식베이스 작성 규격 및 에이전트 라우팅 Frontmatter 표준 지침.
related:
  - ../README.md
  - ../.agents/AGENTS.md
  - ./Mermaid_Diagram_Guidelines.md
---
# Knowledge Base Authoring Guidelines


본 문서는 `Obsidian.Agent` 지식베이스 내에서 문서를 신규 생성하거나 수정할 때, **인간 개발자와 AI 에이전트가 모두 공통으로 준수해야 하는 파일 작성 표준**을 정의합니다. 일관된 구조는 에이전트의 환각(Hallucination)을 줄이고 탐색 효율성을 극대화합니다.

## 1. 파일 및 디렉토리 명명 규칙 (Naming Conventions)
에이전트의 터미널 도구, 스크립트 파싱 및 크로스 플랫폼(OS) 호환성을 위해 다음 규칙을 엄격히 적용합니다.

* **영문 및 표준 케이스 규칙 (ASCII only, zero spaces):** 모든 경로는 공백이 없는 순수 영문(ASCII)으로 작성하며 디렉토리 목적에 따라 다음 표준 케이스를 적용합니다:
  * **최상위 지침서 (`ai_agent_refs/`)**: 단어 첫 글자 대문자와 언더스코어가 결합된 `Title_Snake_Case.md`를 사용합니다 (예: `Agent_Runtime_Operations_Protocol.md`, `Knowledge_Base_Authoring_Guidelines.md`).
  * **프로젝트 문서 및 런북 (`<Project>/docs/`, `troubleshooting/`, `memo/`)**: 소문자 `snake_case.md`를 사용합니다 (예: `01_system_architecture.md`, `phalanx.md`, `entity_types.md`).
  * **프로젝트 폴더**: 공식 프로젝트 고유명(Canonical Names)을 유지합니다 (예: `Phalanx/`, `LLVM/`, `GRC/`, `MundusVivens/`).
  * [Bad] `기타 메모/엔터티 종류.md` (공백/비ASCII 포함), `AgentRuntimeProtocol.md` (언더스코어 누락)
  * [Good] `memo/entity_types.md` (ASCII snake_case), `ai_agent_refs/Agent_Runtime_Operations_Protocol.md` (Title_Snake_Case)

## 2. 필수 YAML Frontmatter (AI Metadata Standard)
모든 마크다운 파일의 최상단에는 반드시 **에이전트 지식 식별 및 탐색(Progressive Disclosure)에 필요한 최소 메타데이터**인 `description`과 `related` 두 가지 속성만을 유지합니다. 이는 불필요한 토큰 소모를 방지하고 단일 제목(H1) 원칙을 유지합니다.

```yaml
---
description: >-
  [문서가 다루는 핵심 주제 및 주요 기술 키워드를 1~2줄로 명확히 요약].
related:
  - ../README.md
  - ./01_game_server_architecture.md
---
```

### 주요 속성 정의
* **`description` (필수)**: 문서가 다루는 핵심 주제와 주요 기술 키워드를 1~2줄로 명확히 기술합니다. 에이전트의 문서 식별 및 RAG 시맨틱 검색 정확도를 보장하는 핵심 메타데이터입니다. 
* **`related` (필수)**: 에이전트가 무분별한 전체 검색 대신 상대 경로 링킹을 통해 필요한 문서만 단계적으로 탐색(Progressive Disclosure)할 수 있도록 연관 문서의 상대 경로를 적어줍니다.

## 3. 링크 및 경로 작성 규칙 (Cross-Referencing)
* **표준 마크다운 상대 경로 사용:** 옵시디언 위키링크(`[[문서명]]`) 대신, 에이전트와 GitHub 시스템이 모두 정확히 인식할 수 있는 표준 상대 마크다운 링크(`.md`) 구문만을 사용합니다.
  * [Bad] `[[entity_types]]` (옵시디언 위키링크 사용 금지)
  * [Good] `[Entity Types](./memo/entity_types.md)` (표준 상대 경로 마크다운 링크)
* **다이어그램 작성 시 참조:** 지식베이스 내 아키텍처 및 상태 머신 등 다이어그램(Mermaid) 작성 시 [Mermaid Diagram Guidelines](./Mermaid_Diagram_Guidelines.md)를 단일 진실 공급원(SSOT)으로 직접 참조합니다.

## 4. 디렉토리 인덱싱 (Indexing)
* 새로운 주요 프로젝트나 대형 폴더를 생성할 경우, 해당 폴더 최상단에 반드시 `README.md`를 작성하여 하위 문서들의 지도를 제공해야 합니다. 에이전트는 특정 프로젝트 진입 시 이 인덱스를 가장 먼저 읽도록 설계되었습니다.
