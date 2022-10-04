---
title: "깃헙 페이지에서 블레이저 웹어셈블리의 404 페이지 출력하기"
slug: 404-page-of-blazor-wasm-on-gh-pages
description: "이 포스트에서는 깃헙 페이지로 블레이저 웹어셈블리 앱을 배포할 경우 깃헙 고유의 404 페이지 대신 블레이저 웹어셈블리의 404 페이지를 출력하는 방법에 대해 알아봅니다."
date: "2022-10-05"
author: Justin-Yoo
tags:
- dotnet
- blazor-wasm
- gh-pages
- 404-page
cover: /2022/10/404-page-of-blazor-wasm-on-gh-pages-00.png
fullscreen: true
---

[블레이저 웹어셈블리][blazor wasm] 앱을 개발하다 보면 자체적으로 404 페이지를 구현한다. 그런데, 이 블레이저 웹어셈블리 앱을 [깃헙 페이지][gh pages]로 배포하면 블레이저의 404 페이지가 아닌 깃헙의 404 페이지를 보여준다. 이를 해결할 수 있는 다양한 방식이 있지만, 여기서는 커스텀 페이지를 이용해 아주 간단하게 구현해 보기로 한다.

> 이 포스트에 사용한 샘플 애플리케이션은 [이곳][gh sample]에서 다운로드 받을 수 있다.


## 블레이저 웹어셈블리 &ndash; 기본 404 페이지 ##

블레이저 웹어셈블리 앱 프로젝트를 생성한 후에 기본으로 보여주는 404 페이지는 아래와 같다.

![Default 404 page][image-01]

이는 기본적으로 `App.razor` 파일에서 아래와 같이 정의하고 있기 때문이다.

```razor
@* BlazorApp1.App.razor *@

<Router AppAssembly="@typeof(App).Assembly">
    ...
    <NotFound>
        <PageTitle>Not found</PageTitle>
        <LayoutView Layout="@typeof(MainLayout)">
            @* ⬇️⬇️⬇️ This is the content ⬇️⬇️⬇️*@
            <p role="alert">Sorry, there's nothing at this address.</p>
            @* ⬆️⬆️⬆️ This is the content ⬆️⬆️⬆️*@
        </LayoutView>
    </NotFound>
</Router>
```

그렇다면, 이를 좀 더 상황에 맞게 커스터마이징을 해 보기로 하자.


## 블레이저 웹어셈블리 &ndash; 커스텀 404 페이지 ##

이 포스트에서 사용한 샘플 앱은 좀 더 상황에 맞게 커스터마이징을 했다. 그러기 위해 아래와 같이 `NotFound.razor` 라는 이름의 컴포넌트를 만든다.

```razor
@* Fitability.Home.Components.NotFound.razor *@

<div class="row">
    <div class="col-6">
        <img src="images/banner-3840x1920.png" class="img-fluid" alt="banner" />
    </div>
    <div class="col-6 d-flex align-items-center">
        <div class="row">
            <div class="col-12">
                <h1 class="text-center">Not Found!</h1>
                <p class="text-center fs-2">Your requested page doesn't exist.</p>
                <p class="text-center fs-3"><a href="/" class="btn btn-primary btn-lg">Home</a></p>
            </div>
        </div>
    </div>
</div>
```

그리고, 이 컴포넌트를 아래와 같이 `App.razor` 페이지의 `LayoutView` 컴포넌트 안에서 호출한다.

```razor
@* Fitability.Home.App.razor *@

<Router AppAssembly="@typeof(App).Assembly">
    ...
    <NotFound>
        <LayoutView Layout="@typeof(MainLayout)">
            @* ⬇️⬇️⬇️ Add the NotFound component here ⬇️⬇️⬇️*@
            <NotFound />
            @* ⬆️⬆️⬆️ Add the NotFound component here ⬆️⬆️⬆️*@
        </LayoutView>
    </NotFound>
</Router>
```

이렇게 한 후 앱을 실행시켜 보면 아래와 같이 404 페이지가 나타난다.

![Custom 404 page][image-02]


## 깃헙 페이지 &ndash; 기본 404 페이지 ##

그렇다면, 이렇게 만들어진 블레이저 웹어셈블리 앱을 깃헙 페이지로 배포해 보자. 그러면 아래와 같이 깃헙의 기본 404 페이지가 보인다.

![GitHub default 404 page][image-03]

이는 깃헙의 경우 존재하지 않는 URL로 접근할 경우 자체 404 페이지를 출력하게끔 설정되어 있기 때문이다. 이제 이를 커스텀 404 페이지로 바꿔보자.


## 깃헙 페이지 &ndash; 커스텀 404 페이지 ##

깃헙 페이지에서 커스텀 404 페이지를 보여주기 위해서는 물리적인 `404.html` 파일이 존재해야 한다. 이를 위해 하드코딩을 해 둔 `404.html` 페이지를 보여줘도 좋지만, 여기서는 블레이저 웹어셈블리의 라우팅 파이프라인 안에서 해결해 보기로 하자.

1. 먼저 기존의 `index.html` 파일을 `404.html` 파일로 복사한다. 이 두 파일은 결과적으로 동일한 파일이다.
2. 그리고 아래와 같이 `404.razor` 페이지를 생성한 후 앞서 만들어 둔 `NotFound` 컴포넌트를 호출한다. 이 때 이 `404.razor` 페이지가 가리키는 라우팅 URL은 `/404.html`이 된다.

    ```razor
    @* Fitability.Home.Pages.404.razor *@

    @page "/404.html"

    <NotFound />
    ```

이렇게 한 후 다시 깃헙 페이지로 배포를 하고 존재하지 않는 URL을 호출해 보자. 그러면 아래와 같이 내가 만든 커스텀 404 페이지가 보일 것이다.

![Custom 404 page on GH Pages][image-04]

깃헙 페이지는 물리적인 `404.html` 파일을 찾고, 이 파일은 블레이저 웹어셈블리의 컴포넌트 라우팅 규칙을 따라가므로 모든 페이지가 블레이저 웹어셈블리의 애플리케이션 파이프라인을 따라가는 셈이다.

---

지금까지 [블레이저 웹어셈블리][blazor wasm] 앱을 [깃헙 페이지][gh pages]로 배포할 때 커스텀 404 페이지를 다루는 방법에 대해 알아보았다.


## 블레이저 앱에 대해 더 알고 싶다면? ##

만약 블레이저 앱에 대해 좀 더 알고 싶다면 아래 튜토리얼을 참조해 보면 좋다.

* [블레이저][blazor] (영문)
* [블레이저 튜토리얼][blazor tutorial] (영문)
* [블레이저 Learn][blazor learn] (한국어)


[image-01]: /2022/10/404-page-of-blazor-wasm-on-gh-pages-01.png
[image-02]: /2022/10/404-page-of-blazor-wasm-on-gh-pages-02.png
[image-03]: /2022/10/404-page-of-blazor-wasm-on-gh-pages-03.png
[image-04]: /2022/10/404-page-of-blazor-wasm-on-gh-pages-04.png


[gh sample]: https://github.com/fitability/fitability.github.io
[gh pages]: https://pages.github.com/

[blazor]: https://dotnet.microsoft.com/apps/aspnet/web-apps/blazor?WT.mc_id=dotnet-77749-juyoo
[blazor tutorial]: https://dotnet.microsoft.com/learn/aspnet/blazor-tutorial/intro?WT.mc_id=dotnet-77749-juyoo
[blazor learn]: https://learn.microsoft.com/ko-kr/training/paths/build-web-apps-with-blazor/?WT.mc_id=dotnet-77749-juyoo
[blazor wasm]: https://learn.microsoft.com/ko-kr/aspnet/core/blazor/?WT.mc_id=dotnet-77749-juyoo#blazor-webassembly
