# Mundus Vivens: 트러블슈팅 및 런북 (Troubleshooting & Runbook)

본 문서는 Mundus Vivens 프로젝트(C# AI Server & C++ Game Server) 개발 과정에서 발생한 시스템 환경 오류, 빌드 에러, gRPC 프로토콜 및 아키텍처 정합성 관련 주요 문제와 그 해결 과정을 누적 기록하는 문서입니다.

---

## 2026-07-06: MSBuild pre-build 환경에서 `Get-FileHash` Cmdlet 미인식 오류

### 1. 현상 (Symptom)
* 새로운 PC 환경에서 C# 프로젝트(`MundusVivens.Prototype`) 빌드 시 pre-build 이벤트의 프로토 파일 동기화 프로세스에서 아래 오류가 발생하며 빌드 실패:
  ```text
  Get-FileHash : The term 'Get-FileHash' is not recognized as the name of a cmdlet, function, script file, or operable program.
  At line:1 char:193
  ... MSBuild Task failed (Exit Code 1)
  ```

### 2. 원인 (Root Cause)
* MSBuild가 pre-build 작업을 위해 백그라운드에서 구동하는 PowerShell 호스트 환경에 따라, 기본 유틸리티 모듈(`Microsoft.PowerShell.Utility`)이 로드되지 않거나 PowerShell 하위 호환성 모드로 실행되어 `Get-FileHash` 명령을 찾지 못함.

### 3. 해결책 (Resolution)
* `.csproj`의 `<Target Name="SyncProto">` 내부 실행 스크립트에서 외부 Cmdlet 종속성을 완전히 제거.
* 모든 .NET 기반 PowerShell 호스트에서 100% 동작이 보장되는 .NET 표준 API `[System.IO.File]::ReadAllText` 메소드를 사용하여 파일 내용을 비교하도록 스크립트 변경:
  ```powershell
  if ([System.IO.File]::ReadAllText($Cpp) -ne [System.IO.File]::ReadAllText($Cs)) { ... }
  ```
* 수정 후 `dotnet build` 시 정상 컴파일되는 것을 확인하고 변경점을 원격 저장소에 즉시 반영함.

---
