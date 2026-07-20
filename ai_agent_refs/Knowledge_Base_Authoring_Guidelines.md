---
type: guideline
title: Knowledge Base Authoring Guidelines
tags: [knowledge-base, authoring, guidelines, obsidian, context-engineering]
related:
  - ../README.md
  - ../.agents/AGENTS.md
status: stable
---
# Knowledge Base Authoring Guidelines

> **부제**: Obsidian Knowledge Base Authoring Guidelines (문서 작성 지침)

본 문서는 `Obsidian.Agent` 지식베이스 내에서 문서를 신규 생성하거나 수정할 때, **인간 개발자와 AI 에이전트가 모두 공통으로 준수해야 하는 파일 작성 표준**을 정의합니다. 일관된 구조는 에이전트의 환각(Hallucination)을 줄이고 탐색 효율성을 극대화합니다.

## 1. 파일 및 디렉토리 명명 규칙 (Naming Conventions)
에이전트의 터미널 도구, 스크립트 파싱 및 크로스 플랫폼(OS) 호환성을 위해 다음 규칙을 엄격히 적용합니다.

* **영문 및 스네이크/카멜 케이스 사용:** 파일명과 폴더명에 한글이나 공백을 절대 사용하지 않습니다.
  * [Bad] `잘못된 예시: 기타 메모/엔터티 종류.md`
  * [Good] `올바른 예시: memo/entity_types.md`

## 2. 필수 YAML Frontmatter (Obsidian Properties)
모든 마크다운 파일의 최상단에는 반드시 아래 형식의 YAML 메타데이터를 포함해야 합니다. 이는 에이전트가 문서 전체를 읽기 전에 문맥(Context)을 즉시 파악하게 해 주며, Obsidian 앱에서 Properties UI로 깔끔하게 렌더링됩니다.

```yaml
---
type: [문서 성격]
title: [사람이 읽기 쉬운 실제 제목]
tags: [tag1, tag2]
related:
  - [연관 문서 상대 경로 1]
  - [연관 문서 상대 경로 2]
status: [stable | draft | deprecated]
---
```

### 주요 속성 정의
* **`type`**: 문서의 성격을 규정합니다. 
  * `spec` (공식 명세), `guideline` (행동 지침), `runbook` (장애 조치), `reference` (참고용 메모/백서), `index` (목차) 중 택 1.
* **`title`**: 문서의 공식 영문 제목. **반드시 본문 최상단 `#` (H1) 헤더와 대소문자 및 띄어쓰기까지 100% 동일하게 일치**해야 합니다. 에이전트의 시맨틱 매핑 오차를 줄이고 이식성을 보장하기 위해 순수 영문(English) 명칭 사용을 원칙으로 합니다. (인간 개발자를 위한 직관적 한글 설명은 H1 바로 밑에 `> **부제**: ...` 양식을 활용해 부기합니다.)
* **`tags`**: Obsidian 태그 문법을 준수합니다. 공백이나 특수기호(`+`, `@` 등)를 사용할 수 없습니다. (예: `c++` 대신 `cpp` 사용)
* **`related`**: **매우 중요합니다.** 에이전트가 무분별한 전체 탐색을 하지 않도록, 상위 인덱스([README.md](./README.md))나 직접적으로 관련된 문서의 상대 경로를 적어 길잡이(Progressive Disclosure) 역할을 하게 합니다.
* **`status`**: 문서의 신뢰도를 나타냅니다. (`stable`: 공식/확정, `draft`: 초안/개인 메모, `deprecated`: 폐기됨)

## 3. 링크 및 경로 작성 규칙 (Cross-Referencing)
* **표준 마크다운 상대 경로 사용:** 옵시디언 고유의 위키링크(`[[문서명]]`) 대신, 에이전트와 GitHub 시스템이 모두 정확히 인식할 수 있는 표준 마크다운 링크 문법을 사용합니다.
  * [Bad] `[[entity_types]]`
  * [Good] `[Entity Types](./memo/entity_types.md)`

## 4. 디렉토리 인덱싱 (Indexing)
* 새로운 주요 프로젝트나 대형 폴더를 생성할 경우, 해당 폴더 최상단에 반드시 [README.md](./README.md) (type: `index`)를 작성하여 하위 문서들의 지도를 제공해야 합니다. 에이전트는 특정 프로젝트 진입 시 이 인덱스를 가장 먼저 읽도록 설계되었습니다.
