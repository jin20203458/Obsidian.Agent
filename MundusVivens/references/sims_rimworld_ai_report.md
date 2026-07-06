# 심즈(The Sims) & 림월드(RimWorld) AI 아키텍처 상세 분석 보고서

이 보고서는 전통적 시뮬레이션 인공지능의 양대 산맥인 Maxis의 *심즈(The Sims)*와 Ludeon Studios의 *림월드(RimWorld)*의 AI 아키텍처 설계와 작동 방식을 코드 레벨에서 분석한 백서입니다.

---

## 1. 개요: 에이전트 시뮬레이션의 두 가지 패러다임

가상 월드에서 NPC들의 자율 사회를 시뮬레이션할 때, 에이전트의 의사결정과 실제 행동 제어를 처리하는 전통적인 근본 아키텍처는 크게 두 갈래로 나뉩니다.

```mermaid
flowchart TD
    A["게임 에이전트 아키텍처 설계"] --> B["A. 욕구 기반 유틸리티 AI (심즈)"]
    A --> C["B. 행동 트리 및 Toil 시퀀스 (림월드)"]

    B --> B1["스마트 오브젝트 (Smart Objects)"]
    B --> B2["욕구 곡선 가중치 (Utility Curve)"]

    C --> C1["씽크트리 (ThinkTrees)"]
    C --> C2["잡 드라이버 및 토일 (Job Drivers & Toils)"]
    C --> C3["기분/생각 메모리 (Mood & Thoughts)"]
```

1.  **스마트 오브젝트 기반 유틸리티 AI (*심즈*)**: 뇌가 아닌 월드의 오브젝트들이 행동 지식을 가지고 광고(Affordance)하며, 에이전트가 자신의 실시간 욕구 점수를 기반으로 무엇을 할지 선택합니다. 에이전트의 연산 부담이 적어 확장성이 뛰어납니다.
2.  **행동 트리 및 행동 조각(Toil) 순차 실행 (*림월드*)**: 에이전트가 상태 트리(ThinkTree)를 타고 올라가며 우선순위(Emergency ➔ Work)를 결정하고, 선택된 Job을 물리적인 아주 작은 단위의 행동 조각(Toils) 시퀀스로 쪼개어 단계별로 실행합니다. 정밀한 물리 제어 및 감정 전이에 특화되어 있습니다.

---

## 2. 심즈 (The Sims) AI 아키텍처: 스마트 오브젝트 & 유틸리티 AI

심즈는 **욕구 기반 유틸리티 AI(Needs-Based Utility AI)**와 **스마트 오브젝트(Smart Object/Affordance)**를 결합하여 에이전트의 자율성을 극한으로 끌어올렸습니다. 새로운 행동을 추가할 때 에이전트의 핵심 코드를 수정할 필요 없이, 오브젝트에 논리를 추가하기만 하면 되는 완벽한 디커플링 구조입니다.

### A. 욕구 기반 의사결정 (Motive & Utility Curve)
*   **Motive**: 심들은 배고픔, 피로, 위생, 사교, 재미 등 6~8가지 핵심 욕구 수치(-100 ~ +100)를 가집니다.
*   **유틸리티 곡선 (Utility Curves)**: 욕구 수준에 따라 수학적 가중치 곡선이 달라집니다.
    *   *생리적 욕구 (배고픔, 피로)*: 수치가 떨어질수록 지수 함수(Exponential) 형태로 가중치 점수가 급격하게 올라가, 다른 모든 행동을 무시하고 생존 행동을 즉시 강제하게 설계됩니다.
    *   *심리적 욕구 (재미, 사교)*: 비교적 완만한 대수 함수(Logarithmic)나 선형 곡선으로 동작하여, 생리적 욕구가 어느 정도 만족되었을 때만 발현되도록 유도합니다 (매슬로의 욕구 단계설 구현).

### B. 스마트 오브젝트와 행동 제공 (Smart Objects & Affordances)
심즈의 가장 핵심적인 혁신은 **"에이전트가 아니라 사물(Object)이 똑똑하다"**는 개념입니다.
*   **Affordance (행동 제공성)**: 침대는 가만히 있지 않고 주변 에이전트들에게 *"나를 사용하면 에너지 욕구를 틱당 +10 만족시켜 줄게"*라고 광고(Advertise)합니다. 샤워기는 *"위생 +20"*을 광고합니다.
*   **의사결정 공식**: 에이전트는 주변 스마트 오브젝트들이 보내는 광고 목록을 스캔하고, 자신의 욕구 수준과 성격을 대입해 최적의 효율을 계산합니다.
    $$\text{Utility} = \text{AdvertisedValue} \times f(\text{CurrentMotiveLevel}) \times \text{PersonalityWeight} \times \frac{1}{\text{Distance}}$$
*   **예외 룰 (Random Top Choice)**: 기계적이고 뻔한 움직임을 방지하기 위해 계산된 유틸리티 점수가 가장 높은 상위 2~3개 행동 중 하나를 무작위(Random)로 선택해 동작시킵니다.

### C. 특성(Trait)과 상황(Lot)에 따른 동적 욕구 조절
*   **특성을 이용한 변수 제어**: 예컨대 '소파 감자(Couch Potato)' 특성을 가진 심은 소파가 광고하는 편안함 점수에 가중치 멀티플라이어를 크게 부여하여 자연스럽게 소파에만 앉아 있도록 유도합니다. 심즈 3부터는 성격 자체를 임시 욕구로 취급하는 방식을 도입했습니다.
*   **부지(Lot) 기반 임시 욕구 주입**: 체육관(Gym) 부지에 들어가면, 월드 매니저가 심에게 '운동 욕구'를 임시로 주입합니다. 이를 통해 심들은 체육관에 들어서자마자 트레드밀을 이용하려는 행동을 취하게 됩니다. 부지를 벗어나면 해당 욕구는 수거(Garbage Collect)됩니다.

---

## 3. 림월드 (RimWorld) AI 아키텍처: ThinkTree, JobDriver, 그리고 Mood

림월드는 거주지 건설이라는 복잡한 환경에서 멀티스텝 물류 처리(예: 나무 베기 ➔ 목재 운반 ➔ 테이블 제작)를 완벽하게 수행해야 합니다. 이를 위해 **행동 트리(ThinkTree)**로 현재 할 일(Job)을 고르고, **잡 드라이버(JobDriver)**와 **토일(Toils)**을 사용하여 행동을 물리적으로 쪼개며, **생각(Thought) 시스템**으로 멘탈 붕괴와 감정 상태를 통제합니다.

### A. 씽크트리 (ThinkTrees)
모든 폰(Pawn)은 주기적으로 XML로 정의된 트리 구조의 **ThinkTree**를 탐색합니다.
*   **ThinkNode_Priority**: 구형 Behavior Tree의 Selector 노드입니다. 자식 노드들을 위에서부터 아래로 순차적으로 평가하여 최초로 유효한 `Job`을 리턴하는 노드가 당첨되면 탐색을 종료합니다. (예: 긴급 도망 ➔ 화재 진압 ➔ 전투 ➔ 자가 치료 ➔ 수면 ➔ 식사 ➔ 작업 ➔ 여가)
*   **ThinkNode_Conditional**: 조건 검사 노드로 데코레이터 패턴과 비슷하게 `IsDrafted` (소집 상태인가?), `OnFire` (몸에 불이 붙었는가?) 등을 확인해 탐색 분기를 제어합니다.

### B. 행동 실행 파이프라인 (JobDrivers & Toils)
ThinkTree가 어떤 `Job` (예: "저장고 A의 철광석을 제련기 B로 운반하라")을 채택하면, 그 구체적인 실무는 **JobDriver** 클래스가 인계받아 실행합니다.
*   **사전 자원 예약 (TryMakePreToilReservations)**:
    *   JobDriver는 행동을 시작하기 전, 필요한 아이템이나 작업 공간 타일을 반드시 **예약(Reserve)**해야 합니다. 
    *   이를 통해 두 폰이 하나의 작업대를 동시에 쓰려고 하거나, 한 개밖에 없는 음식을 동시에 집으려다 발생하는 코드 충돌 및 논리적 오류를 원천 방지합니다.
*   **토일(Toils)의 결합**: `MakeNewToils()`는 `Toil`이라는 미세 단계(행동 조각)들의 리스트를 순차 리턴합니다.
    *   *예시 (요리하기)*: `[오븐으로 걸어가기] ➔ [재료 가져오기] ➔ [오븐에서 대기/조작(Tick 재생)] ➔ [음식 생성]`
    *   **Toil의 구성**: Toil은 시작 시 1번 실행되는 `initAction`, 매 프레임(60Hz) 실행되는 `tickAction`, 완료 시 실행되는 `finishAction`으로 구성됩니다.
    *   **실패 예외 처리**: Toil 실행 도중 누군가 대상 물체를 파괴하거나 문이 잠기면, `FailOnCannotTouch()`나 `FailOnDespawnedOrNull()` 훅이 걸려 있어 즉시 Job을 중단하고 안전하게 다음 ThinkTree로 회귀합니다.

### C. 기분 및 생각 시스템 (Mood & Thoughts)
림월드의 심리 시스템은 크게 '욕구(Needs)'가 '생각(Thoughts)'을 낳고, 생각들의 합이 '기분(Mood)'을 결정하는 2계층 구조를 띕니다.
*   **ThoughtDef (XML)**: 생각의 데이터적 정의 및 가중치(버프/디버프)를 관리합니다 (예: `"테이블 없이 식사함: 무드 -4"`, `"친구가 죽음: 무드 -20"`).
*   **Thought_Memory (휘발성 기억)**: 사건에 의해 생성되어 시간이 지남에 따라 점차 효과가 감쇠(Decay)되고 타이머가 끝나면 소멸합니다. (중복 발생 시 Stack 제약 조건 적용)
*   **Thought_Situational (상황성 기억)**: 어두운 곳에 있거나 방이 지저분한 동안에만 실시간 조건 검사로 계속 유지되며, 그 장소를 벗어나면 즉시 제거됩니다.
*   **기분(Mood) ➔ 정신 붕괴**: 모든 생각들의 무드 가중치 합산이 정신 붕괴 임계치(Minor, Major, Extreme) 이하로 떨어지면, 폰은 확률 판정 후 고유의 **광기/정신이상 상태(Mental State)**에 돌입합니다. 이 상태는 기존 ThinkTree를 MentalState 트리로 강제 스왑시켜 방화, 방황, 폭력 등의 상태 머신으로 돌입하게 합니다.

---

## 4. 아키텍처 비교 요약

| 비교 항목 | 심즈 (The Sims) AI | 림월드 (RimWorld) AI |
| :--- | :--- | :--- |
| **코어 아키텍처** | Utility AI + 스마트 오브젝트 | ThinkTrees (BT) + JobDrivers / Toils |
| **행동 로직의 위치** | 오브젝트 내부 (사물이 주체) | C# 코드 및 XML 트리 (에이전트가 주체) |
| **실행 방식** | 단발성 욕구 충족 연산 | Toil 시퀀스 기반의 다단계 물리 흐름 제어 |
| **동시성 제어** | 타깃 락킹 (Target Locking) | 예약 매니저를 통한 사전 타일/자원 선점 |
| **무드 및 심리** | 욕구가 곧 유틸리티 점수로 직결됨 | 욕구가 생각을 만들고, 생각이 기분을 결정함 |
| **성격(Personality)** | 특성이 유틸리티 욕구 가중치를 보정 | 특성이 행동 트리 분기 및 생각 가중치를 보정 |

---

## 5. 아키텍처 구현 모델 예시 (Code Snippets)

### A. 유틸리티 욕구 및 스마트 오브젝트 연산 모델 (C++ 컨셉)

에이전트가 방 안의 여러 스마트 오브젝트를 스캔하고, 자신의 욕구 상태에 기하급수적 가중치 곡선을 적용하여 동적으로 행동을 선택하는 로직 예제입니다.

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <cmath>
#include <algorithm>
#include <random>

struct Affordance {
    std::string motiveName;
    float value; // 해소 수치
};

class SmartObject {
public:
    std::string name;
    std::vector<Affordance> affordances;

    SmartObject(std::string name, std::vector<Affordance> affordances)
        : name(name), affordances(affordances) {}
};

struct Need {
    std::string name;
    float value; // -100 (고갈) ~ 100 (충족)
};

class UtilityAgent {
public:
    std::string name;
    std::vector<Need> needs;

    UtilityAgent(std::string name) : name(name) {
        needs = { {"Hunger", -20.0f}, {"Energy", 40.0f} }; // 배고픔이 약간 고갈된 상태
    }

    // 욕구 상태에 따른 가중치 곡선 연산
    float GetNeedMultiplier(const std::string& needName) {
        auto it = std::find_if(needs.begin(), needs.end(), [&](const Need& n) { return n.name == needName; });
        if (it == needs.end()) return 1.0f;
        
        // 지수 함수 곡선: 욕구가 고갈될수록 절박함이 기하급수적으로 상승
        float normalized = (it->value + 100.0f) / 200.0f; 
        return std::pow(1.0f - normalized, 2.0f) * 10.0f + 0.1f;
    }

    void EvaluateAndAct(const std::vector<SmartObject>& roomObjects) {
        struct ActionOption {
            const SmartObject* obj;
            Affordance aff;
            float utility;
        };

        std::vector<ActionOption> options;

        for (const auto& obj : roomObjects) {
            for (const auto& aff : obj.affordances) {
                float multiplier = GetNeedMultiplier(aff.motiveName);
                float utility = aff.value * multiplier;
                options.push_back({ &obj, aff, utility });
            }
        }

        // 유틸리티 점수 내림차순 정렬
        std::sort(options.begin(), options.end(), [](const ActionOption& a, const ActionOption& b) {
            return a.utility > b.utility;
        });

        if (options.empty()) return;

        // 기계적 반복을 막기 위해 상위 N개 중 무작위 선택
        int limit = std::min(2, (int)options.size());
        std::random_device rd;
        std::mt19937 gen(rd());
        std::uniform_int_distribution<> dis(0, limit - 1);
        
        const auto& selected = options[dis(gen)];
        std::cout << name << "의 의사결정: " << selected.obj->name 
                  << " 사용 (대상 욕구: " << selected.aff.motiveName 
                  << ", 연산된 유틸리티: " << selected.utility << ")\n";
    }
};
```

### B. 다단계 Toil 제어 구조 모델 (C# 컨셉)

목표 작업을 예약부터 세부 실행까지 작은 Toil 단계들로 쪼개어 연속 프레임으로 가동하는 JobDriver 예제입니다.

```csharp
using System;
using System.Collections.Generic;

namespace AgentSimulation {
    public class Job {
        public string DefName { get; set; }
        public Type DriverClass { get; set; }
    }

    public enum ToilCompleteMode { Instant, Never }

    public class Toil {
        public Action InitAction { get; set; }
        public Action TickAction { get; set; }
        public ToilCompleteMode CompleteMode { get; set; } = ToilCompleteMode.Never;
        public bool Initialized { get; set; } = false;
    }

    public abstract class JobDriver {
        protected Job job;
        private List<Toil> toils;
        private int activeToilIndex = 0;

        public JobDriver(Job job) {
            this.job = job;
            toils = new List<Toil>(MakeNewToils());
        }

        public abstract IEnumerable<Toil> MakeNewToils();

        public void TickDriver() {
            if (activeToilIndex >= toils.Count) return;

            Toil toil = toils[activeToilIndex];
            if (!toil.Initialized) {
                toil.InitAction?.Invoke();
                toil.Initialized = true;
            }

            toil.TickAction?.Invoke();

            if (toil.CompleteMode == ToilCompleteMode.Instant) {
                activeToilIndex++;
            }
        }

        public void MoveNext() {
            activeToilIndex++;
        }
    }

    public class JobDriver_Sleep : JobDriver {
        public float RestLevel { get; set; } = 25.0f;

        public JobDriver_Sleep(Job job) : base(job) {}

        public override IEnumerable<Toil> MakeNewToils() {
            // Toil 1: 침대로 물리적 이동
            yield return new Toil {
                InitAction = () => Console.WriteLine("침대로 이동을 시작합니다."),
                TickAction = () => {
                    Console.WriteLine("침대 방향으로 이동 중...");
                    MoveNext(); // 조건 만족 시 다음 Toil로 전환
                },
                CompleteMode = ToilCompleteMode.Never
            };

            // Toil 2: 수치 증가 및 대기 애니메이션
            yield return new Toil {
                InitAction = () => Console.WriteLine("침대에 누웠습니다."),
                TickAction = () => {
                    RestLevel += 25.0f;
                    Console.WriteLine($"취침 중... 에너지: {RestLevel}%");
                    if (RestLevel >= 100.0f) {
                        Console.WriteLine("작업 완료.");
                        MoveNext();
                    }
                },
                CompleteMode = ToilCompleteMode.Never
            };
        }
    }
}
```
