---
description: >-
  Phalanx C++ 센서 및 C# 코어 트러블슈팅 런북. Phalanx 프로젝트 버그, ETW 수집 오류 및 gRPC 장애 발생 시 참조.
related:
  - ../README.md
  - ../Phalanx/README.md
---
# Phalanx Troubleshooting Runbook

본 문서는 `Phalanx` EDR 솔루션(C++ 센서, gRPC 스트리밍, C# 코어 및 AI 에이전트) 개발 및 실전 모의 침투 테스트 중 발생하는 시스템 예외 현상과 해결 방안을 상세히 기록하는 중앙 런북입니다.

---

## 트러블슈팅 기록 템플릿 (작성 표준)

```markdown
## YYYY-MM-DD: [발생 이슈 요약]

### [현상 (Symptom)]
* 오류 메시지, 로그 내용, 비정상 동작 양상

### [원인 (Root Cause)]
* 코드, 시스템 콜, 동시성 또는 OS API 동작 메커니즘 차원의 근본 원인 분석

### [해결책 (Resolution)]
* 적용된 코드 패치, 구조 변경 및 해결 증빙 (Exit Code 0 / 빌드 확인)
```

---

## 사전 주의사항 및 알려진 기술적 고려점 (Known Constraints)

### 1. ETW 커널 세션 생성 권한 (Administrator Elevation)
* **현상**: 관리자 권한이 없는 일반 사용자 권한으로 센서 실행 시 `krabs-etw` 세션 생성 단계에서 `ACCESS_DENIED (0x5)` 예외 발생.
* **대응책**: `Phalanx.Sensor.exe`의 매니페스트 파일(`app.manifest`)에 `requireAdministrator` 실행 수준을 필수 명시할 것.

### 2. SyspendThread 데드락 예외 방지
* **현상**: 타깃 프로세스가 크리티컬 섹션이나 로더 락(LdrpLoaderLock)을 쥐고 있는 상태에서 강제 동결(Freeze) 시 시스템 전체 리소스 경합 발생 가능성.
* **대응책**: 프로세스 생성 직후 초기 진입점(Entry Point) 단계에서 빠르게 스레드를 인터럽트하거나, 타임아웃(최대 500ms) 가드를 둘 것.
