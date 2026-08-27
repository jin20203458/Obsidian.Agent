---
description: >-
  EPPlus / OpenXML 기반 Excel 엑셀 파일 생성 시 발생하는 sharedStrings.xml 손상, 복구 경고,
  제어문자 오염 등 범용 트러블슈팅 런북. Excel 내보내기 오류 발생 시 참조.
related:
  - ../README.md
  - ./git_and_os.md
---
# Excel / EPPlus Troubleshooting
> **부제**: EPPlus 기반 .xlsx 파일 생성 시 Excel 복구 경고(Repaired Records) 및 XML 손상 해결 런북

본 문서는 C# EPPlus 라이브러리(또는 OpenXML SDK)로 `.xlsx` 파일을 생성할 때 Microsoft Excel에서
"파일에 오류가 있습니다 → 복구된 레코드: /xl/sharedStrings.xml 부분의 문자열 속성"
경고가 발생하는 원인과 해결 방법을 누적 기록합니다.

---

## 2026-08-27: EPPlus sharedStrings.xml "문자열 속성" 복구 경고

### 1. 현상 (Symptom)
* EPPlus 7.x로 생성한 `.xlsx` 파일을 Microsoft Excel로 열면 다음 경고가 표시됨:
  ```
  '파일명.xlsx' 파일에 오류가 있습니다.
  ```
* "예"를 누르면 Excel이 파일을 복구하고, `%TEMP%\errorNNNNNN_01.xml`에 복구 로그가 생성됨:
  ```xml
  <repairedRecord>복구된 레코드: /xl/sharedStrings.xml 부분의 문자열 속성 (문자열)</repairedRecord>
  ```
* 영문 환경에서는 `Repaired Records: String properties from /xl/sharedStrings.xml (String)`.
* 파일 자체는 열리고 데이터도 보이지만, 매번 복구 경고 팝업이 발생하여 사용자 경험이 저해됨.

### 2. 원인 (Root Cause)

**2단계에 걸친 복합 원인이 확인됨.**

#### 원인 A: Carriage Return (`\r`) → EPPlus `_x000D_` 이스케이프
* Windows 줄바꿈(`\r\n`)이 포함된 문자열을 EPPlus 셀에 삽입하면,
  EPPlus가 `\r`(0x0D)을 `_x000D_`라는 특수 토큰으로 이스케이프하여 `sharedStrings.xml`에 기록함.
* Excel의 OpenXML 파서는 `<t>` 태그 내부의 `_x000D_` 토큰을 손상된 문자열 속성으로 판정하고 복구를 시도.
* **해결**: 셀에 값을 넣기 전에 `\r\n` → `\n`, 단독 `\r` → `\n` 정규화 수행.

#### 원인 B: Raw Line Feed (`\n`) + `xml:space="preserve"` 조합
* 원인 A를 해결한 뒤에도 동일한 경고가 재발.
* 정밀 포렌식 분석 결과, **이것이 진짜 근본 원인(True Root Cause)**:
  1. 멀티라인 텍스트(코드 스니펫 등)를 셀에 삽입하면 EPPlus가 `\n`을 raw 바이트(0x0A)로 기록.
  2. EPPlus가 해당 공유 문자열에 `xml:space="preserve"` 속성을 자동 부여:
     ```xml
     <si><t xml:space="preserve">→   1 | /*
             ^
         2 |  *  FIPS-197 compliant AES
     </t></si>
     ```
  3. XML 1.0 규격상 raw LF는 유효하지만, **Excel의 OpenXML 파서는 이 조합을 "손상된 문자열 속성"으로 판정**.
  4. 전체 8,600개 공유 문자열 중 7,496개(87%)가 이 패턴에 해당하여 매번 복구 경고 발생.

**진단 과정 요약:**
```
(1) error*.xml 로그 확인 → sharedStrings.xml 문자열 속성 에러 확인
(2) xlsx를 zip으로 변환 → xl/sharedStrings.xml 추출
(3) _x000D_ 패턴 검색 → 0건 (원인 A는 이미 해결됨)
(4) raw 제어문자(0x00~0x1F) 바이트 스캔 → 0건
(5) _xHHHH_ EPPlus 유니코드 이스케이프 검색 → 0건
(6) xml:space="preserve" + raw LF(\n) 포함 <si> 엔트리 카운트 → 7,496건 / 8,600건
(7) 32,767 글자 초과 엔트리 검색 → 0건 (최대 228자)
(8) EPPlus 최소 재현 테스트 xlsx 생성 → 동일 현상 재현 확인
(9) 결론: EPPlus가 sharedStrings.xml에 기록하는 raw LF + xml:space="preserve" 조합이 근본 원인
```

### 3. 해결책 (Resolution)

#### 3-1. `\r` 정규화 (원인 A 대응)
셀 값 세정 유틸리티에서 모든 `\r\n` 및 단독 `\r`을 `\n`으로 치환:
```csharp
string normalized = text.Replace("\r\n", "\n").Replace("\r", "\n");
```

#### 3-2. 멀티라인 텍스트의 줄바꿈 제거 (원인 B 대응 — **핵심 해결책**)
셀에 삽입되는 멀티라인 문자열에서 모든 줄바꿈을 ` | ` 구분자로 치환하여 **단일 라인 문자열로 변환**:
```csharp
public static string SanitizeForExcelCell(string? text)
{
    if (string.IsNullOrEmpty(text)) return string.Empty;

    // 모든 줄바꿈을 " | " 구분자로 치환
    string singleLine = text
        .Replace("\r\n", " | ")
        .Replace("\r", " | ")
        .Replace("\n", " | ");

    // 연속 구분자 정리, 선행/후행 구분자 제거
    while (singleLine.Contains(" |  | "))
        singleLine = singleLine.Replace(" |  | ", " | ");
    singleLine = singleLine.Trim().TrimStart('|').TrimEnd('|').Trim();

    // XML 1.0 저위 제어문자 제거
    return Regex.Replace(singleLine, @"[\x00-\x08\x0B\x0C\x0E-\x1F]", "");
}
```

이렇게 하면 EPPlus가 `xml:space="preserve"` 없이 일반 `<t>` 태그로 기록하게 되어 Excel 복구 경고가 완전히 사라짐.

#### 3-3. 적용 범위
* **일반 텍스트 필드** (파일명, 체커명, 경로 등): `Sanitize()` 사용 (줄바꿈 정규화만).
* **멀티라인 필드** (코드 스니펫, 메시지 등): `SanitizeForExcelCell()` 사용 (줄바꿈 완전 제거).

#### 3-4. 추가 방어 코드
* XML 1.0 금지 제어문자(`0x00~0x08`, `0x0B~0x0C`, `0x0E~0x1F`) 일괄 제거 Regex 적용.
* 셀당 32,767자 초과 여부도 확인 가능 (본 건에서는 해당 없었으나, 대용량 데이터 시 유의).

### 4. 참고 사항
* **EPPlus 버전**: 7.4.2 (7.x 계열). 버전에 무관하게 동일한 XML 기록 방식 사용.
* **진단 도구**: `.xlsx` → `.zip` 확장자 변환 후 `xl/sharedStrings.xml` 추출 비교 (원본 vs Excel 복구본 diff).
* **WrapText 설정**: `Style.WrapText = true`를 설정해도 `xml:space="preserve"` + raw LF 문제는 해결되지 않음. 근본적으로 줄바꿈 자체를 제거해야 함.

---
