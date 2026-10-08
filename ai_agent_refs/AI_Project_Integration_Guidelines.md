---
description: 신규 개발 저장소와 Obsidian.Agent 지식베이스 간의 연동 및 AGENTS.md 행동 강령 구축 지침.
related:
  - ../README.md
  - ./Knowledge_Base_Authoring_Guidelines.md
  - ./Agent_Runtime_Operations_Protocol.md
  - ./AI_Prompt_Engineering_Guidelines.md
  - ./Troubleshooting_Pruning_Guidelines.md
---
# AI Project Integration Guidelines


본 문서는 새로운 개발 저장소(Repository)를 `Obsidian.Agent` 지식베이스 및 AI 에이전트 협업 환경과 연계하기 위해 수행해야 하는 표준 연동 절차와 행동 강령을 정의합니다.

---

## 프로젝트 연동 3단계 워크플로우

### 1단계. 개발 저장소 내 에이전트 행동 강령 (`.agents/AGENTS.md`) 구성
연동할 개발 프로젝트 저장소 루트에 `.agents/` 디렉토리를 생성하고 `AGENTS.md` 파일을 작성합니다.
* **작성 규칙**: 시스템 프롬프트의 지시 준수율을 극대화하고 오염을 방지하기 위해 **[AI_Prompt_Engineering_Guidelines.md](./AI_Prompt_Engineering_Guidelines.md)의 핵심 원칙(XML 태그 경계 격리, Junk 토큰 배제, 긍정 프레이밍, 직교성 분리)**을 준수하며 다음 불변식을 적용합니다:
  - **Style 1 고밀도 화살표 매핑 (`Keywords -> Path`)**: 자연어 상투어를 배제하고, 핵심 엔티티 키워드 목록과 파일 경로를 `->`로 직결하는 High-SNR 시맨틱 인덱스 구조를 적용합니다. (토큰 바이트 ~20% 절감, 어텐션 헤드의 코사인 유사도 매칭 최적화).
  - **경로 이식성 (Path Portability)**: 모든 파일 및 문서 참조는 저장소 루트 또는 형제 저장소 기준의 OS/CI 독립적인 **상대 경로(`../Obsidian.Agent/`, `../../../Users/user/...`)**로만 구성합니다.
  - **불변식 금지선과 긍정 대안 결합 (Bounded Invariant Fences & Positive Pairing)**: 모호한 부정("환각하지 마라", "실수 금지")은 모델 어텐션을 금지 대상에 집중시켜 역효과(Pink Elephant Problem)를 초래하므로 배제합니다. 반면 시스템 크래시나 정합성 파괴를 막는 치명적 안티패턴은 `NEVER`/`금지` 형태의 **유한하고(3~5개 이내) 구체적인 불변식 경계선(Hard Guardrails)**으로 선언하되, 반드시 **대체할 긍정적 실행 대안(What to do instead)**을 한 쌍으로 결합하여 명시합니다 (예: `NEVER block synchronously; use async/await end-to-end`, `NEVER use Obsidian wikilinks; use standard Markdown links (.md)`).
  - **CLI 명령어 구조화 (Structured CLI Commands)**: 복합 테스트 스위트의 경우 단일 긴 줄 대신 계층형 서브 불렛을 적용하여 Fast QA 필터와 격리 옵션을 에이전트가 기계적으로 오인 없이 파싱하도록 구성합니다.
* **표준 구조 및 5대 필수 태그**:
  1. **`<project_philosophy>`**: 프로젝트 핵심 미션 및 아키텍처 설계 지향점 (`Focus:` 단일 라인으로 고밀도 선언)
  2. **`<engineering_rules>`**: 언어별 코딩 규약, 메모리/동시성 모델, 기술적 제약조건 (순수 텍스트 마크다운 서식 유지)
  3. **`<critical_rules>`**: 빌드 명령어, 계층형 Fast QA 테스트 필터, 실행 권한/자격증명 격리 및 상대 경로 강제
  4. **`<context_triggers>`**: Style 1 화살표 매핑 기반의 High-SNR 온디맨드(JIT) 지식베이스 로딩 게이트웨이 (2~6개 이내)
  5. **`<post_action>`**: 작업 완료 후 수행할 트러블슈팅 로깅, 사양서(SSOT) 동기화 및 링크 무결성 검증 규칙

* **AGENTS.md 표준 템플릿 예시**:
  ```markdown
  <project_philosophy>
  Focus: [Core architectural mission and primary system design goals]
  </project_philosophy>

  <engineering_rules>
  - **Architecture**: [Core design patterns, modular boundaries, and dependency rules]
  - **Concurrency/Resource**: [Concurrency model, resource lifecycle (RAII/Dispose), anti-pattern prevention]
  - **Formatting**: Strictly follow target file style and indentation. Technical plain-text markdown only.
  </engineering_rules>

  <critical_rules>
  - **Build**: `<standard_build_command>`
  - **Test**:
    - Fast QA: `<fast_qa_unit_test_command>` (Enforce category/filter)
    - Requirement: Always specify the fast QA filter to isolate local tests from long-running cloud suites.
    - Full-Chain: `<fullchain_test_script_command>`
  - **Paths**: Use relative paths exclusively (`../Obsidian.Agent/`, etc.).
  </critical_rules>

  <context_triggers>
  - **Architecture**: System architecture, pipelines, IPC schemas -> `../Obsidian.Agent/<Project>/docs/01_architecture.md`
  - **Troubleshooting**: Bugs, runtime crashes, edge cases, runbook -> `../Obsidian.Agent/troubleshooting/<project>.md`
  </context_triggers>

  <post_action>
  - **Log**: Document resolved bugs in `../Obsidian.Agent/troubleshooting/<project>.md`.
  - **Sync**: Update specifications in `../Obsidian.Agent/<Project>/docs/` if architecture changes.
  </post_action>
  ```

---

### 2단계. 지식베이스 저장소 (`Obsidian.Agent`) 리소스 생성
새 프로젝트와 관련된 지식을 저장하고 트래킹할 문서를 작성합니다.
모든 문서의 파일 명명, YAML Frontmatter 형식, 링크 문법 등 **작성 표준은 [Knowledge_Base_Authoring_Guidelines.md](./Knowledge_Base_Authoring_Guidelines.md)를 준수**합니다.

1. **프로젝트 폴더 및 인덱스 생성**:
   * `Obsidian.Agent/<Project_Name>/` 디렉토리를 생성합니다. 폴더명은 **PascalCase 또는 대문자 약어**를 사용합니다 (예: `Phalanx/`, `LLVM/`, `GRC/`, `MundusVivens/`).
   * 폴더 최상단에 `README.md`를 작성하여 하위 문서 지도를 제공합니다.
   * 폴더 내 `docs/` 하위에 시스템의 핵심 설계 사상, 동작 원리, 모듈 간 구조(SSOT)를 정의하는 기술 명세 문서를 생성합니다. 파일명은 **ASCII snake_case**를 사용합니다 (예: `docs/00_project_overview.md`, `docs/01_system_architecture.md`).
   * 모든 문서는 필수 YAML Frontmatter(`description` 및 상대경로 `related` 목록)를 포함하여 점진적 탐색(Progressive Disclosure)을 지원합니다.
2. **트러블슈팅 로그 생성**:
   * `troubleshooting/<project_name>.md` 경로에 전용 로그 문서를 생성합니다 (파일명은 소문자 snake_case).
   * 작성 서식 및 구조는 [Agent_Runtime_Operations_Protocol.md](./Agent_Runtime_Operations_Protocol.md) 제6.1조 및 [Troubleshooting_Pruning_Guidelines.md](./Troubleshooting_Pruning_Guidelines.md)의 표준 서식(H2 에피소드 + H3 `[Resolved]` / `[Unresolved]`)과 증류 체크포인트 블록(`<!-- PRUNING_CHECKPOINT: ... -->`)을 엄격히 준수합니다.

---

### 3단계. 지식베이스 [README.md](../README.md) 통합 동기화 및 사후 검증
루트 [README.md](../README.md) 파일을 업데이트하고 링크 무결성을 검증합니다.
* **디렉토리 구조 (Directory Structure)** 섹션에 새로 추가한 프로젝트 폴더(예: `- **<Project_Name>/**: ...`)와 하위 명세서들을 설명과 함께 링크로 정식 등록합니다.
* **troubleshooting** 섹션에 신규 생성한 프로젝트 트러블슈팅 로그 문서 링크를 등록합니다.
* **링크 무결성 감사 (Mandatory Link Audit)**: 신규 추가된 모든 상대 마크다운 링크(`.md`)가 실제 디스크에 존재하는지 터미널 스크립트를 통해 물리적 실존성을 전수 검증합니다 (Exit Code 0).

---

## 에이전트용 프로젝트 환경 자동 탐색 가이드

신규 프로젝트 연동을 지시받은 에이전트는 해당 프로젝트의 소스 코드를 분석하여 아래 규칙에 따라 [.agents/AGENTS.md](../.agents/AGENTS.md) 파일의 `Build` 및 `Test/Execution` 명령어를 도출합니다.

### 1. 빌드 도구 식별 및 명령어 도출 (Build Directive)
* **CMake 기반 프로젝트** (루트에 `CMakeLists.txt` 존재):
  * 빌드 폴더(`build` 또는 `out`)의 존재 여부를 스캔합니다.
  * 명령어 예시: `cmake --build <build_path> --config <config> --target <target>`
* **Node.js/Frontend 프로젝트** (루트에 `package.json` 존재):
  * 명령어 예시: `npm run build` 또는 프로젝트 유형에 맞는 빌드 명령어
* **.NET/C# 프로젝트** (루트에 `.sln` 또는 `.csproj` 존재):
  * 명령어 예시: `dotnet build <solution_or_project_file> --configuration <config>`
* **Makefile/Make 기반 프로젝트** (루트에 `Makefile` 존재):
  * 명령어 예시: `make -j<cpu_cores>`

### 2. 테스트/실행 도구 도출 (Test & Execution Directive)
* **단위 테스트 및 Fast QA 필터 도출**:
  * 테스트 프로젝트(예: `tests/`, `*.Tests.csproj`, `ctest`) 구조를 분석하여 2~3초 내에 완료되는 Fast QA 필터(예: `dotnet test --filter "Category=Unit"`, `ctest -R Unit`)를 식별합니다.
  * 전체 테스트 실행 시 클라우드 API 호출이나 극심한 지연이 발생하는 경우 경고(Warning) 플래그를 명시합니다.
* **바이너리 아티팩트 및 실행 추적**:
  * 빌드된 바이너리 아티팩트의 생성 경로(예: `build/Release/bin/`, `bin/Debug/`)를 추적하여 실행 명령어를 작성합니다.
  * 정적 분석 툴(Clang, Clang-Tidy 등)의 경우, 대상 파일 검증을 위한 표준 인자 구성을 파악하여 템플릿(예: `--checks=<checks>`, `-analyzer-checker=<checker>`) 형태로 추상화합니다.
