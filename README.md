# Obsidian Agent: AI Collaboration Knowledge Base

본 저장소는 개발자와 AI 에이전트 간의 협업을 위한 공유 지식 베이스(Knowledge Base)입니다.

## 저장소 구조 (Directory Structure)

- **[ai_agent_refs](ai_agent_refs/)**: AI 에이전트 작동 및 프롬프트 최적화를 위한 공통 설계 지침
  - [AI_Agent_Orchestration_Guidelines.md](ai_agent_refs/AI_Agent_Orchestration_Guidelines.md): 다중 에이전트 분산 설계 및 오케스트레이션 규칙
  - [AI_Prompt_AntiPatterns_Guidelines.md](ai_agent_refs/AI_Prompt_AntiPatterns_Guidelines.md): 프롬프트 작성 시 피해야 할 반면교사 체크리스트
  - [AI_Prompt_Engineering_Guidelines.md](ai_agent_refs/AI_Prompt_Engineering_Guidelines.md): C# 문자열 보간 버그 방지 등 공식 프롬프트 엔지니어링 규칙
  - [AI_Prompt_Workflow_Guidelines.md](ai_agent_refs/AI_Prompt_Workflow_Guidelines.md): 프롬프트 조립, 감사 및 테스트 수명 주기
  - [Gemini_Cognitive_Workspace_Guidelines.md](ai_agent_refs/Gemini_Cognitive_Workspace_Guidelines.md): 제미나이 특화 컨텍스트 캐싱 및 CAG 최적화 가이드라인
  - [GitHub_Profile_Readme_Guidelines.md](ai_agent_refs/GitHub_Profile_Readme_Guidelines.md): 깃허브 프로필 리드미 템플릿 구성 지침
  - [Mermaid_Diagram_Guidelines.md](ai_agent_refs/Mermaid_Diagram_Guidelines.md): 에이전트 생성용 락프리 및 시퀀스 다이어그램 작성법
  - [WPF_Architecture_Guidelines.md](ai_agent_refs/WPF_Architecture_Guidelines.md): WPF MVVM 아키텍처 및 컨트롤 바인딩 규칙
- **[MundusVivens](MundusVivens/)**: Mundus Vivens 프로젝트 아키텍처 및 기획 의도 상세 노트
  - [CPP_Evolution_Notes.md](MundusVivens/CPP_Evolution_Notes.md): C++ 서버 로컬 BT 및 지각 시스템 고도화 로드맵
  - [Current_Architecture.md](MundusVivens/Current_Architecture.md): C++ 물리 서버와 C# 인지 브레인 동기화 파이프라인
  - [Smallville_Dialogue_API.md](MundusVivens/Smallville_Dialogue_API.md): 스몰빌 생성 에이전트 대화 API 호출 분석 메모
  - [State_Machine_Limitations.md](MundusVivens/State_Machine_Limitations.md): 상태 머신 기반 시뮬레이션의 한계 및 자율 에이전트 전환 당위성
  - [Under_Consideration.md](MundusVivens/Under_Consideration.md): 현재 아키텍처에서 고도화를 검토 중인 과제 목록

## 에이전트 준수 규칙 (Agent Directives)

- [.agents/AGENTS.md](.agents/AGENTS.md)
