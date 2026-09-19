---
description: >-
  유니티(Unity) 클라이언트 렌더링 및 gRPC 연동 트러블슈팅 런북. 유니티 클라이언트 버그/통신 에러 발생 시 참조.
related:
  - ../README.md
---
# Unity Client Troubleshooting
> **부제**: 유니티 클라이언트 렌더링 및 gRPC 연동 장애 조치 로그

본 문서는 유니티(Unity) 게임 클라이언트의 렌더링, 씬 로딩, gRPC 통신 연동 및 기타 클라이언트 전용 에러와 해결 방법을 누적 기록하는 문서입니다.

---

## 2026-06-18: [Resolved] YetAnotherHttpHandler Safe Mode 컴파일 에러 (PipeReader/Pipelines 누락)

### 1. 현상 (Symptom)
* 프로젝트 클린 클론 후 처음 열었을 때 유니티가 Safe Mode에 갇히며 다음과 같은 컴파일 에러가 발생함:
  ```text
  Library\PackageCache\com.cysharp.yetanotherhttphandler@3171cb204b00\YetAnotherHttpHttpContent.cs(16,75): error CS0246: The type or namespace name 'PipeReader' could not be found (are you missing a using directive or an assembly reference?)
  Library\PackageCache\com.cysharp.yetanotherhttphandler@3171cb204b00\NativeHttpHandlerCore.cs(4,17): error CS0234: The type or namespace name 'Pipelines' does not exist in the namespace 'System.IO'
  ```
* 유니티 에디터가 Safe Mode 상태로 교착되어 백그라운드가 멈추고 `NuGetForUnity` 패키지 복원이 자동으로 수행되지 못함.

### 2. 원인 (Root Cause)
* `/Assets/Packages/` 폴더가 `.gitignore`에 등록되어 있어 NuGet DLL이 유입되지 않음.
* UPM(Unity Package Manager)을 통해 받은 `YetAnotherHttpHandler` 패키지는 하위 NuGet 종속성(System.IO.Pipelines 등)을 자동으로 내려받아주지 못하므로 종속성 결핍으로 컴파일이 깨짐.

### 3. 해결책 (Resolution)
* 유니티 루트 디렉토리에 **`restore_nuget.ps1`** 복원 스크립트를 생성하고 수동 가동하여 필요한 `.NET Standard 2.1` NuGet 어셈블리(총 12개 DLL)를 `Assets/Packages/`에 강제 빌드/복사 주입해 Safe Mode를 해제함.
* **자동 복원 스크립트 가동 명령어**:
  ```powershell
  powershell -ExecutionPolicy Bypass -File .\restore_nuget.ps1
  ```

---

## 2026-06-18: [Resolved] gRPC 프로토콜 HTTP/1.1 다운그레이드 에러 (Http2Only 강제화)

### 1. 현상 (Symptom)
* 로컬 C# gRPC 서버(`http://localhost:5001`)로 테스트 연결 시도 시 통신이 차단되며 아래 에러 발생:
  ```text
  [gRPC] Test call failed: Status(StatusCode="Internal", Detail="Bad gRPC response. Response protocol downgraded to HTTP/1.1.")
  ```

### 2. 원인 (Root Cause)
* TLS(HTTPS) 암호화가 적용되지 않은 순수 HTTP 주소(`http://`)를 통해 통신을 시도할 때, `YetAnotherHttpHandler`가 첫 핸드셰이크 단계에서 HTTP/2가 아닌 HTTP/1.1로 통신 사양을 하향(Downgrade) 요청하여 gRPC 규격 자체가 무산됨.

### 3. 해결책 (Resolution)
* `YetAnotherHttpHandler` 설정 옵션 내부에 **`Http2Only = true`** 속성을 활성화하여, 비암호화 통신 환경에서도 무조건 HTTP/2 Cleartext(h2c) 연결을 고수하도록 강제함.
  ```csharp
  var handler = new YetAnotherHttpHandler()
  {
      Http2Only = true // TLS 비활성화 환경(http://)에서 HTTP/2 통신 강제
  };
  ```

---

## 2026-07-12: [Resolved] 유니티 클라이언트 UI 업데이트 중단 및 UIManager 싱글톤 하이재킹 해결

### 1. 현상 (Symptom)
* 유니티 클라이언트를 실행했을 때, 3D 뷰어 상의 NPC 캡슐들은 정상적으로 움직이고 백그라운드에서는 스냅샷 패킷을 계속 수신 중임에도 불구하고, 화면 상단의 Current Tick, 좌측의 실시간 로그 패널, NPC 클릭 시 노출되어야 할 상세 정보 패널 등이 전혀 갱신되지 않고 초기 상태(혹은 멈춘 상태)로 방치되는 현상.
* 유니티 콘솔 및 `Editor.log` 상에 NullReferenceException 등 관련 에러나 예외 발생이 전혀 없이 침묵함.

### 2. 원인 (Root Cause)
* `SampleScene.unity` 씬 파일 내에 **동일한 `UIManager` 스크립트를 포함하는 서로 다른 두 개의 GameObject가 중복 존재**하고 있었음:
  1. 공백이 포함된 이름의 `"UIManager "` (ID 146213223) - 인스펙터 상의 UI 요소(`tickText`, `logText` 등)가 전혀 할당되지 않은 미완성 상태의 빈 오브젝트.
  2. 정상적인 이름의 `"UIManager"` (ID 583645022) - UI 오브젝트가 올바르게 할당된 실제 연동 오브젝트.
* 유니티가 씬을 로드하고 각 오브젝트의 `Awake()`를 실행할 때, 미완성 상태인 `"UIManager "`의 `Awake`가 먼저 호출되어 static 변수인 `UIManager.Instance` 싱글톤 참조를 선점(하이재킹)함.
* 실제 동작해야 하는 `"UIManager"`의 `Awake`가 실행되었을 때는 이미 `Instance`가 null이 아니므로 싱글톤 갱신을 생략함.
* 이로 인해 `PacketProcessor` 등 외부에서 `UIManager.Instance`를 통해 UI 변경을 시도할 때마다 필드가 전부 null인 첫 번째 인스턴스를 호출하게 됨.
* 각 UI 메서드는 null 체크(`if (tickText != null)` 등)를 안전하게 수행하고 있었기에 예외를 던지지 않고 실행을 종료하여 에러 로그도 없이 화면만 갱신되지 않는 침묵 장애가 발생함.

### 3. 해결책 (Resolution)
* `SampleScene.unity` 파일을 직접 수정하여 미완성 상태였던 중복 오브젝트 `"UIManager "` (ID 146213223) 및 하부의 MonoBehaviour, Transform 컴포넌트를 완전히 삭제함.
* 씬의 루트 목록(`SceneRoots.m_Roots`)에서도 해당 오브젝트의 Transform ID (`146213225`) 엔트리를 삭제함.
* 이를 통해 씬 상에 유일한 UIManager 오브젝트만 남겨 싱글톤 선점 문제를 제거하였으며, 클라이언트 실행 시 Tick 수치, 실시간 대사 로그, NPC 상세 패널이 실시간 갱신되는 것을 확인함.

---

## 2026-07-12: [Resolved] 한국어 폰트(malgun SDF) 깨짐 및 URP 셰이더 미호환 오류

### 1. 현상 (Symptom)
* 월드 상의 NPC 머리 위에 출력되는 한글 텍스트(이름, 상태, 말풍선 대사 등)가 모두 깨져서 노란색/주황색 점선이나 박스 모양의 그래픽 노이즈 형태로 출력됨.

### 2. 원인 (Root Cause)
* 한국어 폰트 파일인 `malgun SDF.asset`이 참조하는 기본 셰이더가 이전 빌트인 파이프라인 전용 셰이더(`TextMeshPro/Distance Field`)로 설정되어 있어, URP(Universal Render Pipeline) 환경과 호환되지 않아 그래픽이 완전히 깨져 렌더링됨.

### 3. 해결책 (Resolution)
* `malgun SDF.asset` 파일의 셰이더 참조 GUID를 URP 호환 셰이더(`TextMeshPro/Mobile/Distance Field` 또는 `TextMeshPro/Distance Field SSD` 등)인 `fe393ace9b354375a9cb14cdbbc28be4`로 파일 직접 수정을 통해 변경 및 재임포트 완료.

---

## 2026-07-12: [Resolved] 탑다운 뷰 3D 텍스트 겹침 및 동적 말풍선 바코드(찌그러짐) 렌더링 오류

### 1. 현상 (Symptom)
* 카메라가 수직으로 월드를 바라보는 탑다운(Top-down) 뷰 형태이므로, 텍스트들의 월드 높이(Y축)에 차이를 주더라도 화면상에서는 같은 중심선에 겹쳐서 투영되어 모든 글씨가 중복되어 뭉쳐 보이던 현상.
* 동적으로 생성되는 말풍선 대사(`SpeechBubbleText`)가 정상적으로 읽히지 않고 얇은 노란색 세로줄(바코드 형태)로 심하게 압축되거나 비정상적으로 줄바꿈되어 출력되는 현상.
* `술집(Tavern)`과 `뒷골목(Back Alley)` 등 인접한 구역의 랜드마크 파란색 큐브 위에 출력되는 이름 글자(예: `"술집 (Tavern)"`)가 심하게 겹치고 뭉개져 식별하기 어려운 현상.

### 2. 원인 (Root Cause)
* 탑다운 카메라 시점에서의 수직 화면 공간은 월드의 Z축에 대응하므로, Y축 오프셋은 깊이에만 영향을 줄 뿐 화면상에서는 겹치게 됨.
* 동적으로 생성한 GameObject에 `TextMeshPro` 컴포넌트를 추가하면 기본 텍스트 박스 크기(`sizeDelta`)가 `(2, 2)`로 매우 좁게 생성되며, 기본 폰트도 한국어를 지원하지 않는 `Liberation Sans SDF`로 설정됨. 여기에 한국어 긴 문자열이 유입되면 글자 단위 줄바꿈(Word Wrapping)이 일어나 수십 줄의 텍스트가 한 곳에 겹쳐 바코드처럼 보이게 됨. 또한 부모 NPC 오브젝트의 크기(Scale)가 비균등하게 왜곡되어 있으면 자식 텍스트도 동일한 왜곡 비율을 상속함.
* `GameManager.cs`에서 랜드마크 스폰 시 글씨 크기(`fontSize`)가 초기 원거리 뷰용 거대 크기인 `12.0f`로 고정되어 있어, 인접 건물 간 글자가 수백 유닛 길이로 침범하여 겹침.

### 3. 해결책 (Resolution)
1. **Z축 오프셋 재정렬**: 탑다운 시점 상/하 정렬을 위해 텍스트 배치 오프셋을 Y축이 아닌 Z축 오프셋으로 전면 변경 (이름: Z +1.2, 상태: Z -1.2, 대화 말풍선: Z +2.5).
2. **동적 말풍선 규격 교정**:
   * 동적 생성 말풍선 객체의 폰트에 이미 URP 수정 폰트가 적용된 `nameText.font` 속성을 런타임에 직접 복사하여 한글 깨짐 방지.
   * `transform.localScale = Vector3.one` 명시로 왜곡 스케일 상속 차단.
   * `rectTransform.sizeDelta = new Vector2(30f, 5f)` 및 적절한 `enableWordWrapping` 적용으로 가로폭 확보.
3. **랜드마크 폰트 크기 조정**: 랜드마크 글자 크기를 `12.0f`에서 `3.0f`로 축소하고 `enableWordWrapping = false`를 설정하여 깔끔한 단일 행 라인으로 정렬.

---

## 2026-07-12: [Resolved] NPC 마우스 클릭 상세 정보 패널 활성화 오류 (CapsuleCollider 계층 구조)

### 1. 현상 (Symptom)
* 유니티 뷰어 내에서 3D 캡슐 모양의 NPC를 마우스로 직접 클릭해도, 좌측 하단의 NPC 상세 상태 정보 패널(NPC Details Panel)이 열리지 않거나 아무런 작동도 일어나지 않는 현상.

### 2. 원인 (Root Cause)
* 유니티의 내장 클릭 이벤트 수신 메서드인 `OnMouseDown`은 `Collider` 컴포넌트가 부착되어 있는 동일한 GameObject의 스크립트에서만 동작함.
* 스폰 구조상 물리 클릭을 처리하는 `NpcController` 스크립트는 부모 GameObject(`go`)에 부착되어 있으나, 실제 충돌판정을 결정하는 3D `CapsuleCollider`는 `GameObject.CreatePrimitive`를 통해 자동 생성된 자식 GameObject(`capsule`)에 부착되어 있었음.
* 이 때문에 Raycast 충돌 이벤트가 부모로 전달되지 못하고 자식 레벨에서 삼켜져, 부모의 `OnMouseDown()` 메서드가 호출되지 않는 구조적 결함이었음.

### 3. 해결책 (Resolution)
* `GameManager.cs`의 NPC 스폰 로직 내에서 자식 캡슐 객체에 붙어있던 콜라이더를 `Destroy()`로 파괴함.
* `NpcController`가 부착되어 있는 부모 GameObject에 직접 `CapsuleCollider`를 추가하여 물리 판정 및 Raycast가 부모 레벨에서 직접 트리거되도록 교정함 (`center = (0, 1.5, 0)`, `height = 3.0f`, `radius = 0.75f`).
* 이를 통해 NPC 클릭 시 정상적으로 `OnMouseDown`이 호출되어 좌측 하단 정보 패널이 활성화됨을 확인함.
