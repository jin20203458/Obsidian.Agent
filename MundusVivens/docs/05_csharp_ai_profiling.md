---
type: spec
title: C# AI Server Profiling & Optimization
tags: [mundus-vivens, profiling, optimization, csharp, ai-server, smallville]
related:
  - ../README.md
  - ./02_agent_design.md
  - ./04_cpp_server_profiling.md
last_updated: 2026-07-15
status: draft
---
# Mundus Vivens: C# AI 서버 성능 프로파일링 및 Smallville 비교 보고서

본 문서는 Mundus Vivens **C# AI 대뇌 서버**의 LLM API 비용 효율성, 기억 계층 구조, 비동기 스케줄링 성능을 정량적으로 계측하고, 스탠퍼드 *Generative Agents (Smallville)* 논문의 아키텍처와 직접 대조하여 설계 우위를 실증하는 보고서입니다.

> **자매 문서**: C++ 물리 서버의 Tracy 프로파일링 및 극한 벤치마크 결과는 [04_cpp_server_profiling.md](./04_cpp_server_profiling.md)를 참조하십시오.

---

## 1. 프로파일링 개요

### 1) 도입 목적
* **LLM API 비용 정량 계측**: 대화, 성찰, 일과 계획 등 각 인지 파이프라인 단계별 실제 API 호출 횟수와 토큰 소비량을 측정하고, Smallville 논문의 이론적 호출 횟수와 1:1 대조합니다.
* **기억 계층(Hot/Cold) 안정성 검증**: 대량 기억 주입(Eviction Storm) 상황에서 인메모리(RAM) 상한선이 완벽히 사수되는지, LiteDB 콜드 아카이브로의 도태(Eviction)가 정상 동작하는지 실증합니다.
* **비동기 스케줄링 비차단성 검증**: 다수 에이전트의 동시 성찰 요청이 메인 시뮬레이션 틱에 주는 실시간 영향(Contention)이 0ms인지 계측합니다.
* **결정론적 판정 파이프라인 속도**: 위협 판정, 인과 캐스케이드 등 LLM을 거치지 않는 수학적 결정론 연산의 처리량(throughput)을 마이크로초 단위로 계측합니다.

### 2) 벤치마크 인프라 설계
프로덕션 코드와 벤치마크 코드를 완전히 분리하여, 일반 서버 구동에 벤치마크 오버헤드가 유입되지 않도록 설계되었습니다.
* **CLI 스위치 (`--benchmark-mode`)**: `Program.cs`의 진입점에서 아규먼트를 인터셉트하여, 지정된 벤치마크 모드만 단독 실행 후 프로세스를 강제 종료합니다.
* **지원 모드**: `dialogue`, `memory`, `reflection`, `full-sim`, `threat`, `cascade`
* **Smallville 대조군 카운터**: 각 벤치마크 내부에서 동일 시나리오를 Smallville 방식으로 처리했을 경우 발생했을 이론적 API 호출 횟수 및 예상 비용을 병행 집계합니다.

---

## 2. 벤치마크 실행 절차 (Runbook)

### 1) 개별 축 벤치마크 실행
```powershell
# 축 1: 프롬프트 통합 경제성
dotnet run --project MundusVivens.Prototype -- --benchmark-mode dialogue

# 축 2: 기억 저장소 계층화
dotnet run --project MundusVivens.Prototype -- --benchmark-mode memory

# 축 3: 성찰 스케줄링
dotnet run --project MundusVivens.Prototype -- --benchmark-mode reflection

# 축 4 & 6: 종합 시뮬레이션 (에이전트 N명, M일)
dotnet run --project MundusVivens.Prototype -- --benchmark-mode full-sim <N> <M>

# 축 5a: 위협 판정 배치
dotnet run --project MundusVivens.Prototype -- --benchmark-mode threat

# 축 5b: 인과 캐스케이드
dotnet run --project MundusVivens.Prototype -- --benchmark-mode cascade
```

### 2) 유의 사항
* `world_config.json`의 `ForceResetDatabase: true` 설정을 확인하여, 이전 세션의 잔여 기억 데이터가 벤치마크 결과를 오염시키지 않도록 합니다.
* 비지성체(`IsSentient = false`)는 성찰 큐에서 자동 바이패스되어 API 호출 0회가 보장됩니다.

---

## 3. Smallville 비교 기준선 (Baseline)

벤치마크 결과를 해석하기 위한 Smallville 논문의 아키텍처 특성 및 비용 기준선입니다.

### 논문 인용 데이터
<!-- TODO: 논문 재정독 후 정확한 수치 기입 -->

| 항목 | Smallville (논문/추정) | 출처 |
|---|---|---|
| 25명 2일 운영 비용 | ~$1,000 (GPT-3.5-turbo) | Park et al., 2023 |
| 대화 1회당 API 호출 | 12 ~ 20 회 (턴별 개별 호출) | 논문 Figure 7 추정 |
| 1일 계획 수립 API 호출 | 3단계 × 에이전트당 ~10회 | 논문 Section 5.2 추정 |
| 성찰 트리거 | importance 합산 ≥ 150 시 즉시 (블로킹) | 논문 Section 5.4 |
| 기억 저장 | Flat List, 전수 스캔 O(N) | 논문 Section 5.1 |
| 위협/전투 판정 | N/A (미지원) | - |

---

## 4. 운영 환경 실측 성능 통계

<!-- TODO: 재측정 후 실측 데이터로 교체 -->

### [비교 요약표]
| 벤치마크 차원 | Smallville (이론/논문) | Mundus Vivens (실측) | 개선 효과 |
|---|---|---|---|
| **대화 1회당 API 호출** | - 회 | - 회 | - |
| **하루 성찰+계획 API 호출** | - 회 | - 회 | - |
| **기억 중요도 채점** | - 회 | - 회 | - |
| **기억 회상(Recall)** | - | - | - |
| **원정 중 일과 계획** | - | - | - |
| **위협 판정** | N/A | - 회 | - |
| **에이전트당 일일 운영비** | - | - | - |

---

## 5. 축별 정량 검증 데이터

### 1) 축 1: 프롬프트 통합 경제성 (Prompt Consolidation)

* **가설 및 설계**: Smallville는 대화 턴마다 개별 API 호출(발화 생성, 감정 판정, 관계 갱신 등)을 수행하여 8턴 대화에 12~20회의 호출이 발생한다. Mundus Vivens는 단일 JSON 통합 스키마로 1회 호출에 모든 출력을 병합하여 91~95%의 API 호출을 절감할 수 있다.
* **실험 조건**: <!-- TODO -->
* **실측 결과**:
    ```
    <!-- TODO: 재측정 결과 붙여넣기 -->
    ```
* **아키텍처 의의**: <!-- TODO -->

---

### 2) 축 2: 기억 저장소 계층화 (Memory Architecture)

* **가설 및 설계**: Smallville는 모든 기억을 단일 Flat List에 저장하여 에이전트 수 × 기억 수에 비례하는 RAM을 점유한다. Mundus Vivens는 Hot Memory(RAM, 상한 40개)와 Cold Archive(LiteDB 디스크)로 이원화하여 RAM 점유를 고정(Clamped)한다.
* **실험 조건**: <!-- TODO -->
* **실측 결과**:
    ```
    <!-- TODO: 재측정 결과 붙여넣기 -->
    ```
* **아키텍처 의의**: <!-- TODO -->

---

### 3) 축 3: 성찰 스케줄링 (Reflection Scheduling)

* **가설 및 설계**: Smallville는 중요도 합산 ≥ 150 시 메인 루프를 블로킹하며 즉시 성찰을 수행한다. Mundus Vivens는 비동기 우선순위 큐에 삽입하고 백그라운드 워커 스레드에서 스로틀링(10 TPS) 처리하여 메인 틱 영향 0ms를 달성한다.
* **실험 조건**: <!-- TODO -->
* **실측 결과**:
    ```
    <!-- TODO: 재측정 결과 붙여넣기 -->
    ```
* **아키텍처 의의**: <!-- TODO -->

---

### 4) 축 4: 일과 생성 및 이동 비용 최적화 (Planning & Travel)

* **가설 및 설계**: Smallville는 매일 3단계 계획 수립(하루 전체 → 시간대별 → 5~15분 단위)을 에이전트마다 수행하여 30~40+회의 API 호출을 소비한다. Mundus Vivens는 1회 통합 성찰로 24시간 일과를 생성하며, 원정 이동 중에는 Auto-Continue로 100% 바이패스한다.
* **실험 조건**: <!-- TODO -->
* **실측 결과**:
    ```
    <!-- TODO: 재측정 결과 붙여넣기 -->
    ```
* **아키텍처 의의**: <!-- TODO -->

---

### 5) 축 5: 전투 및 위협 인지 파이프라인 (Combat & Threat)

Smallville 모델에는 부재하는 Mundus Vivens 독자 영역입니다.

#### 5a) 위협 판정 배치 (Threat Assessment Batch)

* **가설 및 설계**: 호감도, 신뢰도, 트라우마 기억 가중치를 합산하는 결정론적 수학 공식으로 API 0회 판정을 달성한다.
* **실험 조건**: <!-- TODO -->
* **실측 결과**:
    ```
    <!-- TODO: 재측정 결과 붙여넣기 -->
    ```

#### 5b) 인과 캐스케이드 (Causal Cascade)

* **가설 및 설계**: 연쇄 신념 갱신 시 재귀 알고리즘으로 확신도를 감쇠시켜, 다수 노드에 걸친 인과 전파를 API 0회 로컬 연산으로 해결한다.
* **실험 조건**: <!-- TODO -->
* **실측 결과**:
    ```
    <!-- TODO: 재측정 결과 붙여넣기 -->
    ```

---

### 6) 축 6: 종합 운영 비용 비교 (Total Cost Efficiency)

* **Smallville 25명 2일**: <!-- TODO -->
* **Mundus Vivens 25명 2일 (추정)**: <!-- TODO -->
* **비용 절감률**: <!-- TODO -->

---

## 6. 발견된 병목 및 개선 로드맵

### 1) 병목 분석

<!-- TODO: 재측정 결과를 바탕으로 실제 병목 기술 -->

* **LiteDB 동기 Eviction I/O**: <!-- TODO -->
* **Gemini API 네트워크 RTT**: <!-- TODO -->
* **기억 Recall 인덱스 성능**: <!-- TODO -->

### 2) 최적화 로드맵

#### ① 단기 최적화 방안 (Quick Win)
<!-- TODO -->

#### ② 중장기 최적화 방안 (Architecture Evolution)
<!-- TODO -->

---

## 부록 A. 비지성체(Non-Sentient) 인지 배제 아키텍처

늑대(`npc_wolf`)와 같은 야수/동물 계열은 지성체가 아니므로 LLM 대뇌 연산(성찰, 일과 계획, 대화)이 불필요합니다. 이를 아키텍처 수준에서 완전히 격리하였습니다.

### C++ 물리 서버 (척수)
* `SentienceComp.is_sentient = false`로 지정된 엔티티는 `SystemSocial.cpp`의 대화 주도자/타겟 후보군에서 원천 배제.
* 행동은 오직 로컬 Behavior Tree(Needs 센서, 공격 본능)에 의해서만 구동.

### C# AI 서버 (대뇌)
* `Persona.IsSentient = false`로 지정된 에이전트는 `DailyPlanService.EnqueueReflection()` 진입 시 즉시 바이패스.
* LLM API 호출 0회, 고정된 배회/습격 로컬 스케줄을 즉시 주입.

### 비용 절감 효과
<!-- TODO: 비지성체 N마리 기준 절감 수치 -->

---

## 부록 B. Smallville 논문 아키텍처 상세 비교표

<!-- TODO: Smallville 논문 재정독 후 차원별 상세 비교표 작성 -->

| 아키텍처 차원 | Smallville | Mundus Vivens | 코드 참조 |
|---|---|---|---|
| 기억 저장 | Flat List (RAM 무제한) | Hot/Cold 2계층 (RAM 상한 고정) | `MemoryBox`, `PersistenceService` |
| 대화 처리 | 턴별 개별 API | 세션당 단일 JSON 통합 | `DialogueOrchestrator` |
| 성찰 트리거 | importance ≥ 150 (블로킹) | 비동기 큐 + 스로틀링 (비차단) | `DailyPlanService` |
| 일과 계획 | 3단계 세분화 (다회 API) | 1회 통합 생성 + Auto-Continue | `DailyPlanService` |
| 위협/전투 | 미지원 | 결정론적 수학 공식 (API 0회) | `InteractionScheduler` |
| 인과 전파 | 미지원 | 재귀 Cascade + 확신도 감쇠 | `BeliefEngine` |
