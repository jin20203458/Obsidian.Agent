# Mundus Vivens: 트러블슈팅 및 런북 (Troubleshooting & Runbook)

본 문서는 Mundus Vivens 프로젝트(C# AI Server & C++ Game Server) 개발 과정에서 발생한 시스템 환경 오류, 빌드 에러, gRPC 프로토콜 및 아키텍처 정합성 관련 주요 문제와 그 해결 과정을 누적 기록하는 문서입니다.

---

## 2026-07-07: 생체 욕구(Hunger/Fatigue) 및 계획 행동 연동 오작동

### 1. 무제한 가구 탐색 거리로 인한 영구 유턴/이동 루프 (Infinite-Distance Loop)

#### 현상 (Symptom)
* 황무지 장거리 원정(예: 1.5km 이동) 중인 NPC가 생체 욕구 수치 위기(15% 미만)에 봉착했을 때, 자율 야영을 시작하지 않고 다시 아주 먼 거리에 있는 마을 내 빈 가구(성당 의자, 술집 침대 등)를 목적지로 설정하여 반대 방향으로 유턴을 시작하는 문제.
* 원정 스케줄 시간 만료 ➔ AI 서버 스케줄 연장(오토런) ➔ 감쇠 ➔ 기아 위기로 인한 마을 복귀 시도 ➔ AI 스케줄 재연장의 루프에 빠져 영원히 목적지에 도달하지 못함.

#### 원인 (Root Cause)
* C++ `SystemSurvivalOverride`에서 생존용 가구를 탐색할 때, 월드에 등록된 전체 가구 목록 중 비어 있는 것을 거리 제한 없이 최단 거리(`min_dist`) 기준으로만 선별함.
* 주변에 가구가 전혀 없더라도 1km 이상 멀리 떨어진 가구가 매칭되어 척수가 이동 명령을 덮어써 버림.

#### 해결책 (Resolution)
* 가구 매칭 시 최대 거리 필터(**`min_dist < 50.0f`**)를 적용하여, 50m 이내에 사용 가능한 가구가 없을 경우 마을 밖 황무지(Wilderness)로 판정하도록 제한을 둠.
* 50m 이내에 가구가 없을 경우, 현 위치에 `임시 야영지` 또는 `임시 모닥불`을 동적으로 스폰하여 제자리에서 자율 야영을 하도록 예외 처리를 유도함.

---

### 2. 허기/피로 욕구 비동기 엇박자로 인한 출발 즉시 유턴 (Journey Prep Desync Loop)

#### 현상 (Symptom)
* NPC가 마을 여관에서 수면(피로 충전)을 마치고 아르카디아 제국 등으로 출발하자마자 10~20틱 이내에 다시 허기 위기가 터져 마을로 되돌아오고, 밥을 먹고 출발하면 다시 피로 위기가 터져 되돌아오는 무한 왕복 서성임 현상.

#### 원인 (Root Cause)
* 허기와 피로 회복/소모는 물리 틱 상에서 완전히 비동기적으로 처리됨.
* 피로가 95%가 되어 잠에서 깼을 때 허기가 이미 25~30% 근처였다면, C# AI 대뇌가 장거리 원정 계획을 할당하고 마을을 미처 탈출하기도 전에 허기가 15% 임계점을 돌파하게 됨.
* 또한, 계획 수립 후 대화 및 대기 시간 동안 마을 안에서 실시간으로 수치가 지속 감쇠되어 최악의 타이밍에 국경을 넘기 시작함.

#### 해결책 (Resolution)
* `SystemJobDriver`에서 150m를 초과하는 장거리 이동(`Moving`) 명령이 새롭게 주입되는 시점에 한하여, 척수 레벨에서 **허기와 피로도를 즉시 100% 완충**해 주는 **🎒 원정 준비(Journey Preparation)** 단계를 추가함.
* 이를 통해 장거리 길을 떠나기 전 신체 스태미나 정렬(Alignment)을 보장함.

---

### 3. C# AI 대뇌 - C++ 물리 서버 간 자연어 계획(Intent) 키워드 불일치

#### 현상 (Symptom)
* C# AI 서버에서 NPC 스케줄에 맞춰 `"취침 및 휴식"`, `"취침 및 정비"` 등의 잠자리 Job을 정상 발급했고 NPC가 목적지 구역에 도착했음에도, 침대를 점유하지 않고 구석에 멍하니 서 있는 채로 피로도가 계속 감쇠되는 현상.

#### 원인 (Root Cause)
* C++ `SystemAffordanceResolver`에서 일반 일과(Schedule) 행동이 수면/식사인지 판단하기 위해 `job.intent` 내의 자연어 문자열 키워드를 매칭함.
* 1차 구현 시 `"Sleep"`, `"sleep"`, `"잠"`, `"수면"` 등의 키워드만 등록해 둔 반면, C# 스케줄러(및 Gemini LLM)가 생성하는 기본 하루 계획 템플릿에는 **`"취침"` (Korean: Sleeping)** 단어가 주로 사용되어 매칭에 실패함.

#### 해결책 (Resolution)
* C++ 서버 내의 수면 및 식사 식별 키워드 풀을 한국어 동의어 및 유사어까지 확장 적용함.
  * **수면(Sleep) 매칭 키워드:** `"Sleep"`, `"sleep"`, `"잠"`, `"수면"`, **`"취침"`**, `"휴식"`, `"노숙"`, `"rest"`
  * **식사(Eat) 매칭 키워드:** `"Eat"`, `"eat"`, `"밥"`, `"식사"`, `"음식"`, `"먹기"`, `"취식"`
* 향후 유지보수 시, 단순히 자연어 파싱에 의존하기보다 gRPC Job 프로토콜에 명시적인 `JobCategory` enum 필드를 도입하는 아키텍처 개선을 권장함.

---

## 2026-07-07: 감정 쇠퇴 및 전염 시스템의 정수 ID 기반 고속화 최적화

### 현상 (Symptom)
* 20Hz(50ms)로 작동하는 물리 메인 루프(`SystemEmotionDecay`) 내에서, 모든 NPC의 감정 상태를 스캔하고 부정 감정을 필터링할 때 매 틱마다 `std::string::find` 및 `std::string` 비교 연산이 다수 발생하여 CPU 연산 부하가 누적됨.
* 특히 LLM(C#)의 창발적 감정 문자열 대응을 위해 감정을 동적으로 레지스트리에 추가함에 따라, 고정된 문자열 매칭으로는 신규 감정(예: `"분노함"`, `"엄청분노함"`)의 자연 쇠퇴 및 전염 조건 처리에 한계가 있었음.

### 해결책 (Resolution)
* **카테고리 기반 O(1) 분류 및 정수화**:
  * `EmotionCategory` 열거형(`Neutral`, `Anger`, `Hostility`, `Fear`)을 추가하고, `EmotionRegistry`에 감정 ID별 카테고리를 저장하는 `category_table`(`std::vector<EmotionCategory>`)을 도입함.
  * C# gRPC 응답으로 신규 감정 문자열이 수신되어 레지스트리에 등록되는 시점(최초 1회)에만 `.find("분노")` 등의 문자열 검사를 수행하여 카테고리를 분류하고 `category_table`에 저장함.
  * 물리 틱 루프(`SystemEmotionDecay`) 내부에서는 문자열 연산 없이 `category_table[current_emotion_id]`를 통해 O(1) 정수 비교로 부정 감정 여부를 식별함.
* **출력 감정 ID 캐싱**:
  * 전염의 결과로 주입되는 표준 감정인 `"불안"`, `"경계"`의 정수 ID를 `SystemEmotionDecay` 진입 시 `static` 로컬 변수에 최초 1회 캐싱하여 틱 내 문자열 연산을 0으로 만듦.
* **쇠퇴 판정**:
  * `current_emotion_id != base_emotion_id` 정수 비교 판정을 수행하고, 로그 출력 시에만 역방향 조회로 문자열을 출력하도록 개선함.

---

## 2026-07-10: C++ 물리 전투 및 도주(BT) 구현 과정의 오류들

### 1. 도주 목적지 좌표의 맵 경계 이탈 및 `(0, 0)` 수렴으로 인한 Wilderness 거점 에러

#### 현상 (Symptom)
* 전투 중 HP가 30% 미만인 NPC가 `ActionFlee` (도주) 상태로 진입했을 때, 콘솔에 `❌ [GridMap 에러] 거점 좌표 조회 실패: 'Wilderness'을 찾을 수 없습니다.` 경고 로그가 지속적으로 발생하고 캐릭터가 비정상적인 경로로 이동하는 현상.

#### 원인 (Root Cause)
* 타겟의 반대 방향으로 도주 벡터를 계산할 때, 맵 구석이나 경계면 바깥(x < 0 또는 z < 0)으로 목적지가 잡히면 `IsWalkable` 검사가 실패함.
* 45도 회전 우회 테스트 결과도 모두 맵 바깥이라 `!found` 상태에 도달하면 `loc.x`/`loc.z`를 그대로 대입하는데, 캐릭터가 이미 `(0,0)` 구석으로 밀린 상태라면 도주 대상이 `(0,0)`으로 잡힘.
* `SystemPathfinding`은 목적지가 정확히 `(0,0)`이고 `target_location` 이름이 지정되어 있으면 거점 검색을 유발하는데, `Wilderness`의 거점 데이터가 존재하지 않아 에러를 출력하고 길찾기가 오작동함.

#### 해결책 (Resolution)
* `ActionFlee` 내부에서 도주 목적지(`dest_x`, `dest_z`) 및 회전 우회 경로(`test_x`, `test_z`), 그리고 최종 실패 폴백(`dest_x = loc.x`) 대입 부분 전체에 **`std::clamp(값, 5.0f, 1995.0f)`** 제약을 걸어 맵 안의 안전 영역 내로 강제 보정함.

### 2. 전투 중인 NPC의 잡담(대화) 및 고차원 스케줄 전환에 의한 교전 인터럽트

#### 현상 (Symptom)
* 두 NPC가 강제로 전투(교전) 상태에 들어갔음에도, 근접했을 때 갑자기 무기를 거두고 1:1 일상 잡담 대화에 참여하거나 C#에서 발급하는 스케줄을 처리하기 위해 맵 반대편으로 떠나는 현상.

#### 원인 (Root Cause)
* C++ `SystemSocialInteraction` (소셜 잡담) 및 `SystemJobDriver` (고차원 스케줄링) 시스템에서 NPC의 교전 여부(`CombatComp.target_entity != entt::null`)를 확인하지 않아 전투 행동보다 스케줄이나 대화 요청이 우선되어 일어나는 뇌 정지 및 인터럽트 현상.

#### 해결책 (Resolution)
* `SystemSocial.cpp`에서 대화 참여 후보 수집 시 `CombatComp`에 타겟을 둔 캐릭터를 원천적으로 `continue`로 배제함.
* `SystemJobDriver.cpp`에서도 사망자 체크와 더불어 교전 중인 NPC는 고차원 스케줄 상태 전환을 스킵하도록 예외 조항을 추가하여 척수 반사 수준의 물리 전투 흐름의 영속성을 보장함.

---

## 2026-07-12: 몬스터 이동 로그 개선 및 사망 개체 대화 스팸 루프 해결

### 1. 비지성체(몬스터/동물) 목적지 도달 시 "작업" 표현 노출 오류

#### 현상 (Symptom)
* 굶주린 늑대(Wolf) 등의 비지성체 몬스터가 목적지에 도착했을 때, 콘솔에 `🏁 [목적지 도착] 굶주린 늑대이(가) 목적지 [술집]에 도달하여 작업을 시작합니다.`라는 어색한 용어의 로그가 출력됨.

#### 원인 (Root Cause)
* C++ `SystemMovement.cpp`의 목적지 도착 판정 및 ToilState 전환부 로그에 `"작업을 시작합니다"`라는 표현이 모든 엔티티(지성체/비지성체 구분 없음)에 공용으로 사용되어 발생한 표현의 불일치.

#### 해결책 (Resolution)
* [SystemMovement.cpp](file:///C:/Users/adg01/Documents/GitHub/MundusVivens.GameServer.Cpp/SystemMovement.cpp)의 Direct-Seek 및 A* Path 도달 처리 로직 내에 `SentienceComp` 조회 단계를 추가함.
* `SentienceComp::is_sentient` 필드가 `false`인 비지성체 개체의 경우, 목적지 도달 로그의 명사를 `"작업"` 대신 `"행동"`으로 동적 대체하여 `"행동을 시작합니다"`로 보다 자연스럽게 출력되도록 보정함.

---

### 2. [치명적 버그] 사망한 개체에 대한 대화 트리거로 인한 20Hz 사망 처리 루프 스팸 (Dead-Entity Dialogue Loop)

#### 현상 (Symptom)
* 굶주린 늑대가 사살되어 사망한 이후, C++ 서버 콘솔에 `💀 [사망 처리 완료] 굶주린 늑대의 물리 시뮬레이션이 중단되었습니다.` 로그가 매 틱마다 폭포수처럼 무한히 반복 출력되는 런타임 성능 저하 및 로그 오염 현상 발생.

#### 원인 (Root Cause)
1. `SystemSocial.cpp` 내의 `SystemSocialInteraction` (1:1 및 다자간 대화 시작) 시스템에서 대화 주도자(`initiator`) 수집 단계에서는 사망 여부(`is_dead`)가 검사되었으나, **주변 이웃 후보자(`neighbor`) 수집 단계에서는 `is_dead` 검사가 누락**되어 있었음.
2. 이로 인해 살아있는 약탈자 고블린이 죽은 늑대를 대화 대상(Target)으로 선정하여 대화를 걸었고, 늑대의 `ActivityComp`가 `"대화 요청 중"`으로 바뀌고 `BusyTag`가 주입됨.
3. 20Hz로 작동하는 `SystemDeath`는 매 틱마다 늑대가 죽어있는데 액티비티가 `"사망"`이 아닌 것을 감지하고 다시 `"사망"`으로 강제 복구 및 로그 출력을 수행했고, C# AI 서버의 gRPC 배치 상태 동기화 응답 등과 얽혀 틱마다 강제 덮어쓰기 루프가 반복됨.

#### 해결책 (Resolution)
* [SystemSocial.cpp](file:///C:/Users/adg01/Documents/GitHub/MundusVivens.GameServer.Cpp/SystemSocial.cpp)의 대화 대상 이웃 후보군(`neighbor`) 스캔 루프 내에 아래와 같이 **사망자 제외(Filter-out) 조건식**을 추가함.
  ```cpp
  if (reg.all_of<HealthComp>(neighbor) && reg.get<HealthComp>(neighbor).is_dead) continue;
  ```
* 이를 통해 사망한 개체에게 불필요한 대화 요청 및 상태 변경이 차단되어, 스팸 루프 및 틱 지연 현상이 완벽히 해결됨.

---

## 2026-07-12: 생체 위기(Hunger/Fatigue) 로컬 BT 처리 시 틱 갱신 교착 상태 해결

### 현상 (Symptom)
* NPC가 생체 욕구(허기/피로) 위기로 인해 로컬 Behavior Tree(BT)를 통해 가상 Job(999000/999001)을 발급받아 식사/취침을 시작할 때, C++ 서버에서 `🏁 [목적지 도착 - Direct Seek] ...` 및 `🔄 [Toil Transition] ...` 로그가 20Hz 물리 틱마다 폭발적으로 반복 출력되며 시뮬레이션의 논리 틱(Tick) 및 실시간 로그 진행이 완전히 정체되는 현상.

### 원인 (Root Cause)
1. **C# 서버 스케줄 덮어쓰기**: C++ 서버의 `GetPendingJobsAsync` gRPC 콜백에서 C# 서버로부터 미완료 스케줄을 받아올 때, NPC가 로컬 생존 위기를 처리 중인 상태(`is_resolving_survival == true`)인지 여부를 검사하지 않음. 이로 인해 C++의 로컬 가상 Job(999000)이 매 틱마다 C#의 정기 스케줄 Job으로 덮어써지며 `toil.state`가 `Idle`로 강제 리셋됨.
2. **도착 판정 비교 오류**: `ActionMoveToTarget`에서 도착 완료 검사를 오직 `loc.location_name == job.target_location` 문자열 비교에만 의존함. 황무지(`Wilderness`) 등에서 로컬 가상 Job 실행 시 타겟 좌표로 직접 찾아가더라도 영역 식별 불일치(예: Wilderness vs 고블린 초소)로 인해 문자열 매칭이 영구 실패하여 `ActionMoveToTarget`이 `toil.state`를 계속 `Moving`으로 강제 전환하고 `SystemMovement`가 도착 처리(`Working` 전환 및 로그 출력)를 무한 반복함.
3. **가상 Job 타겟 초기화 누락**: `ConditionIsHungry` 및 `ConditionIsFatigued`에서 C++ 가상 Job을 주입할 때 기존에 할당되어 있던 `target_location` 및 좌표(`target_x/y/z`) 데이터를 비워주지 않아, `ActionFindFurniture`가 이미 타겟 설정이 완료된 것으로 판단하여 자율 야영(임시 모닥불/야영지 생성) 탐색 단계를 통째로 스킵함.

### 해결책 (Resolution)
1. **Pending Job 갱신 스킵**: `main.cpp`의 `GetPendingJobsAsync` 콜백 내부에 수신 예외 조건식을 추가하여, 현재 생체 욕구 위기를 로컬에서 처리 중인 NPC(`is_resolving_survival == true`)는 C# 스케줄 주입 대상에서 즉시 제외(`continue`)함.
2. **좌표 기반 도착 판정 추가**: [BehaviorTrees.h](file:///C:/Users/adg01/Documents/GitHub/MundusVivens.GameServer.Cpp/BehaviorTrees.h)의 `ActionMoveToTarget`에 목적지 좌표와의 물리적 거리 연산(`dist < 0.8f`)을 추가하여, 구역명 불일치 상황에서도 목적지에 도달 시 완벽하게 `Success`를 반환하도록 개선함.
3. **가상 Job 정보 초기화**: `ConditionIsHungry` 및 `ConditionIsFatigued` 가상 Job 주입 시 `target_location = ""` 및 target 좌표들을 `0.0f`로 완전 초기화하고 명시적으로 `JobCategory::Eat/Sleep`을 주입하여 가구 탐색 및 모닥불 자율 야영 스폰이 정상 동작하도록 수정함.

---

## 2026-07-12: 격자 구역 오븐베이크 음수 좌표 누락 및 맨바닥 행동(노숙) 로그 스팸 해결

### 1. 2D Bake Map 내의 음수 영역 좌표계 누락 및 Wilderness 판정 오류

#### 현상 (Symptom)
* 늑대 서식지(`Wolf Nest`)와 고블린 초소(`Goblin Outpost`)에 NPC들이 정상 스폰되었으나, 캐릭터들의 구역 컴포넌트(`LocationComp::location_name`)가 거점 구역명이 아닌 `"Wilderness"`(황무지)로 잘못 매핑되는 현상.
* 이로 인해 C# 서버의 본래 계획 구역과 C++ 물리적 구역의 이름이 일치하지 않아 행동 완료 여부 판단 및 로컬 대처에 부작용이 발생함.

#### 원인 (Root Cause)
* C++ `BakeRegionMap` 및 `GetLocationNameAt`에서 사용되는 격자 맵(`region_map_`) 배열은 양수 범위(`[0, 2000)`)만을 지원하도록 구현되었으나, `world_config.json` 상에서 늑대 서식지는 `(250, -250)`, 고블린 초소는 `(-250, 250)`과 같이 음수 영역 좌표를 사용하고 있었음.
* 음수 좌표 영역은 `std::max(0, ...)` 범위 클램핑 시 격자 인덱스 외부에 걸쳐 누락(`Bake` 누락)되었으며, 실시간 위치 갱신 조회 시에도 강제로 `0`으로 클램핑되어 구역명이 무조건 `"Wilderness"`로 어긋나 출력됨.

#### 해결책 (Resolution)
* C# `world_config.json` 파일의 `발레리아 왕국` 바운더리 하한(`MinBounds`) 및 고블린 초소와 늑대 서식지의 좌표를 모두 안전한 양수 공간으로 전면 조정함.
  * **늑대 서식지 (Wolf Nest)**: `(250.0, 0.0, -250.0)` ➔ **`(250.0, 0.0, 10.0)`**
  * **고블린 초소 (Goblin Outpost)**: `(-250.0, 0.0, 250.0)` ➔ **`(10.0, 0.0, 250.0)`**
* C# 데이터베이스 재설정 및 C++ 서버 동기화를 완료하여 양수 격자 공간 내에서 정상적인 거점명(`Wolf Nest`/`Goblin Outpost`) 매핑이 완료됨을 검증함.

---

### 2. 맨바닥 생체 욕구 해결(노숙 식사/취침) 시의 20Hz 로그 스팸 루프

#### 현상 (Symptom)
* NPC 또는 야생 동물이 근처에 침대나 식탁 등 가구가 없어 맨바닥에서 노숙 취침(`Sleeping_On_Floor`) 또는 바닥 식사(`Eating_On_Floor`)를 할 때, 20Hz 물리 틱마다 `"⛺ [BT 노숙 취침] ..."` 또는 `"🥪 [BT 바닥 식사] ..."` 로그가 쏟아져 나오는 로그 오염 현상.

#### 원인 (Root Cause)
* [BehaviorTrees.h](file:///C:/Users/adg01/Documents/GitHub/MundusVivens.GameServer.Cpp/BehaviorTrees.h)의 `ActionInteractFurniture::Tick`에서 `needs.occupied_furniture == entt::null` 검사를 매 틱마다 시도함.
* 맨바닥 행동 시에는 실제로 점유(Occupy)할 가구 자체가 없으므로 `occupied_furniture`가 계속 `entt::null`로 유지되어, 최초 프레임 판단 조건문 내의 로그 출력 및 Toil 초기화 코드가 매 틱 실행되었기 때문임.

#### 해결책 (Resolution)
* `ActionInteractFurniture::Tick`의 최초 진입(가구 점유 시도) 조건문에 아래와 같이 바닥 식사/취침 행동 중인지를 가려내는 필터식을 보완함.
  ```cpp
  if (needs.occupied_furniture == entt::null && 
      toil.current_action != "Eating_On_Floor" && 
      toil.current_action != "Sleeping_On_Floor")
  ```
* 이미 바닥에서 식사나 수면 모션을 유지 중인 캐릭터는 점유 시도 단계를 스킵하여 중복 로그 및 Toil 상태 강제 초기화가 유발되지 않고 1회만 기록되도록 교정함.

---

## 2026-07-12: 유니티 클라이언트 UI 업데이트 중단 및 UIManager 싱글톤 하이재킹 해결

### 현상 (Symptom)
* 유니티 클라이언트를 실행했을 때, 3D 뷰어 상의 NPC 캡슐들은 정상적으로 움직이고 백그라운드에서는 스냅샷 패킷을 계속 수신 중임에도 불구하고, 화면 상단의 Current Tick, 좌측의 실시간 로그 패널, NPC 클릭 시 노출되어야 할 상세 정보 패널 등이 전혀 갱신되지 않고 초기 상태(혹은 멈춘 상태)로 방치되는 현상.
* 유니티 콘솔 및 `Editor.log` 상에 NullReferenceException 등 관련 에러나 예외 발생이 전혀 없이 침묵함.

### 원인 (Root Cause)
* `SampleScene.unity` 씬 파일 내에 **동일한 `UIManager` 스크립트를 포함하는 서로 다른 두 개의 GameObject가 중복 존재**하고 있었음:
  1. 공백이 포함된 이름의 `"UIManager "` (ID 146213223) - 인스펙터 상의 UI 요소(`tickText`, `logText` 등)가 전혀 할당되지 않은 미완성 상태의 빈 오브젝트.
  2. 정상적인 이름의 `"UIManager"` (ID 583645022) - UI 오브젝트가 올바르게 할당된 실제 연동 오브젝트.
* 유니티가 씬을 로드하고 각 오브젝트의 `Awake()`를 실행할 때, 미완성 상태인 `"UIManager "`의 `Awake`가 먼저 호출되어 static 변수인 `UIManager.Instance` 싱글톤 참조를 선점(하이재킹)함.
* 실제 동작해야 하는 `"UIManager"`의 `Awake`가 실행되었을 때는 이미 `Instance`가 null이 아니므로 싱글톤 갱신을 생략함.
* 이로 인해 `PacketProcessor` 등 외부에서 `UIManager.Instance`를 통해 UI 변경을 시도할 때마다, 필드가 전부 null인 첫 번째 인스턴스를 호출하게 됨.
* 각 UI 메서드는 null 체크(`if (tickText != null)` 등)를 안전하게 수행하고 있었기에 예외를 던지지 않고 그냥 실행을 종료하여 아무런 에러 로그도 없이 화면만 갱신되지 않는 현상이 발생함.

### 해결책 (Resolution)
* [SampleScene.unity](file:///C:/Users/adg01/Documents/GitHub/MundusVivens.Unity/Assets/Scenes/SampleScene.unity) 파일을 직접 수정하여 미완성 상태였던 중복 오브젝트 `"UIManager "` (ID 146213223) 및 하부의 MonoBehaviour, Transform 컴포넌트를 완전히 삭제함.
* 씬의 루트 목록(`SceneRoots.m_Roots`)에서도 해당 오브젝트의 Transform ID (`146213225`) 엔트리를 삭제함.
* 이를 통해 씬 상에 유일한 UIManager 오브젝트만 남겨 싱글톤 선점 문제가 근본적으로 제거되었으며, 클라이언트 실행 시 정상적으로 Tick 수치, 실시간 연극 대사 로그, NPC 상세 패널이 실시간 갱신되는 것을 확인함.

---

## 2026-07-12: 한국어 폰트 깨짐, 3D 텍스트 겹침, 카메라 조작 및 대화 범위 한계 개선

### 1. 한국어 폰트(malgun SDF) 깨짐 및 셰이더 미호환 오류
#### 현상 (Symptom)
* 월드 상의 NPC 머리 위에 출력되는 한글 텍스트(이름, 상태, 말풍선 대사 등)가 모두 깨져서 노란색/주황색 점선이나 박스 모양의 심각한 그래픽 노이즈 형태로 출력됨.
#### 원인 (Root Cause)
* 한국어 폰트 파일인 `malgun SDF.asset`이 참조하는 기본 셰이더가 이전 빌트인 파이프라인 전용 셰이더(`TextMeshPro/Distance Field`)로 설정되어 있어, URP(Universal Render Pipeline) 환경과 호환되지 않아 그래픽이 완전히 깨져 렌더링됨.
#### 해결책 (Resolution)
* `malgun SDF.asset` 파일의 셰이더 참조 GUID를 URP 호환 셰이더(`TextMeshPro/Mobile/Distance Field` 또는 `TextMeshPro/Distance Field SSD` 등)인 `fe393ace9b354375a9cb14cdbbc28be4`로 파일 직접 수정을 통해 변경 및 재임포트 완료.

### 2. 카메라 조작성 및 줌인 한계 조절
#### 현상 (Symptom)
* 맵 전체가 너무 멀리(Y=150)에서 렌더링되어 관찰이 어렵고, 기존 줌 로직은 `transform.forward`에 의존하여 카메라 부모 오프셋 등에 의해 오작동함.
#### 해결책 (Resolution)
* `CameraController.cs` 내에서 카메라 높이 조절 기준의 최소 높이(`minHeight`)를 `20.0f`에서 `5.0f`로 낮춰 더 가까운 거리까지 줌인 가능하게 변경.
* 마우스 휠 외에 `Q`/`E` 및 `Page Up`/`Page Down` 키를 통한 키보드 줌 조작 추가.
* `transform.forward` 대신 카메라의 월드 Y 좌표를 물리적으로 직접 증감시키는 방식으로 줌 방식을 전면 개편하여 카메라 오프셋/회전 상태와 무관하게 부드럽게 높낮이가 조절되도록 수정.

### 3. 3D 공간 상의 이름, 상태, 대화 말풍선 중복 겹침 및 말풍선 글자 바코드 현상(찌그러짐) 오류
#### 현상 (Symptom)
* 카메라가 수직으로 월드를 바라보는 탑다운(Top-down) 뷰 형태이므로, 텍스트들의 월드 높이(Y축)에 차이를 주더라도 화면상에서는 같은 중심선에 겹쳐서 투영되어 모든 글씨가 중복되어 뭉쳐 보이던 현상.
* 또한, 동적으로 생성되는 말풍선 대사(`SpeechBubbleText`)가 정상적으로 읽히지 않고 얇은 노란색 세로줄(바코드 형태)로 심하게 압축되거나 비정상적으로 줄바꿈되어 출력되는 현상.
* `술집(Tavern)`과 `뒷골목(Back Alley)` 등 인접한 구역의 랜드마크 파란색 큐브 위에 출력되는 이름 글자(예: `"술집 (Tavern)"`)가 심하게 겹치고 뭉개져 식별하기 어려운 현상.
#### 원인 (Root Cause)
* 탑다운 카메라 시점에서의 수직 화면 공간은 월드의 Z축에 대응하므로, Y축 오프셋은 깊이(크기)에만 영향을 줄 뿐 화면상에서는 겹치게 됨.
* 동적으로 생성한 GameObject에 `TextMeshPro` 컴포넌트를 추가하면 기본 텍스트 박스 크기(`sizeDelta`)가 `(2, 2)`로 매우 좁게 생성되며, 기본 폰트도 한국어를 지원하지 않는 `Liberation Sans SDF`로 설정됩니다. 여기에 한국어 긴 문자열이 들어가면 글자 한두 개 단위로 줄바꿈(Word Wrapping)이 일어나며 수십 개의 텍스트 라인이 한 곳에 수직으로 겹쳐 뭉개짐에 따라 바코드처럼 보이게 되고, 폰트 파일 미지정으로 글자가 보이지 않거나 네모 칸 등의 깨짐 기호로 대체됩니다. 또한, 부모 NPC 오브젝트의 크기(Scale)가 비균등(Non-uniform)하게 찌그러져 있을 경우 하위 자식들도 강제로 동일한 찌그러짐 비율을 상속함.
* `GameManager.cs`에서 랜드마크 스폰 시 글씨 크기(`fontSize`)가 초기 Y=150 카메라 시점용 거대 크기인 **`12.0f`**로 고정되어 있었음. 랜드마크의 물리적 배치 간격에 비해 텍스트 크기가 압도적으로 커서, 인접한 술집(30, 40)과 뒷골목(15, 50)의 글자가 서로 수백 유닛 길이로 침범하여 뭉개지고 겹쳐진 현상임.
#### 해결책 (Resolution)
* 탑다운 카메라 시점에서의 상/하 정렬을 위해 텍스트의 배치 오프셋을 Y축이 아닌 Z축 오프셋으로 전면 변경.
  * **이름 텍스트**: 캐릭터 뒤쪽(Z축 +1.2)
  * **상태 텍스트**: 캐릭터 앞쪽(Z축 -1.2)
  * **대화 말풍선**: 캐릭터 한참 뒤쪽(Z축 +2.5)
* 줌인에 대응하도록 폰트 크기(`fontSize`)를 기존 대비 2~3배 축소(`1.3f`~`1.8f`)하여 화면에 겹침 없이 출력되도록 개선.
* 동적으로 생성한 말풍선 객체의 폰트에 이미 프리프레젠테이션 프리팹을 통해 URP 수정 폰트가 잘 입혀져 있는 `nameText`나 `statusText`의 `font` 속성(`tmpro.font = nameText.font`)을 런타임에 직접 복사하여 한국어 깨짐 현상을 근본적으로 차단함.
* 동적 생성 말풍선의 스케일에 `transform.localScale = Vector3.one`을 명시하여 부모의 왜곡된 스케일 상속을 방지하고, `tmpro.rectTransform.sizeDelta = new Vector2(30f, 5f)`로 가로 크기를 충분히 크게 늘리고 자동 줄바꿈(`enableWordWrapping = true`) 조건을 적절히 지정하여 한글 대사가 찌그러짐 없이 한눈에 한 줄로 들어오도록 보정함.
* `GameManager.cs` 내에서 랜드마크 이름 표시 글자 크기(`fontSize`)를 기존 `12.0f`에서 **`3.0f`**로 대폭 축소하고, `enableWordWrapping = false`를 주어 텍스트가 여러 줄로 쪼개져 큐브 주위를 어지럽히지 않고 깔끔한 한 줄 라인으로 정렬되도록 수정함.




### 4. 대화 범위 비현실적 길이(20m) 및 서로 다른 구역 간 순간이동 대화(Telepathic Dialogue) 오류
#### 현상 (Symptom)
* `술집(Tavern)`에 있는 에바와 `뒷골목(Back Alley)`에 있는 발락이 서로 완전히 떨어져 있음에도 불구하고 원거리에서 서로 대화를 수행하는 어색한 현상 발생.
#### 원인 (Root Cause)
* C++ `SystemSocial.cpp`에서 대화 가능 이웃을 찾을 때, 구역 구분 없이 단순 물리적 직선거리(`MAX_DIALOGUE_DISTANCE = 20.0f`)만을 기준으로 대상을 선정했기 때문. 술집과 뒷골목의 실제 물리적 거리는 약 18m로 20m 한계값 이내에 있었음.
#### 해결책 (Resolution)
* [SystemSocial.cpp](file:///C:/Users/adg01/Documents/GitHub/MundusVivens.GameServer.Cpp/SystemSocial.cpp)의 이웃 검색 루프에 `location_name` 일치 여부 검사(`loc_i.location_name == loc_c.location_name`)를 추가하여, 물리적으로 가깝더라도 다른 랜드마크 구역에 있을 경우 대화 상대에서 배제하도록 설정.
* 현실적인 실소리 수준의 대화 거리를 반영하기 위해 대화 가능 한계 거리(`MAX_DIALOGUE_DISTANCE`)를 기존 `20.0f`에서 **`3.5f`**로 축소 조율 완료.

---

## 2026-07-12: NPC 마우스 클릭 상세 정보 패널 활성화 오류

### 현상 (Symptom)
* 유니티 뷰어 내에서 3D 캡슐 모양의 NPC를 마우스로 직접 클릭해도, 좌측 하단의 NPC 상세 상태 정보 패널(NPC Details Panel)이 열리지 않거나 아무런 작동도 일어나지 않는 현상.

### 원인 (Root Cause)
* 유니티의 내장 클릭 이벤트 수신 메서드인 `OnMouseDown`은 **`Collider` 컴포넌트가 붙어 있는 동일한 GameObject**의 스크립트에서만 동작함.
* 스폰 구조상 물리 클릭을 처리하는 `NpcController` 스크립트는 부모 GameObject(`go`)에 부착되어 있으나, 실제 충돌판정을 결정하는 3D `CapsuleCollider`는 `GameObject.CreatePrimitive`를 통해 자동 생성된 자식 GameObject(`capsule`)에 부착되어 있었음.
* 이 때문에 레이캐스트(클릭) 충돌 이벤트가 부모로 전달되지 못하고 자식 레벨에서 삼켜져, 부모의 `OnMouseDown()` 메서드가 전혀 호출되지 못하고 씹히는 구조적 버그였음.

### 해결책 (Resolution)
* [GameManager.cs](file:///C:/Users/adg01/Documents/GitHub/MundusVivens.Unity/Assets/Scripts/GameManager.cs)의 NPC 스폰 로직 내에서 자식 캡슐 객체에 붙어있던 기존 콜라이더를 `Destroy()`로 즉시 파괴함.
* `NpcController`가 부착되어 있는 부모 GameObject에 직접 `CapsuleCollider`를 추가하여 물리 판정 및 Raycast가 부모 레벨에서 직접 트리거되도록 조정함.
  * 부모 콜라이더 규격: `center = (0, 1.5, 0)`, `height = 3.0f`, `radius = 0.75f`로 캡슐 크기에 맞춰 정밀 조율.
* 이를 통해 유니티 실행 후 NPC를 클릭하면 정상적으로 `OnMouseDown`이 호출되어 좌측 하단에 스냅샷 데이터 기반의 감정, 생체 욕구, 관계망 정보 패널이 온전하게 활성화되는 것을 확인함.


