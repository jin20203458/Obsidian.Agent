# Mermaid 다이어그램 작성 및 파싱 오류 해결 가이드

Obsidian, GitHub, VS Code 등의 마크다운 환경에서 Mermaid 다이어그램을 작성할 때 발생하는 대표적인 구문 오류(Syntax Error)의 원인과 예방 규칙을 정리한 가이드라인입니다.

---

## 1. 핵심 설계 규칙 (Core Rules)

###  Rule 1: `graph` 대신 `flowchart` 엔진 사용하기
Mermaid에는 구형 엔진인 `graph`와 신형 엔진인 `flowchart`가 있습니다. 
* **`graph TD`**: 오래된 파서 엔진을 사용하여 특수문자, 괄호, 따옴표가 조금만 들어가도 구문 에러를 쉽게 발생시킵니다.
* **`flowchart TD`**: 렌더링 속도가 빠르고 괄호, 기호의 이스케이프 처리가 훨씬 유연하며 버그가 적습니다.
* **권장**: 다이어그램 작성 시 항상 `flowchart TD` 또는 `flowchart LR`로 시작하세요.

---

###  Rule 2: 특수문자/공백이 포함된 노드는 반드시 큰따옴표(`""`) 사용
노드 텍스트에 다음과 같은 문자가 포함되어 있다면 반드시 큰따옴표로 감싸야 합니다.
* **대상 문자**: 괄호 `()`, 대괄호 `[]`, 슬래시 `/`, 더하기 `+`, 공백 ` `, 한글/영문 혼용 등
* **올바른 사용법**: `ID["텍스트 내용 (상세 정보)"]`

> [!WARNING]
> 큰따옴표 없이 괄호 등을 사용하면 파서가 이를 노드의 형태 정의(예: `ID(둥근 노드)`)로 잘못 인식하여 `got 'PS' (Parenthesis Start)` 에러를 뿜어냅니다.

---

###  Rule 3: 연결선 레이블(Edge Label)의 `&` (Ampersand) 예약어 피하기
Mermaid에서 `&` 기호는 여러 노드를 동시에 연결하는 **문법 예약어**입니다.
* **예 예시**: `A & B --> C` (A와 B 노드 둘 다 C로 연결)
* 이로 인해 연결선 설명 `|A & B 제어|` 내부에 `&`를 그냥 쓰면 파서가 문법 구조 자체를 오해하여 전혀 엉뚱한 라인에서 에러를 표시합니다.
* **해결책**:
  1. `&` 대신 한글 `및` 또는 영문 `and`를 사용합니다. (권장)
  2. 불가피하게 사용해야 할 경우 엣지 레이블 전체를 큰따옴표로 감싸 텍스트 리터럴로 만듭니다: `-->|"A & B 제어"|`

---

###  Rule 4: 레이블 내부에 따옴표를 표시할 때는 HTML 엔티티 사용
노드 텍스트 내부에 강조 등의 이유로 리터럴 큰따옴표(`"`)를 넣고 싶다면, 이스케이프(`\"`)가 작동하지 않으므로 HTML 엔티티 코드를 사용해야 합니다.
* **HTML 엔티티**: `&quot;` 또는 `#quot;`
* **사용 예시**: `Node["이것은 &quot;강조&quot;된 노드입니다"]`

---

## 2. 올바른 표현 (Good vs Bad)

| 상태 | 잘못된 예시 (Bad) | 올바른 예시 (Good) | 비고 |
| :--- | :--- | :--- | :--- |
| **괄호/공백 포함** | `Agent[AI 에이전트 (Claude / Cursor)]` | `Agent["AI 에이전트 (Claude / Cursor)"]` | 괄호로 인한 구문 해석 오류 방지 |
| **특수문자 포함** | `LocalHTTP[HttpListener (127.0.0.1:15050+)]` | `LocalHTTP["HttpListener (127.0.0.1:15050+)"]` | `+` 기호 및 IP 주소 괄호 처리 |
| **연결선 특수문자** | `VM <-->\|컨텍스트 & 서버 제어\| Server` | `VM <-->\|"컨텍스트 및 서버 제어"\| Server` | `&` 예약어 우회 및 따옴표 감싸기 |
| **따옴표 내포** | `A["이름: \"홍길동\""]` | `A["이름: &quot;홍길동&quot;"]` | 내부 따옴표 깨짐 방지 |

---

## 3. 종합 실전 템플릿 (Reference Template)

다음은 실제 성공적으로 빌드 및 렌더링이 검증된 **MCP Context Feeder** 시스템 구성도 샘플입니다. 템플릿 작성 시 참고하세요.

```mermaid
flowchart TD
    UserDrag["사용자 (UI 조작 / 드래그 앤 드롭)"] -->|"파일 및 폴더 추가"| View["MainWindow (뷰)"]
    View <-->|"데이터 바인딩 (MVVM)"| VM["MainViewModel (뷰모델)"]
    
    VM -->|"프리셋 저장 및 로드"| PresetService["PresetService (프리셋 서비스)"]
    VM -->|"토큰 및 글자수 계산, 필터링"| FileInspector["FileInspector (파일 검사)"]
    
    PresetService <-->|"JSON 파일 입출력"| presetsJSON[("presets.json (설정 파일)")]
    
    VM <-->|"컨텍스트 전달 및 서버 제어"| Server["LocalContextServer (MCP 서버)"]
    Server -.->|"포트 바인딩"| LocalHTTP["HttpListener (로컬 HTTP 서버)"]
    
    Agent["AI 에이전트 (Claude, Cursor 등)"] -->|"GET /sse (SSE 연결 요청)"| LocalHTTP
    LocalHTTP -->|"SSE Event Stream (메시지 스트림)"| Agent
    Agent -->|"POST /api/message (JSON-RPC 호출)"| LocalHTTP
```
