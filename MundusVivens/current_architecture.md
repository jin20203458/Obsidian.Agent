# Mundus Vivens: 시스템 아키텍처 포털 (Current Architecture Map)

본 문서는 **Mundus Vivens** 프로젝트의 물리/공간 시뮬레이션 서버(C++) 및 인지/기억 시뮬레이션 서버(C#) 간의 분산 아키텍처와 상호작용 흐름을 한눈에 조망하기 위한 지도(Map of Content)입니다. 

실제 작동하는 구현 레벨의 상세 아키텍처 명세 및 기술 수식은 소스 코드 저장소 내부의 **단일 진실의 원천(SSOT, Single Source of Truth)** 문서를 단방향 링크로 참조하여 일치성을 유지합니다.

---

##  아키텍처 상세 명세 바로가기 (Single Source of Truth)

각 영역의 구체적인 클래스 설계, 물리 공식, 데이터 흐름 다이어그램은 아래의 기준 문서들을 참조하십시오. (Obsidian 내에서 링크를 클릭하면 즉시 해당 스펙 문서로 이동하여 편집 및 조회가 가능합니다.)

###  1. C++ 게임 서버 (물리 및 척수 엔진)
* **주요 역할**: 20Hz Lock-Free 물리 시뮬레이션, EnTT ECS 관리, Spatial Hash Grid 인접 스캔 및 A* 경로 탐색, 실시간 감정 전염 연산, 대화 주도 및 수락/합류의 수학적 판정.
* **상세 스펙**: [01_architecture.md](01_game_server_architecture.md.md)
  * [3-스레드 멀티 리액터 스레드 모델](01_game_server_architecture.md.md#1-3-스레드-멀티-리액터-모델-thread-model)
  * [ProcessWorldTick 10초 주기 동기화 흐름도](01_game_server_architecture.md.md#2-틱-동기화-및-관계-델타-데이터-흐름-processworldtick-flow)
  * [대화 주도/수락/다자간 합류 확률 공식 및 트리거 흐름도](01_game_server_architecture.md.md#3-대화-트리거--다자간-합류-로직-dialogue-trigger-logic)

###  2. C# AI 서버 (대뇌 및 인지 엔진)
* **주요 역할**: Gemini LLM API 연동 대화 추론, 소문 전파 및 왜곡, MemoryBox의 Hot 기억 관리, API 429 완화를 위한 스로틀링 큐, 성찰(Reflection) 및 일간 스케줄링(DailyPlan).
* **상세 스펙**: [02_agent_design.md](docs/02_agent_design.md)
  * [뇌-육체 분산 아키텍처 제어 블록도](docs/02_agent_design.md#1-분산-아키텍처-및-역할-분담-구조)
  * [통합 믿음(Belief) 컴포넌트 모델 설계](docs/02_agent_design.md#a-통합-믿음-모델-beliefcs)
  * [현저성 감쇠(Salience Decay) 틱 규칙 테이블](docs/02_agent_design.md#d-틱-기반-현저성-감쇠-salience-decay)
  * [DailyPlan 4시간 전 예측형 백그라운드 큐 및 시차 분산 시스템](docs/02_agent_design.md#b-api-429-완화-및-예측형-큐잉-chunk-a)

###  3. 장기 기억 보관 및 회상 (Cold Archive & Heuristic Recall)
* **주요 역할**: Hot Memory(`MemoryBox` 40개 한도)를 초과해 방출(Eviction)되는 기억의 **LiteDB 영구 저장소(`cold_archive`) 이관**, 조우/장소 진입 시 **Heuristic Scoring 기반의 연상기억 회상(Recall)** 복구 루틴, **신념 인과 전파(Causal Cascade) 도미노 감쇠** 알고리즘.
* **상세 스펙**: [02_agent_design.md](docs/02_agent_design.md#b-믿음-중요도importance-수식-및-예산-도태-eviction-와-영구-보관-archive-chunk-b)
  * [Heuristic Recall 매칭 및 Scoring 가중치 공식](docs/02_agent_design.md#b-믿음-중요도importance-수식-및-예산-도태-eviction-와-영구-보관-archive-chunk-b)
  * [신념 인과망 연쇄 파기 (Causal Cascade) 처리](docs/02_agent_design.md#c-신념-인과-도미노-전파-causal-cascade-chunk-b)

###  4. 물리 본능 FSM 및 스마트 사물 상호작용 (Needs & Affordance)
* **주요 역할**: C++ 20Hz 틱 수준의 허기/피로 실시간 감쇠, 임계치(15.0f) 도달 시 C# 스케줄 일시 중단 및 충전 거점 이동(**Survival Override**), C# 대뇌 락 및 성찰 우회(요금 절약), Smart Object(가구) 점유 및 충전(**Affordance Resolver**).
* **상세 스펙**: [02_agent_design.md](docs/02_agent_design.md#6-물리적-본능-오버라이드-및-사물-어포던스-chunk-c)
  * [Needs FSM 본능 제어 루프 및 감쇠/충전 규칙 테이블](docs/02_agent_design.md#6-물리적-본능-오버라이드-및-사물-어포던스-chunk-c)

---

##  부록: 스탠퍼드 스몰빌(Generative Agents)과의 효율성 비교

Mundus Vivens의 통합 오케스트레이션 설계는 구형 Generative Agent 모델의 치명적인 비용 문제를 개선했습니다.

* **8턴 대화 1회당 API 호출 비교**:
  * **Stanford Smallville**: **최소 12회 ~ 최대 20회** (턴당 LLM 질의, 사후 요약, 연쇄 성찰)
  * **Mundus Vivens**: **단 1회** (Gemini JSON 모드 통합 질의로 대본, 감정 변화, 관계 변화, 스케줄을 일괄 도출)
* **성찰(Reflection) 처리**:
  * **Stanford Smallville**: 누적 150점 도달 시 실시간 강제 API 호출로 연쇄 지연 유발.
  * **Mundus Vivens**: 자정 백그라운드 큐 스로틀링(10 TPS) 및 시차 분산 예측 연산으로 API 429 차단.