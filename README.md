---
type: index
title: Obsidian Agent Collaboration Knowledge Base
tags: [knowledge-base, main-index, ai-agent]
related:
  - ./ai_agent_refs/Knowledge_Base_Authoring_Guidelines.md
status: stable
---
# Obsidian Agent Collaboration Knowledge Base
> **부제**: 에이전트 협업을 위한 지식베이스 최상위 인덱스

본 저장소는 개발자와 AI 에이전트 간의 협업을 위한 공유 지식 베이스(Knowledge Base)입니다.

## 저장소 구조 (Directory Structure)

- **ai_agent_refs/**: AI 에이전트 작동 및 프롬프트 최적화를 위한 공통 설계 지침
  - [Agent_QA_Testing_Guidelines.md](ai_agent_refs/Agent_QA_Testing_Guidelines.md): 에이전트 자동 테스트 및 QA 수행 지침
  - [Agent_Troubleshooting_Guidelines.md](ai_agent_refs/Agent_Troubleshooting_Guidelines.md): 에이전트 내부 결함 디버깅 및 트러블슈팅 지침
  - [AI_Prompt_AntiPatterns_Guidelines.md](ai_agent_refs/AI_Prompt_AntiPatterns_Guidelines.md): 프롬프트 작성 시 피해야 할 반면교사 체크리스트
  - [AI_Prompt_Engineering_Guidelines.md](ai_agent_refs/AI_Prompt_Engineering_Guidelines.md): C# 문자열 보간 버그 방지 등 공식 프롬프트 엔지니어링 규칙
  - [AI_Prompt_Workflow_Guidelines.md](ai_agent_refs/AI_Prompt_Workflow_Guidelines.md): 프롬프트 조립, 감사 및 테스트 수명 주기
  - [mermaid_diagram_guidelines.md](ai_agent_refs/mermaid_diagram_guidelines.md): 에이전트 생성용 락프리 및 시퀀스 다이어그램 작성법
  - [WPF_Architecture_Guidelines.md](ai_agent_refs/WPF_Architecture_Guidelines.md): WPF MVVM 아키텍처 및 컨트롤 바인딩 규칙
  - [AI_Project_Integration_Guidelines.md](ai_agent_refs/AI_Project_Integration_Guidelines.md): 신규 개발 프로젝트를 에이전트 지식베이스 및 협업 환경과 연계하는 표준 절차
  - [Knowledge_Base_Authoring_Guidelines.md](ai_agent_refs/Knowledge_Base_Authoring_Guidelines.md): 에이전트와 인간 개발자가 공통으로 준수해야 할 문서 작성/메타데이터 표준 지침


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











## 에이전트 준수 규칙 (Agent Directives)
- [.agents/AGENTS.md](.agents/AGENTS.md)

