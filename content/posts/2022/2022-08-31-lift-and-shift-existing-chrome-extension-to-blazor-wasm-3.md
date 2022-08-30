---
title: "기존 크롬 익스텐션을 블레이저 웹어셈블리로 이전하기 #3 - 크로스 브라우저 호환"
slug: lift-and-shift-existing-chrome-extension-to-blazor-wasm-3
description: "이 포스트에서는 기존에 자바스크립트 기반으로 작동하던 크롬 익스텐션을 블레이저 웹어셈블리로 이전하면서 크로스 브라우저 지원을 하는 방법에 대해 알아봅니다."
date: "2022-08-31"
author: Justin-Yoo
tags:
- dotnet
- blazor-wasm
- browser-extension
- cross-browser
cover: https://sa0blogs.blob.core.windows.net/aliencube/2022/08/lift-and-shift-existing-chrome-extension-to-blazor-wasm-3-00.png
fullscreen: true
---

[지난 포스트][post 2]에서는 [블레이저 웹어셈블리][blazor wasm]의 강점인 [자바스크립트 상호운용성(JS interop)][blazor wasm jsinterop] 기능을 활용해서 [크롬 익스텐션][chrome extension]을 작성해 보았다. 이번 포스트에서는 이 크롬 익스텐션을 크로미움 계열의 브라우저 뿐만 아니라 모질라 계열의 브라우저에서도 작동할 수 있게끔 웹표준을 적용시키면서 반드시 고려해야 할 부분에 대해 알아보기로 한다.

> 이 포스트에 사용한 샘플 앱은 [이곳][gh sample]에서 다운로드 받을 수 있다.


## 블레이저 웹어셈블리를 활용한 브라우저 익스텐션 만들기 시리즈 ##

* [기존 크롬 익스텐션을 블레이저 웹어셈블리로 이전하기 - 블레이저 웹어셈블리 적용][post 1]
* [기존 크롬 익스텐션을 블레이저 웹어셈블리로 이전하기 #2 - 자바스크립트 상호운용성][post 2]
* ***기존 크롬 익스텐션을 블레이저 웹어셈블리로 이전하기 #3 - 크로스 브라우저 호환*** 👈


## 브라우저 익스텐션 폴리필 ##

[W3.org][w3]의 [브라우저 익스텐션 그룹][w3 browserext]에서는 크롬 익스텐션을 중심으로 해서 모든 웹 브라우저로 익스텐션 기능을 확장할 수 있는 웹표준을 제시한다. 그리고 이를 모질라에서 [프로미스][mozilla promise] 방식으로 구현한 것이 바로 [브라우저 익스텐션 폴리필][mozilla browserext polyfill]이다. 따라서 이 폴리필을 추가하면 기본적으로 크롬 익스텐션은 크로스 브라우징이 가능한 브라우저 익스텐션으로 바뀌게 된다.

따라서 `index.html` 파일에 아래와 같이 폴리필 스크립트를 CDN을 통해 추가해 주면 좋다. 아래와 같이 추가하면 가장 최신 버전의 폴리필을 자동으로 연결해서 사용한다.

```html
<!DOCTYPE html>
<html lang="en">
...
<body>
    <div id="app">Loading...</div>
    ...
    <!-- ⬇️⬇️⬇️ Add this line ⬇️⬇️⬇️ -->
    <script src="https://unpkg.com/browse/webextension-polyfill/dist/browser-polyfill.min.js"></script>
    <!-- ⬆️⬆️⬆️ Add this line ⬆️⬆️⬆️ -->
    <script src="_framework/blazor.webassembly.js"></script>
</body>
</html>
```

하지만 이 상태에서 빌드해서 사용할 수는 없는 것이 외부 URL을 직접 참조로 추가하게 되면 보안 위배사항으로 아래와 같은 에러가 난다.

![Refused to load script][image-01]

이를 해결할 수 있는 방법은 자바스크립트를 다운로드 받아서 로컬에서 참조를 해야 한다. [CDN][mozilla browserext polyfill unpkg] 페이지로 들어가서 하나하나 다운로드 받은 후 `wwwroot/js/dist` 디렉토리에 저장한다. 이후 `index.html` 파일을 아래와 같이 수정한다.

```html
<!DOCTYPE html>
<html lang="en">
...
<body>
    <div id="app">Loading...</div>
    ...
    <!-- ⬇️⬇️⬇️ Add this line ⬇️⬇️⬇️ -->
    <script src="js/dist/browser-polyfill.min.js"></script>
    <!-- ⬆️⬆️⬆️ Add this line ⬆️⬆️⬆️ -->
    <script src="_framework/blazor.webassembly.js"></script>
</body>
</html>
```

이렇게 폴리필을 추가하는 작업을 끝냈다. 이제 기존의 `manifest.json` 파일을 브라우저 익스텐션의 기준에 맞춰 수정해야 한다.


## `manifest.json` 수정하기 ##

가장 먼저 해야 할 일은 크롬 익스텐션 고유의 비호환성을 제거하는 일이다. 특히 [Declarative Content][chrome extension declarativecontent] 부분은 호환성이 없으므로 삭제하고 다른 형태로 대체해야 한다. 이 때 특정 웹사이트 도메인(*developer.chrome.com*, *developer.mozilla.org* 혹은 *docs.microsoft.com*)에서만 이 익스텐션이 작동하게끔 하려면 아래와 같이 `permissions` 속성에 웹사이트 주소를 넣어주면 된다.

```json
{
  ...

  "permissions": [
    "*://developer.chrome.com/*",
    "*://developer.mozilla.org/*",
    "*://docs.microsoft.com/*",
    "activeTab",
    // "declarativeContent",
    "storage"
  ],

  ...
}
```

그리고 백그라운드로 작동하는 스크립트에 방금 다운로드 받은 폴리필을 추가한다.


```json
{
  ...

  "background": {
    "scripts": [
      "js/dist/browser-polyfill.min.js",
      "js/background.js"
    ],
    "persistent": false
  },

  ...
}
```

그리고 `options_ui` 속성을 이용해서 `options.html` 페이지를 구동하는 방식을 독립적인 페이지 대신 익스텐션 화면 안의 팝업으로 변경한다.

```json
{
  ...

  "options_page": "options.html",

  "options_ui": {
    "page": "options.html",
    "browser_style": true
  }

  ...
}
```

마지막으로 모질라 기반의 브라우저에서는 익스텐션별로 고유의 ID를 부여해야 하므로 아래와 같이 브라우저별 특성을 선언해 준다.

```json
{
  ...

  "browser_specific_settings": {
    "gecko": {
      "id": "browser-extension-sample@devkimchi.com"
    }
  },

  ...
}
```

이렇게 `manifest.json` 파일의 수정이 끝났다면, 실제로 기존에 작동하던 자바스크립트 파일을 수정해 줄 차례이다.


## `background.js` 수정하기 ##

가장 먼저 할 일은 `background.js` 파일에 있는 모든 `chrome.` 인스턴스를 `browser.` 인스턴스로 바꾸는 것이다. `chrome` 인스턴스는 크롬 익스텐션에 특화된 개체이고, `browser` 인스턴스는 범용 브라우저 익스텐션에 특화된 개체이기 때문이다. 즉, 현재 `background.js` 파일의 내용이 아래와 같다면,

```javascript
chrome.runtime.onInstalled.addListener(function() {
  chrome.storage.sync.set({color: '#3aa757'}, function() {
    console.log("The color is green.");
  });

  chrome.declarativeContent.onPageChanged.removeRules(undefined, function() {
    chrome.declarativeContent.onPageChanged.addRules([{
      conditions: [
        new chrome.declarativeContent.PageStateMatcher({
          pageUrl: { hostEquals: 'developer.chrome.com' },
        }),
        new chrome.declarativeContent.PageStateMatcher({
          pageUrl: { hostEquals: 'docs.microsoft.com' },
        })
      ],
      actions: [new chrome.declarativeContent.ShowPageAction()]
    }]);
  });
});
```

여기에 언급된 모든 `chrome.`으로 시작하는 부분을 `browser.`로 바꿔줘야 한다는 말이다.

```javascript
// Use 'browser.' instead of 'chrome.'
browser.runtime.onInstalled.addListener(function() {
  // Use 'browser.' instead of 'chrome.'
  browser.storage.sync.set({color: '#3aa757'}, function() {
    console.log("The color is green.");
  });

  // Use 'browser.' instead of 'chrome.'
  browser.declarativeContent.onPageChanged.removeRules(undefined, function() {
    // Use 'browser.' instead of 'chrome.'
    browser.declarativeContent.onPageChanged.addRules([{
      conditions: [
        // Use 'browser.' instead of 'chrome.'
        new browser.declarativeContent.PageStateMatcher({
          pageUrl: { hostEquals: 'developer.browser.com' },
        }),
        // Use 'browser.' instead of 'chrome.'
        new browser.declarativeContent.PageStateMatcher({
          pageUrl: { hostEquals: 'docs.microsoft.com' },
        })
      ],
      // Use 'browser.' instead of 'chrome.'
      actions: [new browser.declarativeContent.ShowPageAction()]
    }]);
  });
});
```

두번째로 할 일은 모든 콜백 메소드를 프로미스 패턴으로 바꿔주는 일이다. 예를 들어 아래와 같이 브라우저의 로컬 스토리지에 접근하는 함수를 아래와 같이 수정한다.

```javascript
  // Before
  browser.storage.sync.set({color: '#3aa757'}, function() {
    console.log("The color is green.");
  });

  // After
  browser.storage.sync.set({color: '#3aa757'})
  .then(() => {
    console.log("The color is green.");
  });
```

하지만 위 내용은 `onInstalled` 라는 이벤트에 핸들러로 추가하는 것이므로 아예 펑션의 형태로 바꾸는 것이 편리하다. 아래와 같이 펑션으로 바꿔준다.

```javascript
function handleRuntimeOnInstalled(details) {
  browser.storage.sync.set({color: '#3aa757'})
  .then(() => {
    console.log("The color is green.");
  });
}
```

마지막으로는 앞서 언급한 바와 같이 [Declarative Content][chrome extension declarativecontent] 속성은 호환성이 없으므로, `declarativeContent` 속성 대신 사용할 이벤트 핸들러를 구현해야 한다. 이 샘플에서는 특정 웹사이트 도메인(*developer.google.com*, *developer.mozilla.org* 혹은 *docs.microsoft.com*)으로 접속할 때에만 이 익스텐션을 작동시키게끔 하는 기능을 담당하므로 이를 아래와 같이 `handleTabs()` 라는 펑션으로 대체한다.

```javascript
function handleTabs() {
  browser.tabs.query({active: true, currentWindow: true})
  .then((tabs) => {
    console.log(tabs[0].url);
    
    let matched = tabs[0].url.includes('developer.chrome.com') || tabs[0].url.includes('developer.mozilla.org') || tabs[0].url.includes('docs.microsoft.com');
    if (matched) {
      browser.pageAction.show(tabs[0].id);
    } else {
      browser.pageAction.hide(tabs[0].id);
    }
  });
}
```

이렇게 한 뒤 파일의 맨 아랫부분에서 모든 이벤트 핸들러를 각각의 이벤트에 등록시킨다.

```javascript
browser.runtime.onInstalled.addListener(handleRuntimeOnInstalled);
browser.tabs.onActivated.addListener(handleTabs);
browser.tabs.onHighlighted.addListener(handleTabs);
browser.tabs.onUpdated.addListener(handleTabs);
```


## `popup.js` 수정하기 ##

이번에는 `popup.js` 파일을 수정해 보자. 원래는 아래와 같았다.

```javascript
let changeColor = document.getElementById('changeColor');

chrome.storage.sync.get('color', function(data) {
  changeColor.style.backgroundColor = data.color;
  changeColor.setAttribute('value', data.color);
});

changeColor.onclick = function(element) {
  let color = element.target.value;
  chrome.tabs.query({active: true, currentWindow: true}, function(tabs) {
    chrome.tabs.executeScript(
        tabs[0].id,
        {code: 'document.body.style.backgroundColor = "' + color + '";'});
  });
};
```

앞서와 마찬가지로 모든 `chrome.`으로 시작하는 부분을 `browser.`로 바꿔준다. 그리고, 콜백으로 된 부분을 프로미스 패턴으로 바꿔준다. 그러면 아래와 같이 바뀔 것이다. 마찬가지로 특정 웹사이트 도메인(*developer.google.com*, *developer.mozilla.org* 혹은 *docs.microsoft.com*)에서만 작동하게끔 설정했다.

```javascript
let changeColor = document.getElementById('changeColor');

// Use 'browser.' instead of 'chrome.'
// Use the promise pattern
browser.storage.sync.get('color')
.then((data) => {
  changeColor.style.backgroundColor = data.color;
  changeColor.setAttribute('value', data.color);
});

changeColor.onclick = function(element) {
  let color = element.target.value;

  // Use 'browser.' instead of 'chrome.'
  // Use the promise pattern
  browser.tabs.query({active: true, currentWindow: true})
  .then((tabs) => {
    let matched = tabs[0].url.includes('developer.chrome.com') || tabs[0].url.includes('developer.mozilla.org') || tabs[0].url.includes('docs.microsoft.com');
    if (matched) {
      // Use 'browser.' instead of 'chrome.'
      browser.tabs.executeScript(
        tabs[0].id,
        {code: 'document.body.style.backgroundColor = "' + color + '";'});
    } else {
      console.log('URL not matched');
    }
  });
};
```

이렇게 해서 `popup.js` 파일을 수정했다.


## `options.js` 수정하기 ##

마지막으로 `options.js` 파일을 수정할 차례이다. 원래는 아래와 같이 구현했다.

```javascript
let page = document.getElementById('buttonDiv');

const kButtonColors = ['#3aa757', '#e8453c', '#f9bb2d', '#4688f1'];

function constructOptions(kButtonColors) {
  for (let item of kButtonColors) {
    let button = document.createElement('button');
    button.className = 'color-button';
    button.style.backgroundColor = item;
    button.style.padding = '10px';

    button.addEventListener('click', function() {
      chrome.storage.sync.set({color: item}, function() {
        console.log('color is ' + item);
      })
    });
    page.appendChild(button);
  }
}

constructOptions(kButtonColors);
```

마찬가지로 `browser.`로 바꾸고, 콜백 대신 프로미스 패턴으로 바꾸면 아래와 같이 된다.

```javascript
let page = document.getElementById('buttonDiv');

const kButtonColors = ['#3aa757', '#e8453c', '#f9bb2d', '#4688f1'];

function constructOptions(kButtonColors) {
  for (let item of kButtonColors) {
    let button = document.createElement('button');
    button.className = 'color-button';
    button.style.backgroundColor = item;
    button.style.padding = '10px';

    button.addEventListener('click', function() {
      // Use 'browser.' instead of 'chrome.'
      // Use the promise pattern
      browser.storage.sync.set({color: item})
      .then(() => {
        console.log('color is ' + item);
      })
    });
    page.appendChild(button);
  }
}

constructOptions(kButtonColors);
```

이렇게 해서 `options.js` 파일을 수정했다.


## 블레이저 컴포넌트 추상화 ##

만약 `Popup.razor` 페이지와 `Options.razor` 페이지에 공통적으로 들어가 있는 `js/main.js` 임포트 모듈을 추상화하고 더불어 `js/dist/browser-polyfill.min.js` 임포트 모듈을 추상화하고 싶다면 아래와 같이 블레이저 공통 컴포넌트를 만들어도 좋다. 먼저 각 페이지별로 `OnAfterRenderAsync(...)` 메소드에서 호출할 메소드를 정의한다.

```csharp
public class PageComponentBase : ComponentBase
{
    protected abstract Task LoadAdditionalJsAsync();
}
```

이후 이를 아래와 같이 `OnAfterRenderAsync(...)` 메소드 마지막에 불러들인다.

```csharp
public class PageComponentBase : ComponentBase
{
    [Inject]
    private IJSRuntime JS { get; set; }

    protected IJSObjectReference Module { get; private set; }

    protected override async Task OnAfterRenderAsync(bool firstRender)
    {
        this.Module = await this.JS.InvokeAsync<IJSObjectReference>("import", "./js/main.js");

        var src = "js/dist/browser-polyfill.min.js";
        await this.Module.InvokeVoidAsync("loadJs", src);

        // Invoke the page specific JavaScript loader
        await this.LoadAdditionalJsAsync();
    }

    protected abstract Task LoadAdditionalJsAsync();
}
```

그리고 `Popup.razor` 페이지에서는 `PageComponentBase` 클래스를 상속받고, 아래와 같이 `LoadAdditionalJsAsync()` 메소드를 구현한다. 기존에 의존성 주입을 위해 사용했던 `IJSRuntime` 인스턴스는 더이상 사용할 필요가 없으므로 제거한다.

```csharp
@page "/popup.html"

@* @inject IJSRuntime JS *@

@using ChromeExtensionV2.Components
@inherits PageComponentBase

...

@code {
    protected override async Task LoadAdditionalJsAsync()
    {
        var src = "js/popup.js";
        await this.Module.InvokeVoidAsync("loadJs", src);
    }
}
```

그리고 `Options.razor` 페이지에서도 `PageComponentBase` 클래스를 상속받고, 아래와 같이 `LoadAdditionalJsAsync()` 메소드를 구현한다. 마찬가지로 기존에 의존성 주입을 위해 사용했던 `IJSRuntime` 인스턴스는 더이상 사용할 필요가 없으므로 제거한다.

```csharp
@page "/options.html"

@* @inject IJSRuntime JS *@

@using ChromeExtensionV2.Components
@inherits PageComponentBase

...

@code {
    protected override async Task LoadAdditionalJsAsync()
    {
        var src = "js/options.js";
        await this.Module.InvokeVoidAsync("loadJs", src);
    }
}
```


## `Run-PostBuild.ps1` 수정하기 ##

모질라 계열 브라우저에 익스텐션을 설치하는 방법은 크롬 계열의 브라우저와는 달라서 `.zip` 파일로 압축한 아티팩트가 필요하다. 따라서, `Run-PostBuild.ps1` 파일의 맨 마지막 단계로서 압축파일을 만드는 단계를 아래와 같이 추가한다.

```powershell
Compress-Archive -Path ./published/wwwroot/* -DestinationPath ./published/wwwroot/wwwroot.zip -Force
```

여기까지 작성한 후, 프로젝트를 빌드하고 위의 파워셸 스크립트를 실행시키면 `wwwroot.zip` 파일이 `published` 디렉토리에 생성된다. 이 파일을 파이어폭스 브라우저에 익스텐션으로 등록시키면 아래와 같은 화면을 볼 수 있다.

![Browser extension on FireFox #1][image-02]

파이어폭스에서는 이렇게 옵션 화면을 선택할 수 있다. 배경색을 노랑색으로 변경한다.

![Browser extension options on FireFox][image-03]

그러면 아래와 같이 노랑색으로 배경색을 바꿀 수 있는 버튼이 나타난다.

![Browser extension on FireFox #2][image-04]

---

지금까지, 블레이저 웹어셈블리 앱으로 크롬 익스텐션을 만든 후, 이를 크로스 브라우저 호환성을 지키면서 모질라의 파이어폭스 브라우저에서도 작동할 수 있게끔 수정했다.


## 블레이저 앱에 대해 더 알고 싶다면? ##

만약 블레이저 앱에 대해 좀 더 알고 싶다면 아래 튜토리얼을 참조해 보면 좋다.

* [블레이저][blazor] (영문)
* [블레이저 튜토리얼][blazor tutorial] (영문)
* [블레이저 Learn][blazor learn] (한국어)


[image-01]: https://sa0blogs.blob.core.windows.net/aliencube/2022/08/lift-and-shift-existing-chrome-extension-to-blazor-wasm-3-01.png
[image-02]: https://sa0blogs.blob.core.windows.net/aliencube/2022/08/lift-and-shift-existing-chrome-extension-to-blazor-wasm-3-02.png
[image-03]: https://sa0blogs.blob.core.windows.net/aliencube/2022/08/lift-and-shift-existing-chrome-extension-to-blazor-wasm-3-03.png
[image-04]: https://sa0blogs.blob.core.windows.net/aliencube/2022/08/lift-and-shift-existing-chrome-extension-to-blazor-wasm-3-04.png


[post 1]: /ko/2022/07/08/lift-and-shift-existing-chrome-extension-to-blazor-wasm/
[post 2]: /ko/2022/07/20/lift-and-shift-existing-chrome-extension-to-blazor-wasm-2/
[post 3]: /ko/2022/08/31/lift-and-shift-existing-chrome-extension-to-blazor-wasm-3/

[gh sample]: https://github.com/devkimchi/blazor-wasm-chrome-extension/tree/cross-browsers
[gh sample v2 blazor]: https://github.com/devkimchi/blazor-wasm-chrome-extension/tree/cross-browsers/src/ChromeExtensionV2

[chrome extension]: https://developer.chrome.com/docs/extensions/
[chrome extension v2]: https://developer.chrome.com/docs/extensions/mv2/getstarted/
[chrome extension declarativecontent]: https://developer.chrome.com/docs/extensions/reference/declarativeContent/

[blazor]: https://dotnet.microsoft.com/apps/aspnet/web-apps/blazor?WT.mc_id=dotnet-70466-juyoo
[blazor tutorial]: https://dotnet.microsoft.com/learn/aspnet/blazor-tutorial/intro?WT.mc_id=dotnet-70466-juyoo
[blazor learn]: https://docs.microsoft.com/ko-kr/learn/paths/build-web-apps-with-blazor/?WT.mc_id=dotnet-70466-juyoo
[blazor wasm]: https://docs.microsoft.com/ko-kr/aspnet/core/blazor/host-and-deploy/webassembly?view=aspnetcore-6.0&WT.mc_id=dotnet-70466-juyoo
[blazor wasm jsinterop]: https://docs.microsoft.com/ko-kr/aspnet/core/blazor/javascript-interoperability/?view=aspnetcore-6.0&WT.mc_id=dotnet-70466-juyoo

[w3]: https://www.w3.org/
[w3 browserext]: https://www.w3.org/community/browserext/

[mozilla promise]: https://developer.mozilla.org/docs/Web/JavaScript/Guide/Using_promises
[mozilla browserext]: https://developer.mozilla.org/docs/Mozilla/Add-ons/WebExtensions
[mozilla browserext polyfill]: https://github.com/mozilla/webextension-polyfill
[mozilla browserext polyfill unpkg]: https://unpkg.com/browse/webextension-polyfill/dist/
