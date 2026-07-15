---
type: index
title: Mundus Vivens Knowledge Base Index
tags: [mundus-vivens, index]
related:
  - ../README.md
last_updated: 2026-07-15
status: stable
---
# Mundus Vivens Knowledge Base Index

이 디렉토리는 Mundus Vivens 프로젝트(C++ Game Server & C# AI Server)와 관련된 아키텍처 명세, 외부 참고 자료, 학습 메모를 포함합니다. 에이전트는 문서 간 참조를 위해 본 인덱스를 시작점으로 활용하십시오.

## 📂 `docs/` (공식 아키텍처 명세)
프로젝트의 뼈대를 이루는 공식 기획 및 시스템 설계 문서입니다.
- [00_project_overview.md](./docs/00_project_overview.md): 전체 비전 및 프로젝트 개요
- [01_game_server_architecture.md](./docs/01_game_server_architecture.md): C++ 기반 게임 서버 및 네트워크 아키텍처
- [02_agent_design.md](./docs/02_agent_design.md): NPC 인지/행동 구조 및 메모리 시스템 설계 (C# AI 서버 중심)
- [03_future_roadmap.md](./docs/03_future_roadmap.md): 향후 구현 과제 및 중장기 기획
- [04_profiling_and_optimization.md](./docs/04_profiling_and_optimization.md): 성능 계측(Tracy Profiler) 및 최적화 전략

## 📂 `references/` (외부 참고 자료)
설계에 영감을 주거나 이론적 기반이 된 논문 및 타 게임 아키텍처 분석 리포트입니다.
- [sims_rimworld_ai_report.md](./references/sims_rimworld_ai_report.md): 심즈, 림월드 AI 구조 비교
- [smallville_behavior_report.md](./references/smallville_behavior_report.md): Generative Agents (Smallville) 논문 심층 구현 분석

## 📂 `memo/` (학습 및 임시 노트)
서버 개발을 위한 C++ 구조적 연구 및 기타 특수 로직 스크랩입니다.
- [cpp_server_architecture.md](./memo/cpp_server_architecture.md): C++ 게임 서버 하위 의존성 및 리뷰 로드맵
- [entity_types.md](./memo/entity_types.md): EnTT ECS 엔터티 분기 정의
- [special_logic_locations.md](./memo/special_logic_locations.md): 물리/소셜 시뮬레이션 외의 독립적 특수 로직 발생처
