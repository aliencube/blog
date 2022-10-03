---
title: "기존 크롬 익스텐션을 블레이저 웹어셈블리로 이전하기 #2 - 자바스크립트 상호운용성"
slug: lift-and-shift-existing-chrome-extension-to-blazor-wasm-2
description: "이 포스트에서는 기존에 자바스크립트 기반으로 작동하던 크롬 익스텐션을 블레이저 웹어셈블리로 별다른 설정 없이 이전하는 방법에 대해 알아봅니다."
date: "2022-07-20"
author: Justin-Yoo
tags:
- dotnet
- blazor-wasm
- chrome-extension
- migration
cover: /2022/07/lift-and-shift-existing-chrome-extension-to-blazor-wasm-00.png
fullscreen: true
---

[지난 포스트][post 1]에서는 최소한의 코드 변경만으로 기존 자바스크립트 기반의 [크롬 익스텐션][chrome extension]을 [블레이저 웹어셈블리][blazor wasm] 기반으로 이전하는 방법에 대해 알아 보았다. 하지만 이 때에는 블레이저 웹어셈블리의 장점인 [자바스크립트 상호운용성(JS interop)][blazor wasm jsinterop] 기능을 제대로 활용하지는 않았다. 이 포스트를 통해 이 자바스크립트 상호운용성 기능을 좀 더 적극적으로 활용하는 방식을 다뤄보기로 한다.

> 이 포스트에 사용한 샘플 앱은 [이곳][gh sample]에서 다운로드 받을 수 있다.


## 블레이저 웹어셈블리를 활용한 브라우저 익스텐션 만들기 시리즈 ##

* [기존 크롬 익스텐션을 블레이저 웹어셈블리로 이전하기 - 블레이저 웹어셈블리 적용][post 1]
* ***기존 크롬 익스텐션을 블레이저 웹어셈블리로 이전하기 #2 - 자바스크립트 상호운용성*** 👈
* [기존 크롬 익스텐션을 블레이저 웹어셈블리로 이전하기 #3 - 크로스 브라우저 호환][post 3]


## 크롬 익스텐션 &ndash; 자바스크립트 상호운용성 적용 전 ##

[지난 포스트][post 1]에서 작성한 `index.html` 파일을 보면 아래와 같다. 블레이저 웹어셈블리 스크립트를 로딩한 후에 추가 스크립트를 통해 각 페이지별 자바스크립트를 로딩하는 방식이다. 아래 코드는 `js/main.js` 파일이지만, 아티팩트를 만드는 과정에서 `js/options.js`, `js/popup.js` 파일을 로딩하는 형식으로 바뀐다.

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

여기서 두 가지 불편한 점이 있다.

1. `blazor.webassembly.js` 파일을 로딩하면서 `autostart="false"` 옵션을 함께 선언해야 한다.
2. 프로미스 패턴의 형태로 `Blazor.start()` 이후 `js/main.js` 파일을 별도로 추가해 줘야 한다.

이 부분을 최대한 `index.html` 파일을 건드리지 않고 자바스크립트 상호운용성을 활용할 수 있는 방향으로 바꿔주면 좀 더 블레이저 앱 스러운 형태로 바꿀 수 있지 않을까?


## 크롬 익스텐션 &ndash; 자바스크립트 상호운용성 적용 후 #1 ##

위의 `index.html` 파일을 아래와 같이 변경한다. 앞서와 달리 `Blazor.start()` 함수를 호출하는 스크립트를 삭제하고 대신 `blazor.webassembly.js` 파일을 호출하기 전에 `js/main.js` 파일을 호출한다.

```html
<!DOCTYPE html>
<html lang="en">
...
<body>
    <div id="app">Loading...</div>
    ...
    <!-- ⬇️⬇️⬇️ Add this line ⬇️⬇️⬇️ -->
    <script src="js/main.js"></script>
    <!-- ⬆️⬆️⬆️ Add this line ⬆️⬆️⬆️ -->
    <script src="_framework/blazor.webassembly.js"></script>
</body>
</html>
```

`js/main.js` 파일은 원래는 아무것도 없는 빈 파일이었지만, 이번에는 아래 내용을 추가한다. 코드를 보면 알 수 있다시피 `script` 태그를 추가해서 주어진 자바스크립트 파일을 로딩하는 함수이다.

```javascript
function loadJs(sourceUrl) {
  if (sourceUrl.Length == 0) {
    console.error("Invalid source URL");
    return;
  }

  var tag = document.createElement('script');
  tag.src = sourceUrl;
  tag.type = "text/javascript";

  tag.onload = function () {
    console.log("Script loaded successfully");
  }

  tag.onerror = function () {
    console.error("Failed to load script");
  }

  document.body.appendChild(tag);
}
```

이번에는 `Popup.razor` 파일을 아래와 같이 수정한다.

1. 먼저 `IJSRuntime` 인스턴스를 의존성 주입으로 선언한다.
2. 이후 `JS.InvokeVoidAsync` 메소드를 통해 앞서 `js/main.js` 파일에 정의한 펑션을 호출해서 `js/popup.js` 파일을 로딩한다.

```razor
@* Popup.razor *@

@page "/popup.html"

@* Inject IJSRuntime instance *@
@inject IJSRuntime JS

...

@code {
    protected override async Task OnAfterRenderAsync(bool firstRender)
    {
        if (!firstRender)
        {
            return;
        }
        
        var src = "js/popup.js";

        // Invoke the `loadJs` function
        await JS.InvokeVoidAsync("loadJs", src).ConfigureAwait(false);
    }
}
```

같은 방식으로 `Options.razor` 파일도 아래와 같이 수정한다.

```razor
@* Options.razor *@

@page "/options.html"

@* Inject IJSRuntime instance *@
@inject IJSRuntime JS

...

@code {
    protected override async Task OnAfterRenderAsync(bool firstRender)
    {
        if (!firstRender)
        {
            return;
        }
        
        var src = "js/options.js";

        // Invoke the `loadJs` function
        await JS.InvokeVoidAsync("loadJs", src).ConfigureAwait(false);
    }
}
```

이렇게 한 후, 마지막으로 파워셸 스크립트에서 `js/main.js` 파일 레퍼런스를 `js/popup.js`, `js/options.js` 파일로 바꾸는 부분을 삭제한다.

```powershell
# Run-PostBuild.ps1

...

# Update-FileContent `
#     -Filename "./published/wwwroot/popup.html" `
#     -Value1 "js/main.js" `
#     -Value2 "js/popup.js"

# Update-FileContent `
#     -Filename "./published/wwwroot/options.html" `
#     -Value1 "js/main.js" `
#     -Value2 "js/options.js"
```

이후 앱을 빌드하고 발행한 후 파워셸로 마무리작업을 한 후 다시 크롬 익스텐션을 실행시켜 보면 아무 문제 없이 작동하는 것을 알 수 있다. 이렇게 하면 `loadJs` 펑션을 자바스크립트 상호운용성 기능을 이용해 호출하고 이를 통해 페이지별로 원하는 자바스크립트 파일을 로딩할 수 있게 된다.

하지만, 여전히 `index.html` 파일을 수정해서 `js/main.js` 파일을 로딩해야 하는 번거로움이 있는데, 이 부분 마저도 자바스크립트 상호운용성 기능을 이용해서 없애버릴 수 있을까?


## 크롬 익스텐션 &ndash; 자바스크립트 상호운용성 적용 후 #2 ##

이번에는 아예 `index.html` 파일을 블레이저 웹어셈블리 프로젝트를 처음 만들었을 때의 모습, 즉 아무 수정도 하지 않은 최초의 모습으로 돌려 놓는다. 그렇게 하면 자바스크립트는 아래와 같이 `blazor.webassembly.js` 파일만 로딩하게 된다.

```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no" />
    <title>ChromeExtensionV2</title>
    <base href="/" />
    <link href="css/bootstrap/bootstrap.min.css" rel="stylesheet" />
    <link href="css/app.css" rel="stylesheet" />
    <link href="ChromeExtensionV2.styles.css" rel="stylesheet" />
</head>

<body>
    <div id="app">Loading...</div>

    <div id="blazor-error-ui">
        An unhandled error has occurred.
        <a href="" class="reload">Reload</a>
        <a class="dismiss">🗙</a>
    </div>
    <script src="_framework/blazor.webassembly.js"></script>
</body>
</html>
```

그리고, `js/main.js` 파일에 정의해 놓은 자바스크립트 함수인 `loadJs` 함수 앞에 아래와 같이 `export` 선언자를 붙여 수정한다.

```javascript
export function loadJs(sourceUrl) {
...
}
```

그리고, `Popup.razor` 파일을 아래와 같이 수정한다.

```csharp
...
var src = "js/popup.js";

// Import the `js/main.js` file
var module = await JS.InvokeAsync<IJSObjectReference>("import", "./js/main.js").ConfigureAwait(false);

// Invoke the `loadJs` function
await module.InvokeVoidAsync("loadJs", src).ConfigureAwait(false);
```

마지막으로 `manifest.json` 파일을 아래와 같이 수정한다. 더이상 `js/popup.js`, `js/options.js` 파일에 대한 해시키가 필요 없기 때문이다.

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

이후 앱을 빌드하고 발행한 후 다시 실행시켜 보면 앞서와 동일한 결과를 얻을 수 있다. 즉, 크롬 익스텐션이 정상적으로 작동한다.

---

지금까지, 블레이저 웹어셈블리의 [자바스크립트 상호운용성][blazor wasm jsinterop] 기능을 좀 더 적극적으로 활용해서 기존의 크롬 익스텐션을 블레이저 웹어셈블리로 이전하는 작업을 해 보았다. 이렇게 하는 것의 잇점은 무엇일까?

1. 블레이저 웹어셈블리 앱의 부트스트랩 코드를 전혀 건드리지 않는다.
2. 필요한 자바스크립트는 자바스크립트 상호운용성 기능을 이용해 페이지별로 필요할 때 로딩해서 사용한다. 이 과정에서 모든 자바스크립트를 C# 코드로 제어하게 된다.

그렇다면, 이 방법이 오로지 잇점만 있을까? 아래와 같은 사항도 고려해 봐야 한다:

1. 이 포스트에서 사용한 방식대로 자바스크립트 상호운용성 기능을 사용하는 것은 어찌보면 불필요한 복잡도를 높이는 것일 수도 있다. 단순히 `index.html`/`popup.html`/`options.html` 파일을 통해 자바스크립트를 로딩하는 방식을 사용한다면 굳이 이런 식의 접근이 필요하지 않을 수도 있다.
2. 이 포스트에서 활용한 동적 JS 로딩 방식이 항상 효과적인 것도 아니다. 반드시 트레이드오프가 있게 마련인데, 부트스트래퍼 파일을 수정하고 싶지 않다면, 이 방식이 도움이 되겠지만, 결국 어떤 식으로든 부트스트래퍼 파일을 건드려야 하는 상황이 온다면 그 땐 적절하게 동적 로딩을 사용하는 것이 필요할 것이다.

결국, 이와 같은 형태로 자바스크립트 상호운용성을 좀 더 적극적으로 그리고 적절하게 사용한다면 블레이저 웹어셈블리 형태로 크롬 익스텐션을 더욱 더 효과적으로 만들 수 있을 것이다. [다음 포스트][post 3]에서는 이렇게 만든 크롬 브라우저를 다양한 브라우저 엔진에서 활용할 수 있게끔 크로스 브라우저 호환성을 추가해 보기로 한다.


## 블레이저 앱에 대해 더 알고 싶다면? ##

만약 블레이저 앱에 대해 좀 더 알고 싶다면 아래 튜토리얼을 참조해 보면 좋다.

* [블레이저][blazor] (영문)
* [블레이저 튜토리얼][blazor tutorial] (영문)
* [블레이저 Learn][blazor learn] (한국어)


[post 1]: /ko/2022/07/08/lift-and-shift-existing-chrome-extension-to-blazor-wasm/
[post 2]: /ko/2022/07/20/lift-and-shift-existing-chrome-extension-to-blazor-wasm-2/
[post 3]: /ko/2022/08/31/lift-and-shift-existing-chrome-extension-to-blazor-wasm-3/

[gh sample]: https://github.com/devkimchi/blazor-wasm-chrome-extension/tree/the-integration
[gh sample v2 blazor]: https://github.com/devkimchi/blazor-wasm-chrome-extension/tree/the-integration/src/ChromeExtensionV2

[chrome extension]: https://developer.chrome.com/docs/extensions/
[chrome extension v2]: https://developer.chrome.com/docs/extensions/mv2/getstarted/

[blazor]: https://dotnet.microsoft.com/apps/aspnet/web-apps/blazor?WT.mc_id=dotnet-70466-juyoo
[blazor tutorial]: https://dotnet.microsoft.com/learn/aspnet/blazor-tutorial/intro?WT.mc_id=dotnet-70466-juyoo
[blazor learn]: https://docs.microsoft.com/ko-kr/learn/paths/build-web-apps-with-blazor/?WT.mc_id=dotnet-70466-juyoo
[blazor wasm]: https://docs.microsoft.com/ko-kr/aspnet/core/blazor/host-and-deploy/webassembly?view=aspnetcore-6.0&WT.mc_id=dotnet-70466-juyoo
[blazor wasm jsinterop]: https://docs.microsoft.com/ko-kr/aspnet/core/blazor/javascript-interoperability/?view=aspnetcore-6.0&WT.mc_id=dotnet-70466-juyoo
