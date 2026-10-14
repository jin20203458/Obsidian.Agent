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
