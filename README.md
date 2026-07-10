# Obsidian Agent: AI Collaboration Knowledge Base

본 저장소는 개발자와 AI 에이전트 간의 협업을 위한 공유 지식 베이스(Knowledge Base)입니다.

## 저장소 구조 (Directory Structure)

- **ai_agent_refs/**: AI 에이전트 작동 및 프롬프트 최적화를 위한 공통 설계 지침
  - [AI_Prompt_AntiPatterns_Guidelines.md](ai_agent_refs/AI_Prompt_AntiPatterns_Guidelines.md): 프롬프트 작성 시 피해야 할 반면교사 체크리스트
  - [AI_Prompt_Engineering_Guidelines.md](ai_agent_refs/AI_Prompt_Engineering_Guidelines.md): C# 문자열 보간 버그 방지 등 공식 프롬프트 엔지니어링 규칙
  - [AI_Prompt_Workflow_Guidelines.md](ai_agent_refs/AI_Prompt_Workflow_Guidelines.md): 프롬프트 조립, 감사 및 테스트 수명 주기
  - [GitHub_Profile_Readme_Guidelines.md](ai_agent_refs/GitHub_Profile_Readme_Guidelines.md): 깃허브 프로필 리드미 템플릿 구성 지침
  - [Mermaid_Diagram_Guidelines.md](ai_agent_refs/Mermaid_Diagram_Guidelines.md): 에이전트 생성용 락프리 및 시퀀스 다이어그램 작성법
  - [WPF_Architecture_Guidelines.md](ai_agent_refs/WPF_Architecture_Guidelines.md): WPF MVVM 아키텍처 및 컨트롤 바인딩 규칙
  - [AI_Project_Integration_Guidelines.md](ai_agent_refs/AI_Project_Integration_Guidelines.md): 신규 개발 프로젝트를 에이전트 지식베이스 및 협업 환경과 연계하는 표준 절차


- **LLVM/**: LLVM/Clang 커스텀 Tidy 체커 및 Static Analyzer 개발 아키텍처와 참고 문서를 다루는 기술 노트 폴더 (세부 문서 목록은 해당 디렉토리 참조)


- **MundusVivens/**: Mundus Vivens 프로젝트의 아키텍처, 기획 의도, 고도화 과제를 다루는 기술 노트 폴더 (세부 문서 목록은 해당 디렉토리 참조)

- **troubleshooting/**: 중앙 집중형 트러블슈팅 및 런북 보관 폴더
  - [llvm_clang.md](troubleshooting/llvm_clang.md): LLVM/Clang 커스텀 체커 및 Static Analyzer 장애 조치 로그
  - [mundus_vivens.md](troubleshooting/mundus_vivens.md): C# AI Server & C++ Game Server 장애 조치 로그
  - [git_and_os.md](troubleshooting/git_and_os.md): 공통 OS 및 Git 환경 오류 장애 조치 로그
  - [unity_client.md](troubleshooting/unity_client.md): 유니티 엔진 및 클라이언트 장애 조치 로그

- **ArqaStatic_Pintos_실험_결과/**: Pintos `threads/synch.c`를 대상으로 국방무기체계 코딩 규칙을 적용한 ArqaStatic(Clang-Tidy) 정적 분석 실험 결과 보관 폴더
  - [ArqaStatic_최종_실험_결과_종합_보고서.md](ArqaStatic_Pintos_실험_결과/ArqaStatic_최종_실험_결과_종합_보고서.md): 수동 검출, Clang-Tidy 정탐/미탐/오탐 목록을 일목요연하게 총괄 정리한 마스터 최종 보고서
  - [Cppcheck_최종_실험_결과_종합_보고서.md](ArqaStatic_Pintos_실험_결과/Cppcheck_최종_실험_결과_종합_보고서.md): 수동 검출, Cppcheck 정탐/미탐/오탐 목록을 일목요연하게 총괄 정리한 마스터 최종 보고서
  - [ArqaStatic_수동_검증_기준_데이터.md](ArqaStatic_Pintos_실험_결과/ArqaStatic_수동_검증_기준_데이터.md): 51종 룰셋에 대하여 synch.c를 개발자가 직접 눈으로 분석하여 판정한 기준 데이터









## 에이전트 준수 규칙 (Agent Directives)
- [.agents/AGENTS.md](.agents/AGENTS.md)

