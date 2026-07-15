---
type: guideline
title: AI Project Integration Guidelines
tags: [ai, project, integration, guidelines]
related:
  - ../README.md
  - ./Knowledge_Base_Authoring_Guidelines.md
last_updated: 2026-07-15
status: stable
---
# AI Project Integration Guidelines: 신규 프로젝트 연동 지침

본 문서는 새로운 개발 저장소(Repository)를 `Obsidian.Agent` 지식베이스 및 AI 에이전트 협업 환경과 연계하기 위해 수행해야 하는 표준 연동 절차와 행동 강령을 정의합니다.

---

##  프로젝트 연동 3단계 워크플로우

### 1단계. 개발 저장소 내 에이전트 행동 강령 (`AGENTS.md`) 구성
연동할 개발 프로젝트 저장소 루트에 `.agents/` 디렉토리를 생성하고 `AGENTS.md` 파일을 작성합니다.
* **작성 규칙**: 글로벌 에이전트 프롬프트와의 정체성 충돌을 방지하고 정확한 지시 이행을 위해 **XML 태그 구조**와 **조건부 트리거(JIT)** 방식을 필수로 도입하며, 토큰 낭비를 막기 위해 문맥을 극도로 압축하여 작성합니다.
* **표준 구조 및 필수 태그**:
  1. **`<assigned_role>`**: 해당 저장소 내에서 에이전트가 위임받을 일시적 전문 역할 (예: Senior C++ Engine Developer 등)
  2. **`<project_philosophy>`**: 에이전트가 코딩 중 엄수해야 할 설계 방향 및 핵심 지향점 (예: Lock-free 지향 등)
  3. **`<engineering_rules>`**: 신규 파일 생성이나 레거시 모방 방지를 위한 기술적 제약조건 및 안티패턴 방지 규칙
  4. **`<critical_rules>`**: 빌드/실행 명령어 및 자격증명 노출 방지 등의 환경적/절대적 제약조건
  5. **`<context_triggers>`**: 토큰 절약을 위해 특정 조건(아키텍처 수정, 버그 발생 등) 만족 시에만 지식베이스를 로드하도록 하는 조건부 트리거
  6. **`<post_action>`**: 작업 완료 후 수행할 옵시디언 트러블슈팅 기록 및 아키텍처 문서 동기화(Sync) 규칙

* **AGENTS.md 작성 예시**:
  ```markdown
  <assigned_role>Senior C++ Physics & ECS Engine Developer</assigned_role>
  <project_philosophy>Focus: 20Hz lock-free simulation main loop, EnTT ECS.</project_philosophy>

  <engineering_rules>
  - Memory: Prefer smart pointers. Use raw pointers only when technically required (non-owning observers).
  - Formatting: Strictly follow the target file's style.
  </engineering_rules>

  <critical_rules>
  - Build: `powershell -ExecutionPolicy Bypass -File .\build_local.ps1` (NO raw CMake)
  - Paths: Use relative paths (`../Obsidian.Agent/`, etc.)
  </critical_rules>

  <context_triggers>
  - **Knowledge Base**: If modifying architecture, read `../MundusVivens/docs/01_architecture.md`.
  - **Troubleshooting**: If debugging, read `../Obsidian.Agent/troubleshooting/mundus_vivens.md` before coding.
  </context_triggers>

  <post_action>
  - **Log**: Document resolved bugs in `../Obsidian.Agent/troubleshooting/mundus_vivens.md`.
  - **Sync**: Update specs in `../MundusVivens/docs/` if architecture changes.
  </post_action>
  ```

---

### 2단계. 지식베이스 저장소 (`Obsidian.Agent`) 리소스 생성
새 프로젝트와 관련된 지식을 저장하고 트래킹할 문서를 작성합니다.
모든 문서의 파일 명명, YAML Frontmatter 형식, 링크 문법 등 **작성 표준은 [Knowledge_Base_Authoring_Guidelines.md](./Knowledge_Base_Authoring_Guidelines.md)를 준수**합니다.

1. **프로젝트 폴더 및 인덱스 생성**:
   * `Obsidian.Agent/<Project_Name>/` 디렉토리를 생성합니다. (예: `LLVM/`, `MundusVivens/`)
   * 폴더 최상단에 `README.md` (type: `index`)를 작성하여 하위 문서 지도를 제공합니다.
   * 폴더 내에 시스템의 핵심 설계 사상, 동작 원리, 모듈 간 구조(SSOT)를 정의하는 기술 명세 문서를 최소 1개 이상 생성합니다. (예: `StaticAnalyzer_Architecture.md`)
2. **트러블슈팅 로그 생성**:
   * `Obsidian.Agent/troubleshooting/<project_name>.md` 경로에 전용 로그 문서를 생성합니다.
   * **필수 템플릿 규격**을 반드시 준수하여 구조화합니다:
     ```markdown
     ## YYYY-MM-DD: 에러 발생명
     ### 1. 현상 (Symptom)
     ### 2. 원인 (Root Cause)
     ### 3. 해결책 (Resolution)
     ```

---

### 3단계. 지식베이스 `README.md` 통합 동기화
루트 `README.md` 파일을 업데이트하여 일관성을 유지합니다.
* **디렉토리 구조 (Directory Structure)** 섹션에 새로 추가한 프로젝트 폴더(예: `- **<Project_Name>/**: ...`)와 하위 명세서들을 설명과 함께 링크로 정식 등록합니다.
* **troubleshooting** 섹션에 신규 생성한 프로젝트 트러블슈팅 로그 문서 링크를 등록합니다.

---

##  에이전트용 프로젝트 환경 자동 탐색 가이드

신규 프로젝트 연동을 지시받은 에이전트는 해당 프로젝트의 소스 코드를 분석하여 아래 규칙에 따라 `.agents/AGENTS.md` 파일의 `Build` 및 `Execution` 명령어를 도출합니다.

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
