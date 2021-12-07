---
title: "애저 펑션 OpenAPI 확장기능을 .NET 6로 버전업하기"
slug: migrating-azfunc-openapi-extension-to-v4
description: "이 포스트에서는 애저 펑션의 OpenAPI 확장기능을 .NET Core 3.1 혹은 .NET 5 에서 .NET 6로 코드 변경 없이 이전하는 방법에 대해 알아봅니다."
date: "2021-12-08"
author: Justin-Yoo
tags:
- azure
- azure-functions
- openapi-extension
- migration
cover: https://sa0blogs.blob.core.windows.net/aliencube/2021/12/migrating-azfunc-openapi-extension-to-v4-00.png
fullscreen: true
---

지난 11월 초에 [애저 펑션][az fncapp]의 [.NET 6 지원 및 런타임 V4 업데이트가 정식으로 GA][az fncapp ga]됐다. 동시에 [애저 펑션의 OpenAPI 확장 기능 패키지 역시 정식으로 GA][az fncapp ga openapi]됐다. 이 확장 기능 패키지는 .NET Core 2.1 LTS 부터 3.1 LTS, .NET 5 및 .NET 6 모든 버전을 지원하는데, 기존에 사용하던 V3 런타임 버전의 애저 펑션을 V4 런타임 버전으로 업그레이드 하기 위해서는 어떤 코드베이스 변경이 필요한지 알아보기로 한다.


## OpenAPI 확장 기능 업데이트 ##

새롭게 GA가 된 것으로 인해 패키지 버전이 `-preview` 딱지를 떼고 `1.0.0`이 되었다. 따라서, 기존에 작동하던 애저 펑션 런타임을 그대로 유지하고 싶다면 패키지 버전만 아래와 같이 변경하면 된다 (line #5). 아래는 .NET Core 2.1 혹은 3.1 버전을 사용하고 있을 경우이다.

https://gist.github.com/justinyoo/8fbd8f285c55a351be2cbf08d8119ce6?file=01-fncapp-31-package-reference.xml&highlights=5

만약 .NET 5의 out-of-process 워커 기능을 사용하고 있었다면 아래와 같이 버전만 바꿔주면 된다 (line #5).

https://gist.github.com/justinyoo/8fbd8f285c55a351be2cbf08d8119ce6?file=02-fncapp-50-package-reference.xml&highlights=5

위와 같이 패키지 버전만 업그레이드 해 주면 기존의 애저 펑션을 코드 변경 없이 그대로 사용할 수 있다.

그런데, 만약 애저 펑션 런타임 자체를 V3에서 V4로 이전하려면 이건 얘기가 달라진다. 이 때는 다음 섹션에서 설명하는 바와 같이 `.csproj` 파일을 수정해야 한다. 더불어 로컬 개발 환경에서 애저 펑션을 개발하려면 [Azure Functions Core Tools][az fncapp core tools] 버전을 V4로 업그레이드 해야 한다.


## .NET Core 3.1 에서 .NET 6 으로 업데이트하기 ##

`.csproj` 파일을 열고 `TargetFramework` 엘리먼트를 찾아서 `netcoreapp3.1`에서 `net6.0`으로 바꾼다. 그리고, `AzureFunctionsVersion` 엘리먼트를 찾아서 `v3`에서 `v4`로 값을 바꿔준다 (*line #4-5, 8-9*).

https://gist.github.com/justinyoo/8fbd8f285c55a351be2cbf08d8119ce6?file=03-fncapp-31-target-framework.xml&highlights=4-5,8-9

이렇게 한 후 새로 솔루션 전체를 빌드한 후 실행시켜 보면 문제 없이 작동하는 것을 확인할 수 있다.

> **NOTE**: 만약 .NET Core 2.1 버전을 사용하고 있었다면 `TargetFramework` 엘리먼트의 값을 `netcoreapp2.1`에서 `net6.0`으로 수정하면 된다.


## .NET 5에서 .NET 6 으로 업데이트하기 ##

마찬가지로 `.csproj` 파일을 열고 `TargetFramework` 엘리먼트를 찾아서 `net5.0`에서 `net6.0`으로 바꾼다. 그리고, `AzureFunctionsVersion` 엘리먼트를 찾아서 `v3`에서 `v4`로 값을 바꿔준다 (*line #4-5, 8-9*).

https://gist.github.com/justinyoo/8fbd8f285c55a351be2cbf08d8119ce6?file=04-fncapp-50-target-framework.xml&highlights=4-5,8-9

이렇게 한 후 새로 솔루션 전체를 빌드한 후 실행시켜 보면 문제 없이 작동하는 것을 확인할 수 있다.

---

이제 모든 애저 펑션 앱이 코드 변경 없이도 .NET 6에서 문제 없이 작동하는 것을 확인할 수 있다. 만약 다른 참조 라이브러리가 .NET Core 2.1/3.1 또는 .NET 5에 의존성을 갖고 있지 않은 이상 이와 같은 방법을 이용하면 가장 최신의 애저 펑션 런타임과 OpenAPI 확장 기능을 활용할 수 있을 것이다.


[az fncapp]: https://docs.microsoft.com/ko-kr/azure/azure-functions/functions-overview?WT.mc_id=dotnet-44909-juyoo
[az fncapp core tools]: https://docs.microsoft.com/ko-kr/azure/azure-functions/functions-run-local?tabs=v4%2Cwindows%2Ccsharp%2Cportal%2Cbash%2Ckeda&WT.mc_id=dotnet-44909-juyoo
[az fncapp ga]: https://techcommunity.microsoft.com/t5/apps-on-azure-blog/azure-functions-4-0-and-net-6-support-are-now-generally/ba-p/2933245?WT.mc_id=dotnet-44909-juyoo
[az fncapp ga openapi]: https://techcommunity.microsoft.com/t5/apps-on-azure-blog/general-availability-of-azure-functions-openapi-extension/ba-p/2931231?WT.mc_id=dotnet-44909-juyoo
