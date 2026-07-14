# Mundus Vivens: 트러블슈팅 및 런북 (Troubleshooting & Runbook)

본 문서는 Mundus Vivens 프로젝트(C# AI Server & C++ Game Server) 개발 과정에서 발생한 시스템 환경 오류, 빌드 에러, gRPC 프로토콜 및 아키텍처 정합성 관련 주요 문제와 그 해결 과정을 누적 기록하는 문서입니다.

---

## 2026-07-07: 생체 욕구(Hunger/Fatigue) 및 계획 행동 연동 오작동

### 1. 무제한 가구 탐색 거리로 인한 영구 유턴/이동 루프 (Infinite-Distance Loop)

#### 현상 (Symptom)
* 황무지 장거리 원정(예: 1.5km 이동) 중인 NPC가 생체 욕구 수치 위기(15% 미만)에 봉착했을 때, 자율 야영을 시작하지 않고 다시 아주 먼 거리에 있는 마을 내 빈 가구(성당 의자, 술집 침대 등)를 목적지로 설정하여 반대 방향으로 유턴을 시작하는 문제.
* 원정 스케줄 시간 만료 ➔ AI 서버 스케줄 연장(오토런) ➔ 감쇠 ➔ 기아 위기로 인한 마을 복귀 시도 ➔ AI 스케줄 재연장의 루프에 빠져 영원히 목적지에 도달하지 못함.

#### 원인 (Root Cause)
* C++ `SystemSurvivalOverride`에서 생존용 가구를 탐색할 때, 월드에 등록된 전체 가구 목록 중 비어 있는 것을 거리 제한 없이 최단 거리 (`min_dist`) 기준으로만 선별함.
* 주변에 가구가 전혀 없더라도 1km 이상 멀리 떨어진 가구가 매칭되어 척수가 이동 명령을 덮어써 버림.

#### 해결책 (Resolution)
* 가구 매칭 시 최대 거리 필터 (**`min_dist < 50.0f`**)를 적용하여, 50m 이내에 사용 가능한 가구가 없을 경우 마을 밖 황무지(Wilderness)로 판정하도록 제한을 둠.
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
* 1차 구현 시 `"Sleep"`, `"sleep"`, `"잠"`, `"수면"` 등의 키워드만 등록해 둔 반면, C# 스케줄러(및 Gemini LLM)가 생성하는 기본 하루 계획 템플릿에는 **`"취침"`** 단어가 주로 사용되어 매칭에 실패함.

#### 해결책 (Resolution)
* C++ 서버 내의 수면 및 식사 식별 키워드 풀을 한국어 동의어 및 유사어까지 확장 적용함.
  * **수면(Sleep) 매칭 키워드**: `"Sleep"`, `"sleep"`, `"잠"`, `"수면"`, **`"취침"`**, `"휴식"`, `"노숙"`, `"rest"`
  * **식사(Eat) 매칭 키워드**: `"Eat"`, `"eat"`, `"밥"`, `"식사"`, `"음식"`, `"먹기"`, `"취식"`

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
* `ActionFlee` 내부에서 도주 목적지 및 회전 우회 경로, 그리고 최종 실패 폴백 대입 부분 전체에 **`std::clamp(값, 5.0f, 1995.0f)`** 제약을 걸어 맵 안의 안전 영역 내로 강제 보정함.

### 2. 전투 중인 NPC의 잡담(대화) 및 고차원 스케줄 전환에 의한 교전 인터럽트

#### 현상 (Symptom)
* 두 NPC가 강제로 전투(교전) 상태에 들어갔음에도, 근접했을 때 갑자기 무기를 거두고 1:1 일상 잡담 대화에 참여하거나 C#에서 발급하는 스케줄을 처리하기 위해 맵 반대편으로 떠나는 현상.

#### 원인 (Root Cause)
* C++ `SystemSocialInteraction` (소셜 잡담) 및 `SystemJobDriver` (고차원 스케줄링) 시스템에서 NPC의 교전 여부(`CombatComp.target_entity != entt::null`)를 확인하지 않아 전투 행동보다 스케줄이나 대화 요청이 우선되어 일어나는 인터럽트 현상.

#### 해결책 (Resolution)
* `SystemSocial.cpp`에서 대화 참여 후보 수집 시 `CombatComp`에 타겟을 둔 캐릭터를 원천적으로 `continue`로 배제함.
* `SystemJobDriver.cpp`에서도 사망자 체크와 더불어 교전 중인 NPC는 고차원 스케줄 상태 전환을 스킵하도록 예외 조항을 추가하여 물리 전투 흐름의 영속성을 보장함.

---

## 2026-07-12: 사망 개체 대화 스팸 루프 해결 및 생체 위기 로컬 BT 처리 시 틱 갱신 교착 상태 해결

### 1. 사망한 개체에 대한 대화 트리거로 인한 20Hz 사망 처리 루프 스팸 (Dead-Entity Dialogue Loop)

#### 현상 (Symptom)
* 굶주린 늑대가 사살되어 사망한 이후, C++ 서버 콘솔에 `💀 [사망 처리 완료] 굶주린 늑대의 물리 시뮬레이션이 중단되었습니다.` 로그가 매 틱마다 폭포수처럼 무한히 반복 출력되는 현상.

#### 원인 (Root Cause)
* `SystemSocial.cpp` 내의 대화 시작 시 주변 이웃 후보자(`neighbor`) 수집 단계에서 `is_dead` 검사가 누락됨.
* 이로 인해 살아있는 약탈자 고블린이 죽은 늑대에게 대화를 걸어 늑대의 `ActivityComp`가 `"대화 요청 중"`으로 바뀌고 `BusyTag`가 주입됨.
* `SystemDeath`는 매 틱마다 늑대가 죽었는데 액티비티가 `"사망"`이 아닌 것을 감지하고 다시 `"사망"`으로 강제 복구 및 로그 출력을 수행하여 강제 덮어쓰기 루프가 무한 반복됨.

#### 해결책 (Resolution)
* `SystemSocial.cpp`의 대화 대상 이웃 후보군 스캔 루프 내에 사망자 제외 조건식을 추가하여 사망한 개체에게 불필요한 대화 요청이 전달되는 것을 원천 차단함.
  ```cpp
  if (reg.all_of<HealthComp>(neighbor) && reg.get<HealthComp>(neighbor).is_dead) continue;
  ```

---

### 2. 생체 위기(Hunger/Fatigue) 로컬 BT 처리 시 틱 갱신 교착 상태 해결

#### 현상 (Symptom)
* NPC가 생체 욕구(허기/피로) 위기로 인해 로컬 Behavior Tree(BT)를 통해 가상 Job(999000/999001)을 발급받아 식사/취침을 시작할 때, C++ 서버에서 `🏁 [목적지 도착 - Direct Seek] ...` 및 `🔄 [Toil Transition] ...` 로그가 20Hz 물리 틱마다 폭발적으로 반복 출력되며 시뮬레이션의 논리 틱(Tick) 및 실시간 로그 진행이 완전히 정체되는 현상.

#### 원인 (Root Cause)
1. **C# 서버 스케줄 덮어쓰기**: `GetPendingJobsAsync` gRPC 콜백에서 NPC가 로컬 생존 위기를 처리 중인 상태(`is_resolving_survival == true`)인지 검사하지 않아 로컬 가상 Job(999000)이 매 틱마다 C#의 정기 스케줄 Job으로 덮어써지며 `toil.state`가 `Idle`로 강제 리셋됨.
2. **도착 판정 비교 오류**: `ActionMoveToTarget`에서 도착 완료 검사를 오직 `loc.location_name == job.target_location` 문자열 비교에만 의존함. 황무지(`Wilderness`) 등에서 로컬 가상 Job 실행 시 구역명 불일치로 인해 문자열 매칭이 영구 실패하여 `ActionMoveToTarget`이 `toil.state`를 계속 `Moving`으로 강제 전환하고도 도착 처리(`Working` 전환 및 로그 출력)를 무한 반복함.
3. **가상 Job 타겟 초기화 누락**: 가상 Job 주입 시 기존의 `target_location` 및 target 좌표들이 그대로 남아있어, 자율 야영(모닥불/야영지 생성) 탐색 단계를 스킵해 버림.

#### 해결책 (Resolution)
1. **Pending Job 갱신 스킵**: `main.cpp`의 `GetPendingJobsAsync` 콜백 내부에 수신 예외 조건식을 추가하여, 생체 욕구 위기를 로컬에서 처리 중인 NPC는 C# 스케줄 주입 대상에서 즉시 제외함.
2. **좌표 기반 도착 판정 추가**: `ActionMoveToTarget`에 목적지 좌표와의 물리적 거리 연산(`dist < 0.8f`)을 추가하여, 구역명 불일치 상황에서도 목적지에 도달 시 완벽하게 `Success`를 반환하도록 개선함.
3. **가상 Job 정보 초기화**: 가상 Job 주입 시 `target_location = ""` 및 target 좌표들을 `0.0f`로 완전 초기화하고 명시적으로 `JobCategory::Eat/Sleep`을 주입하여 가구 탐색 및 모닥불 자율 야영 스폰이 정상 동작하도록 수정함.

---

## 2026-07-12: 격자 구역 오븐베이크 음수 좌표 누락 및 맨바닥 행동(노숙) 로그 스팸 해결

### 1. 2D Bake Map 내의 음수 영역 좌표계 누락 및 Wilderness 판정 오류

#### 현상 (Symptom)
* 늑대 서식지(`Wolf Nest`)와 고블린 초소(`Goblin Outpost`)에 NPC들이 정상 스폰되었으나, 캐릭터들의 구역 컴포넌트(`LocationComp::location_name`)가 거점 구역명이 아닌 `"Wilderness"`(황무지)로 잘못 매핑되는 현상.

#### 원인 (Root Cause)
* 2D 격자 맵(`region_map_`) 배열은 양수 범위(`[0, 2000)`)만을 지원하도록 구현되었으나, `world_config.json` 상에서 늑대 서식지는 `(250, -250)`, 고블린 초소는 `(-250, 250)`과 같이 음수 영역 좌표를 사용하고 있었음.
* 음수 좌표 영역은 격자 인덱스 외부에 걸쳐 누락(`Bake` 누락)되었으며, 실시간 위치 갱신 조회 시에도 강제로 `0`으로 클램핑되어 구역명이 무조건 `"Wilderness"`로 어긋나 출력됨.

#### 해결책 (Resolution)
* `world_config.json` 파일의 고블린 초소와 늑대 서식지의 좌표를 모두 안전한 양수 공간으로 전면 조정하여 정밀 매핑이 완료됨을 검증함.
  * **늑대 서식지 (Wolf Nest)**: `(250.0, 0.0, -250.0)` ➔ **`(250.0, 0.0, 10.0)`**
  * **고블린 초소 (Goblin Outpost)**: `(-250.0, 0.0, 250.0)` ➔ **`(10.0, 0.0, 250.0)`**

---

### 2. 맨바닥 생체 욕구 해결(노숙 식사/취침) 시의 20Hz 로그 스팸 루프

#### 현상 (Symptom)
* NPC 또는 야생 동물이 근처에 침대나 식탁 등 가구가 없어 맨바닥에서 노숙 취침(`Sleeping_On_Floor`) 또는 바닥 식사(`Eating_On_Floor`)를 할 때, 20Hz 물리 틱마다 `"⛺ [BT 노숙 취침] ..."` 또는 `"🥪 [BT 바닥 식사] ..."` 로그가 쏟아져 나오는 현상.

#### 원인 (Root Cause)
* `ActionInteractFurniture::Tick`에서 `needs.occupied_furniture == entt::null` 검사를 매 틱마다 시도하나, 맨바닥 행동 시에는 점유할 가구가 없어 `occupied_furniture`가 계속 `entt::null`로 유지되면서 최초 진입 조건문 내의 로그 출력 및 Toil 초기화가 매 틱 실행됨.

#### 해결책 (Resolution)
* `ActionInteractFurniture::Tick`의 최초 진입 조건문에 아래와 같이 바닥 식사/취침 행동 중인지를 가려내는 필터식을 보완하여, 이미 노숙 수면/식사 모션을 유지 중인 캐릭터는 점유 시도 단계를 스킵하여 중복 로그가 유발되지 않고 1회만 기록되도록 교정함.
  ```cpp
  if (needs.occupied_furniture == entt::null && 
      toil.current_action != "Eating_On_Floor" && 
      toil.current_action != "Sleeping_On_Floor")
  ```

---

## 2026-07-14: Tracy Profiler 연동 후 ENABLE_PROFILING=OFF 빌드 시 컴파일 실패

### 현상 (Symptom)

* `ENABLE_PROFILING=OFF` 옵션으로 빌드 시 `#include <tracy/Tracy.hpp>`를 참조하는 모든 소스 파일에서 `fatal error C1083: 포함 파일을 열 수 없습니다. 'tracy/Tracy.hpp'` 에러가 발생하며 컴파일 실패.
* 영향 파일: `main.cpp`, `GameLoop.cpp`, `GrpcResultQueue.h`, `SystemMovement.cpp`, `SystemPlayer.cpp`, `SystemSocial.cpp`.

### 원인 (Root Cause)

Tracy 헤더를 `#ifdef TRACY_ENABLE` 가드 없이 무조건적으로 `#include <tracy/Tracy.hpp>`하고 있었다. `ENABLE_PROFILING=OFF` 시 CMake의 `target_include_directories`에 Tracy 경로가 추가되지 않으므로 컴파일러가 헤더를 찾지 못한다.

### 해결책 (Resolution)

`TracyIntegration.h` 래퍼 헤더를 신설하여 모든 Tracy `#include`를 단일 지점으로 통합하고, `TRACY_ENABLE` 미정의 시 모든 매크로를 no-op으로 처리:

```cpp
// TracyIntegration.h
#pragma once

#ifdef TRACY_ENABLE
#   define TRACY_ON_DEMAND
#   include <tracy/Tracy.hpp>
#else
#   define FrameMark
#   define ZoneScoped
#   define ZoneScopedN(name)
#   define TracyLockable(type, name) type name
#   define LockableBase(type) type
#endif
```

영향받은 6개 파일의 `#include <tracy/Tracy.hpp>`를 `#include "TracyIntegration.h"`로 일괄 교체.

---

## 2026-07-14: Tracy Profiler 활성화 시 C++ 서버가 main() 진입 전 즉시 종료 (Exit Code 1)

### 현상 (Symptom)

* `ENABLE_PROFILING=ON`으로 빌드한 `MundusVivensGameServer.exe`가 실행 즉시 종료됨.
* Stdout/Stderr 출력이 **0바이트** — `main()` 함수의 첫 번째 `std::cout` 배너조차 출력되지 않음.
* Exit Code = `1` (세그폴트/Access Violation이 아닌 정상적인 `exit()` 호출).
* `%LOCALAPPDATA%\CrashDumps`에 당일자 덤프 파일 없음 → OS가 비정상 종료로 인식하지 않음.
* 포트 8086 점유 프로세스 없음 (`netstat` 확인).
* `ENABLE_PROFILING=OFF` 빌드 후 동일 코드 정상 구동 → **Tracy 초기화가 원인임을 확정**.

### 원인 (Root Cause)

Tracy Profiler는 `TRACY_ENABLE` 매크로가 정의되면 전역 정적 객체 `tracy::Profiler`를 자동 생성한다. 이 생성자는 `main()` 진입 이전의 **정적 초기화(Static Initialization) 단계**에서 실행되며, 내부적으로 TCP 8086 포트 바인딩 및 워커 스레드 생성을 포함한 소켓 인프라를 즉시 가동한다.

`TRACY_ON_DEMAND` 매크로가 정의되지 않은 경우(vcpkg 기본값), Tracy는 프로파일러 GUI 연결 여부와 무관하게 **무조건적으로 전체 인프라를 초기화**한다. Windows 환경에서 이 초기화가 실패하면(소켓 권한 문제, 방화벽 차단, OS 레벨 제약 등) Tracy 내부에서 `exit(1)`이 호출되어 프로세스가 `main()` 진입 전에 종료된다. 이 시점은 `std::cout` 실행 이전이므로 콘솔 출력이 0바이트로 나타난다.

### 해결책 (Resolution)

`TracyIntegration.h` 내에서 Tracy `#include` 직전에 `TRACY_ON_DEMAND`를 정의하여, 프로파일러 GUI가 실제로 접속하기 전까지 소켓 인프라 초기화를 지연시킴:

```cpp
#ifdef TRACY_ENABLE
#   define TRACY_ON_DEMAND   // GUI 접속 전까지 소켓 초기화 지연
#   include <tracy/Tracy.hpp>
#endif
```

### 검증 (Verification)

1. `ENABLE_PROFILING=ON` + `TRACY_ON_DEMAND` 빌드 → 컴파일 및 실행 성공
2. C++ 서버 20Hz 틱 루프, gRPC 배치, BT 전투까지 틱 27 이상 정상 처리 확인
3. Tracy GUI를 `localhost:8086`으로 연결 시 실시간 Flame Graph 수신 가능 상태


