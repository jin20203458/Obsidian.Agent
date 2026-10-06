---
description: 유니티(Unity) 클라이언트 렌더링 및 gRPC 연동 트러블슈팅 런북.
related:
  - ../README.md
  - ../MundusVivens/README.md
---

# Unity Client Troubleshooting

본 문서는 유니티(Unity) 게임 클라이언트의 렌더링, 씬 로딩, gRPC 통신 연동 및 기타 클라이언트 전용 에러와 해결 방법을 누적 기록하는 문서입니다.

---

## 2026-06-18: [Resolved] YetAnotherHttpHandler Safe Mode 컴파일 에러 (PipeReader/Pipelines 누락)

### 1. 현상 (Symptom)
* 프로젝트 클린 클론 후 처음 열었을 때 `PipeReader`, `Pipelines` 누락 CS0246/CS0234 컴파일 에러로 유니티 에디터가 Safe Mode에 고착되어 `NuGetForUnity` 패키지 자동 복원이 중단됨.

### 2. 원인 (Root Cause)
* UPM을 통해 임포트된 `YetAnotherHttpHandler`가 내부적으로 `System.IO.Pipelines` NuGet 패키지에 의존하나, `/Assets/Packages/`가 `.gitignore` 대상이어서 클린 환경에서 DLL이 누락됨.
* Safe Mode 상태에서는 유니티 에디터 스크립트 실행이 중단되어 NuGet 패키지 복원이 자동 트리거되지 못하는 교착 상태 발생.

### 3. 해결책 (Resolution)
* 에디터 외부 PowerShell 복원 스크립트(`Scripts/restore_nuget.ps1`)를 실행하여 `System.IO.Pipelines` 및 `System.Buffers` 어셈블리를 `Assets/Packages/`로 사전 수동 복원 후 에디터 재시작.

---

## 2026-06-18: [Resolved] gRPC 프로토콜 HTTP/1.1 다운그레이드 에러 (Http2Only 강제화)

### 1. 현상 (Symptom)
* 유니티 클라이언트에서 C++ 물리 서버 및 C# AI 서버로 gRPC 채널 연결 시 `RpcException: Status(StatusCode="Internal", Detail="Bad gRPC response. Response protocol downgraded to HTTP/1.1.")` 예외 발생 및 통신 두절.

### 2. 원인 (Root Cause)
* `YetAnotherHttpHandler`는 기본적으로 HTTP/1.1 및 HTTP/2 프로토콜 협상을 시도함.
* 비암호화 통신(`http://`) 환경에서 서버가 ALPN 없이 응답할 경우 클라이언트가 HTTP/1.1로 자동 다운그레이드를 시도하여 gRPC 스트림이 파괴됨.

### 3. 해결책 (Resolution)
* `GrpcChannelOptions.HttpHandler` 생성 시 `Http2Only = true`를 강제 설정하여 HTTP/1.1 다운그레이드를 원천 차단:
  ```csharp
  var handler = new YetAnotherHttpHandler
  {
      Http2Only = true
  };
  var channel = GrpcChannel.ForAddress("http://localhost:50051", new GrpcChannelOptions { HttpHandler = handler });
  ```
* **부정 제약**: 비암호화 로컬 gRPC 통신 환경에서는 절대 `Http2Only = true` 옵션을 누락하지 말 것.

---

> [!NOTE]
> **[정제 완료 기준선]** 2026-10-06 이전 상위 항목은 정제 완료됨. 신규 인시던트는 이 아래에 추가됩니다.

---
