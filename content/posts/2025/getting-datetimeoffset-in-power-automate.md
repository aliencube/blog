---
title: "Power Automate 워크플로우에서 DateTimeOffset 값 추출하기"
slug: getting-datetimeoffset-in-power-automate
description: "이 포스트에서는 Power Automate 워크플로우를 사용할 때 DateTimeOffset 값을 추출하는 방법에 대해 알아봅니다."
date: "2025-01-15"
author: Justin-Yoo
tags:
- power-automate
- excel
- office-scripts
- power-platform
cover: /2025/01/getting-datetimeoffset-in-power-automate-00.png
fullscreen: true
---

[Power Automate][pau] 워크플로우를 작성하다보면 정말 자주 쓰는 함수식/표현식 중 하나가 바로 [날짜계산 관련][pau functions]이다. 날짜 계산은 타임존과 관련이 있어서 UTC 시각을 기준으로 원하는 타임존으로 변환하는 작업이 필요하다. 이 때 `utcNow()`, `convertTimeZone()`, `convertFromUtc()`, `convertToUtc()` 같은 함수들을 사용한다.

그런데, 문제는 변환 전/후의 값을 보면 분명히 UTC 시각이 로컬 시각으로 바뀌긴 했지만, 그 형식은 항상 `2025-01-15T12:34:56Z`과 같아서, 실제 타임존 오프셋을 반영하지 않는다. 즉, 이 값을 [ISO-8601][iso8601] 형식으로 바꾸면 `2025-01-15T12:34:56+00:00`이 된다. 타임존을 변경해도 항상 `+00:00`이 붙는 문제가 있다.

따라서, 날짜 문자열 형식에 타임존 값을 표현하고 싶다면 별도로 타임존 오프셋 값을 구해야 한다. 이번 포스트에서는 Power Automate 워크플로우에서 [Excel][m365 excel]의 [Office Scripts][m365 excel office scripts] 기능을 이용해 타임존 오프셋 값을 추출하는 방법에 대해 알아보기로 한다.

## Office Scripts로 타임존 오프셋 값 추출하기

우선 Excel에서 Office Scripts를 사용해 타임존 오프셋 값을 추출하는 방법을 알아보자. Office Scripts는 기본적으로 JavaScripts와 동일하므로 JavaScript에 익숙하다면 쉽게 사용할 수 있다. 다음은 Office Scripts를 사용해 타임존 오프셋 값을 추출하는 코드이다.

```javascript
function main(
    workbook: ExcelScript.Workbook,
    // timestamp in 'yyyy-MM-dd' format
    timestamp: string,
    // timezone value in IANA format
    timezone: string): string {

    const date = new Date(`${timestamp}T00:00:00Z`);

    const timestampInTZ = date.toLocaleString("en-US", { timeZone: timezone });
    const dateInTZ = new Date(timestampInTZ);

    const timestampInUTC = date.toLocaleString("en-US", { timeZone: 'UTC' });
    const dateInUTC = new Date(timestampInUTC);

    const diffInMS = dateInTZ.getTime() - dateInUTC.getTime();

    const sign = diffInMS >= 0 ? "+" : "-";
    const diffInABS = Math.abs(diffInMS);

    const totalHours = diffInABS / (1000 * 60 * 60);
    const hours = Math.floor(totalHours);
    const mins = Math.round((totalHours - hours) * 60);

    const hoursInFormat = String(hours).padStart(2, "0");
    const minsInFormat = String(mins).padStart(2, "0");

    return `${sign}${hoursInFormat}:${minsInFormat}`;
}
```

이 스크립트는 `timestamp`와 `timezone` 값을 받아서, 해당 `timestamp`를 `timezone`에 맞게 변환한 후, UTC 시간과 비교하여 타임존 오프셋 값을 계산한다. 이제 이 스크립트를 Excel 파일에 저장하고, Power Automate 워크플로우에서 호출하는 방법을 알아보자.

## Power Automate 워크플로우에서 Office Scripts 호출하기

아래와 같이 `Run Script` 액션을 선택한다.

![Run Script 액션 선택][image-01]

이후 아래와 같이 Office Scripts를 저장한 엑셀 파일을 선택하고 스크립트 이름과, `timestamp`, `timezone` 값을 입력한다. 여기서 `timestamp`는 `yyyy-MM-dd` 형식이어야 하며, `timezone`은 IANA 형식이어야 한다.

![Run Script 액션 파라미터 입력][image-02]

그 다음 액션에서는 아래 그림과 같이 몇가지 변환 함수를 활용한다.

![날짜 형식 변환][image-03]

1. `convertFromUtc()` 함수와 `utcNow()` 함수를 이용해 현재 UTC 시각을 로컬 시각으로 변환한다.
1. 변환된 로컬 시각은 타임존 오프셋 정보가 없으므로 형식을 `yyyy-MM-ddTHH:mm:ss.fff`까지만 맞춘다.
1. 이전 액션에서 가져온 타임존 오프셋 값을 `concat()` 함수를 이용해 붙여준다.

이렇게 하고 워크플로우를 실행하면, 아래와 같이 제대로 된 타임존 오프셋 값이 문자열로 붙어있는 것을 볼 수 있다.

![타임존 오프셋 값 호출 결과][image-04]

---

지금까지 Power Automate 워크플로우에서 Office Scripts를 활용해 타임존 오프셋 값을 추출하는 방법에 대해 알아보았다. 이를 활용하면, Power Automate 워크플로우에서 타임존 오프셋 값을 활용할 수 있게 된다. Power Automate 워크플로우에서 타임존 오프셋 값을 제공하는 함수가 생길 때 까지는 이런 방법을 활용하는 것이 현재까지는 최선일 것이다.

[image-01]: /2025/01/getting-datetimeoffset-in-power-automate-01.png
[image-02]: /2025/01/getting-datetimeoffset-in-power-automate-02.png
[image-03]: /2025/01/getting-datetimeoffset-in-power-automate-03.png
[image-04]: /2025/01/getting-datetimeoffset-in-power-automate-04.png

[pau]: https://www.microsoft.com/ko-kr/power-platform/products/power-automate
[pau functions]: https://learn.microsoft.com/ko-kr/azure/logic-apps/workflow-definition-language-functions-reference#date-and-time-functions

[m365 excel]: https://www.microsoft.com/ko-kr/microsoft-365/excel
[m365 excel office scripts]: https://learn.microsoft.com/ko-kr/office/dev/scripts/overview/excel

[iso8601]: https://ko.wikipedia.org/wiki/ISO_8601
