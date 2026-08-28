---
description: Obsidian.Agent 지식베이스의 메인 인덱스 및 아키텍처/가이드라인 전체 지점. 시스템 전반의 아키텍처 탐색 시 참조.
related:
  - ./ai_agent_refs/Knowledge_Base_Authoring_Guidelines.md
---
# Obsidian Agent Collaboration Knowledge Base
> **부제**: 에이전트 협업을 위한 지식베이스 최상위 인덱스

본 저장소는 개발자와 AI 에이전트 간의 협업을 위한 공유 지식 베이스(Knowledge Base)입니다.

## 저장소 구조 (Directory Structure)

- **ai_agent_refs/**: AI 에이전트 작동 및 프롬프트 최적화를 위한 공통 설계 지침
  - [Agent_Runtime_Operations_Protocol.md](ai_agent_refs/Agent_Runtime_Operations_Protocol.md): 의무 QA 검증(Exit Code 0 Ground Truth), 3-Strike 서킷 브레이커, 롤백 및 트러블슈팅 통합 런타임 프로토콜 (SSOT)
  - [AI_Prompt_Engineering_Guidelines.md](ai_agent_refs/AI_Prompt_Engineering_Guidelines.md): 프롬프트 3단계 수명주기(Build-Audit-Test), 13대 핵심 원칙 및 5대 안티패턴을 집대성한 통합 표준 지침서 (SSOT)
  - [mermaid_diagram_guidelines.md](ai_agent_refs/mermaid_diagram_guidelines.md): 에이전트 생성용 락프리 및 시퀀스 다이어그램 작성법
  - [WPF_Architecture_Guidelines.md](ai_agent_refs/WPF_Architecture_Guidelines.md): WPF MVVM 아키텍처 및 컨트롤 바인딩 규칙
  - [AI_Project_Integration_Guidelines.md](ai_agent_refs/AI_Project_Integration_Guidelines.md): 신규 개발 프로젝트를 에이전트 지식베이스 및 협업 환경과 연계하는 표준 절차
  - [Knowledge_Base_Authoring_Guidelines.md](ai_agent_refs/Knowledge_Base_Authoring_Guidelines.md): 에이전트와 인간 개발자가 공통으로 준수해야 할 문서 작성/메타데이터 표준 지침
  - [Independent_Audit_Protocol_Guidelines.md](ai_agent_refs/Independent_Audit_Protocol_Guidelines.md): 고신뢰성 기술 문서 및 분석 사양서의 무결성을 보증하는 순차적 4단계 계쇄 독립감사 표준 지침
  - [AI_Agent_Architecture_Paradigms_Guidelines.md](ai_agent_refs/AI_Agent_Architecture_Paradigms_Guidelines.md): 글로벌 빅테크 및 최신 연구에 기반한 AI 에이전트 아키텍처 및 멀티 에이전트 협동(Teaming) 패러다임 종합 가이드
  - [Agent_Collaboration_Workflow_Guidelines.md](ai_agent_refs/Agent_Collaboration_Workflow_Guidelines.md): 2025~2026 최신 Agentic SE 연구 기반 일상 개발용 에이전트 협동 및 이중 계쇄(Dual-Gated) 오케스트레이션 표준 지침


- **LLVM/**: LLVM/Clang 커스텀 Tidy 체커 및 Static Analyzer 개발 아키텍처와 참고 문서를 다루는 기술 노트 폴더 (세부 문서 목록은 해당 디렉토리 참조)

- **MundusVivens/**: Mundus Vivens 프로젝트의 아키텍처, 기획 의도, 고도화 과제를 다루는 기술 노트 폴더
  - [Mundus Vivens Index](./MundusVivens/README.md): Mundus Vivens 프로젝트 전용 문서 인덱스

- **GRC/**: GenAI Roleplay Chat(GRC) 데스크톱 클라이언트의 아키텍처 및 구현 기술 노트를 다루는 폴더
  - [GRC Index](./GRC/README.md): GRC 프로젝트 전용 문서 인덱스


- **troubleshooting/**: 중앙 집중형 트러블슈팅 및 런북 보관 폴더
  - [llvm_clang.md](troubleshooting/llvm_clang.md): LLVM/Clang 커스텀 체커 및 Static Analyzer 장애 조치 로그
  - [mundus_vivens.md](troubleshooting/mundus_vivens.md): C# AI Server & C++ Game Server 장애 조치 로그
  - [git_and_os.md](troubleshooting/git_and_os.md): 공통 OS 및 Git 환경 오류 장애 조치 로그
  - [unity_client.md](troubleshooting/unity_client.md): 유니티 엔진 및 클라이언트 장애 조치 로그
  - [etc/README.md](troubleshooting/etc/README.md): 프로젝트 비종속 범용 기술(OpenXML, C# 등) 장애 조치 로그



## 에이전트 준수 규칙 (Agent Directives)
- [.agents/AGENTS.md](.agents/AGENTS.md)

