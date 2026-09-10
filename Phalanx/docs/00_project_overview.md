---
description: >-
  Phalanx 프로젝트의 비전, 해결 과제, 3대 핵심 엔지니어링 가치 및 시스템 정량 목표 명세.
related:
  - ../README.md
  - ./01_system_architecture.md
  - ./02_ai_agent_investigation_design.md
---
# Project Overview
> **부제**: Phalanx 프로젝트 개요

> **한 줄 요약**: 0.1ms 초저지연 C++ 네이티브 탐지 엔진과 자율 AI 위협 헌팅 에이전트(Autonomous Hunter Agent)가 결합된 차세대 엔드포인트 탐지 및 대응(EDR) 시스템.

---

## 저장소 안내 (Repository Overview)

Phalanx 프로젝트의 코드 및 아키텍처 구현체는 아래 링크에서 확인할 수 있습니다.

- [C++ 네이티브 엔진 및 C# 관제 솔루션 (Phalanx)](https://github.com/jin20203458/phalanx) - C++20 EDR 엔진(`Phalanx.Engine`), gRPC 보고 스트림, C# WPF 관제 콘솔 및 AI 에이전트(`Phalanx.Cockpit`) 통합 저장소
- [지식베이스 (Obsidian.Agent)](https://github.com/jin20203458/Obsidian.Agent) - 본 문서를 포함한 프로젝트 공식 기술 명세서 및 아키텍처 문서 모음

---

## 1. 프로젝트 비전 (Vision)

**Phalanx**(팔랑크스)는 고대 그리스 중장보병 방진의 빈틈없는 방어와 유기적 협동 체계에서 착안한 **차세대 엔드포인트 탐지 및 대응(EDR, Endpoint Detection & Response) 시스템**입니다.

기존 안티바이러스(AV)가 정적 파일 해시에 의존하여 메모리 인젝션 및 파일리스(Fileless) 공격에 무력화되는 한계를 극복하고, 전통적 EDR의 고질적 문제인 경보 피로(Alert Fatigue)와 탐지-대응 지연(Detection Latency)을 **C++20 네이티브 실시간 차단 엔진**과 **자율 AI 위협 헌팅 에이전트(Autonomous Hunter Agent)**의 하이브리드 결합으로 해결합니다.

---

## 2. 해결하고자 하는 문제 (Problem Statement)

1. **파일리스 및 스크립트 기반 침투 고도화**:
   * 공격자는 디스크에 악성 바이너리를 남기지 않고, 정상 윈도우 유틸리티(`powershell.exe`, `certutil.exe`)를 은밀히 스폰하여 메모리 상에서 직접 페이로드를 실행합니다.
   * 정적 파일 검사로는 실행 시점의 동적 프로세스 체인을 선제 차단할 수 없습니다.

2. **사후 탐지의 한계와 초기 실행 윈도우(Initial Execution Window) 방어**:
   * 대부분의 중앙 집중형 보안 시스템은 이벤트를 원격 분석 파이프라인으로 전송한 뒤 사후 분석(Post-execution)하므로, 탐지 시점에는 이미 랜섬웨어 암호화나 C2 비콘 발송이 시작된 후입니다.
   * 공격자가 첫 시스템 콜을 수행하기 전, 엔드포인트 로컬 현장에서 마이크로초 단위로 프로세스 생성 시점에 즉각 개입하여 실행 윈도우를 원천 차단하는 초저지연 로컬 방어가 필수적입니다.

3. **기존 보안 관제의 경보 피로 (Alert Fatigue) 및 오탐 위험**:
   * 모든 의심 이벤트를 외부 클라우드 LLM에 무차별 질의하는 구조는 막대한 API 비용과 분석 지연을 초래합니다.
   * 정상 관리 도구(LOLBins)를 섣불리 차단하면 업무 마비(오탐)를 유발하고 방치하면 침해로 이어지므로, 명백한 공격은 현장에서 0.1ms 만에 사살하고 모호한 회색지대만 선별하여 AI가 심층 수사하는 지능형 계층화가 필수적입니다.

---

## 3. Phalanx의 3대 핵심 엔지니어링 가치

### A. C++20 네이티브 0.1ms 초저지연 차단 및 인메모리 DAG (The Muscle & Reflex)
* Windows **ETW (Event Tracing for Windows)**를 유저모드 C++20으로 안전하게 구독하여 커널 크래시(Zero BSOD Risk) 없이 프로세스 트리, 네트워크 소켓, 모듈 로드 이벤트를 마이크로초 단위로 수집합니다.
* 피크 시 초당 수만 건의 이벤트 폭주에도 유실률 0%를 보장하는 **더블 버퍼드 락-스왑(Double-Buffered Lock-Swap)** 큐를 적용합니다.
* C++ 내부에서 `std::unordered_map` 기반 인메모리 프로세스 트리(DAG)를 O(1)로 유지하며, 결정론적 룰 엔진을 통해 **0.1ms(100μs) 이내에 고위험 프로세스를 현장 사살(`TerminateProcess`)**합니다.

### B. 24μs 원자적 동결과 자율 AI 위협 헌터 (The Brain & Forensic Preservation)
* 정상 도구를 악용하는 모호한 회색지대(LOLBin) 위협 발생 시, 무작정 죽이지 않고 `ntdll.dll`의 미공개 커널 API인 **`NtSuspendProcess`를 호출해 24μs 만에 프로세스를 원자적으로 동결**합니다.
* 동결된 타깃의 휘발성 메모리(RAM)를 보존한 상태에서, **C# 자율 AI 에이전트가 ReAct(Thought ➔ Action ➔ Observation) 루프를 돌며 5대 OS 수사 도구(메모리 스캔, 페이로드 디코딩, 도메인 평판 조회 등)**를 직접 호출하여 3초 이내에 최종 판결(사형 확정 또는 오탐 복구)을 도출합니다.
* AI 지연이나 장애 발생 시 ntdll 로더 락 데드락을 방지하는 **10초 `SafetyWatchdog` 자가 회복 가드**를 연동합니다.

### C. 비용 최적화 및 고가용성 하이브리드 설계 (Cost-Optimized Dual Defense)
* 명백한 악성 공격 99%는 C++ 엔진이 로컬 룰로 0.1ms 만에 즉시 사살하므로 **LLM API 호출 비용이 0원**입니다.
* 오직 정밀 수사가 필요한 1%의 회색지대 사건에만 AI 에이전트를 선별 가동하여 극단적인 비용 효율성을 달성합니다.
* 네트워크 단절(폐쇄망) 환경에서도 C++ 엔진의 로컬 룰 및 세이프티 워치독을 통해 단독 방어가 완결됩니다.

---

## 4. 시스템 정량적 목표 (Key Performance Metrics)

| 지표 (Metrics) | 목표 기준치 (Target) | 달성 메커니즘 |
| :--- | :--- | :--- |
| **엔진 CPU 점유율** | 평상시 **0.5% 미만** | 더블 버퍼링, 유저모드 이벤트 스트리밍, 비동기 I/O |
| **엔진 메모리 사용량** | **30MB 미만** | C++ POD 구조체 순환 버퍼링, 정적 메모리 관리 |
| **인메모리 프로세스 트리 탐색** | **1μs 미만** | C++ 인메모리 해시맵 O(1) 부모-자식 DAG 매핑 |
| **결정론적 즉각 사살 지연 (Kill)** | 룰 매칭 후 **0.1ms(100μs) 이내** | C++ 현장 판정 및 `TerminateProcess` 즉각 호출 |
| **원자적 선제 동결 지연 (Suspend)** | 감지 후 **50μs 미만** (실측 **24~27μs**) | `ntdll!NtSuspendProcess` 동적 로딩 및 10초 세이프티 워치독 |
| **AI 자율 수사 완료 지연** | 전형적 시나리오 **3초 이내** | 구조화 JSON 모드, 도구 호출 최적화, 동결 타깃 RAM 스캔 |
| **침해사고 리포트 생성 지연** | 판결 완료 후 **1초 이내** | QuestPDF 인메모리 A4 벡터 렌더링 |
