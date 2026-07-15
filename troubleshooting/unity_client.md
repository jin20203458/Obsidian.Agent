---
type: runbook
title: Unity Client Troubleshooting
tags: [troubleshooting, unity, client, runbook]
related:
  - ../README.md
last_updated: 2026-07-15
status: stable
---
# Unity Client: 클라이언트 트러블슈팅 (Unity Client Troubleshooting)

본 문서는 유니티(Unity) 게임 클라이언트의 렌더링, 씬 로딩, gRPC 통신 연동 및 기타 클라이언트 전용 에러와 해결 방법을 누적 기록하는 문서입니다.

---

## 2026-06-18: Unity gRPC 연동 및 YetAnotherHttpHandler 컴파일 에러 해결

### 1. PipeReader/Pipelines 누락 (Safe Mode 교착 상태)

#### 현상 (Symptom)
* 프로젝트 클린 클론 후 처음 열었을 때 유니티가 Safe Mode에 갇히며 다음과 같은 컴파일 에러가 발생함.
  ```text
  Library\PackageCache\com.cysharp.yetanotherhttphandler@3171cb204b00\YetAnotherHttpHttpContent.cs(16,75): error CS0246: The type or namespace name 'PipeReader' could not be found (are you missing a using directive or an assembly reference?)
  Library\PackageCache\com.cysharp.yetanotherhttphandler@3171cb204b00\NativeHttpHandlerCore.cs(4,17): error CS0234: The type or namespace name 'Pipelines' does not exist in the namespace 'System.IO'
  ```
* 유니티 에디터가 Safe Mode 상태로 교착되어 백그라운드가 멈추고 `NuGetForUnity` 패키지 복원이 자동으로 수행되지 못함.

#### 원인 (Root Cause)
* `/Assets/Packages/` 폴더가 `.gitignore`에 등록되어 있어 NuGet DLL이 유입되지 않음.
* UPM(Unity Package Manager)을 통해 받은 `YetAnotherHttpHandler` 패키지는 하위 NuGet 종속성(System.IO.Pipelines 등)을 자동으로 내려받아주지 못하므로 종속성 결핍으로 컴파일이 깨짐.

#### 해결책 (Resolution)
* 유니티 루트 디렉토리에 **`restore_nuget.ps1`** 복원 스크립트를 생성하고 수동 가동하여 필요한 `.NET Standard 2.1` NuGet 어셈블리(총 12개 DLL)를 `Assets/Packages/`에 강제 빌드/복사 주입해 Safe Mode를 해제함.
  
* **자동 복원 스크립트 가동 명령어**:
  ```powershell
  powershell -ExecutionPolicy Bypass -File .\restore_nuget.ps1
  ```

---

### 2. gRPC 프로토콜 HTTP/1.1 다운그레이드 에러 (Http2Only 강제화)

#### 현상 (Symptom)
* 로컬 C# gRPC 서버(`http://localhost:5001`)로 테스트 연결 시도 시 통신이 차단되며 아래 에러 발생:
  ```text
  [gRPC] Test call failed: Status(StatusCode="Internal", Detail="Bad gRPC response. Response protocol downgraded to HTTP/1.1.")
  ```

#### 원인 (Root Cause)
* TLS(HTTPS) 암호화가 적용되지 않은 순수 HTTP 주소(`http://`)를 통해 통신을 시도할 때, `YetAnotherHttpHandler`가 첫 핸드셰이크 단계에서 HTTP/2가 아닌 HTTP/1.1로 통신 사양을 하향(Downgrade) 요청하여 gRPC 규격 자체가 무산됨.

#### 해결책 (Resolution)
* `YetAnotherHttpHandler` 설정 옵션 내부에 **`Http2Only = true`** 속성을 하드코딩으로 활성화하여, 비암호화 통신 환경에서도 무조건 HTTP/2 Cleartext(h2c) 연결을 고수하도록 강제함.
  ```csharp
  var handler = new YetAnotherHttpHandler()
  {
      Http2Only = true // TLS 비활성화 환경(http://)에서 HTTP/2 통신 강제
  };
  ```

---

## 2026-07-12: 유니티 클라이언트 UI 업데이트 중단 및 UIManager 싱글톤 하이재킹 해결

### 1. 현상 (Symptom)
* 유니티 클라이언트를 실행했을 때, 3D 뷰어 상의 NPC 캡슐들은 정상적으로 움직이고 백그라운드에서는 스냅샷 패킷을 계속 수신 중임에도 불구하고, 화면 상단의 Current Tick, 좌측의 실시간 로그 패널, NPC 클릭 시 노출되어야 할 상세 정보 패널 등이 전혀 갱신되지 않고 초기 상태(혹은 멈춘 상태)로 방치되는 현상.
* 유니티 콘솔 및 `Editor.log` 상에 NullReferenceException 등 관련 에러나 예외 발생이 전혀 없이 침묵함.

### 2. 원인 (Root Cause)
* `SampleScene.unity` 씬 파일 내에 **동일한 `UIManager` 스크립트를 포함하는 서로 다른 두 개의 GameObject가 중복 존재**하고 있었음:
  1. 공백이 포함된 이름의 `"UIManager "` (ID 146213223) - 인스펙터 상의 UI 요소(`tickText`, `logText` 등)가 전혀 할당되지 않은 미완성 상태의 빈 오브젝트.
  2. 정상적인 이름의 `"UIManager"` (ID 583645022) - UI 오브젝트가 올바르게 할당된 실제 연동 오브젝트.
* 유니티가 씬을 로드하고 각 오브젝트의 `Awake()`를 실행할 때, 미완성 상태인 `"UIManager "`의 `Awake`가 먼저 호출되어 static 변수인 `UIManager.Instance` 싱글톤 참조를 선점(하이재킹)함.
* 실제 동작해야 하는 `"UIManager"`의 `Awake`가 실행되었을 때는 이미 `Instance`가 null이 아니므로 싱글톤 갱신을 생략함.
* 이로 인해 `PacketProcessor` 등 외부에서 `UIManager.Instance`를 통해 UI 변경을 시도할 때마다, 필드가 전부 null인 첫 번째 인스턴스를 호출하게 됨.
* 각 UI 메서드는 null 체크(`if (tickText != null)` 등)를 안전하게 수행하고 있었기에 예외를 던지지 않고 그냥 실행을 종료하여 아무런 에러 로그도 없이 화면만 갱신되지 않는 현상이 발생함.

### 3. 해결책 (Resolution)
* `SampleScene.unity` 파일을 직접 수정하여 미완성 상태였던 중복 오브젝트 `"UIManager "` (ID 146213223) 및 하부의 MonoBehaviour, Transform 컴포넌트를 완전히 삭제함.
* 씬의 루트 목록(`SceneRoots.m_Roots`)에서도 해당 오브젝트의 Transform ID (`146213225`) 엔트리를 삭제함.
* 이를 통해 씬 상에 유일한 UIManager 오브젝트만 남겨 싱글톤 선점 문제가 근본적으로 제거되었으며, 클라이언트 실행 시 정상적으로 Tick 수치, 실시간 연극 대사 로그, NPC 상세 패널이 실시간 갱신되는 것을 확인함.

---

## 2026-07-12: 한국어 폰트 깨짐, 3D 텍스트 겹침, 카메라 조작 및 대화 범위 한계 개선

### 1. 한국어 폰트(malgun SDF) 깨짐 및 셰이더 미호환 오류
#### 현상 (Symptom)
* 월드 상의 NPC 머리 위에 출력되는 한글 텍스트(이름, 상태, 말풍선 대사 등)가 모두 깨져서 노란색/주황색 점선이나 박스 모양의 심각한 그래픽 노이즈 형태로 출력됨.
#### 원인 (Root Cause)
* 한국어 폰트 파일인 `malgun SDF.asset`이 참조하는 기본 셰이더가 이전 빌트인 파이프라인 전용 셰이더(`TextMeshPro/Distance Field`)로 설정되어 있어, URP(Universal Render Pipeline) 환경과 호환되지 않아 그래픽이 완전히 깨져 렌더링됨.
#### 해결책 (Resolution)
* `malgun SDF.asset` 파일의 셰이더 참조 GUID를 URP 호환 셰이더(`TextMeshPro/Mobile/Distance Field` 또는 `TextMeshPro/Distance Field SSD` 등)인 `fe393ace9b354375a9cb14cdbbc28be4`로 파일 직접 수정을 통해 변경 및 재임포트 완료.

### 2. 카메라 조작성 및 줌인 한계 조절
#### 현상 (Symptom)
* 맵 전체가 너무 멀리(Y=150)에서 렌더링되어 관찰이 어렵고, 기존 줌 로직은 `transform.forward`에 의존하여 카메라 부모 오프셋 등에 의해 오작동함.
#### 해결책 (Resolution)
* `CameraController.cs` 내에서 카메라 높이 조절 기준의 최소 높이(`minHeight`)를 `20.0f`에서 `5.0f`로 낮춰 더 가까운 거리까지 줌인 가능하게 변경.
* 마우스 휠 외에 `Q`/`E` 및 `Page Up`/`Page Down` 키를 통한 키보드 줌 조작 추가.
* `transform.forward` 대신 카메라의 월드 Y 좌표를 물리적으로 직접 증감시키는 방식으로 줌 방식을 전면 개편하여 카메라 오프셋/회전 상태와 무관하게 부드럽게 높낮이가 조절되도록 수정.

### 3. 3D 공간 상의 이름, 상태, 대화 말풍선 중복 겹침 및 말풍선 글자 바코드 현상(찌그러짐) 오류
#### 현상 (Symptom)
* 카메라가 수직으로 월드를 바라보는 탑다운(Top-down) 뷰 형태이므로, 텍스트들의 월드 높이(Y축)에 차이를 주더라도 화면상에서는 같은 중심선에 겹쳐서 투영되어 모든 글씨가 중복되어 뭉쳐 보이던 현상.
* 또한, 동적으로 생성되는 말풍선 대사(`SpeechBubbleText`)가 정상적으로 읽히지 않고 얇은 노란색 세로줄(바코드 형태)로 심하게 압축되거나 비정상적으로 줄바꿈되어 출력되는 현상.
* `술집(Tavern)`과 `뒷골목(Back Alley)` 등 인접한 구역의 랜드마크 파란색 큐브 위에 출력되는 이름 글자(예: `"술집 (Tavern)"`)가 심하게 겹치고 뭉개져 식별하기 어려운 현상.
#### 원인 (Root Cause)
* 탑다운 카메라 시점에서의 수직 화면 공간은 월드의 Z축에 대응하므로, Y축 오프셋은 깊이(크기)에만 영향을 줄 뿐 화면상에서는 겹치게 됨.
* 동적으로 생성한 GameObject에 `TextMeshPro` 컴포넌트를 추가하면 기본 텍스트 박스 크기(`sizeDelta`)가 `(2, 2)`로 매우 좁게 생성되며, 기본 폰트도 한국어를 지원하지 않는 `Liberation Sans SDF`로 설정됩니다. 여기에 한국어 긴 문자열이 들어가면 글자 한두 개 단위로 줄바꿈(Word Wrapping)이 일어나며 수십 개의 텍스트 라인이 한 곳에 수직으로 겹쳐 뭉개짐에 따라 바코드처럼 보이게 되고, 폰트 파일 미지정으로 글자가 보이지 않거나 네모 칸 등의 깨짐 기호로 대체됩니다. 또한, 부모 NPC 오브젝트의 크기(Scale)가 비균등(Non-uniform)하게 찌그러져 있을 경우 하위 자식들도 강제로 동일한 찌그러짐 비율을 상속함.
* `GameManager.cs`에서 랜드마크 스폰 시 글씨 크기(`fontSize`)가 초기 Y=150 카메라 시점용 거대 크기인 `12.0f`로 고정되어 있었음. 랜드마크의 물리적 배치 간격에 비해 텍스트 크기가 압도적으로 커서, 인접한 술집(30, 40)과 뒷골목(15, 50)의 글자가 서로 수백 유닛 길이로 침범하여 뭉개지고 겹쳐진 현상임.
#### 해결책 (Resolution)
* 탑다운 카메라 시점에서의 상/하 정렬을 위해 텍스트의 배치 오프셋을 Y축이 아닌 Z축 오프셋으로 전면 변경.
  * **이름 텍스트**: 캐릭터 뒤쪽(Z축 +1.2)
  * **상태 텍스트**: 캐릭터 앞쪽(Z축 -1.2)
  * **대화 말풍선**: 캐릭터 한참 뒤쪽(Z축 +2.5)
* 줌인에 대응하도록 폰트 크기(`fontSize`)를 기존 대비 2~3배 축소(`1.3f`~`1.8f`)하여 화면에 겹침 없이 출력되도록 개선.
* 동적으로 생성한 말풍선 객체의 폰트에 이미 프리프레젠테이션 프리팹을 통해 URP 수정 폰트가 잘 입혀져 있는 `nameText`나 `statusText`의 `font` 속성(`tmpro.font = nameText.font`)을 런타임에 직접 복사하여 한국어 깨짐 현상을 근본적으로 차단함.
* 동적 생성 말풍선의 스케일에 `transform.localScale = Vector3.one`을 명시하여 부모의 왜곡된 스케일 상속을 방지하고, `tmpro.rectTransform.sizeDelta = new Vector2(30f, 5f)`로 가로 크기를 충분히 크게 늘리고 자동 줄바꿈(`enableWordWrapping = true`) 조건을 적절히 지정하여 한글 대사가 찌그러짐 없이 한눈에 한 줄로 들어오도록 보정함.
* `GameManager.cs` 내에서 랜드마크 이름 표시 글자 크기(`fontSize`)를 기존 `12.0f`에서 `3.0f`로 대폭 축소하고, `enableWordWrapping = false`를 주어 텍스트가 여러 줄로 쪼개져 큐브 주위를 어지럽히지 않고 깔끔한 한 줄 라인으로 정렬되도록 수정함.

---

## 2026-07-12: NPC 마우스 클릭 상세 정보 패널 활성화 오류

### 1. 현상 (Symptom)
* 유니티 뷰어 내에서 3D 캡슐 모양의 NPC를 마우스로 직접 클릭해도, 좌측 하단의 NPC 상세 상태 정보 패널(NPC Details Panel)이 연리지 않거나 아무런 작동도 일어나지 않는 현상.

### 2. 원인 (Root Cause)
* 유니티의 내장 클릭 이벤트 수신 메서드인 `OnMouseDown`은 `Collider` 컴포넌트가 붙어 있는 동일한 GameObject의 스크립트에서만 동작함.
* 스폰 구조상 물리 클릭을 처리하는 `NpcController` 스크립트는 부모 GameObject(`go`)에 부착되어 있으나, 실제 충돌판정을 결정하는 3D `CapsuleCollider`는 `GameObject.CreatePrimitive`를 통해 자동 생성된 자식 GameObject(`capsule`)에 부착되어 있었음.
* 이 때문에 레이캐스트(클릭) 충돌 이벤트가 부모로 전달되지 못하고 자식 레벨에서 삼켜져, 부모의 `OnMouseDown()` 메서드가 전혀 호출되지 못하고 씹히는 구조적 버그였음.

### 3. 해결책 (Resolution)
* `GameManager.cs`의 NPC 스폰 로직 내에서 자식 캡슐 객체에 붙어있던 기존 콜라이더를 `Destroy()`로 즉시 파괴함.
* `NpcController`가 부착되어 있는 부모 GameObject에 직접 `CapsuleCollider`를 추가하여 물리 판정 및 Raycast가 부모 레벨에서 직접 트리거되도록 조정함.
  * 부모 콜라이더 규격: `center = (0, 1.5, 0)`, `height = 3.0f`, `radius = 0.75f`로 캡슐 크기에 맞춰 정밀 조율.
* 이를 통해 유니티 실행 후 NPC를 클릭하면 정상적으로 `OnMouseDown`이 호출되어 좌측 하단에 스냅샷 데이터 기반의 감정, 생체 욕구, 관계망 정보 패널이 온전하게 활성화되는 것을 확인함.
