---
type: reference
title: Special Logic Locations Memo
tags: [mundus-vivens, logic, memo]
related:
  - ../README.md
status: draft
---
# Special Logic Locations Memo
> **부제**: 물리/소셜 시뮬레이션 예외 특수 로직 작동 장소

### 1. 실질적으로 특수한 로직이 작동하는 장소 (Active)

아래 4가지 타입은 C++ 서버의 물리/소셜 시뮬레이션 공식에서 **완벽하게 독립된 특수 로직**으로 작동하고 있습니다.

- **`Tavern` (술집)**: 사회적 에너지가 10배 빠르게 회복되며, 대화 발생 확률이 1.8배 증가하는 '가장 활발한 사교 장소'입니다.
- **`Market` (시장)**: 대화 발생 확률이 1.5배 보정되는 활기찬 장소입니다.
- **`Square` (광장)**: 대화 발생 확률이 1.2배 보정되는 열린 장소입니다.
- **`Church` (성당)**: 대화 발생 확률이 0.3배로 극도로 낮아지는 엄숙한 장소입니다.

---

### 2. 매핑만 되어 있고, 물리적으로는 디폴트(1.0) 처리되는 장소 (Passive)

아래 타입들은 C# 설정 파일과 C++ 컴포넌트 간에 타입 매핑은 완벽히 이루어지지만, C++ 시뮬레이터 공식 상에서는 별도의 특수 가중치가 없어서 **일반 기본 장소(가중치 1.0, 기본 에너지 회복)로 처리**됩니다.

- **`Wilderness` (황무지)**
- **`Residential` (주거지)**
- **`Forge` (대장간)**
- **`Manor` (저택)**
- **`Country` (외곽 시골)**
- **`City` (도시)**
- **`Place` (기타 장소)**
- **`Unspecified` (지정되지 않음)**

```cpp
struct JobComp {
    uint64_t job_id = 0;             // 이 작업(의도)의 고유 ID (C#에서 부여)
    std::string target_location;     // 가고자 하는 목적지 이름 (예: "Tavern", "Eva's Home")
    float target_x = 0.0f;           // 목적지의 실제 물리적 X 좌표
    float target_y = 0.0f;           // 목적지의 실제 물리적 Y 좌표
    float target_z = 0.0f;           // 목적지의 실제 물리적 Z 좌표
    std::string intent;              // LLM이나 계획기에서 생성한 인간용 행동 설명 (예: "에바와 소문 나누기")
    uint32_t target_agent_id = 0;    // 이 행동이 다른 NPC와 얽힌 사회적 작업인 경우, 그 상대방 NPC의 ID
    int32_t priority = 0;            // 작업의 우선순위 (생존 욕구 등 긴급 상황 발생 시 덮어쓰기 위해 판정)
    bool is_active = false;          // 현재 이 작업을 실행 중인지 여부
    JobCategory category = JobCategory::Unspecified; // 대분류 (Sleep, Eat, Social, Work, Travel 등)
};
```