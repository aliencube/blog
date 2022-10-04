---
title: "ASP.NET Core 앱에서 동일 인터페이스로 만들어진 여러 인스턴스에 대한 의존성을 주입하는 다섯 가지 방법"
slug: 5-ways-injecting-multiple-instances-of-same-interface-on-aspnet-core
description: "이 포스트에서는 ASP.NET Core 앱에서 의존성 주입을 구현할 때 동일 인터페이스를 가진 서로 다른 인스턴스를 주입하는 여러 가지 방법에 대해 알아봅니다."
date: "2020-07-01"
author: Justin-Yoo
tags:
- aspnet-core
- dependency-injection
- ioc-container
- multiple-instances
cover: /2020/07/5-ways-injecting-multiple-instances-of-same-interface-on-aspnet-core-00.png
fullscreen: true
---

[ASP.NET Core][aspnet core] 애플리케이션을 개발하다 보면 항상 접하게 되는 것이 바로 [의존성 주입을 위한 IoC 컨테이너 설정이다][aspnet core di]. 자체적으로 제공하는 IoC 컨테이너를 이용하면 별도의 써드파티 라이브러리를 이용하지 않고서도 충분히 구현이 가능한데, 이 포스트에서는 그 중에서 하나의 인터페이스를 공유하는 다양한 인스턴스를 주입했을 때 그 중 하나를 선택해야 하는 경우에 대해 알아보기로 하자.


## 인터페이스 및 인스턴스 구현 ##

아래와 같은 식으로 인터페이스와 인스턴스를 구현한다고 가정하자. `IFeedReader` 인터페이스를 세 개의 다른 클리스, `BlogFeedReader`, `PodcastFeedReader`, `YouTubeFeedReader`로 구현했다 (line #1, 7, 22, 37).

https://gist.github.com/justinyoo/e5b18f907f6a1b22614c4e487cfe185f?file=01-feed-reader.cs&highlights=1,7,22,37


## IoC 컨테이너 등록 #1 &ndash; 콜렉션 이용하기 ##

위와 같이 정의한 클라스를 이제 아래 `Startup.cs`의 `ConfigureServices()` 메소드에 등록시켜 보자.

https://gist.github.com/justinyoo/e5b18f907f6a1b22614c4e487cfe185f?file=02-configure-services.cs

이렇게 하면 이 세 인스턴스가 모두 `IFeedReader`로 등록이 되긴 한다. 그런데, 이를 사용하는 클라스 입장에서는 어떤 인스턴스를 써야 할지 모르기 때문에 이럴 경우에는 아래와 같이 해주면 좋다. 즉, 모든 `IFeedReader` 인스턴스들을 `IEnumerable<IFeedReader>` 형태의 콜렉션으로 받아들인 후에 필요한 것만 필터링해서 가져오는 방법이다 (line #7).

https://gist.github.com/justinyoo/e5b18f907f6a1b22614c4e487cfe185f?file=03-feed-service.cs&highlights=7

이런 식으로 `IFeedReader` 인스턴스의 콜렉션을 사용할 경우에는 위와 같은 방식으로 해도 좋지만, 아래와 같은 방식으로 루프를 돌려서 해결해도 된다 (line #12). 이 경우는 [비지터 패턴][design pattern visitor] 혹은 [이터레이터 패턴][design pattern iterator]을 구현할 때 유용하다.

https://gist.github.com/justinyoo/e5b18f907f6a1b22614c4e487cfe185f?file=04-feed-service.cs&highlights=12


## IoC 컨테이너 등록 #2 &ndash; 해결자(Resolver) 이용하기 ##

첫번째 방법과 비슷한데, 이번에는 해결자(Resolver) 인스턴스를 사용해서 가져오는 방법을 살펴보자. 우선 아래와 같이 `IFeedReaderResolver` 인터페이스와 `FeedReaderResolver` 클라스를 정의한다 (line #1, 6). 이 때 `IServiceProvider`로 표현되는 인스턴스를 의존성으로 주입하는 것을 눈여겨 보자. 이 인스턴스는 ASP.NET Core의 IoC 컨테이너에서 사용하는 것으로 이를 통하면 모든 의존성 객체들에 접근할 수 있다. 또한 이 방식에서는 컨벤션을 이용해서 인스턴스를 가져오기 때문에 앞서 정의한 `Name` 속성은 더이상 필요없다 (line 17-18).

https://gist.github.com/justinyoo/e5b18f907f6a1b22614c4e487cfe185f?file=05-feed-reader-resolver.cs&highlights=1,6,17-18

이렇게 한 후 다시 `Startup.cs` 파일의 `ConfigureServices()` 메소드를 아래와 같이 수정한다. 이 때 기존의 `xxxFeedReader` 인스턴스는 인터페이스 기반이 아닌 실제 구현체 기반으로 의존성을 등록한다. 그렇게 해도 실제 해결자 클라스에서 알아서 `IFeedReader` 형태로 변환해서 반환하기 때문에 상관 없다.

https://gist.github.com/justinyoo/e5b18f907f6a1b22614c4e487cfe185f?file=06-configure-services.cs

마지막으로 `BlogFeedService` 클라스를 아래와 같이 수정한다 (line #5).

https://gist.github.com/justinyoo/e5b18f907f6a1b22614c4e487cfe185f?file=07-feed-service.cs&highlights=5


## IoC 컨테이너 등록 #3 &ndash; 해결자(Resolver) + 팩토리 메소드 패턴 이용하기 ##

이번에는 해결자 클라스를 좀 더 변형해서 팩토리 메소드 패턴을 구현해 보자. 인터페이스는 동일하지만, 이번에는 `IServiceProvider` 인스턴스에 대한 의존성을 없앴다. 대신 `Activator.CreateInstance()` 메소드를 통해 직접 인스턴스를 생성한다 (line #5-6).

https://gist.github.com/justinyoo/e5b18f907f6a1b22614c4e487cfe185f?file=08-feed-reader-resolver.cs&highlights=5-6

위와 같이 해결자 클라스를 정의한다면 굳이 기존의 `xxxFeedReader` 인스턴스들을 IoC 컨테이너에 등록시킬 필요가 없고 `IFeedReaderResolver` 하나만으로도 충분하다. 다만, 이 때에는 `xxxFeedReader` 인스턴스들을 싱글톤 형태로는 등록할 수 없다는 점을 고려하자.

https://gist.github.com/justinyoo/e5b18f907f6a1b22614c4e487cfe185f?file=09-configure-services.cs


## IoC 컨테이너 등록 #4 &ndash; 명시적 대리자(Delegate) 이용하기 ##

앞서 해결자(Resolver) 클라스를 이용했지만, 이 부분을 명시적 [대리자(Delegate) 형태][dotnet delegates]로 바꿔서 사용할 수도 있다. 아래 코드를 살펴보자. 가장 먼저 `Startup.cs` 파일 안에서 `Startup` 클라스 바깥쪽에 대리자 선언을 한다.

https://gist.github.com/justinyoo/e5b18f907f6a1b22614c4e487cfe185f?file=10-feed-reader-delegate.cs

그리고 난 후 `ConfigureServices()` 메소드를 아래와 같이 수정한다. 대리자는 메소드 형태만 정의를 했기 때문에 실제 구현을 아래와 같이 해 줘야 한다. 구현 로직은 앞서와 별반 다르지 않다 (line #9-10).

https://gist.github.com/justinyoo/e5b18f907f6a1b22614c4e487cfe185f?file=11-configure-services.cs&highlights=9-10

실제 이를 활용하려는 `BlogFeedService` 클라스는 아래와 같이 수정한다 (line #5). `FeedReaderDelegate`가 반환하는 객체는 `IFeedReader` 인스턴스이므로, 실제 원하는 결과를 얻기 위해서는 메소드 체이닝을 통해 한번 더 메소드를 호출해야 한다 (line #12).

https://gist.github.com/justinyoo/e5b18f907f6a1b22614c4e487cfe185f?file=12-feed-service.cs&highlights=5,12


## IoC 컨테이너 등록 #5 &ndash; 람다 펑션을 이용한 암시적 대리자(Delegate) 이용하기 ##

이번에는 명시적으로 대리자를 선언하는 대신 암시적으로 람다 펑션을 이용해 보자. 아래와 같이 `ConfigureServices()` 메소드를 수정한다. 대리자 선언이 없었기 때문에 의존성 자체를 아예 람다 펑션으로 만들어 버렸다 (line #7-13).

https://gist.github.com/justinyoo/e5b18f907f6a1b22614c4e487cfe185f?file=13-configure-services.cs&highlights=7-13

이렇게 한 후 실제 `BlogFeedService` 역시도 람다 펑션을 의존성으로 주입해 줘야 한다 (line #5). 위의 경우와 모양은 거의 비슷하지만 대리자 대신 람다 펑션을 의존성 객체로 주입한다는 점이 다르다.

https://gist.github.com/justinyoo/e5b18f907f6a1b22614c4e487cfe185f?file=14-feed-service.cs&highlights=5

---

지금까지 [ASP.NET Core][aspnet core] 앱에서 의존성 객체를 주입할 때 동일한 인터페이스로 다양한 구현체가 있을 경우 선택적으로 구현체를 받아서 사용하는 방법에 대해 알아보았다. 다섯 가지 방법이 거의 다 비슷하면서도 다른데, 어느 방식이 더 낫다 아니다를 말하기는 어려울 듯 하고, 다만 개발자가 선호하는 방식에 따라 한 가지를 취하지 않을까 한다.

[dotnet delegates]: https://docs.microsoft.com/ko-kr/dotnet/csharp/programming-guide/delegates/?WT.mc_id=aliencubeorg-blog-juyoo

[aspnet core]: https://docs.microsoft.com/ko-kr/aspnet/core/?view=aspnetcore-3.1&WT.mc_id=aliencubeorg-blog-juyoo
[aspnet core di]: https://docs.microsoft.com/ko-kr/aspnet/core/fundamentals/dependency-injection?view=aspnetcore-3.1&WT.mc_id=aliencubeorg-blog-juyoo

[design pattern visitor]: https://www.dofactory.com/net/visitor-design-pattern
[design pattern iterator]: https://www.dofactory.com/net/iterator-design-pattern
