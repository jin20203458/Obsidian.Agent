---
description: 신규 개발 저장소와 Obsidian.Agent 지식베이스 간의 연동 및 AGENTS.md 행동 강령 구축 지침.
related:
  - ../README.md
  - ./Knowledge_Base_Authoring_Guidelines.md
  - ./Agent_Runtime_Operations_Protocol.md
  - ./AI_Prompt_Engineering_Guidelines.md
---
# AI Project Integration Guidelines


본 문서는 새로운 개발 저장소(Repository)를 `Obsidian.Agent` 지식베이스 및 AI 에이전트 협업 환경과 연계하기 위해 수행해야 하는 표준 연동 절차와 행동 강령을 정의합니다.

---

##  프로젝트 연동 3단계 워크플로우

### 1단계. 개발 저장소 내 에이전트 행동 강령 (`.agents/AGENTS.md`) 구성
연동할 개발 프로젝트 저장소 루트에 `.agents/` 디렉토리를 생성하고 `AGENTS.md` 파일을 작성합니다.
* **작성 규칙**: 시스템 프롬프트의 지시 준수율을 극대화하고 오염을 방지하기 위해 **[AI_Prompt_Engineering_Guidelines.md](./AI_Prompt_Engineering_Guidelines.md)의 핵심 원칙(XML 태그 경계 격리, Junk 토큰 배제, 긍정 프레이밍)**을 준수하며, 조건부 트리거(JIT)를 활용하여 문맥을 고밀도로 압축 작성합니다.
* **표준 구조 및 5대 필수 태그**:
  1. **`<project_philosophy>`**: 프로젝트 핵심 지향점 및 아키텍처 설계 원칙 (설계 가치 충돌 시 적용할 우선순위 포함)
  2. **`<engineering_rules>`**: 언어별 코딩 규약, 메모리/동시성 모델, 기술적 제약조건 및 안티패턴 방지 규칙
  3. **`<critical_rules>`**: 빌드/실행 명령어, Fast QA 테스트 필터, 실행 권한 및 자격증명 격리 제약
  4. **`<context_triggers>`**: 특정 조건 만족 시에만 지식베이스를 로드하도록 하는 조건부 점진적 탐색(JIT) 트리거
  5. **`<post_action>`**: 작업 완료 후 수행할 트러블슈팅 로깅 및 사양서(SSOT) 동기화 규칙

* **AGENTS.md 표준 템플릿 예시**:
  ```markdown
  <project_philosophy>
  Focus: [Core architectural mission and primary system design goals]
  Priorities: [Trade-off ordering: e.g., Correctness & Security > Code Simplicity]
  </project_philosophy>

  <engineering_rules>
  - Architecture: [Core design patterns, modular boundaries, and dependency rules]
  - Concurrency/Resource: [Concurrency model, resource lifecycle (RAII/Dispose), anti-pattern prevention]
  - Formatting: Strictly follow the target file's style and indentation.
  </engineering_rules>

  <critical_rules>
  - Build: `<standard_build_command>`
  - Test: `<unit_test_command>` (Enforce fast QA filter)
  - Paths: Use relative paths (`../Obsidian.Agent/`, etc.)
  </critical_rules>

  <context_triggers>
  - **Architecture**: If modifying core architecture or IPC, read `../Obsidian.Agent/<Project>/docs/01_architecture.md`.
  - **Troubleshooting**: If debugging or fixing errors, read `../Obsidian.Agent/troubleshooting/<project>.md` before coding.
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
   * `Obsidian.Agent/<Project_Name>/` 디렉토리를 생성합니다. (예: `LLVM/`, `MundusVivens/`)
   * 폴더 최상단에 `README.md`를 작성하여 하위 문서 지도를 제공합니다.
   * 폴더 내 `docs/` 하위에 시스템의 핵심 설계 사상, 동작 원리, 모듈 간 구조(SSOT)를 정의하는 기술 명세 문서를 최소 1개 이상 생성합니다. (예: `docs/01_static_analyzer_architecture.md`, `docs/01_system_architecture.md`)
2. **트러블슈팅 로그 생성**:
   * `troubleshooting/<project_name>.md` 경로에 전용 로그 문서를 생성합니다.
   * 작성 서식 및 구조는 [Agent_Runtime_Operations_Protocol.md](./Agent_Runtime_Operations_Protocol.md) 제6.1조의 표준 템플릿(`[Resolved]` / `[Unresolved]`)을 엄격히 준수합니다.

---

### 3단계. 지식베이스 [README.md](../README.md) 통합 동기화
루트 [README.md](../README.md) 파일을 업데이트하여 일관성을 유지합니다.
* **디렉토리 구조 (Directory Structure)** 섹션에 새로 추가한 프로젝트 폴더(예: `- **<Project_Name>/**: ...`)와 하위 명세서들을 설명과 함께 링크로 정식 등록합니다.
* **troubleshooting** 섹션에 신규 생성한 프로젝트 트러블슈팅 로그 문서 링크를 등록합니다.

---

##  에이전트용 프로젝트 환경 자동 탐색 가이드

신규 프로젝트 연동을 지시받은 에이전트는 해당 프로젝트의 소스 코드를 분석하여 아래 규칙에 따라 [.agents/AGENTS.md](../.agents/AGENTS.md) 파일의 `Build` 및 `Execution` 명령어를 도출합니다.

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

### 2. 실행/테스트 도구 도출 (Execution Directive)
* 빌드된 바이너리 아티팩트의 생성 경로(예: `build/Release/bin/`, `bin/Debug/`)를 추적하여 실행 명령어를 작성합니다.
* 정적 분석 툴(Clang, Clang-Tidy 등)의 경우, 대상 파일 검증을 위한 표준 인자 구성을 파악하여 템플릿(예: `--checks=<checks>`, `-analyzer-checker=<checker>`) 형태로 추상화합니다.
