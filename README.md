# Obsidian Agent: AI Collaboration Knowledge Base

본 저장소는 개발자와 AI 에이전트 간의 협업을 위한 공유 지식 베이스(Knowledge Base)입니다.

## 저장소 구조 (Directory Structure)

- **[ai_agent_refs](ai_agent_refs/)**: AI 에이전트 작동 및 프롬프트 최적화를 위한 공통 설계 지침
  - [Gemini_Cognitive_Workspace_Guidelines.md](ai_agent_refs/Gemini_Cognitive_Workspace_Guidelines.md): 제미나이 특화 컨텍스트 캐싱 및 CAG 최적화 가이드라인
  - [AGENTS_MD_Optimization_Guidelines.md](ai_agent_refs/AGENTS_MD_Optimization_Guidelines.md): 다중 에이전트 분산 설계 규칙
  - [AI_Prompt_Engineering_Guidelines.md](ai_agent_refs/AI_Prompt_Engineering_Guidelines.md) / [AI_Prompt_AntiPatterns_Guidelines.md](ai_agent_refs/AI_Prompt_AntiPatterns_Guidelines.md): 프롬프트 조립 및 디버깅 체크리스트
  - [mermaid_diagram_guidelines.md](ai_agent_refs/mermaid_diagram_guidelines.md): 락프리 및 시퀀스 다이어그램 작성법
- **[MundusVivens](MundusVivens/)**: Mundus Vivens 프로젝트 아키텍처 및 기획 의도 상세 노트
  - [현 구조.md](MundusVivens/현%20구조.md): C++ 물리 서버와 C# 인지 브레인 동기화 파이프라인
  - [C++ 고도화 사항.md](MundusVivens/C++%20고도화%20사항.md): 향후 적용될 반사 BT 및 지각 시스템 확장 포인트

## 에이전트 준수 규칙 (Agent Directives)

1. **텍스트 정돈성 유지**: 본 저장소의 모든 문서는 데코레이션 이모티콘이나 메타적 사족을 배제하고 드라이한 기술 사양서(Technical Specification) 포맷을 유지해야 합니다.
2. **상대 경로 참조**: 자매 코드 리포지토리(`MundusVivens`, `MundusVivens.GameServer.Cpp`)를 참조하거나 링크할 때는 항상 `../` 상대 경로를 사용하십시오.
