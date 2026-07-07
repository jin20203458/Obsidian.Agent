# Mundus Vivens: Project Vision, Purpose & Gameplay Overview

본 문서는 Mundus Vivens 프로젝트의 기획적 비전, 게임플레이 루프, 그리고 개발 철학을 설명합니다.

## 1. 프로젝트 비전 및 장르 (Project Vision & Genre)

- **장르**: 중세 판타지 마을 배경의 관찰형 자율 사회 시뮬레이션 샌드박스
- **세계관 및 분위기**:
  - NPC들은 이익, 호감도, 소문과 평판에 따라 자율적으로 살아가는 현실적이고 유기적인 중세 사회를 이룹니다.
  - 외부인(플레이어)에 대해 경계심을 가지며, 정보 교환과 신뢰 관계를 통해 협동하거나 솔직한 감정을 표출합니다.
- **핵심 가치**:
  - **자율성 (Autonomy)**: NPC는 독자적인 하루 일과를 수행하고, 감정 상태를 갱신하며, 대화 상대에 따라 소문을 변형하여 전파합니다.
  - **유기적 창발성 (Emergence)**: NPC 개인의 성격(외향성 등)과 호감도/신뢰도 관계망에 따라 다양한 사회적 상호작용이 유도됩니다.

## 2. 플레이어 인터랙션 루프 (Gameplay Loop & Player Interaction)

플레이어는 세계의 관찰자이자 개입자(Interrupter) 역할을 수행합니다.

- **관찰 및 추적**: NPC들의 이동(`PlayerMoveRequest`) 및 대화 이벤트(`DialogueEventPayload`)를 실시간으로 모니터링합니다.
- **직접 대화 개입**: NPC와 대화 세션(`PlayerDialogueSession`)을 수립하여 소문(정보)을 주입하거나 이간질, 신뢰 형성 활동을 전개합니다.
- **물리적 환경 개입 (위협 및 인터럽트)**: 플레이어의 공격 등 위협 감지 시, NPC는 진행 중이던 자율 스케줄이나 LLM 대화 추론을 즉시 중단(Interrupt)하고 도망치거나 전투 행동을 시작합니다.

## 3. 개발 및 아키텍처 철학 (Development Philosophy)

- **대규모 확장성 및 고성능 지향 (High Performance & Scalability)**:
  - C++ 게임 서버의 3-스레드 리액터 모델, EnTT ECS, 공간 해시 그리드 등을 바탕으로, 동시 접속자 수 및 NPC 개체 수가 극대화되는 대규모 시뮬레이션을 전제로 코드를 최적화합니다.
  - 성능 병목을 최소화하기 위해 동적 락(Lock)을 배제하고 비동기 네트워크 파이프라인(agrpc, Asio)을 설계 기저로 삼습니다.
- **실용적인 인지 엔진 및 자원 제약 관리**:
  - LLM API 호출 비용 및 컨텍스트 윈도우 한계를 통제하기 위해 NPC의 활성 기억(`MemoryBox`)을 40개 한도로 제한하는 Hot Memory 계층을 설계했습니다.
  - 방출된 기억은 로컬 저장소(**LiteDB**)의 `cold_archive`로 이전하고, 필요할 때만 가중치 연산으로 재소환하는 하이브리드 구조를 통해 장기 구동 안정성을 유지합니다.

## 4. 시스템 구조 개요 (High-Level Framework)

전체 시스템은 **육체(C++ Game Server)**와 **뇌(C# AI Server)**가 물리적으로 분리된 분산 네트워크 구조로 동작합니다.

```
[ C++ Game Server: 육체 (Body) ]
   - 20Hz Lock-Free 물리 시뮬레이션, 길찾기, 인접 구역 감지
   - 플레이어 세션 수립 및 위협 인터럽트 처리
        ▲
        │ (gRPC 통신 파이프라인 - mundus_vivens.proto)
        ▼
[ C# AI Server: 뇌 (Brain) ]
   - Gemini API 기반의 페르소나/감정 왜곡 대화 추론
   - LiteDB 기반의 연상 기억 회상 및 일일 계획 수립(Reflection)
```
