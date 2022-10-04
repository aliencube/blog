---
title: "Blazor 웹 애플리케이션에 React UI 컴포넌트 끼얹기"
slug: adding-react-components-to-blazor-webassembly-app
description: "이 포스트에서는 React 콤포넌트를 Blazor 웹 앱에서 렌더링하는 방법에 대해 알아봅니다."
date: "2020-06-03"
author: Justin-Yoo
tags:
- blazor
- reactjs
- fluent-ui
- js-interop
cover: /2020/06/adding-react-components-to-blazor-webassembly-app-00.png
fullscreen: true
---

* ***Blazor 웹 애플리케이션에 React UI 컴포넌트 끼얹기***
* [Blazor 웹 애플리케이션에 node.js를 이용해 React UI 컴포넌트 끼얹기][post series 2]
* [애저 정적 웹 앱에 블레이저 웹 어셈블리 앱 호스팅하기][post series 3]

지난 [빌드 2020 행사][build]에서 엄청나게 많은 서비스와 기술들이 소개가 되었다. 이 중에서 프론트엔드 애플리케이션 개발에 한 획을 그을만한 내용이 하나 있었는데, 바로 [Blazor 웹 어셈블리][blazor wasm]에 대한 것이다. [웹 어셈블리][wasm]는 쉽게 말해서 웹 브라우저에서 자바스크립트 언어가 아닌 일반 프로그래밍 언어로 만들어진 바이너리를 작동시키는 기술이라고 이해하면 좋다. [Blazor][blazor]는 바로 이 웹 어셈블리를 이용해 닷넷 라이브러리를 웹브라우저에서 직접 실행시킬 수 있게 한다. [Blazor][blazor wasm]에 대한 소개는 [빌드 발표 동영상][build blazor]을 참조하도록 한다.

다른 프론트엔드 프레임워크와 마찬가지로 [Blazor 웹 어셈블리][blazor wasm] 역시 자체 완결성을 갖고 있어서 다른 프론트엔드 프레임워크와 섞어 쓰기에는 까다로울 수 있다. 그럼에도 불구하고 기존의 컴포넌트 기반 프론트엔드 프레임워크라면 가능하기도 한데, 이 포스트에서는 [React][reactjs]를 기반으로 하는 [Fluent UI][fluentui] 컴포넌트를 [Blazor 웹 어셈블리][blazor wasm] 웹 앱에 추가하는 방법에 대해 알아보기로 한다.

> 이 포스트에 쓰인 샘플 코드는 이곳[https://github.com/devkimchi/Blazor-React-Sample][gh sample]에서 다운로드 받을 수 있다.


## 기본 Blazor 웹 어셈블리 앱 만들기 ##

[Blazor 서버][blazor server] 기반의 웹 애플리케이션과 달리 [Blazor 웹 어셈블리][blazor wasm] 앱을 개발하기 위해서는 최신 버전의 [.NET Core 3.1 SDK 3.1.4 버전 이상][netcore sdk 3.1.4]을 설치해야 한다. 자신의 운영체제에 맞는 설치 파일을 다운로드 받아서 설치하도록 하자. 설치가 끝났다면 [Blazor 시작하기][blazor gettingstarted]를 참고해서 간단한 앱을 하나 만들어 보자.

https://gist.github.com/justinyoo/e6a99fffa35d032f70e937c7ccf14ddb?file=01-dotnet-new.sh

> 이 포스트에서는 웹 어셈블리 형태의 앱을 만들 예정이므로 `blazorwasm`을 선택했다. 만약 [Blazor 서버][blazor server] 앱을 만들고자 한다면 `blazorserver`를 선택한다.

이후 `dotnet run` 명령어를 통해 앱을 실행시켜 보면 아래와 같은 화면을 보게 된다. Counter 페이지로 이동해서 `Click me` 버튼을 클릭하면 숫자가 올라가는 것도 확인할 수 있다.

![][image-01]
![][image-02]

여기까지는 [Blazor 시작하기][blazor gettingstarted] 문서에 나와 있는 바와 같다.


## React UI 컴포넌트 추가하기 ##

[Fluent UI][fluentui]는 [Microsoft 365][m365]에 포함된 애플리케이션에 적용된 UI 프레임워크이다. 이 프레임워크에는 현재 [React][reactjs]를 적용했는데, 웹 애플리케이션을 제작할 때 컴포넌트 방식으로 추가하기 쉽게 되어 있다. 이 포스트에서는 [Progress Indicator][fluentui progressindicator] 콘트롤을 이용해서 `Click me` 버튼을 누를 때 마다 콘트롤이 변하는 것을 간단하게 구현해 보자. 해당 콘트롤 문서 페이지에 있는 [CodePen][codepen]을 통해 제공되는 예제 코드를 활용하면 된다.

> 원래 [Hassan Habib][hassan]이 [Blazor 서버][blazor server]를 이용한 예제를 [유튭][hassan video]에 올려놓은 것이 있다. 이를 참고해서 [Blazor 웹 어셈블리][blazor wasm]에 맞게 변형시켰다.

먼저 `index.html` 파일을 열어 아래와 같이 [React][reactjs] 관련 자바스크립트를 CDN으로부터 가져온다.

https://gist.github.com/justinyoo/e6a99fffa35d032f70e937c7ccf14ddb?file=02-add-react-library.html

[Blazor 웹 어셈블리][blazor wasm]는 [C# 코드에서 자바스크립트를 호출][blazor js from dotnet]한다거나, 반대로 [자바스크립트에서 C# 코드를 호출][blazor dotnet from js]할 수 있는데 이는 웹 어셈블리의 자바스크립트 상호운용성을 기반으로 한다. 이제 아래 자바스크립트를 작성한다. Blazor에서 `RenderProgressBar` 라는 펑션으로 `count` 값을 넘겨주면 이 펑션은 `ProgressIndicator` 컴포넌트를 생성하고 `reactProgressBar`라는 영역에 렌더링한다.

> Blazor 에서 호출하는 자바스크립트 펑션은 기본적으로 `window` 아래에 존재하는 글로벌 스코프로 작동한다.

https://gist.github.com/justinyoo/e6a99fffa35d032f70e937c7ccf14ddb?file=03-add-react-component.html

이제 `Counter.razor` 페이지를 수정해 보자. 위에 작성한 자바스크립트 펑션을 호출해야 한다. 자바스크립트 펑션을 닷넷 코드에서 호출하기 위해서는 `IJSRuntime` 인스턴스를 이용해야 하는데, 이는 페이지에서 `@inject` 디렉티브를 통해 주입할 수 있다 (line #2). 그리고 `IncrementCount()` 메소드를 `async`로 바꾸고 (line #15), 그 메소드 안에서 `InvokeVoidAsync()` 메소드를 호출한다. 이 메소드를 통해 위에 작성한 자바스크립트의 `RenderProgressBar` 펑션을 호출하는 것이다 (line #19). 마지막으로 `reactProgressBar` 라는 ID로 컴포넌트가 렌더링될 수 있는 자리를 마련해 놓는다 (line #10).

https://gist.github.com/justinyoo/e6a99fffa35d032f70e937c7ccf14ddb?file=04-update-razor-page.razor&highlights=2,10,15,19

이렇게 한 후 다시 애플리케이션을 실행시켜서 `Click me` 버튼을 클릭해 보자. 그러면 아래와 같이 Progress Indicator가 보인다.

![][image-03]

---

이렇게 해서 [Blazor 웹 어셈블리][blazor wasm] 애플리케이션에 [React][reactjs] 기반의 컴포넌트를 추가하는 방법에 대해 알아보았다. 여기서 고려하지 않은 부분이 몇 가지가 있는데,

* 카운터 값이 바뀔 때 마다 컴포넌트를 새롭게 렌더링한다. 새롭게 렌더링하는 대신, [Blazor][blazor] 애플리케이션이 자체적으로 [상태 관리][blazor statemanagement]가 가능하므로 이를 활용하는 방법이 더 좋을 수도 있다. 다만, 복잡도가 증가한다는 점도 고려해 두도록 하자.
* [React][reactjs] 라이브러리를 CDN에 직접 링크했다. npm 패키지를 이용할 수도 있는데, 이는 [다음 포스트][post next]에서 다뤄보기로 한다.


[image-01]: /2020/06/adding-react-components-to-blazor-webassembly-app-01.png
[image-02]: /2020/06/adding-react-components-to-blazor-webassembly-app-02.png
[image-03]: /2020/06/adding-react-components-to-blazor-webassembly-app-03.png

[post series 1]: /ko/2020/06/03/adding-react-components-to-blazor-webassembly-app/
[post series 2]: /ko/2020/06/10/adding-react-components-to-blazor-webassembly-app-by-nodejs/
[post series 3]: /ko/2020/06/17/hosting-blazor-web-assembly-app-on-azure-static-webapp/

[post next]: /ko/2020/06/10/adding-react-components-to-blazor-webassembly-app-by-nodejs/

[gh sample]: https://github.com/devkimchi/Blazor-React-Sample

[build]: https://mybuild.microsoft.com/?WT.mc_id=aliencubeorg-blog-juyoo
[build blazor]: https://mybuild.microsoft.com/sessions/420ccd3f-6570-4c58-91da-cd760c511171?source=sessions&WT.mc_id=aliencubeorg-blog-juyoo

[blazor]: https://docs.microsoft.com/ko-kr/aspnet/core/blazor/?view=aspnetcore-3.1&WT.mc_id=aliencubeorg-blog-juyoo
[blazor wasm]: https://docs.microsoft.com/ko-kr/aspnet/core/blazor/?view=aspnetcore-3.1&WT.mc_id=aliencubeorg-blog-juyoo#blazor-webassembly
[blazor server]: https://docs.microsoft.com/ko-kr/aspnet/core/blazor/?view=aspnetcore-3.1&WT.mc_id=aliencubeorg-blog-juyoo#blazor-server
[blazor gettingstarted]: https://docs.microsoft.com/ko-kr/aspnet/core/blazor/get-started?view=aspnetcore-3.1&tabs=visual-studio-code&WT.mc_id=aliencubeorg-blog-juyoo
[blazor js from dotnet]: https://docs.microsoft.com/ko-kr/aspnet/core/blazor/call-javascript-from-dotnet?view=aspnetcore-3.1&WT.mc_id=aliencubeorg-blog-juyoo
[blazor dotnet from js]: https://docs.microsoft.com/ko-kr/aspnet/core/blazor/call-dotnet-from-javascript?view=aspnetcore-3.1&WT.mc_id=aliencubeorg-blog-juyoo
[blazor statemanagement]: https://docs.microsoft.com/ko-kr/aspnet/core/blazor/state-management?view=aspnetcore-3.1&WT.mc_id=aliencubeorg-blog-juyoo

[wasm]: https://webassembly.org/
[reactjs]: https://ko.reactjs.org/
[m365]: https://www.office.com/
[netcore sdk 3.1.4]: https://dotnet.microsoft.com/download/dotnet-core/3.1?WT.mc_id=aliencubeorg-blog-juyoo#3.1.4
[codepen]: https://codepen.io/

[fluentui]: https://developer.microsoft.com/fluentui/?WT.mc_id=aliencubeorg-blog-juyoo
[fluentui progressindicator]: https://developer.microsoft.com/fluentui?WT.mc_id=aliencubeorg-blog-juyoo#/controls/web/progressindicator
[fluentui progressindicator codepen]: https://codepen.io/pen/?&editable=true=https%3A%2F%2Fdeveloper.microsoft.com%2Fen-us%2Ffluentui%3FWT.mc_id%3Daliencubeorg-blog-juyoo

[hassan]: https://twitter.com/HassanRezkHabib
[hassan video]: https://youtu.be/E4xUCxOL_PI
