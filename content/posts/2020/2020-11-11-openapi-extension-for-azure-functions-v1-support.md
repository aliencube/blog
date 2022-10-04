---
title: "Open API 익스텐션에서 애저 펑션 v1 런타임 지원하기"
slug: openapi-extension-to-support-azure-functions-v1
description: "이 포스트에서는 애저 펑션 v2 이상을 지원하는 Open API 익스텐션에서 애저 펑션 v1 앱을 지원하기 위한 방법에 대해 논의해봅니다."
date: "2020-11-11"
author: Justin-Yoo
tags:
- azure-functions
- azure-functions-proxy
- openapi-extension
- backward-compatibility
cover: /2020/11/openapi-extension-to-support-azure-functions-v1-00.png
fullscreen: true
---

[애저 펑션][az func]에 [데코레이터를 추가해서 Open API 문서를 자동으로 생성][post azfunc swagger]하는 [오픈 소스][gh azfunc swagger] [라이브러리][nuget azfunc swagger]가 있다. 이 라이브러리는 v1 부터 애저 펑션의 모든 런타임 버전을 지원하지만, v1 런타임 자체가 갖는 한계 때문에 Open API 문서를 자동으로 생성하기 어려운 경우가 많다. 하지만, 언제나 방법은 있는 법. 이 포스트에서는 [애저 펑션 프록시 기능][az func proxy]을 이용해서 이를 해결하는 방법에 대해 알아보기로 한다.


## 레거시 애저 펑션 ##

레거시 엔터프라이즈 애플리케이션의 경우에는 참조 라이브러리 의존성 때문에 v1 런타임으로만 애저 펑션을 구성해야 하는 경우가 왕왕 있다. 여기 레거시 애저 펑션 엔드포인트가 아래와 같은 형태로 구성되어 있다고 가정하자.

https://gist.github.com/justinyoo/2b0b286bbe3e727e17423047cd97f86e?file=01-v1-legacy.cs

애저 펑션 런타임 v1의 경우, [Newtonsoft.Json][nuget jsonnet] 버전 9.0.1에 고정되어 있기 때문에, 예를 들어 이 `MyReturnObject` 클라스가 Newtonsoft.Json 버전 10.0.1 이상에 대한 의존성을 갖고 있다면, 이 경우에는 이 [Open API 확장 기능 라이브러리][gh azfunc swagger]를 사용할 수 없다.


## Open API 문서 정의용 애저 펑션 만들기 ##

이런 경우에는 [애저 펑션 프록시 기능][az func proxy]으로 한 번 감싸주면 완벽하지는 않더라도 동일한 개발자 경험을 제공함으로써 문제를 해결할 수 있다. 먼저 Open API 정의용 애저 펑션을 v3 런타임으로 아래와 같이 만들어 보자. 펑션 앱 이름은 `MyV1ProxyFunctionApp`으로 한다 (line #1). 그리고 기본적으로 모든 펑션 기능은 레거시 v1 앱과 동일하게 작성한다 (line #3-7). 하지만, 이 펑션은 그저 프록시 용도이므로 반환 값은 간단하게 OK 결과만 반환하게끔 한다 (line #10).

https://gist.github.com/justinyoo/2b0b286bbe3e727e17423047cd97f86e?file=02-v1-proxy.cs&highlights=1,3-7,10

이제 Open API 확장 기능 라이브러리를 설치했다면, 데코레이터를 추가할 차례이다. 아래와 같이 `FunctionName(...)` 데코레이터 위에 Open API 메타데이터 관련 데코레이터를 추가한다 (line #5-9).

https://gist.github.com/justinyoo/2b0b286bbe3e727e17423047cd97f86e?file=03-v1-proxy-openapi.cs&highlights=5-9

여기까지 하고 이 프록시 애저 펑션 앱을 실행시키면 성공적으로 Swagger UI 화면을 볼 수 있다. 하지만, 이 앱은 Swagger UI 화면만 보여줄 뿐 실제로 동작하는 화면은 아니므로 추가로 작업을 해 줘야 한다.


## 레거시 애저 펑션 프록시 추가하기 ##

아래와 같이 `proxies.json` 파일을 프로젝트 루트 폴더에 만든다. 레거시 펑션과 프록시 펑션은 동일한 엔드포인트를 갖게 만들어 놨기 때문에 (line #6,11) 사용자 입장에서는 마치 웹서버가 바뀌는 정도의 차이만 느낄 뿐 개발 경험은 동일하게 유지할 수 있다. 또한 쿼리스트링과 요청 헤더 역시도 동일하게 레거시 펑션 앱으로 전달한다 (line #13-14).

https://gist.github.com/justinyoo/2b0b286bbe3e727e17423047cd97f86e?file=04-proxies.json&highlights=6,11,13-14

프록시 펑션을 배포할 때 이 `proxies.json` 파일도 함께 배포해야 하므로 아래와 같이 `.csproj` 파일을 수정해야 한다 (line #10-12).

https://gist.github.com/justinyoo/2b0b286bbe3e727e17423047cd97f86e?file=05-v1-proxy.csproj&highlights=10-12

이렇게 한 후 이 프록시 펑션을 로컬에서 실행시켜 보거나 애저로 배포한 후 프록시 API 엔드포인트로 접속해 보면 원하는 Open API 문서도 생성할 수도 있고, 실제 레거시 API도 프록시를 통해 실행시킬 수도 있다.

---

지금까지 [애저 펑션 프록시][az func proxy] 기능을 이용해서 v1 런타임으로만 구성 가능한 레거시 애저 펑션에 [Open API 확장 기능][gh azfunc swagger]을 구현하는 방법에 대해 알아 보았다. 이 방법의 단점이라면 레거시 API를 한 번 호출할 때 프록시를 거쳐가므로 비용이 두 배로 들어간다는 것인데, 이 부분은 실제 엔터프라이즈 애플리케이션 아키텍팅의 관점에서 신중히 결정하면 될 것이다.


[post azfunc swagger]: /ko/2019/02/02/introducing-swagger-ui-on-azure-functions/

[gh azfunc swagger]: https://github.com/aliencube/AzureFunctions.Extensions

[nuget azfunc swagger]: https://www.nuget.org/packages/Aliencube.AzureFunctions.Extensions.OpenApi/
[nuget jsonnet]: https://www.nuget.org/packages/Newtonsoft.Json/

[az func]: https://docs.microsoft.com/ko-kr/azure/azure-functions/functions-overview?WT.mc_id=dotnet-10866-juyoo
[az func proxy]: https://docs.microsoft.com/ko-kr/azure/azure-functions/functions-proxies?WT.mc_id=dotnet-10866-juyoo
