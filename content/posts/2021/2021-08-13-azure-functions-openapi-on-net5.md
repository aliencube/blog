---
title: ".NET 5 환경에서 애저 펑션 OpenAPI 확장 기능"
slug: azure-functions-openapi-on-net5
description: "이 포스트에서는 애저 펑션의 OpenAPI 확장 기능을 .NET 5의 out-of-process 워커 환경에서 구동시키는 방법에 대해 알아봅니다."
date: "2021-08-13"
author: Justin-Yoo
tags:
- azure
- azure-functions
- openapi
- swagger
cover: /2021/08/azure-functions-openapi-on-net5-00.png
fullscreen: true
---

지난 [//Build 2021][build2021] 행사에서 [공식적으로 발표][build2021 openapi]했던 [애저 펑션][az fncapp]의 [OpenAPI 지원 기능(프리뷰) 기능][azfunc openapi]은 당시에는 v3 런타임, 즉 .NET Core 3.1 환경까지만 지원하고 있었다. 이 확장 기능이 [최근에 .NET 5의 런타임 격리 환경에서 동작하는 패키지][azfunc openapi nuget]를 새롭게 프리뷰로 선보였는데, 이 포스트에서는 해당 기능을 한 번 살펴보고자 한다.

> NOTE: 이 포스트에서 쓰인 샘플 코드는 [이곳][gh sample]에서 확인할 수 있다.


## 애저 펑션 런타임 격리 환경 지원 프로젝트 생성 및 설정 ##

이 포스트에서는 [비주얼 스튜디오][vs]를 이용해서 애저 펑션 앱을 생성하기로 한다. 가장 먼저 애저 펑션 프로젝트를 만들 때, 아래와 같이 .NET 5 (Isolated), Http trigger 를 선택한다.

![.NET 5 (Isolated), Http trigger 선택][image-01]

기본 HTTP 트리거 템플릿을 이용한 HTTP 엔드포인트 코드가 보일 것이다. 이제 솔루션 탐색기에서 NuGet 패키지 매니저를 선택한다.

![NuGet 패키지 매니저 선택][image-02]

NuGet 패키지 매니저 화면에서 "Include prerelease" 체크박스를 선택한 후 검색창에 `Microsoft.Azure.Functions.Worker.Extensions.OpenApi`를 입력한 후 검색된 패키지를 설치한다. 이 포스트를 쓰는 현재 최신 버전은 `v0.8.1-preview`이다.

![NuGet 패키지 매니저 화면에서 애저 펑션 OpenAPI 확장 기능 패키지 검색 및 선택][image-03]

OpenAPI 확장 기능 패키지 설치가 끝났다.


## 호스트 빌더 설정 ##

이제 OpenAPI 확장 기능 패키지를 설치했으니, 애저 펑션 프로젝트를 구동하기 위한 호스트 빌더를 설정할 차례이다. `Program.cs` 파일을 열고 아래와 같이 기존에 있던 `ConfigureFunctionsWorkerDefaults()` 메소드를 삭제한다. 이 메소드는 기본적으로 `System.Text.Json` 패키지를 사용하기 때문이다.

https://gist.github.com/justinyoo/32474da3f9e512b6ab4e34192fbbb4b8?file=00-program.cs

그리고 아래와 같이 `ConfigureFunctionsWorkerDefaults(worker => worker.UseNewtonsoftJson())`, `ConfigureOpenApi()` 메소드를 추가한다. 첫번째 메소드는 `Newtonsoft.Json` 패키지를 사용해서 애저 펑션을 구동하겠다는 선언이고, 두번째 메소드를 통해 OpenAPI 관련 추가 엔드포인트를 설정한다.

https://gist.github.com/justinyoo/32474da3f9e512b6ab4e34192fbbb4b8?file=01-program.cs

> **NOTE**: 현재 `System.Text.Json` 패키지는 정상적인 동작을 보장하지 않으므로 `Newtonsoft.Json` 패키지 사용을 명시적으로 선언하는 것을 권장한다.

이제 OpenAPI 확장 기능 패키지 사용을 위한 환경 설정이 끝났다.


## OpenAPI 데코레이터 설정 ##

아래와 같이 OpenAPI 관련 데코레이터를 입력한다. 이 부분은 기존의 방법과 동일하므로 여기서는 따로 언급하지 않기로 한다.

https://gist.github.com/justinyoo/32474da3f9e512b6ab4e34192fbbb4b8?file=03-run.cs

위와 같이 데코레이터 설정이 끝났다면, 이제 실제로 애플리케이션을 실행시켜 볼 차례이다.


## Swagger UI 실행 ##

[비주얼 스튜디오][vs] 화면에서 `F5` 키 혹은 디버그 버튼을 클릭해서 애저 펑션을 실행시킨다.

![애저 펑션 디버깅 모드 실행][image-04]

애저 펑션 실행 화면에 보면 아래와 같이 OpenAPI 관련 엔드포인트가 자동으로 추가된 것이 보인다.

![애저 펑션 OpenAPI 엔드포인트 자동 추가][image-05]

이제 `http://localhost:7071/api/swagger/ui` 엔드포인트를 웹 브라우저를 통해 확인해 보자. 아래와 같이 보일 것이다.

![Swagger UI 화면][image-06]

여기까지 해서 애저 펑션에 OpenAPI 확장 기능을 잘 설정한 것을 확인했다.


## 애저 펑션 배포 &ndash; Windows ##

이제 이렇게 만든 애저 펑션을 배포해 보도록 하자. 기존의 방법과 큰 차이는 없다. 먼저 솔루션 탐색기에서 프로젝트를 선택하고 Publish 메뉴를 선택한다.

![애저 펑션 배포 메뉴][image-07]

아래와 같이 "Azure"를 선택한다. 그리고 "Azure Functions App (Windows)"를 선택한다.

![애저 펑션 배포 방식 설정 (Windows)][image-08]

아래와 같이 기존에 생성해 놓은 애저 펑션 인스턴스를 선택할 수도 있고, 우측의 ➕ 버튼을 클릭해서 새 인스턴스를 만들 수도 있다. 여기서는 기존의 인스턴스를 선택해서 배포하는 것으로 한다.

![기존 윈도우용 애저 펑션 배포 인스턴스 선택][image-09]

배포가 끝난 후에 애저 펑션 인스턴스가 올라가 있는 URL을 웹 브라우저에서 열어보면 아래와 같은 동일한 결과를 확인할 수 있다.

![애저에 배포한 애저 펑션 Swagger UI][image-10]


## 애저 펑션 배포 &ndash; Linux ##

이번에는 동일한 앱을 애저 펑션 리눅스 인스턴스로 배포해 보자. 더불어 이번에는 깃헙 액션을 이용해서 배포하기로 한다. 그러기 위해서는 우선 이 애저 펑션 앱을 깃헙 리포지토리에 업로드해야 한다. 아래 그림과 같이 "Git Changes" 화면으로 가서 Git 리포지토리를 생성한다.

![Git Changes 화면][image-11]

이미 [비주얼 스튜디오][vs]를 통해 깃헙에 로그인을 했다면 아래와 같이 깃헙 리포지토리를 만들고 바로 업로드할 수 있다. 리포지토리를 생성하고 곧바로 펑션 앱 소스 코드를 업로드한다.

![깃헙 리포지토리 생성][image-12]

업로드가 끝난 후 깃헙으로 이동하면 아래와 같이 새로운 깃헙 리포지토리가 생성되었고, 소스 코드까지 업로드가 된 것이 보인다.

![깃헙 리포지토리 생성 완료][image-13]

다시 배포 메뉴로 돌아가서 아래와 같이 "➕ New" 버튼을 클릭한다.

![새 배포 프로필 생성][image-14]

그러면 앞서와 같은 애저 펑션 배포 메뉴가 나타나는데, 이번에는 "Azure Function App (Linux)"를 선택한다.

![애저 펑션 배포 방식 설정 (Linux)][image-15]

앞서와 마찬가지로 기존에 이미 생성해 놓은 애저 펑션 인스턴스를 선택할 수도 있고, 새롭게 인스턴스를 만들 수도 있다. 여기서는 기존 인스턴스를 선택하기로 한다.

![기존 리눅스용 애저 펑션 배포 인스턴스 선택][image-16]

앞서 윈도우 인스턴스로 배포할 때에는 깃헙 리포지토리 없이 직접 로컬 컴퓨터에서 배포를 했다면, 이번에는 깃헙 리포지토리가 있기 때문에 배포 방식에 대한 선택지가 생겼다. 따라서 기존과 동일한 배포방식을 선택하는 대신, 여기서는 아래와 같이 깃헙 액션을 선택한다.

![깃헙 액션 선택][image-17]

깃헙 액션 워크플로우가 자동으로 생성이 됐지만, 아직 커밋이 되지 않았기 때문에 아래와 같이 커밋 링크를 클릭해서 코드 커밋을 먼저 해야 한다.

![깃헙 액션 커밋후 푸시 링크][image-18]

다시 "Git Changes" 메뉴에서 아래와 같이 커밋 메시지를 작성하고 "Commit All"을 클릭한 후 푸시한다.

![깃헙 액션 커밋후 푸시][image-19]

실제로 깃헙 리포지토리로 이동해 보면 아래와 같이 깃헙 액션이 동작하고 있는 것을 볼 수 있다.

![깃헙 액션으로 배포중][image-20]

배포가 끝난 후 아래와 같이 웹 브라우저를 이용해 리눅스용 애저 펑션 인스턴스로 이동해 보면 제대로 배포가 된 것이 보인다.

![애저에 배포한 애저 펑션 Swagger UI][image-21]

---

지금까지 [비주얼 스튜디오][vs]를 이용해 [애저 펑션][az fncapp]에 [OpenAPI 확장 기능 패키지][azfunc openapi]를 .NET 5의 런타임 격리 환경에서 설치하고 동작시키고 배포하는 방법에 대해 알아 보았다. 이렇게 격리 환경에서 제대로 작동한다면, 다가오는 .NET 6 런타임에서도 이론적으로는 충분히 작동이 가능할 것으로 예상한다. 이 글을 읽는 여러분도 한 번 작동시켜 보자!


[image-01]: /2021/08/azure-functions-openapi-on-net5-01.png
[image-02]: /2021/08/azure-functions-openapi-on-net5-02.png
[image-03]: /2021/08/azure-functions-openapi-on-net5-03.png
[image-04]: /2021/08/azure-functions-openapi-on-net5-04.png
[image-05]: /2021/08/azure-functions-openapi-on-net5-05.png
[image-06]: /2021/08/azure-functions-openapi-on-net5-06.png
[image-07]: /2021/08/azure-functions-openapi-on-net5-07.png
[image-08]: /2021/08/azure-functions-openapi-on-net5-08.png
[image-09]: /2021/08/azure-functions-openapi-on-net5-09.png
[image-10]: /2021/08/azure-functions-openapi-on-net5-10.png
[image-11]: /2021/08/azure-functions-openapi-on-net5-11.png
[image-12]: /2021/08/azure-functions-openapi-on-net5-12.png
[image-13]: /2021/08/azure-functions-openapi-on-net5-13.png
[image-14]: /2021/08/azure-functions-openapi-on-net5-14.png
[image-15]: /2021/08/azure-functions-openapi-on-net5-15.png
[image-16]: /2021/08/azure-functions-openapi-on-net5-16.png
[image-17]: /2021/08/azure-functions-openapi-on-net5-17.png
[image-18]: /2021/08/azure-functions-openapi-on-net5-18.png
[image-19]: /2021/08/azure-functions-openapi-on-net5-19.png
[image-20]: /2021/08/azure-functions-openapi-on-net5-20.png
[image-21]: /2021/08/azure-functions-openapi-on-net5-21.png


[gh sample]: https://github.com/justinyoo/azfunc-openapi-dotnet

[build2021]: https://mybuild.microsoft.com/home?WT.mc_id=dotnet-38365-juyoo
[build2021 openapi]: https://techcommunity.microsoft.com/t5/apps-on-azure/create-and-publish-openapi-enabled-azure-functions-with-visual/ba-p/2381067?WT.mc_id=dotnet-38365-juyoo

[azfunc openapi]: https://github.com/Azure/azure-functions-openapi-extension
[azfunc openapi nuget]: https://github.com/Azure/azure-functions-openapi-extension/releases/tag/v0.8.1-preview

[az fncapp]: https://docs.microsoft.com/ko-kr/azure/azure-functions/functions-overview?WT.mc_id=dotnet-38365-juyoo

[vs]: https://visualstudio.microsoft.com/?WT.mc_id=dotnet-38365-juyoo
