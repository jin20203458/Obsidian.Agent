---
description: >-
  Obsidian 지식베이스 작성 규격 및 에이전트 라우팅 Frontmatter 표준 지침. 마크다운 문서 작성/수정 시 참조.
related:
  - ../README.md
  - ../.agents/AGENTS.md
---
# Knowledge Base Authoring Guidelines

> **부제**: Obsidian Knowledge Base Authoring Guidelines (문서 작성 지침)

본 문서는 `Obsidian.Agent` 지식베이스 내에서 문서를 신규 생성하거나 수정할 때, **인간 개발자와 AI 에이전트가 모두 공통으로 준수해야 하는 파일 작성 표준**을 정의합니다. 일관된 구조는 에이전트의 환각(Hallucination)을 줄이고 탐색 효율성을 극대화합니다.

## 1. 파일 및 디렉토리 명명 규칙 (Naming Conventions)
에이전트의 터미널 도구, 스크립트 파싱 및 크로스 플랫폼(OS) 호환성을 위해 다음 규칙을 엄격히 적용합니다.

* **영문 및 스네이크/카멜 케이스 사용:** 파일명과 폴더명에 한글이나 공백을 절대 사용하지 않습니다.
  * [Bad] `잘못된 예시: 기타 메모/엔터티 종류.md`
  * [Good] `올바른 예시: memo/entity_types.md`

## 2. 필수 YAML Frontmatter (AI Agent Routing Standard)
모든 마크다운 파일의 최상단에는 반드시 **에이전트 라우팅 및 지식 탐색(Progressive Disclosure)에 필요한 최소 메타데이터**인 `description`과 `related` 두 가지 속성만을 유지합니다. 이는 불필요한 토큰 소모를 방지하고 H1 제목 중복 버그(DRY 위반)를 근본 차단합니다.

```yaml
---
description: >-
  [문서 핵심 요약 1줄].
  [에이전트 트리거 조건: 예 - C++ 물리 엔진 수정이나 EnTT ECS 관련 작업 시 활성화]
related:
  - ../README.md
  - ./01_game_server_architecture.md
---
```

### 주요 속성 정의
* **`description` (필수)**: 문서의 핵심 요약과 함께 **"에이전트가 어떤 상황/요청에서 이 문서를 읽어야 하는지(Trigger Condition)"**를 명시합니다. 에이전트가 엉뚱한 문서를 여는 환각(Hallucination)을 차단하는 핵심 라우팅 속성입니다.
* **`related` (필수)**: 에이전트가 무분별한 전체 검색 대신 상대 경로 링킹을 통해 필요한 문서만 단계적으로 탐색(Progressive Disclosure)할 수 있도록 연관 문서의 상대 경로를 적어줍니다.

## 3. 링크 및 경로 작성 규칙 (Cross-Referencing)
* **표준 마크다운 상대 경로 사용:** 옵시디언 고유의 위키링크(`[[문서명]]`) 대신, 에이전트와 GitHub 시스템이 모두 정확히 인식할 수 있는 표준 마크다운 링크 문법을 사용합니다.
  * [Bad] `[[entity_types]]`
  * [Good] `[Entity Types](./memo/entity_types.md)`

## 4. 디렉토리 인덱싱 (Indexing)
* 새로운 주요 프로젝트나 대형 폴더를 생성할 경우, 해당 폴더 최상단에 반드시 [README.md](./README.md)를 작성하여 하위 문서들의 지도를 제공해야 합니다. 에이전트는 특정 프로젝트 진입 시 이 인덱스를 가장 먼저 읽도록 설계되었습니다.
