# AI Project Integration Guidelines: 신규 프로젝트 연동 지침

본 문서는 새로운 개발 저장소(Repository)를 `Obsidian.Agent` 지식베이스 및 AI 에이전트 협업 환경과 연계하기 위해 수행해야 하는 표준 연동 절차와 행동 강령을 정의합니다.

---

##  프로젝트 연동 3단계 워크플로우

### 1단계. 개발 저장소 내 에이전트 행동 강령 (`AGENTS.md`) 구성
연동할 개발 프로젝트 저장소 루트에 `.agents/` 디렉토리를 생성하고 `AGENTS.md` 파일을 작성합니다.
* **작성 규칙**: 설명이나 배경 지식은 철저히 배제하고, 에이전트가 즉각 실행해야 하는 **행동 양식(Directives)만 간결한 불릿 포인트로 작성**합니다.
* **필수 포함 항목**:
  1. **Build**: 대상 프로젝트의 가장 빠른 빌드/증분 빌드 명령어 템플릿
  2. **Execution**: 분석 툴 구동, 테스트 실행 또는 프로젝트 실행 명령어 템플릿
  3. **Knowledge Base**: 수정 작업 전에 읽어야 할 `Obsidian.Agent` 내 관련 명세 및 로그 경로를 지정하여 사전 독해 강제
  4. **Troubleshooting**: 버그 해결 시 기록할 `Obsidian.Agent` 내 트러블슈팅 문서 상대 경로 지정

---

### 2단계. 지식베이스 저장소 (`Obsidian.Agent`) 리소스 생성
새 프로젝트와 관련된 지식을 저장하고 트래킹할 문서를 작성합니다.

1. **아키텍처/기획 기술 노트 생성**:
   * `Obsidian.Agent/<Project_Name>/` 디렉토리를 생성합니다. (예: `LLVM/`, `MundusVivens/`)
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
`Obsidian.Agent` 저장소 루트에 있는 [README.md](file:///C:/Users/user/Documents/GitHub/Obsidian.Agent/README.md) 파일을 업데이트하여 일관성을 유지합니다.
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
