# Mundus Vivens: 프로젝트 비전 및 시스템 개요 (Project Overview)

본 문서는 **Mundus Vivens** 프로젝트의 기획적 비전, 핵심 게임플레이 루프, 그리고 전체 시스템 아키텍처의 큰 틀을 설명합니다.

---

## [1] 프로젝트 비전 및 정체성 (Project Vision)

*   **장르**: 중세 판타지 마을 배경의 관찰형 자율 사회 시뮬레이션 샌드박스
*   **세계관**:
    *   NPC들은 이익, 호감도, 소문과 평판에 따라 자율적으로 삶을 살아가는 현실적이고 유기적인 중세 사회를 형성합니다.
    *   외부인(플레이어)에 대해 경계심을 가지며, 정보 교환과 신뢰 관계를 통해 협동하거나 갈등을 겪습니다.
*   **핵심 가치**:
    *   **자율성 (Autonomy)**: NPC는 독자적으로 하루 일과를 수행하고, 실시간으로 감정 상태를 갱신하며, 대화 상대에 따라 소문을 변형하여 전파합니다.
    *   **유기적 창발성 (Emergence)**: NPC 개인의 성향과 관계망에 따라 다자간 대화나 돌발적인 행동 변화 등 다양한 사회적 상호작용이 유도됩니다.

---

## [2] 핵심 게임플레이 루프 (Gameplay Loop)

플레이어는 세계의 관찰자이자 인지적/물리적 개입자 역할을 수행합니다.

*   **관찰 및 추적**: NPC들의 이동 상태 및 실시간 대화 발생 여부를 모니터링합니다.
*   **대화 개입**: NPC에게 말을 걸어 소문을 퍼트리거나, 이간질을 하거나, 신뢰 관계를 형성하여 사회적 관계망을 뒤흔듭니다.
*   **물리적 개입**: 플레이어가 NPC를 직접 공격하는 등의 위협 상황을 조성하면, NPC는 자율 행동을 즉시 중단하고 도망치거나 방어 행동을 취합니다.

---

## [3] 개발 및 아키텍처 철학 (Development Philosophy)

*   **실시간성과 대규모 확장성**:
    *   초당 수십 번 작동하는 실시간 물리 연산과 병목이 발생할 수 있는 비동기 AI 추론(네트워크)의 라이프사이클을 스레드 레벨에서 엄격하게 격리합니다.
    *   NPC 개체 수와 동시 접속 처리를 극대화할 수 있도록 경량 엔티티 컴포넌트 시스템(ECS) 및 스레드 격리형 비동기 통신을 기저 설계로 삼습니다.
*   **실용적이고 지속 가능한 AI 비용 제어**:
    *   대규모 언어 모델(LLM) API 호출 비용과 컨텍스트 윈도우 한계를 통제하기 위해, 자주 접근하는 기억(단기 기억)과 로컬 데이터베이스에 보관되는 기억(장기 기억)을 계층화하여 캐싱합니다.
    *   물리적 생존 수치가 급격히 저하되거나 단순 원정 이동 중일 때는 불필요한 AI 호출을 생략(우회)하는 등 효율적인 리소스 관리 정책을 유지합니다.

---

## [4] 고수준 시스템 구조 (High-Level Framework)

전체 시스템은 **육체(C++ 게임 서버)**와 **뇌(C# AI 서버)**가 물리적으로 분리된 분산 네트워크 구조로 동작합니다.

```
[ C++ 게임 서버: 육체 (Body) ]
   - 실시간 물리/공간 시뮬레이션, 길찾기, 인접 구역 감지
   - 플레이어 세션 수립 및 위협 인터럽트 처리
        ▲
        │ (비동기 gRPC 통신 파이프라인)
        ▼
[ C# AI 서버: 뇌 (Brain) ]
   - 페르소나 및 감정 왜곡 기반의 AI 대화/성찰 추론
   - 기억 보관소(Memory Box) 및 장기 계획 스케줄러 구동
```

---

## [5] 사양서 지도 (Map of Specs)

Mundus Vivens 개발에 활용되는 주요 설계 문서들의 배치 지도입니다.

*   [00_project_overview.md](file:///c:/Users/user/Documents/GitHub/Obsidian.Agent/MundusVivens/docs/00_project_overview.md): **[현재 문서]** 프로젝트 비전 및 시스템 고수준 개요
*   [01_architecture.md](file:///c:/Users/user/Documents/GitHub/Obsidian.Agent/MundusVivens/docs/01_architecture.md): C++ 물리 엔진 아키텍처 (스레드 모델, 틱 동기화 및 2단계 Job 수거 흐름, 대화 확률 연산, 길찾기 최적화)
*   [02_agent_design.md](file:///c:/Users/user/Documents/GitHub/Obsidian.Agent/MundusVivens/docs/02_agent_design.md): C# 인지/기억 엔진 아키텍처 (MemoryBox 구조, Hot/Cold 기억 보관 규칙, 일일 스케줄러 및 본능 인터럽트 규칙)
*   [03_future_roadmap.md](file:///c:/Users/user/Documents/GitHub/Obsidian.Agent/MundusVivens/docs/03_future_roadmap.md): 프로젝트 미래 백로그 및 C++ vs C# 개발 책임 경계선 명세
