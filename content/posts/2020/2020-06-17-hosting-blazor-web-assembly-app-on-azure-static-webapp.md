---
title: "애저 정적 웹 앱에 블레이저 웹 어셈블리 앱 호스팅하기"
slug: hosting-blazor-web-assembly-app-on-azure-static-webapp
description: "이 포스트에서는 애저 정적 웹 앱 인스턴스에 블레이저 웹 어셈블리 앱을 호스팅할 때 알아두어야 할 것들과 더불어 백엔드 API와 통신하기 위한 애저 펑션 프록시에 대해 알아봅니다."
date: "2020-06-17"
author: Justin-Yoo
tags:
- blazor
- azure-static-web-app
- github-actions
- azure-functions-proxy
cover: /2020/06/hosting-blazor-web-assembly-app-on-azure-static-webapp-00.png
fullscreen: true
---

* [Blazor 웹 애플리케이션에 React UI 컴포넌트 끼얹기][post series 1]
* [Blazor 웹 애플리케이션에 node.js를 이용해 React UI 컴포넌트 끼얹기][post series 2]
* ***애저 정적 웹 앱에 블레이저 웹 어셈블리 앱 호스팅하기***

[지난 포스트][post prev]에서는 [Blazor 웹 어셈블리][blazor wasm] 앱을 로컬에서 개발해 봤다면, 이번 포스트에서는 이렇게 개발한 앱을 실제로 배포해 보는 작업을 진행해 보도록 한다. 이 포스트에서 배포에 사용할 앱은 [애저 정적 웹 앱][az swa]이다.

> 이 포스트에 쓰인 샘플 코드는 이곳 [https://github.com/devkimchi/Blazor-React-Sample][gh sample]에서 다운로드 받을 수 있다.


## 기본 Blazor 웹 어셈블리 앱 만들기 ##

[Blazor 웹 어셈블리][blazor wasm] 앱을 만드는 방법은 [지난 포스트][post prev]를 참고하도록 한다. 이 포스트에서는 이렇게 만들어진 앱을 배포하는 데 중점을 둔다. 실제 앱 소스는 [`BlazorNpmSample` 프로젝트][gh sample blazor]에 있다. 이를 빌드하고 로컬에서 실행시켜 보자.

https://gist.github.com/justinyoo/fcba3e387d240a057e76a28f233fec82?file=01-dotnet-run.sh

그리고 브라우저에서 `Counter` 페이지로 이동해서 `Click me` 버튼을 눌러보자. 그러면 아래와 같은 화면이 나타날 것이다. 여기서 에러가 생길텐데, 예상한 에러기 때문에 괜찮다. 이 에러는 이 포스트의 마지막 부분에 가면 자연스럽게 해결된다.

![][image-01]


## 애저 정적 웹 앱에 Blazor 웹 어셈블리 앱 배포하기 ##

[애저 정적 웹 앱][az swa]은 지난 [빌드 2020][build 2020] 행사에서 최초로 공개된 애저의 새로운 서비스이다. 현재 공개 프리뷰 상태로 제공되고 있는 중이다. 이 [애저 정적 웹 앱][az swa]이 특히 의미가 있는 이유는, 요즘과 같이 프론트엔드 애플리케이션이 사용자와 상호작용을 하는 데 있어서 커다란 역할을 차지하는 만큼, 백엔드 쪽에서는 API 정도로만 데이터를 주고 받는 정도로만 작동한다. 특히 [JAM 스택][jamstack]으로 대변되는 애플리케이션 구조가 지난 몇 년간 두드러진 현상이어서 이러한 시장의 요구에 대응하고자 [애저 정적 웹 앱][az swa] 서비스가 출시되었다. 다만, 현재는 공개 프리뷰 상태여서 JavaScript 기반의 애플리케이션을 우선 지원하기 때문에, [Blazor 웹 어셈블리][blazor wasm] 앱을 이 [애저 정적 웹 앱][az swa] 인스턴스에 배포하기 위해서는 몇 가지 추가적인 절차가 필요하다. 이제부터 하나씩 살펴보기로 한다.

> [Blazor 웹 어셈블리][blazor wasm] 앱을 [애저 정적 웹 앱][az swa] 인스턴스에 배포하기 위한 기본 설정은 [Tim Heuer][tim twitter]의 [포스트][tim post]를 참조했다.

위 리포지토리를 이용해서 [애저 정적 웹 앱][az swa] 인스턴스를 배포하는 방법은 [이 페이지][az swa build]를 참조한다. 그런데, 이 포스트를 쓰는 현 시점에서는 아래와 같이 배포에 실패한다.

![][image-02]

이는 배포에 쓰이는 액션인 [Azure/static-web-apps-deploy@v0.0.1-preview][az swa action]가 [Oryx][oryx] 라는 통합 빌드 도구를 사용하기 때문이다. 현재 [Oryx][oryx]가 지원하는 최신의 .NET Core SDK 버전이 아직 2.2 까지라서 Blazor 앱을 지원하지 않기 때문에 빌드에 실패하는 것이다.

따라서, 이를 해결하기 위해서는 자동으로 생성된 깃헙 액션 워크플로우를 수정해야 한다. 먼저 아래 액션을 `actions/checkout@v2` 바로 다음에 추가한다. 액션에서 볼 수 있다시피 Blazor 앱을 빌드하기 위해서는 .NET Core SDK 버전을 `3.1.300` 이상으로 맞춰줘야 하기 때문에 이 포스트를 쓰는 현재의 최신 버전인 `3.1.300`으로 지정했다 (line #4).

https://gist.github.com/justinyoo/fcba3e387d240a057e76a28f233fec82?file=02-action-dotnet.yaml&highlights=4

위와 같이 .NET Core SDK 버전을 지정한 후, Oryx로 빌드하는 대신 별도로 Blazor 앱을 아래와 같이 빌드해서 아티팩트를 생성한다. 필요하다면 이 전에 빌드와 테스트를 별도로 진행시킬 수도 있지만, 여기서는 생략한다. 생성된 아티팩트는 `published` 라는 디렉토리에 저장된다 (line #4).

https://gist.github.com/justinyoo/fcba3e387d240a057e76a28f233fec82?file=03-action-dotnet-publish.yaml&highlights=4

이제 [Blazor 웹 어셈블리][blazor wasm] 앱이 준비가 됐다. 이제 기존의 `Build And Deploy` 스텝을 수정한다. 아래와 같이 `app_location`, `api_location`, `app_artifact_location` 값을 바꿔준다 (line #6-8).

https://gist.github.com/justinyoo/fcba3e387d240a057e76a28f233fec82?file=04-action-build-deploy.yaml&highlights=6-8

여기까지 한 후 깃헙 액션을 저장하고 리포지토리에 푸시하면, 이번에는 깃헙 액션이 성공적으로 수행될 것이다. 그리고, 앱도 성공적으로 배포된 것을 확인할 수 있다. 하지만 에러는 여전히 발생하는데, 괜찮다.

![][image-03]


## 애저 정적 웹 앱에 프록시 API 배포하기 ##

[애저 정적 웹 앱][az swa]을 포함한 모든 정적 웹사이트 호스팅 서비스의 가장 큰 도전 과제는 바로 프론트엔드 웹사이트가 백엔드 API와 문제없이 소통할 수 있게 해 주는 것이다. 일반적으로 백엔드 API와 통신하기 위해서는 여러 방법이 있지만 OAuth를 사용해서 인증키를 받아오기도 하고 아니면 API 인증키를 이용해서 직접 API를 호출하기도 한다. 이 때 후자의 경우에는 이 인증키를 어딘가에 저장시켜 놓고 호출할 때 마다 가져다가 써야 하는데, 정적 웹사이트의 경우에는 보안 때문에 이를 저장할 곳이 마땅치 않다. 이 때 사용할 수 있는 것이 바로 [API 프록시 기능][az swa api]이다. 이 프록시 기능을 이용해 프론트엔드 앱이 프록시로 API를 호출하면, 이 프록시 API가 인증키를 붙여서 실제 API를 호출하는 방식이다.


### 외부 API 만들기 ###

우선, 외부 API를 하나 만들어 보자. 이 API는 Blazor 앱과는 아무 상관 없이 독립적으로 동작하는 API이다. 편의를 위해 .NET Core를 이용한 [애저 펑션][az func]으로 작성했다. 이 펑션은 `count` 라는 파라미터를 받아 결과값을 반환하는 펑션이다. 앞서 링크한 소스코드 리포지토리에서 [`BlazorApiSample` 프로젝트가][gh sample api] 바로 이 외부 API를 가리킨다. 이 펑션 앱을 아래 명령어를 통해 바로 실행시켜 본다.

https://gist.github.com/justinyoo/fcba3e387d240a057e76a28f233fec82?file=05-api-func-start.sh

이렇게 한 후 아래와 같이 웹 브라우저에서 API 호출을 해 보자.

https://gist.github.com/justinyoo/fcba3e387d240a057e76a28f233fec82?file=06-api-run.txt

그러면 아래와 같은 결과를 볼 수 있다.

![][image-04]


### 프록시 API 만들기 ###

이번에는 [애저 정적 웹 앱][az swa]에서 사용할 프록시 API를 만들어 볼 차례이다. 아직까지는 프리뷰라서 프록시 API를 작성할 때 아래와 같은 제약사항을 염두에 두고 만들어야 한다.

* JavaScript 펑션만 지원한다.
* HTTP 바인딩만 지원한다.

> 이 외에도 제약 사항이 좀 더 있는데, 자세한 사항은 [제약 조건][az swa api constraints]을 참조한다.

따라서, 이 제약 사항을 바탕으로 프록시 API를 만들어야 한다. 이렇게 만들어 놓은 샘플 코드는 [`BlazorProxySample` 프로젝트][gh sample proxy]를 참조한다. 아주 간단한 프록시 API이다. 아래 코드를 보면 이 프록시 API가 어떻게 작동하는지 알 수 있다. 환경 변수에 저장된 `API__BASE_URI`, `API__ENDPOINT`, `API__AUTH_KEY` 값을 통해 외부 API와 연결한다 (line #4-6). 당연하게도 이 환경 변수 값은 외부로 노출되지 않는다. 그리고, 프록시 API가 하는 일은 실제 API로 요청을 재구성해서 날려주는 것에 불과하다 (line #9-11).

https://gist.github.com/justinyoo/fcba3e387d240a057e76a28f233fec82?file=07-proxy-http-trigger.js&highlights=4-6,9,11

그리고, 위 코드에서 보이는 환경 변수 값을 아래 `local.settings.json` 파일에 저장한다 (line #5-7).

https://gist.github.com/justinyoo/fcba3e387d240a057e76a28f233fec82?file=08-proxy-local-settings.json&highlights=5-7

기본적인 설정은 끝났으니 아래 명령어를 통해 프록시 API를 실행시킨다. 이 때 프록시 API는 포트 번호를 `7071`이 아닌 다른 무언가로 바꿔서 실행시켜야 하는데, 앞서 외부 API를 로컬에서 실행시킬 때 이미 `7071`번 포트를 선점했기 때문이다. 여기서는 `7072`로 지정했다.

https://gist.github.com/justinyoo/fcba3e387d240a057e76a28f233fec82?file=09-proxy-func-start.sh

이렇게 실행을 시키면 아래 그림과 같이 두 개의 애저 펑션 인스턴스가 로컬에서 돌아가는 것을 확인할 수 있다.

![][image-05]

실제로 프록시 API로 요청을 날려보면 아래와 같은 결과가 나온다.

![][image-06]


### Blazor 앱과 프록시 API 연결하기 ###

여기서 한가지 더 살펴봐야 하는 부분이 있다. [애저 정적 웹 앱][az swa]에서 프론트엔드 애플리케이션과 이 프록시 API는 기본적으로 동일한 인스턴스이다. 따라서 실재 인스턴스 상에서 실행을 하는 데에는 큰 문제가 없으나, 로컬에서는 이 두 앱이 서로 다른 인스턴스가 된다. Blazor 앱은 `https://localhost:5001`에서, 프록시 API는 `http://localhost:7072` 에서 각각 돌아가기 때문에 API 통신이 불가능하다. 따라서, 프록시 API에 CORS 설정을 해 주어야 한다. 아래와 같이 `local.settings.json` 파일을 수정해서 CORS 설정을 추가한다 (line #6-8).

https://gist.github.com/justinyoo/fcba3e387d240a057e76a28f233fec82?file=10-proxy-local-settings-cors.json&highlights=6-8

이렇게 한 후 애저 펑션 앱, 프록시 API, Blazor 앱을 하나하나 차례로 실행시켜 보면 아래와 같다. 아래 그림에서는 오른쪽부터 애저 펑션 앱, 프록시 API, Blazor 앱이다.

![][image-07]

그리고, Blazor 앱의 `Counter` 페이지에서 `Click me` 버튼을 눌러보자. 그러면 아래와 같은 결과를 확인할 수 있다. 이제 아무런 에러 없이 앱이 잘 돌아간다!

![][image-08]


### Blazor 앱과 프록시 API 함께 배포하기 ###

이제 로컬에서의 모든 개발은 끝났고, 이를 한번에 배포해 보자. 앞서 수정했던 깃헙 액션을 다시 한 번 수정해야 한다. 먼저 아래 액션을 `actions/setup-dotnet@v1` 액션 바로 다음에 추가한다. 이는 Blazor 앱이 참조하는 `appsettings.json` 파일을 프록시 API를 참조하게끔 바꿔주는 역할을 한다 (line #4).

https://gist.github.com/justinyoo/fcba3e387d240a057e76a28f233fec82?file=11-action-appsettings.yaml&highlights=4

그리고 아래와 같이 `Azure/static-web-apps-deploy@v0.0.1-preview` 액션에서 프록시 API를 함께 배포한다고 선언해 줘야 한다 (line #6).

https://gist.github.com/justinyoo/fcba3e387d240a057e76a28f233fec82?file=12-action-build-deploy.yaml&highlights=6

이렇게 깃헙 액션까지 수정한 후 원격 리포지토리로 푸시한다. 그러면 자동으로 깃헙 액션이 돌면서 모든 애플리케이션은 [애저 정적 웹 앱][az swa]으로 배포하게 된다. 배포가 끝난 후 앱을 실행시켜 보자. 여전히 에러가 발생한다.

![][image-09]

이는 아직 프록시 API에 환경 설정을 해주지 않아서 생긴 일이므로 아래와 같이 `Configuration` 블레이드에서 환경 변수를 설정해준다.

![][image-10]

그리고, 다시 한 번 앱을 리프레시한 후에 실행시켜 보자. 이제는 에러 없이 모든 것이 제대로 작동한다!

![][image-11]

---

지금까지 [Blazor 웹 어셈블리][blazor wasm] 앱을 [애저 정적 웹 앱][az swa] 인스턴스에 배포하는 방법에 대해 알아 보았다. Blazor 앱만 배포하는 과정은 깃헙 액션 워크플로우를 살짝만 수정하면 되므로 그닥 복잡하지는 않은데, 프록시 API까지 함께 배포하려면 좀 더 손이 많이 간다. 아직 프리뷰 상태이고 여전히 개선 중이니, 정식 출시가 되는 시점에서는 Blazor 앱 역시도 작동할 수 있기를 기대한다.


[image-01]: /2020/06/hosting-blazor-web-assembly-app-on-azure-static-webapp-01.png
[image-02]: /2020/06/hosting-blazor-web-assembly-app-on-azure-static-webapp-02.png
[image-03]: /2020/06/hosting-blazor-web-assembly-app-on-azure-static-webapp-03.png
[image-04]: /2020/06/hosting-blazor-web-assembly-app-on-azure-static-webapp-04.png
[image-05]: /2020/06/hosting-blazor-web-assembly-app-on-azure-static-webapp-05.png
[image-06]: /2020/06/hosting-blazor-web-assembly-app-on-azure-static-webapp-06.png
[image-07]: /2020/06/hosting-blazor-web-assembly-app-on-azure-static-webapp-07.png
[image-08]: /2020/06/hosting-blazor-web-assembly-app-on-azure-static-webapp-08.png
[image-09]: /2020/06/hosting-blazor-web-assembly-app-on-azure-static-webapp-09.png
[image-10]: /2020/06/hosting-blazor-web-assembly-app-on-azure-static-webapp-10.png
[image-11]: /2020/06/hosting-blazor-web-assembly-app-on-azure-static-webapp-11.png

[gh sample]: https://github.com/devkimchi/Blazor-React-Sample
[gh sample blazor]: https://github.com/devkimchi/Blazor-React-Sample/tree/master/BlazorNpmSample
[gh sample api]: https://github.com/devkimchi/Blazor-React-Sample/tree/master/BlazorApiSample
[gh sample proxy]: https://github.com/devkimchi/Blazor-React-Sample/tree/master/BlazorProxySample

[post series 1]: /ko/2020/06/03/adding-react-components-to-blazor-webassembly-app/
[post series 2]: /ko/2020/06/10/adding-react-components-to-blazor-webassembly-app-by-nodejs/
[post series 3]: /ko/2020/06/17/hosting-blazor-web-assembly-app-on-azure-static-webapp/

[post prev]: /ko/2020/06/10/adding-react-components-to-blazor-webassembly-app-by-nodejs/

[blazor wasm]: https://docs.microsoft.com/ko-kr/aspnet/core/blazor/?view=aspnetcore-3.1&WT.mc_id=aliencubeorg-blog-juyoo#blazor-webassembly

[az swa]: https://docs.microsoft.com/ko-kr/azure/static-web-apps/overview?WT.mc_id=aliencubeorg-blog-juyoo
[az swa build]: https://docs.microsoft.com/ko-kr/azure/static-web-apps/getting-started?tabs=vanilla-javascript&WT.mc_id=aliencubeorg-blog-juyoo
[az swa action]: https://github.com/Azure/static-web-apps-deploy
[az swa api]: https://docs.microsoft.com/ko-kr/azure/static-web-apps/apis?WT.mc_id=aliencubeorg-blog-juyoo
[az swa api constraints]: https://docs.microsoft.com/ko-kr/azure/static-web-apps/apis?WT.mc_id=aliencubeorg-blog-juyoo#constraints

[az func]: https://docs.microsoft.com/ko-kr/azure/azure-functions/functions-overview?WT.mc_id=aliencubeorg-blog-juyoo

[build 2020]: https://mybuild.microsoft.com/?WT.mc_id=aliencubeorg-blog-juyoo
[jamstack]: https://jamstack.org/
[oryx]: https://github.com/microsoft/Oryx

[tim twitter]: https://twitter.com/timheuer
[tim post]: https://timheuer.com/blog/hosting-blazor-in-azure-static-web-apps/
