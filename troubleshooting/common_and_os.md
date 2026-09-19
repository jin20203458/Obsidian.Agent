---
description: >-
  특정 프로젝트에 종속되지 않는 공통 OS(Windows/Linux), Git 버전 관리, 파일 인코딩 및 범용 라이브러리(OpenXML, 직렬화 등) 트러블슈팅 런북. 공통 환경/OS/라이브러리 문제 발생 시 참조.
related:
  - ../README.md
---
# Common & OS Troubleshooting

본 문서는 특정 프로젝트에 국한되지 않고, 전역 OS 환경, 파일 시스템 권한, Git 버전 관리, 인코딩 변환 및 범용 기술 스택(OpenXML, 직렬화 등)에서 발생하는 문제와 해결 방법을 누적 기록하는 공통 런북입니다.

---

## 2026-08-27: [Resolved] OpenXML/EPPlus 엑셀 내보내기 파일 손상 및 sharedStrings.xml 복구 오류

### 1. 현상 (Symptom)
- .NET 애플리케이션에서 EPPlus 또는 OpenXML 라이브러리를 통해 대량 데이터(수천~수만 건)를 엑셀 파일로 내보낸 후 Microsoft Excel로 열 때 다음과 같은 경고창 발생:
  > "'filename.xlsx' 파일에 문제가 있는 것을 발견했습니다. 이 통합 문서의 내용을 최대한 복구하시겠습니까?"
- 복구 시도 시 `AppData/Local/Temp/error*.xml` 파일이 생성되며 다음과 같은 로그 기록:
  ```xml
  <recoveryLog xmlns="http://schemas.openxmlformats.org/spreadsheetml/2006/main">
    <summary>'filename.xlsx' 파일에 오류가 있습니다.</summary>
    <repairedRecords>
      <repairedRecord>복구된 레코드: /xl/sharedStrings.xml 부분의 문자열 속성 (문자열)</repairedRecord>
    </repairedRecords>
  </recoveryLog>
  ```

### 2. 원인 (Root Cause)
1. **Microsoft Excel 단일 셀 최대 글자 수 한계(32,767자) 초과**:
   - Microsoft Excel 공식 사양상 단일 셀에 저장 가능한 최대 문자 수는 **32,767자($2^{15}-1$)**입니다.
   - 소스 코드 스니펫, 대용량 로그 텍스트, 호출 스택(Stack Trace) 등 긴 멀티라인 텍스트를 셀에 삽입할 때 일부 셀이 40,000자 이상으로 비대해지는 경우가 발생합니다.
   - OpenXML 스키마 검증기(`OpenXmlValidator`) 상으로는 XML 구조 자체에 결함이 없더라도, Microsoft Excel 내부 메모리 할당기가 32,767자를 초과하는 셀을 수용하지 못해 `/xl/sharedStrings.xml` 레코드를 강제 절단 및 복구 대상으로 처리합니다.
2. **Carriage Return(`\r`)의 `_x000D_` 인코딩 및 XML 1.0 제어문자**:
   - Windows 줄바꿈(`\r\n`)이 포함된 문자열을 EPPlus에 전달하면, EPPlus 내부 직렬화기가 `\r` (ASCII 13)을 `_x000D_` 토큰으로 치환하여 `<si><t>..._x000D_...</t></si>` 형태로 저장합니다.
   - Microsoft Excel의 OpenXML 파서는 `<t>` 태그 내부의 `_x000D_`를 유효하지 않은 문자열 속성으로 취급하여 복구 오류를 발생시킵니다.
   - 또한 XML 1.0 규격상 `0x00~0x08`, `0x0B~0x0C`, `0x0E~0x1F` 범위의 ASCII 저위 제어문자는 XML 문서 내에 포함될 수 없습니다.

### 3. 해결책 (Resolution)
1. **엑셀 셀 전용 문자열 세정 유틸리티(`XmlSanitizer`) 구현**:
   - 모든 셀 데이터를 엑셀 패키지에 할당하기 전에 3단계 정규화 파이프라인(줄바꿈 `\n` 통일, 불법 제어문자 제거, 32,000자 Clamping)을 거치도록 처리했습니다.
   ```csharp
   using System;
   using System.Text.RegularExpressions;

   public static class XmlSanitizer
   {
       // Excel 단일 셀 최대 글자 수 한계(32,767자)에 대한 안전 상한선
       private const int MaxExcelCellLength = 32000;

       private static readonly Regex InvalidXmlCharsRegex = new(
           @"[\x00-\x08\x0B\x0C\x0E-\x1F]",
           RegexOptions.Compiled);

       /// <summary>
       /// 1) 줄바꿈 정규화 (CRLF -> LF)
       /// 2) XML 1.0 금지 제어문자 제거
       /// 3) 32,000자 안전 상한선 적용 (Excel 복구 오류 방지)
       /// </summary>
       public static string Sanitize(string? text)
       {
           if (string.IsNullOrEmpty(text)) return string.Empty;

           // 1. CRLF 및 단독 CR을 LF로 변환
           string normalized = text.Replace("\r\n", "\n").Replace("\r", "\n");

           // 2. XML 1.0 저위 제어문자 제거
           string clean = InvalidXmlCharsRegex.Replace(normalized, "");

           // 3. 32,000자 초과 시 안전 절단
           if (clean.Length > MaxExcelCellLength)
           {
               clean = clean.Substring(0, MaxExcelCellLength) + "\n... [Truncated: Exceeded Excel 32k Cell Limit]";
           }

           return clean;
       }
   }
   ```
2. **엑셀 생성 파이프라인 전수 적용**:
   - 엑셀 시트 생성 시 셀 값을 할당하는 모든 코드 경로에서 `XmlSanitizer.Sanitize(value)`를 호출하여 오류를 원천 방어했습니다.
   - 멀티라인 텍스트를 담는 열에는 `sheet.Cells[row, col].Style.WrapText = true;`를 명시적으로 활성화했습니다.
