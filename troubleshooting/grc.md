---
description: GRC(GenAI Roleplay Chat) C# WPF 데스크톱 클라이언트 트러블슈팅 런북.
related:
  - ../README.md
  - ../GRC/README.md
---
# GRC Troubleshooting

본 문서는 GRC (GenAI Roleplay Chat) 데스크톱 클라이언트(C#/.NET 8.0 WPF, CommunityToolkit.Mvvm, Google Gemini API, Vertex AI TTS, System.Threading.Channels 스트리밍 및 O(1) 바이너리 탐색 파일 로거) 개발 및 운영 중 발생하는 시스템 예외, 동시성 버그, 라이브러리 충돌 및 트러블슈팅 내역을 누적하여 관리하는 중앙 기술 런북입니다.

---

## 사전 주의사항 및 알려진 기술적 제약 (Known Constraints)

### 1. WPF UI 스레드 (STA) 비차단 불변식
* **현상**: 대규모 LLM 토큰 스트리밍 또는 디스크 쓰기 작업 시 UI 스레드 블로킹으로 인한 창 멈춤(Freezing) 발생 가능성.
* **대응책**: 모든 API 호출 및 I/O는 `System.Threading.Channels` 기반 백그라운드 큐로 오프로드하고, UI 렌더링은 디스패처 배치 업데이트를 통해서만 수행.

### 2. Vertex AI Global 엔드포인트 단일화 및 리전 필드 제거
* **현상**: Vertex AI 연동 시 특정 리전(예: `us-central1`, `asia-northeast3`)을 지정할 경우 최신 주력 모델(`gemini-3.8-flash`, `gemini-4-argon`) 엔드포인트 404/미지원 오류 발생.
* **대응책**: 사용자 혼선을 원천 방지하기 위해 UI 및 설정에서 `Location` 입력을 전면 제거하고, 모든 LLM API 호출을 `aiplatform.googleapis.com/.../locations/global/...` 단일 엔드포인트로 고정. (단, 음성 TTS 모델의 경우 지원 리전인 `us-central1`로 내부 자동 고정)

---

> [!NOTE]
> **[정제 완료 기준선]** 2026-10-14 이전 상위 항목은 정제 완료됨. 신규 인시던트는 이 아래에 추가됩니다.

---

### 3. LLM JSON 파서 내결함성 및 역직렬화 트랩 (LlmJsonParser)
* **현상**:
  1. **사전/사후 잡담 괄호 트랩**: LLM이 JSON 페이로드 앞뒤에 설명용 괄호(예: `Format {template}. Payload: {"key": "val"}`)를 출력할 때 단순 `IndexOf('{')` 방식은 비정상 괄호를 파싱 대상으로 삼아 실패.
  2. **문자열형 숫자 및 Enum 불일치**: LLM이 숫자 필드를 큰따옴표로 감싸거나(`"count": "42"`), DTO의 Enum을 문자열(`"Planning"`)로 반환할 때 기본 System.Text.Json 역직렬화 예외 발생.
  3. **토큰 상한에 의한 JSON 절단(Truncation)**: 긴 서술문이나 배열 출력 도중 토큰 제한으로 문자열이나 괄호가 닫히지 않고 끊겨 전체 응답이 유실됨.
* **대응책**:
  1. `LenientOptions`에 `NumberHandling = JsonNumberHandling.AllowReadingFromString` 및 `JsonStringEnumConverter`를 등록하고 정적 캐싱하여 매 요청마다 옵션 재생성 오버헤드를 제거.
  2. 마크다운 코드블록(```` ```json ````) 우선 파싱, Index 0 기준 문자열 리터럴/이스케이프 추적 및 다중 후보군(Candidate) 순차 역직렬화 파이프라인 구현.
  3. 불완전하게 종료된 스트림에 대해 `RepairTruncatedJson`을 적용하여 열린 따옴표 및 스택에 남은 미닫힘 괄호(`]`, `}`)를 역순으로 닫고, 불완전한 trailing key/comma를 정리하여 데이터 복구율 극대화.

