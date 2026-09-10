---
description: >-
  Phalanx (AI-Augmented C++ ETW & C# WPF EDR Solution) 지식베이스 메인 인덱스. Phalanx 시스템 설계 및 개발 작업 시작 시 참조.
related:
  - ../README.md
---
# Phalanx Knowledge Base Index

이 디렉토리는 Phalanx 프로젝트(AI 증강형 엔드포인트 탐지 및 대응 솔루션, EDR)의 시스템 아키텍처 명세, AI 자율 위협 헌팅 에이전트(Autonomous Hunter Agent) 설계, 구현 방법론 및 개발 로드맵을 포함합니다. 에이전트는 문서 간 참조를 위해 본 인덱스를 시작점으로 활용하십시오.

## `docs/` (공식 아키텍처 명세)
프로젝트의 뼈대를 이루는 공식 엔지니어링 설계 및 시스템 명세 문서입니다.
- [00_project_overview.md](./docs/00_project_overview.md): 비전, 해결 과제 및 3대 엔지니어링 의의
- [01_system_architecture.md](./docs/01_system_architecture.md): 2계층 시스템 토폴로지 (C++ 네이티브 실시간 탐지/방어 엔진 + C# WPF AI 관제 콘솔)
- [02_ai_agent_investigation_design.md](./docs/02_ai_agent_investigation_design.md): ReAct 자율 위협 헌터 에이전트, 5대 OS 수사 도구 및 포렌식 아카이브 설계
- [03_implementation_roadmap.md](./docs/03_implementation_roadmap.md): 단계별(Phase) 기능 구현 마일스톤 및 완료 정의(DoD)
- [04_performance_benchmarks.md](./docs/04_performance_benchmarks.md): EDR 시스템 전체 실측 벤치마크 및 성능 프로파일링 통합 레지스트리 (큐 동시성, 프로세스 동결, 족보 역추적, 100μs 룰 엔진, 향후 E2E 공격 누수 실측)
