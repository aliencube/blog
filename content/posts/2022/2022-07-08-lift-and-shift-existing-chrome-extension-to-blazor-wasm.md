---
title: "기존 크롬 익스텐션을 블레이저 웹어셈블리로 이전하기 - 블레이저 웹어셈블리 적용"
slug: lift-and-shift-existing-chrome-extension-to-blazor-wasm
description: "이 포스트에서는 기존에 자바스크립트 기반으로 작동하던 크롬 익스텐션을 블레이저 웹어셈블리로 별다른 설정 없이 이전하는 방법에 대해 알아봅니다."
date: "2022-07-08"
author: Justin-Yoo
tags:
- dotnet
- blazor-wasm
- chrome-extension
- migration
cover: /2022/07/lift-and-shift-existing-chrome-extension-to-blazor-wasm-00.png
fullscreen: true
---

[크롬 익스텐션][chrome extension]은 크로미움 기반의 웹 브라우저에서 다양한 용도로 활용할 수 있는 작은 애플리케이션이다. 기본적으로 웹서버가 필요하지 않은, 정적 컨텐츠로 만들어진 하나의 웹 애플리케이션이라고 생각하면 편하다. 그렇다면, 이 크롬 익스텐션을 [블레이저 웹어셈블리][blazor wasm] 기반으로 만들 수도 있지 않을까? 이 포스트를 통해 기존의 크롬 익스텐션을 최소한의 코드 변경만으로 블레이저 웹어셈블리 앱으로 이전하는 방법에 대해 알아보기로 하자.

> 이 포스트에 사용한 샘플 앱은 [이곳][gh sample]에서 다운로드 받을 수 있다.


## 블레이저 웹어셈블리를 활용한 브라우저 익스텐션 만들기 시리즈 ##

* ***기존 크롬 익스텐션을 블레이저 웹어셈블리로 이전하기 - 블레이저 웹어셈블리 적용*** 👈
* [기존 크롬 익스텐션을 블레이저 웹어셈블리로 이전하기 #2 - 자바스크립트 상호운용성][post 2]
* [기존 크롬 익스텐션을 블레이저 웹어셈블리로 이전하기 #3 - 크로스 브라우저 호환][post 3]


## 크롬 익스텐션 &ndash; 기존 자바스크립트 기반 ##

기존의 [매니페스트 v2][chrome extension v2] 기반으로 만들어진 [크롬 익스텐션][gh sample v2 original]이 하나 있다. 이 익스텐션을 크로미움 기반의 [엣지 브라우저][edge]에 등록시키면 아래와 같이 보인다.

![Chrome Extension on Microsoft Edge][image-01]

그리고 실제로 이 익스텐션을 페이지에서 실행시켜보면 아래와 같이 배경화면 색상을 바꿀 수 있다.

![Chrome Extension to change background colour #1][image-02]
![Chrome Extension that has changed the background colour #1][image-03]

이렇게 잘 작동하는 크롬 익스텐션을 이제 블레이저 웹어셈블리 앱으로 이전해 보기로 하자.

> 2022년 1월 17일 이후 매니페스트 v2 기반의 크롬 익스텐션은 더이상 정식으로 등록할 수 없다. v3 기반의 익스텐션은 다음 포스트에서 다룰 예정이다.


## 크롬 익스텐션 &ndash; 블레이저 웹어셈블리 기반 ##

우선 블레이저 웹 어셈블리 프로젝트를 하나 만든다. 그리고 나서 `index.html` 파일을 열어보면 아래와 같이 `app` 이라는 ID 값을 가진 `div` 태그가 하나 보인다.

![Visual Studio][image-04]

블레이저 웹어셈블리 애플리케이션은 기본적으로 이 `app` ID를 가진 `div` 태그 아래에 생기는 [버추얼 DOM][vdom]을 다루는 방식이다. 따라서, 크롬 익스텐션 역시도 이 버추얼 DOM을 다룰 수 있게끔 해야 한다. 이와 관련해서는 다음 포스트에서 다루기로 한다.

블레이저 웹어셈블리 앱과 크롬 익스텐션의 차이점 중 기억할 것이 하나 더 있다. 블레이저 웹어셈블리 앱에서 페이지 네비게이션은 라우팅에 기반한다. 즉 앞서 언급한 버추얼 DOM을 통해 각 페이지가 렌더링되는 방식이다. 따라서 물리적인 파일은 `index.html` 하나면 충분하다.

기존에 있던 `Counter.razor`와 `FetchData.razor`를 삭제한다. 그리고 `Popup.razor`을 추가한다.

```razor
@* Popup.razor *@

@page "/popup.html"

<PageTitle>Pop Up</PageTitle>

<h1>Pop Up - Chrome Extension with Blazor WASM</h1>

<button id="changeColor"></button>
```

마찬가지로 `Options.razor`를 추가한다.

```razor
@* Options.razor *@

@page "/options.html"

<PageTitle>Options</PageTitle>

<h1>Options - Chrome Extension with Blazor WASM</h1>

<div id="buttonDiv">
</div>
<div>
    <p>Choose a different background color!</p>
</div>
```

이 두 페이지를 통해 `popup.html`, `options.html` 페이지를 구현하면 된다.

여기까지 한 후 블레이저 웹어셈블리 앱을 퍼블리시한다.

```powershell
dotnet publish ./src/ChromeExtensionV2 -c Release -o published
```

그렇게 해서 만들어진 아티팩트는 아래와 같다.

![Chrome Extension published #1][image-05]

이 아티팩트를 엣지 브라우저에 등록해 보자. 그러면 아래와 같은 에러메시지를 보게 되는데, 이는 아티팩트가 만들어지면서 블레이저 웹어셈블리 앱이 `_framework`이라는 디렉토리를 자동으로 만들기 때문이다.

![Chrome Extension registration error #1][image-06]

크롬 익스텐션에서 이처럼 밑줄로 시작하는 이름은 허용하지 않기 때문에 이 이름을 바꿔줘야 한다. 마찬가지로 이미 만들어진 파일들 중에서 이 디렉토리를 참조하는 것이 있다면 모두 같이 바꿔줘야 한다. 이는 파워셸이나 bash 스크립트로 손쉽게 바꿀 수 있다. 여기서는 파워셸 스크립트를 통해 변경하기로 한다.

```powershell
# Run-PostBuild.ps1

function Update-FileContent {
    param (
        [string] $Filename,
        [string] $Value1,
        [string] $Value2
    )

    $content = Get-Content -Path $Filename -Raw
    $updated = $content -replace $Value1, $Value2
    Set-Content -Path $Filename -Value $updated -Force
}

Update-FileContent `
    -Filename "./published/wwwroot/_framework/blazor.webassembly.js" `
    -Value1 "_framework" `
    -Value2 "framework"

Update-FileContent `
    -Filename "./published/wwwroot/index.html" `
    -Value1 "_framework" `
    -Value2 "framework"

mv ./published/wwwroot/_framework ./published/wwwroot/framework
```

위 파워셸 파일을 실행시키면 아래와 같이 디렉토리 이름이 바뀐 것이 보인다.

![Chrome Extension published #2][image-07]

다시 등록을 시도해 보자. 그런데 이번에는 `options.html` 파일이 없다는 에러가 나타난다. 즉, 물리적으로 `options.html` 파일이 반드시 존재해야 한다는 말과 같다.

![Chrome Extension registration error #2][image-08]

이 에러로 미루어 볼 때, 크롬 익스텐션은 `popup.html`, `options.html` 파일이 반드시 존재한다는 것을 가정한다. 결국 앞서 작성한 두 Razor 페이지만으로는 충분하지 않기 때문에 두 파일을 같이 만들어야 한다. 그런데, 결국 `index.html`, `popup.html`, `options.html` 파일은 이름만 다를 뿐 버추얼 DOM 핸들링을 통해 각각의 파일에 있는 `app` ID를 가진 `div` 태그에 렌더링을 하는 방식이므로, 굳이 새로 만들기보다는 앞의 파워셸 스크립트를 조금만 수정해서 `index.html` 파일을 복제하면 된다.

```powershell
# Run-PostBuild.ps1
...
cp ./published/wwwroot/index.html ./published/wwwroot/popup.html
cp ./published/wwwroot/index.html ./published/wwwroot/options.html
```

다시 이 파워셸 스크립트를 실행시켜 보면 아래와 같이 `popup.html`, `options.html` 파일이 복제된 것이 보이고, 두 파일의 내용은 `index.html` 파일과 동일하다.

![Chrome Extension published #3][image-09]

이제 다시 익스텐션을 등록시켜보자. 이번엔 제대로 등록이 되었다.

![Chrome Extension registered #1][image-10]

그러면 실제로 작동하는지 한 번 보자. 아래 그림과 같이 "Inspect pop-up window" 메뉴를 클릭한다.

![Chrome Extension inspect pop-up window][image-11]

그러면 아래와 같이 개발자 도구 창이 나타나면서, 에러가 생겼다고 한다. [Content Security Policy][mdn content security policy] 쪽에 에러가 있다는 메시지이다.

![Chrome Extension pop-up error #1][image-12]

따라서 이를 해결해 주기 위해서는 아래와 같이 `manifest.json` 파일에 `content_security_policy` 속성을 추가해줘야 한다.

```json
{
  "manifest_version": 2,
  "version": "1.0",
  "name": "Getting Started Example (Blazor WASM)",
  "description": "Build an Extension!",

  ...

  "content_security_policy": "script-src 'self' 'unsafe-eval' 'wasm-unsafe-eval' 'sha256-v8v3RKRPmN4odZ1CWM5gw80QKPCCWMcpNeOmimNL2AA='; object-src 'self'",

  ...
}
```

* `unsafe-eval`: 자바스크립트의 `Function()` 개체, `setTimeout()`, `setInterval()` 함수 등을 사용하기 위해서는 반드시 추가해야 한다.
* `wasm-unsafe-eval`: 웹어셈블리 바이너리를 사용하기 위해서는 반드시 추가해야 한다.
* `sha256-v8v3RKRPmN4odZ1CWM5gw80QKPCCWMcpNeOmimNL2AA=`: 블레이저 웹어셈블리 라이브러리를 사용하기 위해서는 반드시 추가해야 한다.

이렇게 `manifest.json` 파일을 수정한 후 다시 익스텐션을 등록한 후 실행시켜보자. 아무 문제 없이 등록이 잘 됐다.

![Chrome Extension registered #2][image-13]

그런데, 에러는 없어졌지만 배경색을 바꿀 버튼이 안 보이기 때문에 원하는 대로 작동하지는 않는다. 그 이유는 아래 그림과 같이 `popup.html`, `options.html` 파일에 자바스크립트가 연결되어 있지 않기 때문이다.

![Chrome Extension popup.html #1][image-14]

이 파일들은 파워셸을 통해 `index.html` 파일로부터 자동으로 만들어진 것들이므로, 우선 `index.html` 파일에 아래와 같이 `js/main.js` 파일 레퍼런스를 추가한다. 그리고, 비어있는 `main.js` 파일을 `js` 디렉토리에 추가한다.

```html
<!DOCTYPE html>
<html lang="en">
...
<body>
    <div id="app">Loading...</div>
    ...
    <script src="_framework/blazor.webassembly.js"></script>
    <!-- ⬇️⬇️⬇️ Add this line ⬇️⬇️⬇️ -->
    <script src="js/main.js"></script>
    <!-- ⬆️⬆️⬆️ Add this line ⬆️⬆️⬆️ -->
</body>
</html>
```

이렇게 한 후, 파워셸 스크립트에 아래 내용을 추가해서 `popup.html`, `options.html` 파일 복제시 `popup.js`, `options.js` 파일로 수정한다.

```powershell
# Run-PostBuild.ps1
...
cp ./published/wwwroot/index.html ./published/wwwroot/popup.html
cp ./published/wwwroot/index.html ./published/wwwroot/options.html

Update-FileContent `
    -Filename "./published/wwwroot/popup.html" `
    -Value1 "js/main.js" `
    -Value2 "js/popup.js"

Update-FileContent `
    -Filename "./published/wwwroot/options.html" `
    -Value1 "js/main.js" `
    -Value2 "js/options.js"
```

이제 자바스크립트 참조가 끝났으니, 다시 익스텐션을 설치하고 실행시켜보자. 이번에는 아래와 같은 에러가 나타난다. 분명히 블레이저 웹어셈블리 앱이 아닌 상태에서는 제대로 작동하는 자바스크립트였는데, 지금은 기대한 바와 같은 결과를 보이지 않는다. 어떻게 된 일일까?

![Chrome Extension pop-up error #2][image-15]

문제는 바로 `blazor.webassembly.js` 파일에 있다. 이 자바스크립트를 통해 블레이저 웹어셈블리 앱을 시작하고 그 이후에 `popup.js` 혹은 `options.js` 파일을 로딩해서 돌아가야 하는데, 지금은 `blazor.webassembly.js` 파일을 통해 블레이저 웹어셈블리 앱을 시작하기도 전에 이 파일들이 실행이 됐기 때문에 위와 같은 에러가 생기는 것이다.

따라서, 이 문제를 해결하기 위해서는 `index.html` 파일의 자바스크립트 참조 부분을 아래와 같이 바꿔준다. 이 문제는 블레이저 웹어셈블리 앱의 [자바스크립트 상호운용성(JS interop)][blazor wasm jsinterop] 문서를 참조하면 더 자세한 내용을 알 수 있다.

```html
<!DOCTYPE html>
<html lang="en">
...
<body>
    <div id="app">Loading...</div>
    ...
    <!-- Add the 'autostart' attribute and set its value to 'false' -->
    <script src="_framework/blazor.webassembly.js" autostart="false"></script>
    <!-- ⬇️⬇️⬇️ Add these lines ⬇️⬇️⬇️ -->
    <script>
        Blazor.start().then(function () {
            var customScript = document.createElement('script');
            customScript.setAttribute('src', 'js/main.js');
            document.head.appendChild(customScript);
        });
    </script>
    <!-- ⬆️⬆️⬆️ Add these lines ⬆️⬆️⬆️ -->
</body>

</html>
```

앱을 빌드한 후 익스텐션을 다시 설치해 보자. Content Security Policy 관련 또 다른 에러가 나타난다.

![Chrome Extension pop-up error #3][image-16]

이 문제를 해결하기 위해서는 에러 화면에서 제시한 해시 값을 `manifest.json` 파일에 추가해 주면 된다.

```json
{
  "manifest_version": 2,
  "version": "1.0",
  "name": "Getting Started Example (Blazor WASM)",
  "description": "Build an Extension!",

  ...

  "content_security_policy": "script-src 'self' 'unsafe-eval' 'wasm-unsafe-eval' 'sha256-v8v3RKRPmN4odZ1CWM5gw80QKPCCWMcpNeOmimNL2AA=' 'sha256-DnTH4SKCYpHBGu1OxDOqoYLsvmZTiYIWJVQ1Ava7Kig=' 'sha256-b9roSuk6Pa7l0Hl/LWXGQlupw8fMc6ME2+82/N3qM0Q='; object-src 'self'",

  ...
}
```

* `sha256-DnTH4SKCYpHBGu1OxDOqoYLsvmZTiYIWJVQ1Ava7Kig=`: `popup.js` 파일을 가리키는 해시값이다.
* `sha256-b9roSuk6Pa7l0Hl/LWXGQlupw8fMc6ME2+82/N3qM0Q=`: `options.js` 파일을 가리키는 해시값이다.

위 두 해시값을 추가한 후 다시 익스텐션을 실행시켜보자. 이제는 제대로 실행되는 것을 확인할 수 있다.

![Chrome Extension to change background colour #2][image-17]
![Chrome Extension that has changed the background colour #2][image-18]

이제 기존의 크롬 익스텐션 앱이 블레이저 웹어셈블리로 성공적으로 이전할 수 있게 됐다. 블레이저 웹어셈블리로 완성된 크롬 익스텐션 앱은 [이곳][gh sample v2 blazor]에서 확인한다.

---

지금까지 [블레이저 웹어셈블리][blazor wasm] 앱을 이용해 기존의 자바스크립트 기반 크롬 익스텐션을 최소한의 코드 변경만으로 이전하는 것에 대해 알아보았다. 이 방식을 사용한다면 별다른 수고 없이도 블레이저 웹어셈블리 앱을 활용할 수 있겠지만, 사실 블레이저 웹어셈블리의 강력한 [자바스크립트 상호운용성(JS interop)][blazor wasm jsinterop] 기능을 십분 활용하진 못한 셈이다. [다음 포스트][post 2]에서는 동일한 크롬 익스텐션을 이용하지만 자바스크립트 상호운용성을 조금 더 적극적으로 활용하는 방법에 대해 알아보기로 한다.


## 블레이저 앱에 대해 더 알고 싶다면? ##

만약 블레이저 앱에 대해 좀 더 알고 싶다면 아래 튜토리얼을 참조해 보면 좋다.

* [블레이저][blazor] (영문)
* [블레이저 튜토리얼][blazor tutorial] (영문)
* [블레이저 Learn][blazor learn] (한국어)


[image-01]: /2022/07/lift-and-shift-existing-chrome-extension-to-blazor-wasm-01.png
[image-02]: /2022/07/lift-and-shift-existing-chrome-extension-to-blazor-wasm-02-ko.png
[image-03]: /2022/07/lift-and-shift-existing-chrome-extension-to-blazor-wasm-03-ko.png
[image-04]: /2022/07/lift-and-shift-existing-chrome-extension-to-blazor-wasm-04.png
[image-05]: /2022/07/lift-and-shift-existing-chrome-extension-to-blazor-wasm-05.png
[image-06]: /2022/07/lift-and-shift-existing-chrome-extension-to-blazor-wasm-06.png
[image-07]: /2022/07/lift-and-shift-existing-chrome-extension-to-blazor-wasm-07.png
[image-08]: /2022/07/lift-and-shift-existing-chrome-extension-to-blazor-wasm-08.png
[image-09]: /2022/07/lift-and-shift-existing-chrome-extension-to-blazor-wasm-09.png
[image-10]: /2022/07/lift-and-shift-existing-chrome-extension-to-blazor-wasm-10.png
[image-11]: /2022/07/lift-and-shift-existing-chrome-extension-to-blazor-wasm-11-ko.png
[image-12]: /2022/07/lift-and-shift-existing-chrome-extension-to-blazor-wasm-12.png
[image-13]: /2022/07/lift-and-shift-existing-chrome-extension-to-blazor-wasm-13-ko.png
[image-14]: /2022/07/lift-and-shift-existing-chrome-extension-to-blazor-wasm-14.png
[image-15]: /2022/07/lift-and-shift-existing-chrome-extension-to-blazor-wasm-15-ko.png
[image-16]: /2022/07/lift-and-shift-existing-chrome-extension-to-blazor-wasm-16-ko.png
[image-17]: /2022/07/lift-and-shift-existing-chrome-extension-to-blazor-wasm-17-ko.png
[image-18]: /2022/07/lift-and-shift-existing-chrome-extension-to-blazor-wasm-18-ko.png


[post 1]: /ko/2022/07/08/lift-and-shift-existing-chrome-extension-to-blazor-wasm/
[post 2]: /ko/2022/07/20/lift-and-shift-existing-chrome-extension-to-blazor-wasm-2/
[post 3]: /ko/2022/08/31/lift-and-shift-existing-chrome-extension-to-blazor-wasm-3/

[gh sample]: https://github.com/devkimchi/blazor-wasm-chrome-extension/tree/the-migration
[gh sample v2 original]: https://github.com/devkimchi/blazor-wasm-chrome-extension/tree/the-migration/src/chrome-extension-v2
[gh sample v2 blazor]: https://github.com/devkimchi/blazor-wasm-chrome-extension/tree/the-migration/src/ChromeExtensionV2

[chrome extension]: https://developer.chrome.com/docs/extensions/
[chrome extension v2]: https://developer.chrome.com/docs/extensions/mv2/getstarted/

[blazor]: https://dotnet.microsoft.com/apps/aspnet/web-apps/blazor?WT.mc_id=dotnet-70466-juyoo
[blazor tutorial]: https://dotnet.microsoft.com/learn/aspnet/blazor-tutorial/intro?WT.mc_id=dotnet-70466-juyoo
[blazor learn]: https://docs.microsoft.com/ko-kr/learn/paths/build-web-apps-with-blazor/?WT.mc_id=dotnet-70466-juyoo
[blazor wasm]: https://docs.microsoft.com/ko-kr/aspnet/core/blazor/host-and-deploy/webassembly?view=aspnetcore-6.0&WT.mc_id=dotnet-70466-juyoo
[blazor wasm jsinterop]: https://docs.microsoft.com/ko-kr/aspnet/core/blazor/javascript-interoperability/?view=aspnetcore-6.0&WT.mc_id=dotnet-70466-juyoo

[edge]: https://www.microsoft.com/ko-kr/edge?WC.mc_id=dotnet-70466-juyoo

[vdom]: https://en.wikipedia.org/wiki/Virtual_DOM

[mdn content security policy]: https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/manifest.json/content_security_policy
