---
title: "Blazor 웹 애플리케이션에 node.js를 이용해 React UI 컴포넌트 끼얹기"
slug: adding-react-components-to-blazor-webassembly-app-by-nodejs
description: "이 포스트에서는 node.js를 이용해 React 콤포넌트를 Blazor 웹 앱에서 렌더링하는 방법에 대해 알아봅니다."
date: "2020-06-10"
author: Justin-Yoo
tags:
- blazor
- reactjs
- fluent-ui
- js-interop
cover: /2020/06/adding-react-components-to-blazor-webassembly-app-by-nodejs-00.png
fullscreen: true
---

* [Blazor 웹 애플리케이션에 React UI 컴포넌트 끼얹기][post series 1]
* ***Blazor 웹 애플리케이션에 node.js를 이용해 React UI 컴포넌트 끼얹기***
* [애저 정적 웹 앱에 블레이저 웹 어셈블리 앱 호스팅하기][post series 3]

[지난 포스트][post prev]에서는 [Blazor 웹 어셈블리][blazor wasm] 앱에 자바스크립트 CDN을 활용해서 [React][reactjs] 기반의 [Fluent UI][fluentui] 컴포넌트를 추가해 봤다. 이런 식으로도 자바스크립트를 많이 활용을 하지만, 요즘 웹 프론트엔드 개발은 주로 [node.js][nodejs]와 [npm 패키지][npmjs]를 활용해서 하는 편이다. 이 포스트에서는 지난 포스트의 개발 경험을 node.js와 npm 패키지로 바꿔보기로 한다.

> 이 포스트에 쓰인 샘플 코드는 이곳[https://github.com/devkimchi/Blazor-React-Sample][gh sample]에서 다운로드 받을 수 있다.


## 기본 Blazor 웹 어셈블리 앱 만들기 ##

[Blazor 서버][blazor server] 기반의 웹 애플리케이션과 달리 [Blazor 웹 어셈블리][blazor wasm] 앱을 개발하기 위해서는 최신 버전의 [.NET Core 3.1 SDK 3.1.4 버전 이상][netcore sdk 3.1.4]을 설치해야 한다. 자신의 운영체제에 맞는 설치 파일을 다운로드 받아서 설치하도록 하자. 설치가 끝났다면 [Blazor 시작하기][blazor gettingstarted]를 참고해서 간단한 앱을 하나 만들어 보자.

https://gist.github.com/justinyoo/224fa5fe1bfa2dca8dcdd0fc83c17251?file=01-dotnet-new.sh

> 이 포스트에서는 웹 어셈블리 형태의 앱을 만들 예정이므로 `blazorwasm`을 선택했다. 만약 [Blazor 서버][blazor server] 앱을 만들고자 한다면 `blazorserver`를 선택한다.

이후 `dotnet run` 명령어를 통해 앱을 실행시켜 보면 아래와 같은 화면을 보게 된다. Counter 페이지로 이동해서 `Click me` 버튼을 클릭하면 숫자가 올라가는 것도 확인할 수 있다.

![][image-01]
![][image-02]

여기까지는 [Blazor 시작하기][blazor gettingstarted] 문서에 나와 있는 바와 같다.


## React UI 컴포넌트 추가하기 ##

> 이 npm 패키지를 작성하는 부분은 [Kedren Villena][kedren]의 [포스트][kedren post]를 참고했다.

[이전 포스트][post prev]에서는 CDN을 통해 자바스크립트 라이브러리를 링크하고, 직접 이를 이용해서 [Fluent UI][fluentui] 컴포넌트를 작성했다면, 이번에는 [node.js][nodejs]와 [npm 패키지][npmjs]를 사용해 보자. 프로젝트의 루트 디렉토리에 `JsLibraries`라는 이름으로 디렉토리를 하나 만들고 그 안에서 아래와 같은 명령어를 실행시킨다.

https://gist.github.com/justinyoo/224fa5fe1bfa2dca8dcdd0fc83c17251?file=02-npm-init.sh

기본적인 npm 패키지 스카폴딩이 된 상태이다. 여기서 아래 명령어를 통해 우리가 개발에 필요한 [React][reactjs] 관련 npm 패키지들을 설치한다.

https://gist.github.com/justinyoo/224fa5fe1bfa2dca8dcdd0fc83c17251?file=03-npm-install-save.sh

그리고, 패키징에 필요하지만 배포에는 굳이 필요하지 않은 npm 패키지들을 아래와 같이 설치한다.

https://gist.github.com/justinyoo/224fa5fe1bfa2dca8dcdd0fc83c17251?file=04-npm-install-save-dev.sh

> **참고** 맥OS 사용자의 경우, 만약 `webpack` 또는 `webpack-cli`를 설치하는 과정에서 `node-gyp` 관련 에러가 난다면, [이 이슈 페이지][node-gyp issue]를 참조해서 문제를 해결한다.

여기서 다시 `src` 디렉토리를 만들고 난 후 그 아래 `index.js` 파일과 `progressbar.js` 파일을 생성한다. `progressbar.js` 파일에는 실제 로직이 들어가고, `index.js` 파일은 이 npm 패키지를 빌드해서 [Blazor 웹 어셈블리][blazor wasm]앱에서 사용하기 위한 일종의 루트 디렉토리라고 생각하면 좋다. 그러면 `progressbar.js` 파일에 아래와 같이 로직을 입력한다.

https://gist.github.com/justinyoo/224fa5fe1bfa2dca8dcdd0fc83c17251?file=05-progressbar.js&highlights=1-3,5

아래는 [이전 포스트][post prev]의 내용이다. 위 코드와 아래 코드를 비교해 보면 어느 부분이 달라졌는지 알 수 있을 것이다. 위 코드에서는 먼저 라이브러리들을 임포트했고 (line #1-3), `renderProgressBar`라는 펑션을 익스포트했다 (line #5). 또한 `window` 객체로 편입시키지 않았는데, 이는 아래에서 별도로 언급하기로 한다.

https://gist.github.com/justinyoo/e6a99fffa35d032f70e937c7ccf14ddb?file=03-add-react-component.html

이렇게 한 후 `index.js` 파일을 통해 이 펑션을 사용할 수 있게끔 한 번 더 감싸준다 (line #3). 이렇게 굳이 한 번 더 감싸줄 필요가 있나 싶기도 하지만, 개인적으로는 이 `index.js`에서 선언하는 부분이 마치 IoC 컨테이너 같다는 느낌이어서 선호하는 편이다.

https://gist.github.com/justinyoo/224fa5fe1bfa2dca8dcdd0fc83c17251?file=06-index.js&highlights=3

이렇게 `index.js`와 `progressbar.js` 파일을 작성했다. 이제 이를 컴파일해서 하나의 번들로 만들어야 하는데, 이는 webpack 으로 할 수 있다. 아래와 같이 `webpack.config.js` 파일을 만들어 보자. 여기서 `babel-loader`는 ES2015 문법과 호환을 유지하면서 번들링을 해주기 위한 일종의 번역기(?) 정도가 된다 (line #10). 모든 `.js`, `.jsx` 파일에 대해서 바벨 번역기를 실행시키고 (line #7) 번들링을 한 다음 [Blazor 웹 어셈블리][blazor wasm] 앱의 `wwwroot/js` 디렉토리에 `bundle.js` 라는 이름으로 결과물을 저장시키도록 하는 설정이다 (line #16-17). 여기서 `library`와 `libraryTarget` 옵션이 흥미로운데 (line #18-19), 일단 [Blazor 웹 어셈블리][blazor wasm] 앱에서 자바스크립트 모듈들은 `window` 객체 아래 존재하고, 이를 `FluentUiComponents` 라는 일종의 네임스페이스로 묶어준다고 이해하면 좋다.

https://gist.github.com/justinyoo/224fa5fe1bfa2dca8dcdd0fc83c17251?file=07-webpack-config.js&highlights=7,10,16-19

이렇게 작업한 후 [Blazor 웹 어셈블리][blazor wasm] 애플리케이션에서는 두 곳을 바꿔주면 된다. 먼저 `index.html` 파일을 열어 아래와 같이 `js/bundle.js` 파일을 추가한다 (line #2).

https://gist.github.com/justinyoo/224fa5fe1bfa2dca8dcdd0fc83c17251?file=08-index.html&highlights=2

여기까지 한 후 `package.json` 파일을 열어 아래와 같이 수정한다 (line #4). 이렇게 수정하면 `npm run build` 명령어를 통해 npm 패키지 빌드가 돌아갈 것이다.

https://gist.github.com/justinyoo/224fa5fe1bfa2dca8dcdd0fc83c17251?file=09-package.json&highlights=4

이후 `Counter.razor` 파일에서 아래와 같이 자바스크립트를 호출하면 된다. [이전 포스트][post prev]에서는 단순히 `RenderProgressBar` 펑션을 호출했다면, 이번에는 `FluentUiComponents.RenderProgressBar`와 같은 형태로 네임스페이스를 붙여주는 차이가 있다 (line #4).

https://gist.github.com/justinyoo/224fa5fe1bfa2dca8dcdd0fc83c17251?file=10-counter.razor&highlights=4


## Blazor 프로젝트에서 npm 패키지 한꺼번에 빌드하기 ##

여기서 팁이 한 가지 더 있다. `dotnet build` 명령어를 통해 Blazor 앱을 빌드할 수는 있지만 앞서 만들어 놓은 npm 패키지는 따로 빌드를 해야 한다. 하지만, `.csproj` 파일을 적당히 수정하면 npm 패키지도 함께 빌드할 수 있다. 아래와 같이 `.csproj` 파일을 수정한다. `PropertyGroup` 엘리먼트에 `JsLibraryRoot` 엘리먼트와 `DefaultItemExcludes` 엘리먼트를 추가한다 (line #7-8). 그리고, 빌드에 불필요한 `node_modules` 디렉토리를 제외하기 위해서 `Content`, `None` 엘리먼트를 추가한다 (line #18-22). 마지막으로 `Target` 엘리먼트를 통해 npm 패키지를 빌드한다 (line #24-28).

https://gist.github.com/justinyoo/224fa5fe1bfa2dca8dcdd0fc83c17251?file=11-app.csproj&highlights=7-8,18-22,24-28

이렇게 빌드가 끝난 후 애플리케이션을 실행시켜서 `Click me` 버튼을 클릭해 보자. 그러면 아래와 같이 Progress Indicator가 보인다.

![][image-03]

---

이렇게 해서 [Blazor 웹 어셈블리][blazor wasm] 애플리케이션에 [React][reactjs] 기반의 컴포넌트를 npm 패키지를 이용해 추가하는 방법에 대해 알아보았다. [다음 포스트][post next]에서는 이렇게 만들어진 [Blazor 웹 어셈블리][blazor wasm] 애플리케이션을 [애저 정적 웹 앱][az swa]에 배포하는 방법에 대해 알아보기로 하자.


[image-01]: /2020/06/adding-react-components-to-blazor-webassembly-app-by-nodejs-01.png
[image-02]: /2020/06/adding-react-components-to-blazor-webassembly-app-by-nodejs-02.png
[image-03]: /2020/06/adding-react-components-to-blazor-webassembly-app-by-nodejs-03.png

[gh sample]: https://github.com/devkimchi/Blazor-React-Sample

[post series 1]: /ko/2020/06/03/adding-react-components-to-blazor-webassembly-app/
[post series 2]: /ko/2020/06/10/adding-react-components-to-blazor-webassembly-app-by-nodejs/
[post series 3]: /ko/2020/06/17/hosting-blazor-web-assembly-app-on-azure-static-webapp/

[post prev]: /ko/2020/06/03/adding-react-components-to-blazor-webassembly-app/
[post next]: /ko/2020/06/17/hosting-blazor-web-assembly-app-on-azure-static-webapp/

[kedren]: https://www.linkedin.com/in/kedrenvillena/
[kedren post]: https://medium.com/swlh/using-npm-packages-with-blazor-2b0310279320

[blazor]: https://docs.microsoft.com/ko-kr/aspnet/core/blazor/?view=aspnetcore-3.1&WT.mc_id=aliencubeorg-blog-juyoo
[blazor wasm]: https://docs.microsoft.com/ko-kr/aspnet/core/blazor/?view=aspnetcore-3.1&WT.mc_id=aliencubeorg-blog-juyoo#blazor-webassembly
[blazor server]: https://docs.microsoft.com/ko-kr/aspnet/core/blazor/?view=aspnetcore-3.1&WT.mc_id=aliencubeorg-blog-juyoo#blazor-server
[blazor gettingstarted]: https://docs.microsoft.com/ko-kr/aspnet/core/blazor/get-started?view=aspnetcore-3.1&tabs=visual-studio-code&WT.mc_id=aliencubeorg-blog-juyoo
[blazor js from dotnet]: https://docs.microsoft.com/ko-kr/aspnet/core/blazor/call-javascript-from-dotnet?view=aspnetcore-3.1&WT.mc_id=aliencubeorg-blog-juyoo
[blazor dotnet from js]: https://docs.microsoft.com/ko-kr/aspnet/core/blazor/call-dotnet-from-javascript?view=aspnetcore-3.1&WT.mc_id=aliencubeorg-blog-juyoo
[blazor statemanagement]: https://docs.microsoft.com/ko-kr/aspnet/core/blazor/state-management?view=aspnetcore-3.1&WT.mc_id=aliencubeorg-blog-juyoo

[wasm]: https://webassembly.org/
[reactjs]: https://ko.reactjs.org/
[netcore sdk 3.1.4]: https://dotnet.microsoft.com/download/dotnet-core/3.1?WT.mc_id=aliencubeorg-blog-juyoo#3.1.4
[nodejs]: https://nodejs.org/
[npmjs]: https://www.npmjs.com/

[node-gyp issue]: https://github.com/nodejs/node-gyp/issues/569

[fluentui]: https://developer.microsoft.com/fluentui/?WT.mc_id=aliencubeorg-blog-juyoo
[fluentui progressindicator]: https://developer.microsoft.com/fluentui?WT.mc_id=aliencubeorg-blog-juyoo#/controls/web/progressindicator
[fluentui progressindicator codepen]: https://codepen.io/pen/?&editable=true=https%3A%2F%2Fdeveloper.microsoft.com%2Fen-us%2Ffluentui%3FWT.mc_id%3Daliencubeorg-blog-juyoo

[az swa]: https://docs.microsoft.com/ko-kr/azure/static-web-apps/overview?WT.mc_id=aliencubeorg-blog-juyoo
