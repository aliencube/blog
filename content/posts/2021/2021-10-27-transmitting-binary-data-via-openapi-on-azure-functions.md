---
title: "애저 펑션 OpenAPI 확장 기능을 이용해 바이너리 데이터 전송하기"
slug: transmitting-binary-data-via-openapi-on-azure-functions
description: "이 포스트에서는 애저 펑션의 OpenAPI 확장 기능을 이용해 바이너리 데이터를 전송하는 방법에 대해 알아보겠습니다."
date: "2021-10-27"
author: Justin-Yoo
tags:
- azure-functions
- openapi
- binary-data
- byte-array
cover: /2021/10/transmitting-binary-data-via-openapi-on-azure-functions-00.png
fullscreen: true
---

[애저 펑션][az fncapp]의 [OpenAPI 확장기능][az fncapp openapi]은 [0.9.0-preview][az fncapp openapi nuget] 버전부터 바이트 배열 타입을 지원한다. 이를 이용하면 이미지 파일과 같은 바이너리 데이터를 OpenAPI 문서에 정의할 수 있다. 이 포스트에서는 OpenAPI 확장 기능으로 바이너리 데이터를 정의하고 전송하는 방법에 대해 알아보기로 하자.

> 이 포스트에 사용된 코드는 [이 깃헙 리포지토리][gh sample]에서 다운로드 받을 수 있다.


## `text/plain` 형태로 바이너리 데이터 전송 ##

우선 아래 코드를 보자. 이미지 데이터를 요청 페이로드에 직접 base64 인코딩한 문자열로 보내는 방법이다. 이는 이미 API를 호출하기 전에 바이너리 데이터가 base64 문자열로 인코딩되어 있다고 가정하는 것에서 출발한다. 아래 코드를 보면 요청 페이로드에 컨텐츠 형식을 `text/plain`으로 설정하고 데이터 형식을 `byte[]`로 지정했다 *(line #7)*. 그리고 응답 페이로드를 위해서는 컨텐츠 형식을  `image/png`로 설정하고 데이터 형식을 `byte[]`로 지정했다 *(line #9)*. 실제 데이터는 base64 형식으로 인코딩된 문자열이므로 내부적으로 바이트 배열 형태로 변환한 후 전송하면 된다 *(line #15-21)*.

https://gist.github.com/justinyoo/9d875913235af8b7636810e888fb0013?file=01-run-byte-array.cs&highlights=7,9,15-21

위의 구성을 불필요한 부분은 생략하고 OpenAPI 문서로 렌더링해 보면 아래와 같다. 요청 페이로드 형식이 `text/plain` 컨텐츠로서 base64 형식으로 인코딩된 문자열이라는 것이 명시되어 있다 *(line #7-10)*. 그리고, 응답 페이로드 형식 역시도 `image/png` 형식으로서 base64 형식으로 인코딩된 문자열이라고 명시되어 있다 *(line #16-19)*.

https://gist.github.com/justinyoo/9d875913235af8b7636810e888fb0013?file=02-run-byte-array.yaml&highlights=7-10,16-19

이를 Swagger UI 페이지를 통해 실행시켜 보면 아래와 같다. 이미지 데이터가 제대로 전송이 되었다.

![Byte Array][image-01]


## `multipart/form-data` 형태로 바이너리 데이터 및 텍스트 데이터 전송 ##

이번에는 `multipart/form-data` 형태로 바이너리 데이터와 텍스트 데이터를 한번에 전송해 보자. 바이너리 데이터를 직접 API로 전송할 때 가장 일반적으로 사용하는 방식이 될 것이다. 아래 요청 페이로드를 보면 컨텐츠 형식을 `multipart/form-data`로 지정하고, 데이터 형식은 `MultiPartFormDataModel`로 지정했다 *(line #7)*. 실제 데이터 구조체를 보면 이미지 데이터가 바이트 배열 형태로 들어가 있는 것이 보인다 *(line #34)*. 이 때 이 바이너리 데이터는 `req.Form.Files[0]`을 통해 참조하고 바이트배열로 풀어낸다 *(line #15-23)*.

https://gist.github.com/justinyoo/9d875913235af8b7636810e888fb0013?file=03-run-multi-part-form-data.cs&highlights=7,15-23,35

위 구성을 불필요한 부분을 생략하고 OpenAPI 문서로 렌더링해 보면 아래와 같다. 요청 페이로드의 참조 형식에 보면 *(line #9)*, `image` 데이터가 base64 형식으로 인코딩된 문자열로 지정이 되었음을 알 수 있다 *(line #31-33)*.

https://gist.github.com/justinyoo/9d875913235af8b7636810e888fb0013?file=04-run-multi-part-form-data.yaml&highlights=9,31-33

이를 실제로 Swagger UI 페이지를 통해 실행시켜 보자. 이미지 데이터가 제대로 전송이 된 것을 확인할 수 있다.

![Multi-Part Form-Data][image-02]

---

지금까지 애저 펑션의 OpenAPI 확장 기능을 이용해 바이너리 데이터 형식을 정의하고, 이를 Swagger UI 페이지를 이용해 직접 전송해 보는 방법에 대해 알아보았다. 가장 기다려왔던 기능이 이번 릴리즈로 해결된 만큼 많은 곳에서 활용할 수 있기를 바란다.


[image-01]: /2021/10/transmitting-binary-data-via-openapi-on-azure-functions-01.png
[image-02]: /2021/10/transmitting-binary-data-via-openapi-on-azure-functions-02.png


[gh sample]: https://github.com/devkimchi/azure-functions-binary-data-via-swagger-ui

[az fncapp]: https://docs.microsoft.com/ko-kr/azure/azure-functions/functions-overview?WT.mc_id=dotnet-47576-juyoo
[az fncapp openapi]: https://aka.ms/azfunc-openapi
[az fncapp openapi nuget]: https://www.nuget.org/packages/Microsoft.Azure.WebJobs.Extensions.OpenApi
