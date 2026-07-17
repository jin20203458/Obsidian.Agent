---
type: spec
title: Project Overview
tags: [grc, overview, architecture, ai-client, wpf]
related:
  - ../../MundusVivens/docs/00_project_overview.md
  - ./01_grc_architecture.md
last_updated: 2026-07-17
status: stable
---
# GRC (GenAI Roleplay Chat): 프로젝트 개요

> **한 줄 요약**: 3단계 압축 기억 메커니즘과 자율 세션 빌더를 탑재하여, 한계를 초월한 대규모 서사 관리가 가능한 고성능 AI 역할극 데스크톱 클라이언트.

---

## 저장소 안내 (Repository Overview)

GRC 클라이언트 소스코드 및 문서 저장소는 다음과 같습니다.

- [GRC 소스코드 (GitHub)](https://github.com/jin20203458/GRC) - C# 11.0, WPF, System.Threading.Channels 기반 비동기 스트리밍 클라이언트
- [지식베이스 (Obsidian.Agent)](https://github.com/jin20203458/Obsidian.Agent) - 본 문서를 포함한 프로젝트 공식 기술 명세서 모음

---

## 왜 만들었는가 (Why)

생성형 AI(LLM)의 발전으로 AI와 텍스트 기반의 역할극(TRPG, 캐릭터 롤플레잉)을 즐기는 유저들이 폭발적으로 증가했습니다. 기존에도 훌륭한 오픈소스 웹 클라이언트들이 존재하지만, **서버 엔지니어의 관점**에서 볼 때 기존 툴들은 명확한 한계를 가지고 있었습니다.

1.  **컨텍스트 윈도우의 한계와 비용 문제**: 긴 서사를 진행할수록 누적된 프롬프트가 모델의 입력 한계를 초과하며, 매 턴마다 엄청난 토큰 비용을 발생시킵니다.
2.  **수동적이고 번거로운 설정**: 캐릭터 스탯, 세계관, 로어북, 시스템 프롬프트를 유저가 일일이 텍스트로 깎아야(Prompt Engineering) 하는 진입 장벽이 존재합니다.
3.  **UI 동시성 제어의 부재**: 무거운 JSON 역직렬화와 텍스트 스트리밍이 메인 스레드를 막아버려, UI가 간헐적으로 굳거나(Freeze) 버벅이는 현상이 고질적으로 발생합니다.

GRC는 이러한 기술적 병목을 **서버 아키텍처 레벨의 접근(계층형 캐시, 비동기 큐, 생산자-소비자 패턴)**으로 타파하고자 시작된 데스크톱 애플리케이션 프로젝트입니다.

---

## 무엇을 하는 프로젝트인가 (What)

**장르**: 생성형 AI 기반 대규모 서사 관리 및 자율 콘텐츠 저작(Authoring) 클라이언트

*   **3단계 메모리 파이프라인**: 18턴의 단기 기억 버퍼링, 10턴 주기의 챕터 요약, 1,500자 단위의 장기 연대기 압축이라는 계층적 메모리 소거법(Eviction)을 도입하여, 아무리 긴 소설을 써도 컨텍스트 길이를 상수로 고정합니다.
*   **AI Session Architect**: 유저가 "사이버펑크 하수구 아포칼립스"라는 한 줄만 입력하면, AI 에이전트가 6단계의 상태 기계(State Machine)를 거쳐 세계관, 로어북, 스탯, 초기 시나리오를 자가 검수(Self-Review)하며 자동으로 세션을 빌드합니다.
*   **비차단(Non-blocking) 렌더링 파이프라인**: O(1) 파일 쓰기 시스템과 `System.Threading.Channels` 기반 비동기 스트리밍, 60FPS 배치 렌더링을 적용하여 대용량 서사 처리 시에도 단 1프레임의 UI 끊김조차 허용하지 않습니다.
*   **멀티모달 TTS**: 단순한 TTS가 아닌, AI가 현재 서사의 맥락과 캐릭터 감정을 분석하여 감정이 실린 성우 연기(Audio Modality)를 스트리밍과 완벽한 싱크로 재생합니다.
