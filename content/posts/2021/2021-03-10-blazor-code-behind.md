---
title: "블레이저 앱 UI에서 코드 분리하기"
slug: blazor-code-behind
description: "이 포스트에서는 블레이저 앱을 만들 때, UI와 코드를 분리하는 방법에 대해 알아봅니다."
date: "2021-03-10"
author: Justin-Yoo
tags:
- blazor-server
- blazor-webassembly
- code-behind
- separate-of-concerns
cover: /2021/03/blazor-code-behind-00.png
fullscreen: true
---

기존에 운영하던 [ASP.NET WebForm][aspnet webform] 앱이 있다고 가정해 보자. [닷넷 프레임워크][dotnet framework] 전용이기도 하고 더이상 새로운 기능이 추가되지 않을 예정이므로 마이크로소프트에서는 [.NET 5][dotnet 5]로 이전을 권장하고 있다. 하지만, 웹폼으로 만들어진 앱을 .NET 5로 이전하기 위해서는 어떻게 해야 할까? 여러 대안이 있지만, 웹폼이 사용하는 방식과 비슷한 블레이저 앱으로 이전하는 것이 가장 현실적인 대안이 될 수 있다.

[블레이저 앱][blazor]을 처음 생성했을 때 자동으로 만들어주는 `Counter.razor` 파일을 보면 대략 아래와 같이 생겼다.

https://gist.github.com/justinyoo/adf1539f817c4b604c5cedc421e96aed?file=01-counter.razor

위 코드에 보이다시피 HTML 부분과 코드 부분이 하나의 `.razor` 파일 안에 공존하고 있다. 애플리케이션이 잘 작동하기만 하면 크게 문제가 되지는 않는 부분이지만, 점점 비지니스 요구사항이 복잡해지고 애플리케이션 개발 규모가 커지다 보면, HTML 부분과 코드 부분을 분리해서 개발하는 것이 여러모로 편리하다. 실제로 현대적인 개발 방법론에서도 [문제의 분리 원칙][oop soc]을 적용시키는 것을 권장하고 있다. 이 포스트에서는 블레이저 앱 개발시 HTML과 코드를 분리하는 방법에 대해 알아보기로 한다.

> * 이 포스트는 [블레이저 구성요소 Partial 클래스 지원][blazor components partial] 문서를 바탕으로 한다.
> * 이 포스트에서 블레이저라고 언급하는 경우 [블레이저 서버][blazor server]와 [블레이저 웹 어셈블리][blazor wasm] 앱 모두를 가리킨다. 필요할 경우, 서버와 웹 어셈블리 앱을 명시적으로 언급할 것이다.
> * 이 포스트에서 사용한 샘플 코드는 이곳 [깃헙 리포지토리][gh sample]에서 다운로드 받을 수 있다.


## Partial 클래스 만들기 ##

기본적으로 모든 `.razor` 파일은 그 자체로 하나의 클래스가 된다. 즉, 앞서 언급한 `Counter.razor` 파일은 컴파일시 `Counter` 클래스가 된다고 보면 된다. 따라서, HTML과 코드를 분리하기 위해서는 새로운 클래스를 만들 때 아래와 같이 [`partial` 한정자][dotnet partial]를 붙여줘야 한다.

https://gist.github.com/justinyoo/adf1539f817c4b604c5cedc421e96aed?file=02-counter.cs

그리고 위의 `@code` 블록 안에 있는 코드 부분을 방금 생성한 `Counter` 클래스로 옮겨 놓는다.

https://gist.github.com/justinyoo/adf1539f817c4b604c5cedc421e96aed?file=03-counter.cs

이렇게 한 후 다시 컴파일하고 실행을 시켜서 아무 문제 없이 실행되는 것을 확인해 보자.


## 컴포넌트 모델 상속하기 ##

위와 같이 `partial` 한정자를 이용해 사용할 수도 있지만, 블레이저가 자체적으로 제공하는 컴포넌트 모델을 사용할 수도 있다. 이렇게 할 경우 컴포넌트 모델에서 제공하는 다양한 속성과 이벤트들을 활용할 수 있다. 아래와 같이 `ComponentBase` 클래스를 상속 받아 `CounterBase` 클래스를 생성한다.

https://gist.github.com/justinyoo/adf1539f817c4b604c5cedc421e96aed?file=04-counterbase.cs

이렇게 한 후 원래 `Counter.razor` 파일에 `@inherits` 지시자를 아래와 같이 추가한다.

https://gist.github.com/justinyoo/adf1539f817c4b604c5cedc421e96aed?file=05-counter.razor

그런데, 아래와 같이 `@currentCount`와 `IncrementCount`에서 오류가 발생한 것이 보일 것이다.

![Counter.razor 에러][image-01]

이는 `CounterBase` 클래스의 `currentCount` 필드와 `IncrementCount()` 메소드가 `private` 스코프로 되어 있기 때문이다. 따라서, 이를 적어도 `protected` 수준까지는 해 놓아야 한다. 그러면 `Counter.razor` 클래스는 `CounterBase` 클래스를 상속 받아 사용하므로 필드와 메소드에 접근이 가능해진다. 위의 코드를 아래와 같이 `private`에서 `protected`로 수정한다. 그리고, 필드는 속성으로 바꾼다.

https://gist.github.com/justinyoo/adf1539f817c4b604c5cedc421e96aed?file=06-counterbase.cs

기왕 바꾸는 김에 한 가지를 더 추가해 보자. `ComponentBase` 클래스는 컴포넌트의 생애주기에 관여하는 다양한 메소드를 포함하고 있다. 따라서, 그 중 하나인 `OnInitializedAsync()` 메소드를 추가해서 컴포넌트를 시작할 때 필요한 것들을 정의해 보자. 이 메소드는 마치 웹폼의 `Page_Init()` 혹은 `Page_Load()`메소드와 비슷하다는 것을 느낄 수 있는가? 여기서는 `CurrentCount` 값을 `10`으로 초기화한다.

https://gist.github.com/justinyoo/adf1539f817c4b604c5cedc421e96aed?file=07-counterbase.cs

그리고, `Counter.razor` 파일도 필드를 참조하는 대신 속성을 참조하도록 아래와 같이 변경한다. 그러면 앞서 보였던 오류가 사라질 것이다.

https://gist.github.com/justinyoo/adf1539f817c4b604c5cedc421e96aed?file=08-counter.razor

이제 다시 컴파일한 후 다시 앱을 실행시켜 보자. 그러면 예상한 대로 잘 실행이 될 것이다. 또한, 10으로 초기화 된 `Current count`값이 보일 것이다.

![Counter.razor - Current count 값 10으로 초기화][image-02]


## Partial + 컴포넌트 모델 ##

위에 기술한 컴포넌트 모델의 단점이라면 `@inherits` 지시자를 이용해서 명시적으로 컴포넌트를 상속 받아야 한다는 점인데, 그렇다면 이를 피할 수 있는 방법은 없을까? 물론 있다. 바로 `partial` 한정자를 사용하면 된다. 아래와 같이 클래스에 `partial` 한정자를 붙이고 클래스 이름을 `Counter`로 바꾼다. 또한 이 `Counter` 클래스는 `ComponentBase` 클래스를 상속받게끔 설정한다.

https://gist.github.com/justinyoo/adf1539f817c4b604c5cedc421e96aed?file=09-counter.cs

그리고, `Counter.razor` 파일에서 `@inherits` 지시자를 없앤다.

https://gist.github.com/justinyoo/adf1539f817c4b604c5cedc421e96aed?file=10-counter.razor

이제 다시 컴파일한 후 앱을 실행시켜 보자. 그러면 예상한 대로 잘 실행이 될 것이다.

---

지금까지 블레이저 앱에서 HTML과 코드를 분리하는 방법에 대해 알아보았다. 사실, 이 포스트는 어떤 특별힌 트릭이나 팁을 제공한다기 보다는 효율적인 개발을 위해 코드와 UI를 분리하는 방법에 대해 기술한 것이다. 그리고, 만약 [ASP.NET WebForm][aspnet webform]에서 [블레이저][blazor]로 이전을 고려하고 있고, 기존의 개발 경험을 최대한 유지하고 싶다면 WebForm의 [코드 비하인드 모델][aspnet webform codebehind]을 블레이저에서도 거의 동일하게 적용시킬 수 있으니 적극적으로 활용해 보길 바란다.


[image-01]: /2021/03/blazor-code-behind-01.png
[image-02]: /2021/03/blazor-code-behind-02.png

[gh sample]: https://github.com/devkimchi/Blazor-Code-Behind-Sample

[oop soc]: https://docs.microsoft.com/ko-kr/dotnet/architecture/modern-web-apps-azure/architectural-principles?WT.mc_id=dotnet-19728-juyoo#separation-of-concerns

[dotnet framework]: https://dotnet.microsoft.com/download/dotnet-framework?WT.mc_id=dotnet-19728-juyoo
[dotnet 5]: https://dotnet.microsoft.com/download/dotnet/5.0?WT.mc_id=dotnet-19728-juyoo
[dotnet partial]: https://docs.microsoft.com/ko-kr/dotnet/csharp/programming-guide/classes-and-structs/partial-classes-and-methods?WT.mc_id=dotnet-19728-juyoo

[aspnet webform]: https://docs.microsoft.com/ko-kr/aspnet/web-forms/what-is-web-forms?WT.mc_id=dotnet-19728-juyoo
[aspnet webform codebehind]: https://docs.microsoft.com/ko-kr/troubleshoot/aspnet/code-behind-model?WT.mc_id=dotnet-19728-juyoo

[blazor]: https://docs.microsoft.com/ko-kr/aspnet/core/blazor/?view=aspnetcore-5.0&WT.mc_id=dotnet-19728-juyoo
[blazor server]: https://docs.microsoft.com/ko-kr/aspnet/core/blazor/hosting-models?view=aspnetcore-5.0&WT.mc_id=dotnet-19728-juyoo#blazor-server
[blazor wasm]: https://docs.microsoft.com/ko-kr/aspnet/core/blazor/hosting-models?view=aspnetcore-5.0&WT.mc_id=dotnet-19728-juyoo#blazor-webassembly
[blazor components partial]: https://docs.microsoft.com/ko-kr/aspnet/core/blazor/components/?view=aspnetcore-5.0&WT.mc_id=dotnet-19728-juyoo#partial-class-support
