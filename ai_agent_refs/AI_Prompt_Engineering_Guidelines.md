---
type: guideline
title: AI Prompt Engineering Guidelines
tags: [ai, prompt, engineering, guidelines, grc]
related:
  - ../README.md
last_updated: 2026-07-15
status: stable
---
# AI 프롬프트 엔지니어링 지침서 (GRC Project)
# AI Prompt Engineering Guidelines
# 
# 최종 갱신일: 2026-06-12 (검증 및 심층 업데이트 완료)
# 근거: 2024~2026년 최신 학술 논문 18편 및 Google/Anthropic/OpenAI 공식 가이드라인 기반
# 검증: 전체 인용 출처 18건 웹 검증 완료 (2026-06-12)
#
# 본 문서는 GRC 프로젝트의 모든 AI 프롬프트 작성 시 참조해야 할 핵심 원칙과 실전 규칙을 정리합니다.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## 1. 구조적 프롬프팅 (Structured Prompting)

### 원칙
XML 태그를 사용하여 프롬프트의 각 구성 요소(지시문, 데이터, 예시, 출력 형식)를 
물리적으로 분리하면, LLM의 지시 준수율이 30% 이상 향상된다.

### 근거
- [Source] "The Delimiter Hypothesis: Does Prompt Format Actually Matter?" (Systima, 2026.03)
  → 마크다운 헤더(`#`, `---`) 구조는 악의적인 사용자 입력(Trojan Delimiters)에 의해 20% 이상의 파싱 실패 및 인젝션 취약성을 보이나, XML 태그는 이를 100% 방어함. 구조적 분리와 보안을 위해 XML 태그를 최우선 권장.
- [Source] "XML Prompting as Grammar-Constrained Interaction" (arXiv:2509.08182, 2025.09)
  → XML 태그 기반 프롬프팅이 문법 제약 상호작용임을 수학적으로 증명. 수렴 보장 제공.
- [Source] Anthropic 공식 문서 (2024-2026, 지속 업데이트)
  → "XML tags help Claude parse complex prompts unambiguously"
- [Source] "The PICCO Framework" (arXiv:2604.14197, 2026)
  → Persona, Instructions, Context, Constraints, Output의 명시적 분리 프레임워크.

### 실전 규칙
- <system_directive>: 역할(Role)과 과제(Task) 정의
- <rules> 또는 <constraints>: 행동 제약 및 금지사항
- <context>: 참고 데이터 (세계관, 이전 서사 등)
- <output_format>: 출력 규격 (JSON 스키마, 태그 형식 등)
- <example>: Few-shot 예시
- 태그 형식은 한 프롬프트 내에서 XML 또는 Markdown 중 하나로 통일할 것

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## 2. 데이터-지시문 격리 (Data-Instruction Separation)

### 원칙
프롬프트 내에서 데이터(사용자 입력, 외부 텍스트)와 지시문(규칙, 명령)을 
분리하지 않으면, 데이터가 지시문으로 오해석되어 환각(hallucination)이나 
프롬프트 인젝션 취약점이 발생한다.

### 근거
- [Source] "How Not to Detect Prompt Injections with an LLM" (ACM AISec 2025 Workshop, 2025.10)
  → LLM 기반 인젝션 탐지(KAD)는 구조적 취약점(DataFlip 공격 등)이 존재함.
- [Source] "Defense Against Prompt Injection Attack by Leveraging Attack Techniques" (ACL 2025)
  → XML 기반 격리(Isolation)는 단일 프롬프트 환경에서 필수적인 1차 방어선이지만, 적응형 공격(Adaptive attacks)에 의해 우회될 수 있음.
- [Source] "Agent Privilege Separation in OpenClaw" (arXiv:2603.13424, 2026)
  → 자율 에이전트 시스템의 경우, '저권한 데이터 읽기 에이전트'와 '고권한 액션 에이전트'를 물리적으로 분리하는 다중 에이전트 권한 분리 아키텍처가 가장 근본적이고 완벽한 방어책(ASR 0%)임.

### 실전 규칙
- 사용자 입력 데이터는 반드시 <user_input> 또는 <context> 태그로 감싸서 격리
- 각 프롬프트는 자기 영역 밖의 데이터를 생성하지 않도록 명시적으로 차단
  예: "서사 요약에 수치 데이터를 포함하지 마십시오" (무수치/Zero-Numeric 원칙)
- 한 모델이 생성한 데이터를 다른 모델의 지시문 내에 삽입할 때는 
  반드시 데이터 태그로 감싸서 지시문과 구별

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## 3. 소비자 주도 계약 설계 (Consumer-Driven Contract)

### 원칙
프롬프트의 출력 형식은, 그 출력을 실제로 소비하는 코드(파서, 다음 단계 에이전트)의 
기대 규격에 의해 결정되어야 한다. "좋은 문장"이 아니라 "파싱 가능한 문장"이 목표.

### 근거
- [Source] BoundaryML Benchmark (2026)
  → 텍스트 프롬프트 내에 JSON Schema를 명시할 경우, TypeScript 인터페이스를 사용하는 것보다 토큰을 4배 더 낭비하며, 하위 모델에서는 스키마 메타키(`properties` 등) 오염 에러율이 6%에 달함. 반면 TypeScript 인터페이스는 0% 오류율 달성.
- [Source] "Software Engineering for Prompt-Enabled Systems" (arXiv:2503.02400, 2025-2026)
  → 프롬프트를 타입 기반 인터페이스(typed interface)로 취급. 계약 기반 설계(contract-driven).
- [Source] "JSONSchemaBench: A Rigorous Benchmark of Structured Outputs for Language Models" 
  (arXiv:2501.10868, ICML 2025 ES-FoMo Workshop)
  → 문법 기반 제약 디코딩(Guidance, XGrammar 등)으로 단순 스키마에서 거의 100% 달성 가능하나, 복잡한 재귀/중첩 스키마에서는 프레임워크별 편차 큼.
- [Source] "Chain-of-Collaboration Prompting Framework" (arXiv, 2025.05)
  → 멀티 에이전트 간 출력은 파싱 가능하고 검증된 형식이어야 함.

### 실전 규칙
- 출력이 JSON으로 파싱될 예정이면:
  → responseMimeType: "application/json" 설정 필수
  → **복잡한 구조의 경우 텍스트 프롬프트 내에 Raw JSON Schema를 적지 말고 TypeScript 인터페이스를 사용하여 구조와 타입을 정의할 것.** (주석 활용 용이, 토큰 절약)
  → 단, 엣지 케이스 설명을 위해 간단한 Raw JSON Few-shot 예시 1~2개를 덧붙이는 것은 강력히 권장됨.
  → 방어적 파싱 코드(ExtractJson, DeserializeSafe) 반드시 구비
- 출력이 특정 태그로 파싱될 예정이면:
  → <plot>, <chronicle> 등 파싱 대상 태그를 프롬프트에 명시
  → Fallback 파싱 로직 구비 (태그 없을 때 전체 텍스트 사용)
- 멀티 스텝 파이프라인에서:
  → 각 단계의 출력 규격을, 다음 단계가 기대하는 입력 규격에 맞춰 설계
  → 에이전트에게 "이 출력이 어디서 어떻게 소비되는지" 맥락을 제공

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## 4. 수치 연산 안전 장치 (Numerical Safety Clamping)

### 원칙
LLM은 산술 연산에서 체계적 오류(2~15%)를 범한다. 
수치를 다루는 프롬프트에는 반드시 유효 범위(Clamping)와 이탈 시 처리 규칙을 명시한다.

### 근거
- [Source] "Steering Large Language Models between Code Execution and Textual Reasoning" 
  (ICLR 2025, CodeSteer)
  → LLM이 코드 실행 대신 텍스트 추론을 선호하는 경향이 있으며, 이 경우 산술 오류 발생.
     코드 기반 도구 사용(Code Interpreter) 시 100% 성공률 달성 가능한 과제에서도
     텍스트 추론 시 성능 저하. 모델/과제 복잡도 증가 시 역스케일링 관찰.
- [Source] "Demystifying Errors in LLM Reasoning Traces: An Empirical Study of Code Execution Simulation" 
  (ACM TOSEM, 2026 / arXiv:2512.00215, 2025.11)
  → 최첨단 추론 모델 4종(Claude 4, DeepSeek R1, Gemini, GPT-4o) 평가.
     9가지 오류 유형 분류 중 Computation Error가 가장 빈발.
     도구 보강(calculator 플러그인) 시 계산 오류의 58% 교정 가능.

### 실전 규칙
- "100/100" 같은 범위형 스탯: [0~최대치] 이탈 불가 명시 (Clamping)
- 단일 숫자(예: 골드 1000): 무제한 연산 허용 여부 명시
- 증감폭 가이드: "5~10 상승", "20% 감소" 등 구체적 범위 제시
- 변화가 없는 항목은 이전 값을 그대로 유지한다는 규칙 명시
- 가능하다면 수치 연산은 LLM 대신 코드에서 처리 (Tool Use 활용)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## 5. System Instruction vs User Prompt 배치 전략

### 원칙
역할 정의, 행동 제약, 출력 형식, Few-shot 예시는 System Instruction에 배치하고,
실제 질문/과제만 User Prompt에 배치하면 성능이 최적화된다.

### 근거
- [Source] "LLM Shots: Best Fired at System or User Prompts?" (Halil et al., 2025, Huawei)
  → "질문은 User Prompt에, 나머지 모든 것은 System Prompt에 배치"가 최적.
- [Source] Google 공식 문서 (Gemini 3, 2026.06)
  → "핵심 행동 제약, 역할, 출력 형식은 System Instruction 또는 프롬프트 최상단에 배치"
- [Source] OpenAI 공식 문서 (2026)
  → Developer 메시지 권장 순서: Identity → Instructions → Examples → Context

### 실전 규칙 (GRC 적용)
- System Instruction에 배치할 것:
  → GM 페르소나/성격 정의
  → 세계관 (<master_setting>)
  → 로어북 주입
  → 서식 규칙 (<syntax_rules>)
- User Prompt에 배치할 것:
  → 장/중기 서사 컨텍스트 (<background_context>)
  → 현재 유저 행동 (<current_action>)
  → 최종 실행 지시 (<final_instruction>)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## 6. Few-Shot 예시 전략

### 원칙
Few-shot 예시는 항상 포함하되, 2~4개가 최적이며 형식 일관성이 핵심이다.
예시가 충분히 명확하면 명시적 지시문을 줄일 수 있다.

### 근거
- [Source] Google 공식 문서 (2026)
  → "Few-shot 예시를 프롬프트에 항상 포함할 것을 권장. 예시 없는 프롬프트는 덜 효과적."
  → "예시가 충분히 명확하면 지시문을 줄여도 된다."
- [Source] "LLM Shots" (2025)
  → 대부분의 모델에서 2~4개의 예시가 최적.
- [Source] OpenAI 공식 문서 (2026)
  → "다양한 입력 범위를 보여주는 예시 제공. 엣지 케이스 포함 권장."

### 실전 규칙
- 예시 수: 2~4개 (너무 많으면 과적합, 너무 적으면 효과 미미)
- 형식 일관성: 모든 예시의 XML 태그, 들여쓰기, 구분자가 동일해야 함
- 다양성: 정상 케이스 + 엣지 케이스를 함께 포함
- 배치: System Instruction 내에 배치 (User Prompt가 아님)
- 현재 GRC 적용 상태:
  → 챗뷰 서식 규칙에 <example> 태그로 대사/독백 예시 1개 포함 [Good]
  → 에이전트 프롬프트에는 Few-shot 예시 없음 [Warning] (개선 여지)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## 7. Temperature 및 Generation Config 최적화

### 원칙
Gemini 3.x 모델 제품군에서는 Temperature를 비롯한 샘플링 매개변수(top_p, top_k)를 명시적으로 조작하지 않고 생략(Omit/Null)하여 API 기본값(Default = 1.0)으로 구동한다.

### 근거
- [Source] Google 공식 문서 (Gemini 3.5 Flash & 3.1 Pro 개발자 가이드, 2026.06 검증 완료)
  → [Warning] "Gemini 3.x 모델에서는 temperature, top_p, top_k를 기본값으로 유지할 것을 강력히 권장. 변경 시 루핑(동일 문장 반복 출력)이나 성능 저하 등 예기치 못한 동작이 발생할 수 있음."
  → 특히 정밀 JSON 구조화 출력이나 코드 생성 등의 과제에서도 온도를 임의로 낮추는 대신, System Instruction의 규칙 정의 및 Response Schema(Structured Output)를 활용해 일관성을 제어하도록 설계됨.

### Temperature 가이드라인 (Gemini 2.x 이하 또는 타사 모델 참고용)
| 과제 유형             | 권장 Temperature |
|----------------------|------------------|
| JSON/구조화 출력      | 0.0 ~ 0.2       |
| 코드 생성             | 0.0 ~ 0.3       |
| 분류/추출             | 0.0 ~ 0.2       |
| 요약                 | 0.3 ~ 0.5       |
| 일반 Q&A             | 0.5 ~ 0.7       |
| 창작/서사             | 0.7 ~ 1.0       |
| 브레인스토밍          | 0.8 ~ 1.2       |

### 실전 규칙 (GRC 적용 상태)
- **모든 API 호출부 일괄 적용**: `GenerationConfig` 내 `Temperature` 프로퍼티를 nullable(`float?`)로 변환하고 `null`로 지정하여 API 요청 시 필드 자체가 전송되지 않도록(Omit) 구성 완료.
- **에이전트(SessionArchitect)**: `null` 적용 (기본값 구동) [Good]
- **상태창 갱신(StatusAPI)**: `null` 적용 (JSON 출력 작업이지만 무한 루프 예방을 위해 기존 `0.6`에서 기본값으로 전환 완료) [Good]
- **중기/장기 서사 요약 및 일어 번역**: `null` 적용 (기본값 구동) [Good]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## 8. Thinking(사고) 모델 프롬프트 전략

### 원칙
Thinking이 내장된 모델(Gemini 2.5/3, Claude Extended Thinking)에게는
"어떻게(HOW)" 대신 "무엇을(WHAT)" 지시하라. CoT 지시문은 불필요하다.

### 근거
- [Source] Google 공식 문서 (Gemini Thinking, 2026.06.04)
  → "Gemini 2.5/3 시리즈는 자동으로 내부 사고 텍스트를 생성하므로, 
     응답에서 추론 단계를 설명하도록 하는 것은 일반적으로 불필요."
  → "복잡한 문제에 'Think very hard before answering'이라고 하면 성능 향상 가능."
- [Source] OpenAI 공식 문서 (2026)
  → 추론 모델(o-시리즈): CoT 지시 불필요. 간결하고 직접적인 목표 제시가 최적.
     Developer 메시지에 핵심 지시를 배치하고, 프로세스 세부사항은 모델에 위임.
  → 표준 GPT 모델: 명시적이고 단계적인 지시가 효과적. 구조화된 프롬프트 필수.

### Thinking Level 가이드라인 (Gemini 3)
| 과제 복잡도          | 권장 ThinkingLevel |
|---------------------|-------------------|
| 간단 (사실 검색, 분류)| minimal 또는 OFF  |
| 보통 (비교, 유추)     | low ~ medium     |
| 복잡 (수학, 코딩)     | high             |

### 실전 규칙 (GRC 적용)
- 상태창 갱신: ThinkingLevel.minimal [Good] (단순 JSON 변환)
- 중기 서사 요약: ThinkingLevel.high [Good] (복잡한 서사 융합)
- 장기 기억 병합: ThinkingLevel.medium [Good] (중간 복잡도의 압축)
- "단계별로 생각하세요" 같은 CoT 지시문은 사용하지 말 것 (이미 내장)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## 9. 프롬프트 길이와 토큰 효율성

### 원칙
프롬프트가 길어질수록 성능 수익은 급격히 체감한다. 
핵심 정보는 프롬프트의 시작과 끝에 배치하고, 중간 영역은 피한다.

### 근거
- [Source] "Lost in the Middle: How Language Models Use Long Contexts" (Liu et al., TACL 2024)
  → LLM은 긴 컨텍스트의 중간에 있는 정보에 주의를 기울이지 못함.
     시작과 끝에 있는 정보가 가장 가장 잘 활용됨.
     장기 컨텍스트 전용 모델에서도 이 현상 관찰.
- [Source] "Incorporating Token Usage into Prompting Strategy Evaluation" (arXiv:2505.14880, 2025.05)
  → 토큰 사용량 증가는 급격히 체감하는 성능 수익을 가져온다.
     Few-shot 예시 3개→8개 시 Token Cost가 10배 이상 증가하나 성능 향상은 미미.
     Big-Otok 프레임워크 제안: 효율성 인식(efficiency-aware) 프롬프트 평가 필요.

### 실전 규칙
- 핵심 지시문: 프롬프트 최상단에 배치
- 대량의 컨텍스트 데이터: 중간에 배치하되 태그로 격리
- 최종 실행 지시: 프롬프트 최하단에 배치 (<final_instruction>)
- 연결 구문 사용: "위 정보를 바탕으로..." 등의 브릿지 문구로 컨텍스트와 과제를 연결
- 과도한 프롬프팅 금지: 간결하고 핵심적인 지시가 장황한 설명보다 효과적

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## 10. 부정 지시문 (Negative Prompting)

### 원칙
"~하지 마시오"보다 "~하시오"가 더 신뢰성이 높다.
단, 기본 행동 차단에는 부정 지시문이 효과적이다.

### 근거
- 부정 지시문은 LLM에게 "금지된 행동을 먼저 표상한 후 억제"하도록 요구하므로,
  역설적으로 금지된 출력의 발생 확률을 높일 수 있다.
- Google, OpenAI, Anthropic 모두 부정 지시문을 사용하되,
  긍정적 대안과 함께 결합할 것을 권장.

### 실전 규칙
- 우선 긍정 프레이밍: "불릿 포인트를 사용하지 마시오" → "번호 매긴 문단을 사용하시오"
- 부정 지시문이 효과적인 경우:
  → 특정 기본 행동 차단: "마크다운 서식을 사용하지 마십시오"
  → 안전 제약: "PC의 대사, 행동, 감정을 임의로 묘사하지 마십시오"
  → 형식 제한: "JSON 외의 텍스트를 출력하지 마십시오"
- 모호한 부정 금지: "환각하지 마시오"는 비효과적. 대신 근거 컨텍스트를 제공

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## 11. 다국어(한국어) 프롬프트 고려사항

### 원칙
최신 토크나이저의 발전으로 CJK 언어의 토큰 효율성은 오히려 영어보다 뛰어난 경우가 많다. 단, 문법적으로 한국어(SOV)는 단순한 띄어쓰기보다 명시적인 '조사(Postpositions)' 및 '격 표지'를 활용하여 구조적 마커를 제어하는 프롬프팅이 필수적이다.

### 근거
- [Source] 토큰 효율성: "Is Sanskrit the most token-efficient language?" (arXiv:2601.06142, 2026)
  → CJK 언어가 2~3배 토큰을 더 소모한다는 과거의 오해를 반박함. Qwen 등 최신 모델은 표의문자의 높은 의미 밀도를 활용하여 영어보다 최대 40% 적은 토큰으로 추론 가능.
- [Source] 구조적 마커: "Towards harnessing the most of ChatGPT for Korean grammatical error correction" (Applied Sciences, 2024)
  → 성공적인 한국어 프롬프팅을 위해서는 조사와 격 표지를 명시적으로 제어하여 모델이 문법적 의도를 정확히 파악하도록 강제해야 함을 입증.

### 실전 규칙 (GRC 적용)
- System Instruction의 역할/규칙 정의: 한국어 사용 [Good] (사용자 대상 프로젝트)
- 태그와 구분자: XML 태그는 영어로 유지 (<rules>, <output_format> 등) [Good]
- 토큰 효율: 최신 모델에서는 CJK 토큰 패널티가 사실상 해소되었으므로, 인위적인 압축보다는 명확한 조사 사용과 의미 전달에 집중할 것.
- 출력 언어 명시: 명시적으로 "반드시 한국어로 작성하십시오" 포함 [Good]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## 12. 시스템 프롬프트 코드 구현 패턴 (C# 11)

### 원칙
C# 백엔드 코드 내에 시스템 프롬프트를 작성할 때는, 이중 따옴표(`""`)나 이중 중괄호(`{{`)의 이스케이프(Escape) 지옥을 피하고 JSON 원본 가독성을 유지하기 위해 반드시 **C# 11 다중 달러 기호 기반의 Raw String Literal (`$$"""..."""`)**을 사용한다.

### 실전 규칙 (GRC 적용)
- `$$"""..."""` 패턴 적용 시:
  → 단일 중괄호 `{` 와 `}` 는 C# 보간 문자가 아닌 **순수 JSON 중괄호** 문자로 인식됨. (TypeScript 스키마나 JSON 예시 작성 시 그대로 복붙 가능)
  → C# 변수를 프롬프트에 동적으로 삽입할 때만 이중 중괄호 `{{변수명}}` 사용.
  → 큰따옴표 `"` 를 탈출 문자 없이 자유롭게 사용.
- 이는 프롬프트 엔지니어와 백엔드 개발자 간의 프롬프트 텍스트 이관/유지보수 비용을 획기적으로 낮춰준다.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## 13. Self-Correction 패턴 (향후 적용 고려)

### 원칙
"생성 → 검토 → 정제"의 자기 교정 패턴은 
Anthropic이 공식 권장하는 가장 일반적인 에이전트 체이닝 패턴이다.

### 근거
- [Source] Anthropic 공식 문서 (2024-2025)
  → "가장 일반적인 체이닝 패턴은 self-correction: 
     초안 생성 → 기준에 대해 검토 → 검토 결과 기반으로 정제"

### GRC 적용 가능성
- 에이전트의 최종 단계(PromptGen) 직후에 검토 단계를 삽입하여,
  생성된 시스템 프롬프트가 런타임 환경과 충돌하지 않는지 자동 검증 가능
- 현재 구현: 미적용 (향후 고도화 과제)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## 참고 문헌 (References)

 1. "XML Prompting as Grammar-Constrained Interaction" — Alpay & Alpay (arXiv:2509.08182, 2025.09)
 2. Anthropic Official Docs — Prompting Best Practices (2024-2026, 지속 업데이트)
 3. "The PICCO Framework for LLM Prompting" — Cook (arXiv:2604.14197, 2026.04)
 4. "Promptware Engineering: Software Engineering for Prompt-Enabled Systems" — Chen et al. (arXiv:2503.02400, ACM TOSEM, 2025)
 5. "Defense Against Prompt Injection Attack by Leveraging Attack Techniques" — Chen et al. (ACL 2025)
 6. "JSONSchemaBench: A Rigorous Benchmark of Structured Outputs for Language Models" — Geng et al. (arXiv:2501.10868, ICML 2025 ES-FoMo)
 7. "Steering Large Language Models between Code Execution and Textual Reasoning" — CodeSteer (ICLR 2025)
 8. "How Not to Detect Prompt Injections with an LLM" — Choudhary et al. (ACM AISec 2025 Workshop, 2025.10)
 9. "Connecting the Dots: A Chain-of-Collaboration Prompting Framework" — Zhao et al. (arXiv:2505.10936, 2025.05)
10. "Demystifying Errors in LLM Reasoning Traces" — Abdollahi et al. (ACM TOSEM, 2026 / arXiv:2512.00215)
11. "LLM Shots: Best Fired at System or User Prompts?" — Halil et al. (ACM WWW '25 / Huawei Research, 2025)
12. Google AI Official — Gemini Prompting Strategies (2026.06)
13. OpenAI Official — Prompt Engineering Guide (2026)
14. Google AI Official — Gemini Thinking (2026.06)
15. "Lost in the Middle: How Language Models Use Long Contexts" — Liu et al. (TACL 2024)
16. "Incorporating Token Usage into Prompting Strategy Evaluation" (arXiv:2505.14880, 2025.05)
17. "Is Sanskrit the most token-efficient language? A quantitative study using GPT, Gemini, and SentencePiece" — Kumar (arXiv:2601.06142, 2026.01)
18. "The Order Effect: Investigating Prompt Sensitivity to Input Order in LLMs" (arXiv:2502.04134, 2025)
19. "Towards harnessing the most of ChatGPT for Korean grammatical error correction" (Applied Sciences, 2024)
