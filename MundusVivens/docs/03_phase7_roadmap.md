# Mundus Vivens: Phase 7 Realism Expansion Roadmap & Progress Report

본 문서는 AI 에이전트 아키텍처 연구를 바탕으로 수립한 **Phase 7 개발 과제의 구현 성과 보고 및 잔여 로드맵(Progress & Living Roadmap)**입니다. 

---

## 📊 Phase 7 구현 성과 보고 (Implementation Status)

대규모 구동 안정성 확보(Chunk A), 기억 및 인지 정합성 고도화(Chunk B), ECS 기반 생체 FSM 및 사물 상호작용(Chunk C)을 통해 기존에 수립된 Phase 7 목표의 핵심 기능들이 **성공적으로 전면 구현 완료**되었습니다.

### 🧱 Chunk A (인프라 & 시각적 안정성) ➔ `[IMPLEMENTED]`
* **지연 시간 마스킹 (`SystemBusyAmbient`)**: gRPC/성찰 통신 동안 NPC가 제자리에 돌부처처럼 굳어 마비되는 현상을 없애기 위해 C++ 서버 ECS 내에 `BusyTag`를 설계하고, 대기 시간 동안 서성이거나 생각하는 로컬 행동 연출을 실시간 동기화 마스킹 처리 완료.
* **예측형 더블 버퍼링 및 API 스로틀링 워커**: 자정 API 폭주(API Spike)를 예방하기 위해 첫날 일과 시간을 시차 분산하였으며, 일정 만료 4시간 전에 디바이스 큐(`PriorityQueue`)에 성찰 요청을 밀어 넣고 최대 10 TPS 스로틀링 및 지수 백오프를 통해 순차 처리 후 스케줄을 즉시 Swap 처리하도록 설계 완료.

### 🧠 Chunk B (기억 & 자아 파이프라인 개편) ➔ `[IMPLEMENTED]`
* **LiteDB Cold Archive & Heuristic Recall**: 주 기억(`MemoryBox`) 한도 초과(40개) 시 중요도가 낮은 기억을 LiteDB 영구 보관소(`cold_archive`)로 이관하고, 조우 시 대상/장소/최신성 가중치 점수를 합산해 동적으로 Working Memory에 재회상(Recall)하는 엔진 구현 완료.
* **신념 인과 도미노 전파 (Causal Cascade)**: 신념 데이터에 `DerivedFrom`, `SupersededBy` 필드를 연동해 상위 핵심 사실이 변경될 시 하위 자식 신념들의 확신도(`Confidence`)를 자동으로 비례 감쇠시키는 인과망 전파 알고리즘 구현 완료.
* **Zero-Cost 관계 인상 주입 (`ImpressionSummary`)**: 자정 성찰 LLM 호출 DTO를 최적화하여 별도의 API 호출 비용 없이 대화 참가자에 대한 관계 인상 요약본을 누적 업데이트하고 대화 시 프롬프트에 주입하는 하이브리드 중기기억 구축 완료.

### 🏃 Chunk C (실시간 환경 상호작용 및 생체 FSM) ➔ `[IMPLEMENTED]`
* **스마트 오브젝트 (Affordances)**: `mundus_vivens.proto` 내 가구 정보(`FurnitureInfo`) 전송 규격을 구현하여 C++ 서버가 각 Zone 내의 사물(의자, 침대, 식탁 등)의 3D 물리 공간 좌표를 완벽히 인지하도록 연동 완료.
* **Need FSM & Survival Override**: C++ 서버 20Hz 틱 수준에서 허기/피로가 임계치(15) 아래로 하락 시 C# Job을 즉시 중단(gRPC `ReportJobStatus`)하고 가까운 거점의 스마트 오브젝트로 이동하는 척수 반사 오버라이드 구현 완료.
* **대뇌 락 및 성찰 우회**: 생체 해결 시점에 C# 서버는 고비용의 LLM 성찰을 우회하여 비용을 절약하고, NPC를 즉시 해제하여 해결이 완료될 때까지 C# 스케줄 배포를 보류하는 안정화 로직 구축 완료.

---

## 🗺️ Future Roadmap (향후 개발 과제)

Chunk A~C가 완벽히 다져진 하이브리드 토대 위에서, 한 단계 높은 창발적 디테일을 위해 다음 페이즈에 구현할 과제 리스트입니다.

### 1. C++ 척수 엔진 추가 증축 과제 (C++ Focus)
* **로컬 행동 트리 (Behavior Tree/Utility AI) 엔진 탑재**: 척수 레벨의 이동/생체 FSM을 체계화하기 위해 가벼운 행동 트리 엔진을 C++ 내부에 심어, 돌발 위험(공격, 폭발 등) 시 C# 명령을 즉시 강제 취소하고 로컬 대피/방어를 50ms 이내로 처리.
* **지각/감각 시스템 (Perception FOV)**: 공간 인접 판단을 넘어, 에이전트의 시야각(Field of View), 장애물 차폐(Raycasting), 소음 전파 알고리즘을 20Hz로 처리하여 "등 뒤의 소리를 들었는지", "내 시야 범위 내에서 훔쳤는지" 등의 입체 감각 정보 전달.

### 2. 고차원 인지 및 사회성 확장 과제 (C# Focus)
* **의견 분기 (Value Axes) 및 선호 장소 (Place Attachment)**: 에이전트마다 3대 가치관 수치(전통vs진보 등)와 선호 장소 필드를 부여하여, 아침 스케줄 생성 시 선호 장소 중심의 **생활 반경**이 자연스레 형성되게 하고 대화 시 가치관 차이에 의한 **사상/종교 갈등 대화**를 묘사.
* **대화 맥락 힌트 (Context Seed)**: C++ 서버가 각 Zone별 최근 발생한 물리 사건 로그(원형 버퍼)를 수집해 gRPC로 넘겨주면, 대화 시작 시 해당 사건을 자연스럽게 첫마디(Seed)로 언급하게 프롬프트 가이드 주입.
* **자발적 동적 목표 (Autonomous Goal Tracking)**: C#에서 발급한 `JobComp.intent` 및 `target_agent_id` 데이터를 활용해, C++ 서버에서 A* 경로 탐색의 목적지가 고정된 좌표가 아니라 **추적 대상 NPC의 실시간 3D 위치**를 동적으로 쫓아가 만나게 하는 알고리즘 고도화.
* **소셜 네트워크 분석 기반 지위 부여 (Social Role Graph)**: NPC 간의 호감도 매트릭스를 분석하여 인기인, 소외자, 라이벌 등의 동적 사회적 지위(Status) 태그를 연산하고 이를 대화 톤앤매너 프롬프트에 동적 반영.
* **소문의 꽃 (Gossip Legend)**: 왜곡 횟수(`MutationCount`)가 누적(예: 3회 이상)되거나 중요도 0.95 이상인 사건을 '전설'로 승격시켜 시간 감쇠(Decay) 면제권을 세팅하고, 장기 기억(Cold Archive)에 박제하여 마을의 고유 역사(Lore)로 남기는 시스템 구축.
