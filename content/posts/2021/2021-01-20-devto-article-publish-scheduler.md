---
title: "Dev.To 웹사이트에 블로그 포스트 예약 발행해주는 도구 만들어 보기"
slug: devto-article-publish-scheduler
description: "이 포스트에서는 애저 듀어러블 펑션을 이용해서 Dev.To 웹사이트에 블로그 포스트를 예약 발행하는 방법에 대해 알아봅니다."
date: "2021-01-20"
author: Justin-Yoo
tags:
- azure-functions
- azure-durable-functions
- devto
- frontmatter
cover: /2021/01/devto-article-publish-scheduler-00-ko.png
fullscreen: true
---

직장 동료인 [Todd][todd]가 예전에 만들어서 현재 잘 사용하고 있는 [Dev.To 블로그 포스트 예약 발행 도구][todd publishtodev]가 있다. 이 앱은 자바스크립트로 만들어져 있어서, 개인적으로 이를 닷넷으로 다시 만들어 보고 싶었다. 가장 큰 이유는

* 웹 페이지 스크래핑
* [Dev.To API][devto api] 사용법
* 프론트매터 다루기
* [애저 듀어러블 펑션][az fncapp durable]

정도를 연습해 보고 싶었기 때문이다. 실제로 이 포스트를 통해 어떻게 구현을 했는지 하나하나 짚어가기로 하자.

> 실제 구현한 애플리케이션의 전체 소스 코드는 [이곳][gh sample]에서 확인해 보도록 하자.


## 웹 페이지 스크래핑 ##

[Dev.To][devto] 사이트에서 블로그를 작성하면 임시로 접근할 수 있는 프리뷰 페이지 URL을 준다. URL의 모양은 대략 `https://dev.to/<username>/xxxx-****-temp-slug-xxxx?preview=xxxx`와 같이 생겼는데, 이 URL로 들어가서 HTML 문서보기를 선택한 후에 아래와 같이 `id="article-body"`를 포함한 HTML 엘리먼트를 찾는다.

![Dev.To 페이지 HTML 문서 보기][image-01]

위 그림에 보면 `data-article-id` 속성값이 보이는데, 바로 이 값이 내 블로그 포스트의 고유 ID 값이다.

요즘은 [Puppeteer][pptr] 혹은 [Playwright][pwr] 같은 헤드리스 브라우저를 사용하면 코딩을 통해 손쉽게 웹 페이지 스크래핑을 할 수 있다. 또한 이 둘은 각각 [Puppeteer Sharp][pptr csharp]과 [Playwright Sharp][pwr csharp] 같이 닷넷 버전으로도 라이브러리를 제공하고 있어서 닷넷 개발자 역시도 더더욱 손쉽게 사용할 수 있다. 그런데, 문제는 이 라이브러리가 애저 펑션에서는 예상한 것과 같이 작동하지 않는다. [node 버전][pwr node]을 사용하게 되면 [이 문서][pwr anthony]를 참조해서 애저 펑션에서도 작동하게 할 수 있겠지만, 닷넷 버전의 경우에는 로컬 개발 환경에서는 제대로 작동을 하는 반면, 애저로 배포를 했을 때 아직 제대로 작동시키는 방법을 찾지 못했다. *이 부분은 좀 더 연구를 해 봐야 할 듯 하다.*

따라서, 아쉽지만 좀 더 전통적인 방법을 통해 웹 페이지를 스크래핑하는 방식으로 전환했다. 아래와 같은 형식으로 [`HttpClient`][dotnet core httpclient]와 정규표현식을 이용했다 (line #1-2, 8).

https://gist.github.com/justinyoo/e94e83b82d1fc5851935bde157eb7d71?file=01-scraping-article-id.cs&highlights=1-2,8

위와 같이 내 블로그 포스트의 고유 ID 값을 알아내면 이제 다음 단계로 넘어갈 차례이다.


## Dev.To API 래퍼 라이브러리 ##

[Dev.To][devto]는 개발자 커뮤니티를 위한 블로그 발행 플랫폼이다. 개발과 관련해서 꽤 다양한 주제로 하루에도 수백개씩 글이 올라오는데, 블로그 글을 발행하기 위해서 굳이 웹사이트를 방문해서 글을 쓸 필요도 없이 API 호출로도 충분히 가능하게끔 플랫폼 구성이 잘 되어 있다. 그만큼 [API 문서화][devto api]도 잘 되어 있는 셈이다. 이 문서 페이지에 가 보면 Open API 문서도 제공하고 있기 때문에 손쉽게 래퍼 SDK를 만들 수 있었다.


### AutoRest로 래퍼 SDK 만들기 ###

Open API 문서만 있다면 [AutoRest][autorest]를 이용해서 굉장히 손쉽게 래퍼 SDK를 만들 수 있다. 여기서는 닷넷 버전의 SDK를 만들 예정이므로 아래와 같은 명령어를 사용하면 SDK가 만들어진다. 네임스페이스를 `Aliencube.Forem.DevTo`로 하고, SDK가 만들어지는 디렉토리를 `output`으로 설정했다. 마지막의 `--v3` 옵션은 Open API 문서가 v3 사양을 따라 만들어졌다는 것을 가리킨다.

https://gist.github.com/justinyoo/e94e83b82d1fc5851935bde157eb7d71?file=02-generate-sdk.sh

[AutoRest][autorest]는 닷넷 이외에도 Go, Java, Python, node.js, TypeScript, Ruby, PHP 등의 언어로 SDK를 만들 수 있으므로 필요한대로 만들어 쓰면 된다. 이와 관련해서 래퍼 SDK를 저장해 놓은 리포지토리는 아래와 같다.

[https://github.com/aliencube/forem-sdk][devto api wrapper]


## 블로그 포스트 마크다운 받아오기 ##

먼저 API를 이용하기 위해서는 API 키 값을 받아와야 한다. [계정 설정 페이지][devto account]에서 아래 그림과 같이 API 키 값을 생성한다.

![Dev.To API 키][image-02]

그리고 위에 만들어 놓은 API 래퍼 SDK를 이용하면 아래와 같이 손쉽게 블로그 포스트 마크다운 데이터를 받아올 수 있다 (line #4-6).

https://gist.github.com/justinyoo/e94e83b82d1fc5851935bde157eb7d71?file=03-get-markdown.cs&highlights=4-6


## 프론트매터 수정하기 ##

Dev.To에 발행하는 모든 포스트는 프론트매터라고 불리는 블로그 포스트별 메타데이터를 마크다운 최상단에 저장한다. 프론트매터는 YAML 형식으로 되어 있는데, 프론트매터를 포함한 마크다운 문서는 대략 아래와 같은 식으로 보인다.

https://gist.github.com/justinyoo/e94e83b82d1fc5851935bde157eb7d71?file=04-frontmatter.yaml

여기서 보면 `pbulished: false`라는 부분이 보인다. 이 값을 `true`로 바꿔주고 저장하면 블로그 포스트가 발행되는 것이므로, 여기서는 이 프론트매터를 수정해야 한다. 아래 코드를 보자. 마크다운 문서에서 프론트매터를 분리해 냈다.

https://gist.github.com/justinyoo/e94e83b82d1fc5851935bde157eb7d71?file=05-extract-frontmatter.cs

다음은 문자열 형태로 되어 있는 프론트매터를 `FrontMatter`라는 강타입 인스턴스 형식으로 비직렬화했다. 여기서는 [YamlDotNet][ydn] 라이브러리를 사용해서 이 YAML 문서를 비직렬화했다. 이후 `published` 값을 `true`로 수정했다.

https://gist.github.com/justinyoo/e94e83b82d1fc5851935bde157eb7d71?file=06-deserialise-frontmatter.cs

프론트매터를 수정했으니, 이를 다시 직렬화해서 마크다운 본문과 합쳐야 한다.

https://gist.github.com/justinyoo/e94e83b82d1fc5851935bde157eb7d71?file=07-update-markdown.cs


## 블로그 포스트 마크다운 업데이트하기 ##

이제 이 수정된 마크다운 포스트를 다시 API를 통해 업데이트하면 블로그 포스트가 발행된다.

https://gist.github.com/justinyoo/e94e83b82d1fc5851935bde157eb7d71?file=08-update-article.cs

이제 기본적인 블로그 포스트 발행 절차가 완성되었다. 이제 이를 예약 발행해주는 워크플로우를 구현해 보자.


## 애저 듀어러블 펑션으로 예약 발행하기 ##

[애저 듀어러블 펑션][az fncapp durable]은 크게 세 펑션의 조합이라고 생각하면 좋다. [API 엔드포인트 펑션][az fncapp durable client], [오케스트레이션 펑션][az fncapp durable orchestrator], [액티비티 펑션][az fncapp durable activity]이 각각의 역할을 담당하고 있는데, 대략 아래와 같은 형식으로 운영된다고 생각하면 쉽다.

1. 사용자는 HTTP API 요청을 엔드포인트 펑션으로 보내고, 엔드포인트 펑션은 전체 워크플로우를 관장하는 오케스트레이션 펑션을 호출한 후 202 응답을 반환한다.
2. 오케스트레이션 펑션은 언제, 어느 시점에 액티비티 펑션을 호출해야 하는지 결정하고, 이에 맞춰 개별 액티비티 펑션을 호출한다.
3. 개별 액티비티 펑션은 주어진 역할을 수행하고 그 결과를 오케스트레이션 펑션과 공유한다.

![애저 듀어러블 펑션 작동 구조][image-03]

오케스트레이션 펑션에는 개별 액티비티 펑션 제어를 위한 방법 중 하나로 [타이머 기능][az fncapp durable timer]을 구현할 수 있는데, 이를 통해 스케줄링이 가능하다. 즉, 블로그 포스트를 임시 저장해 둔 상태로 두고, 내가 원하는 시간에 자동으로 발행이 되게끔 예약 발행 기능을 구현하는 것이다.


### API 엔드포인트 펑션 ###

애저 듀어러블 펑션 전체 워크플로우에서 외부로 공개된 유일한 부분이 바로 이 [API 엔드포인트 펑션][az fncapp durable client]이다. 엔드포인트 펑션은 기본적으로 [HTTP 트리거 펑션][az fncapp trigger http]과 같지만, [듀어러블 펑션 바인딩][az fncapp durable binding] 파라미터가 하나 추가되어 있다 (line #4).

https://gist.github.com/justinyoo/e94e83b82d1fc5851935bde157eb7d71?file=09-api-endpoint-function-1.cs&highlights=4

엔드포인트 펑션이 하는 일을 간단하게 정리하면 아래와 같다.

1. 외부에서 요청 개체와 함께 공개된 엔드포인트로 API를 호출한다. 요청 개체의 형식은 아래와 같다. 여기서 `schedule` 값은 [ISO8601][iso8601] 형식(예: `2021-01-20T07:30:00+09:00`)을 따른다.

    https://gist.github.com/justinyoo/e94e83b82d1fc5851935bde157eb7d71?file=10-api-request-payload.json

2. 요청 개체를 비직렬화한다.

    https://gist.github.com/justinyoo/e94e83b82d1fc5851935bde157eb7d71?file=09-api-endpoint-function-2.cs

3. 오케스트레이션 펑션 인스턴스를 생성한다. 이 때 앞서 비직렬화한 요청 개체를 오케스트레이션 펑션으로 보낸다.

    https://gist.github.com/justinyoo/e94e83b82d1fc5851935bde157eb7d71?file=09-api-endpoint-function-3.cs

4. 오케스트레이션 펑션은 비동기식으로 동작하므로 요청자에게 202 응답을 반환한다.

    https://gist.github.com/justinyoo/e94e83b82d1fc5851935bde157eb7d71?file=09-api-endpoint-function-4.cs


### 오케스트레이션 펑션 ###

[오케스트레이션 펑션][az fncapp durable orchestrator]은 전체 워크플로우를 제어한다. 아래는 오케스트레이션 펑션에 쓰이는 바인딩이다 (line #3).

https://gist.github.com/justinyoo/e94e83b82d1fc5851935bde157eb7d71?file=10-orchestraction-function-1.cs&highlights=3

이 `IDurableOrchestrationContext` 인스턴스를 통해 앞서 엔드포인트 펑션에서 보내온 요청 개체를 받아올 수 있다.

https://gist.github.com/justinyoo/e94e83b82d1fc5851935bde157eb7d71?file=10-orchestraction-function-2.cs

이 요청 개체 안에 들어있는 예약 발행일을 이용해 타이머를 활성화 시킨다.

https://gist.github.com/justinyoo/e94e83b82d1fc5851935bde157eb7d71?file=10-orchestraction-function-3.cs

위와 같이 타이머가 활성화가 되면, 이 오케스트레이션 펑션은 타이머가 종료되는 시점까지 활동을 멈춘다. 타이머가 종료되면 다시 오케스트레이션 펑션이 실행되면서 그 안에서 아래 액티비티 펑션을 호출한다.

https://gist.github.com/justinyoo/e94e83b82d1fc5851935bde157eb7d71?file=10-orchestraction-function-4.cs

마지막으로 액티비티 펑션이 실행시킨 결과를 반환한다.

https://gist.github.com/justinyoo/e94e83b82d1fc5851935bde157eb7d71?file=10-orchestraction-function-5.cs


## 액티비티 펑션 ##

앞서 실행시켰던 엔드포인트 펑션, 오케스트레이션 펑션은 실제로 블로그 포스트 자체를 다루지는 않았다. 실제로 웹 페이지 스크래핑 및 DevTo API 호출 및 마크다운 문서 업데이트는 바로 이 [액티비티 펑션][az fncapp durable activity]을 통해 이루어진다. 아래는 액티비티 펑션에 사용되는 바인딩이다 (line #3).

https://gist.github.com/justinyoo/e94e83b82d1fc5851935bde157eb7d71?file=11-activity-function-1.cs&highlights=3

그리고 이 펑션 안에 앞서 언급한 웹 페이지 스크래핑, DevTo API 호출, 마크다운 핸들링 등과 같은 코드를 추가하면 된다.

https://gist.github.com/justinyoo/e94e83b82d1fc5851935bde157eb7d71?file=11-activity-function-2-ko.cs

마지막으로 처리 결과를 아래와 같이 반환한다.

https://gist.github.com/justinyoo/e94e83b82d1fc5851935bde157eb7d71?file=11-activity-function-3.cs

---

지금까지 [애저 듀어러블 펑션][az fncapp durable]을 이용해 Dev.To 플랫폼에 블로그 포스트를 예약 발행하는 도구를 구현해 보았다. 이를 통해 애저 듀어러블 펑션의 핵심적인 워크플로우인 API 호출 ➡️ 오케스트레이션 ➡️ 액티비티의 세 가지 단계를 살펴보았다. 이 워크플로우를 이용하면 서버리스 펑션의 제약사항 중 하나인 Stateless를 해결하고 Stateful한 서버리스 기능을 구현할 수 있는데, 애저 듀어러블 펑션의 강력함과 편리함을 느껴볼 수 있기를 바란다.


[image-01]: https://user-images.githubusercontent.com/1538528/89647854-3a9c4e80-d8f9-11ea-8415-a71aac67168c.png
[image-02]: /2021/01/devto-article-publish-scheduler-02.png
[image-03]: /2021/01/devto-article-publish-scheduler-03-ko.png

[todd]: https://twitter.com/toddanglin
[todd publishtodev]: https://github.com/toddanglin/PublishToDev

[gh sample]: https://github.com/devrel-kr/devto-article-publish-scheduler

[pptr]: https://pptr.dev/
[pptr csharp]: https://www.puppeteersharp.com/

[pwr]: https://playwright.dev/
[pwr node]: https://www.npmjs.com/package/playwright
[pwr csharp]: https://github.com/microsoft/playwright-sharp
[pwr anthony]: https://anthonychu.ca/post/azure-functions-headless-chromium-puppeteer-playwright/

[dotnet core httpclient]: https://docs.microsoft.com/ko-kr/dotnet/api/system.net.http.httpclient?view=netcore-3.1&WT.mc_id=dotnet-12868-juyoo

[devto]: https://dev.to/
[devto account]: https://dev.to/settings/account
[devto api]: https://docs.forem.com/api/
[devto api wrapper]: https://github.com/aliencube/forem-sdk

[autorest]: https://github.com/Azure/autorest

[ydn]: https://github.com/aaubry/YamlDotNet

[iso8601]: https://ko.wikipedia.org/wiki/ISO_8601

[az fncapp]: https://docs.microsoft.com/ko-kr/azure/azure-functions/functions-overview?WT.mc_id=dotnet-12868-juyoo
[az fncapp trigger http]: https://docs.microsoft.com/ko-kr/azure/azure-functions/functions-bindings-http-webhook-trigger?tabs=csharp&WT.mc_id=dotnet-12868-juyoo
[az fncapp durable]: https://docs.microsoft.com/ko-kr/azure/azure-functions/durable/durable-functions-overview?tabs=csharp&WT.mc_id=dotnet-12868-juyoo
[az fncapp durable timer]: https://docs.microsoft.com/ko-kr/azure/azure-functions/durable/durable-functions-timers?tabs=csharp&WT.mc_id=dotnet-12868-juyoo
[az fncapp durable binding]: https://docs.microsoft.com/ko-kr/azure/azure-functions/durable/durable-functions-bindings?WT.mc_id=dotnet-12868-juyoo
[az fncapp durable query]: https://docs.microsoft.com/ko-kr/azure/azure-functions/durable/durable-functions-instance-management?tabs=csharp&WT.mc_id=dotnet-12868-juyoo
[az fncapp durable client]: https://docs.microsoft.com/ko-kr/azure/azure-functions/durable/durable-functions-types-features-overview?WT.mc_id=dotnet-12868-juyoo#client-functions
[az fncapp durable orchestrator]: https://docs.microsoft.com/ko-kr/azure/azure-functions/durable/durable-functions-types-features-overview?WT.mc_id=dotnet-12868-juyoo#orchestrator-functions
[az fncapp durable activity]: https://docs.microsoft.com/ko-kr/azure/azure-functions/durable/durable-functions-types-features-overview?WT.mc_id=dotnet-12868-juyoo#activity-functions
