---
title: "OpenAPI 인증 플로우를 이용한 애저 펑션 엔드포인트 접근 권한 관리"
slug: securing-azure-function-endpoints-via-openapi-auth
description: "이 포스트에서는 애저 펑션 엔드포인트를 OpenAPI 문서 정의를 통해 외부 접근 권한을 관리하는 방법에 대해 알아봅니다."
date: "2021-10-06"
author: Justin-Yoo
tags:
- azure-functions
- openapi
- security
- authentication
cover: https://sa0blogs.blob.core.windows.net/aliencube/2021/10/securing-azure-function-endpoints-via-openapi-auth-00.png
fullscreen: true
---


[애저 펑션][az fncapp]의 보안에 관련한 전반적인 내용은 [애저 펑션에 적용해야 하는 보안 최소요구사항][az funcapp security baseline] 페이지에 잘 기술되어 있다. 이와 더불어 애플리케이션의 API 엔드포인트 각각에 대해 접근 권한을 제어할 수도 있는데, [애저 펑션][az fncapp] 앱은 기본적으로 자체 발급한 API 키를 이용한다. 이 때 [OpenAPI 확장 기능][az fncapp extensions openapi]을 사용하면 API 키 뿐만 아니라 다양한 형태로 접근 권한을 정의할 수 있고 이를 Swagger UI 페이지를 통해 확인할 수도 있다. 이 포스트에서는 애저 펑션에서 사용할 수 있는 다양한 접근 제어 방법을 OpenAPI 확장 기능과 연동시켜 알아보기로 한다.

> 이 포스트에서 사용한 샘플 코드는 [이 깃헙 리포지토리][gh sample]에서 다운로드 받을 수 있다.


## 인증 관련 OpenAPI 스펙 ##

우선, [OpenAPI 인증 관련 스펙][openapi spec security]을 간략하게 살펴보자.

* `type`: 인증을 위한 형식이다. 현재 API key, HTTP, OAuth2, OpenID Connect 방식을 지원한다. 단, OpenAPI v2 스펙에서는 OpenID Connect 방식을 지원하지 않는다.
* `name`: 인증 키 이름이다. API Key 방식으로 사용할 때 필요하다.
* `in`: 인증 키의 위치를 지정한다. API Key 방식으로 사용할 때 필요하며, `query`, `header`, `cookie` 중 하나가 된다.
* `scheme`: 인증 방식을 지정한다. HTTP 인증 방식으로 사용할 때 필요하며, 주로 `Basic` 또는 `Bearer` 중 하나를 사용한다.
* `bearerFormat`: HTTP 인증 방식으로 Bearer 토큰 방식을 사용할 때 지정한다. 거의 대부분의 경우에서 `JWT`를 사용하면 된다.
* `flows`: OAuth2 방식으로 사용할 때 필요하다. `implicit`, `password`, `clientCredentials`, `authorizationCode` 중 하나를 사용한다.
* `openIdConnectUrl`: OpenID Connect 방식으로 사용할 때 필요하다. OpenAPI v2 스펙 지원을 위해서는 OAuth2 방식 혹은 Bearer 토큰 방식으로 대체하는 것이 좋다.

위의 내용을 바탕으로 애저 펑션의 접근 권한을 OpenAPI 확장 기능에 정의해 보기로 하자.


## 쿼리스트링에 API Key 사용 ##

애저 펑션의 기본 기능만 이용한 방법이다. 아래 코드를 한 번 보자. OpenAPI 확장 기능을 설치했다면 아래와 같이 코드를 작성하면 된다. 가장 눈여겨 봐야 할 부분은 `OpenApiSecurityAttribute(...)` 데코레이터인데, 아래와 같이 설정했다 (line #6-9).

* `Type`: `SecuritySchemeType.ApiKey`
* `In`: `OpenApiSecurityLocationType.Query`
* `Name`: `code`

https://gist.github.com/justinyoo/7a9dab5304ef36c8747a3c0cf8430b71?file=01-apikey-in-query-auth-flow-httptrigger.cs&highlights=6-9

이 상태에서 애저 펑션 앱을 실행하면 아래와 같은 Swagger UI 화면이 나타난다.

![Swagger UI - Query][image-01]

위 그림에서 자물쇠 모양을 클릭하면 아래와 같이 API Key 값을 입력하는 화면이 나오는데, 쿼리스트링의 `code` 파라미터로 API Key 값을 넘겨주는 것이 보인다.

![Swagger UI - Query - API Key][image-02]

실제로 앱을 실행시키면 아래와 같이 쿼리스트링에 `code` 파라미터가 붙은 것이 보인다.

![Swagger UI - Query - Result][image-03]


## 요청 헤더에 API Key 사용 ##

이 역시 애저 펑션의 기본 기능이다. 이번에는 `OpenApiSecurityAttribute(...)` 데코레이터를 아래와 같이 지정했다 (line #6-9)

* `Type`: `SecuritySchemeType.ApiKey`
* `In`: `OpenApiSecurityLocationType.Header`
* `Name`: `x-functions-key`

https://gist.github.com/justinyoo/7a9dab5304ef36c8747a3c0cf8430b71?file=02-apikey-in-header-auth-flow-httptrigger.cs&highlights=6-9

이 상태에서 애저 펑션 앱을 실행하면 아래와 같은 Swagger UI 화면이 나타난다.

![Swagger UI - Header][image-04]

위 그림에서 자물쇠 모양을 클릭하면 아래와 같이 API Key 값을 입력하는 화면이 나오는데, 요청 헤더의 `x-functions-key`를 통해 API Key 값을 넘겨주는 것이 보인다.

![Swagger UI - Header - API Key][image-05]

실제로 앱을 실행시키면 아래와 같이 요청 헤더를 통해 `x-functions-key`가 추가된 것이 보인다.

![Swagger UI - Header - Result][image-06]


## Basic 인증 토큰 사용 ##

이번에는 Basic 인증 토큰을 사용하는 방법에 대해 알아보자. 아래와 같이 `OpenApiSecurityAttribute(...)` 데코레이터를 설정한다 (line #6-8).

* `Type`: `SecuritySchemeType.Http`
* `Scheme`: `OpenApiSecuritySchemeType.Basic`

이는 애저 펑션의 기본 인증 기능에 더해 사용하거나 기본 인증 대신 사용하는 방법이기 때문에, 만약 기본 인증 기능을 사용하지 않으려면, `HttpTrigger` 바인딩의 AuthLevel 값을 `AuthorizationLevel.Anonymous`로 지정해 주는 것이 좋다 (line #12).

https://gist.github.com/justinyoo/7a9dab5304ef36c8747a3c0cf8430b71?file=03-http-basic-auth-flow-httptrigger.cs&highlights=6-8,12

이 상태에서 애저 펑션 앱을 실행하면 아래와 같은 Swagger UI 화면이 나타난다.

![Swagger UI - Basic Auth][image-07]

위 그림에서 자물쇠 모양을 클릭하면 아래와 같이 Username 값과 Password 값을 입력하는 화면이 나오는데, 이 값은 향후 `Authorization` 헤더에 추가된다.

![Swagger UI - Basic Auth - Details][image-08]

실제로 앱을 실행시키면 아래와 같이 요청 헤더를 통해 `Authorization` 헤더에 Base64 인코딩 된 값으로 보내진다.

![Swagger UI - Basis Auth - Result][image-09]

이렇게 보내진 토큰 값을 실제 인증 서버에 보내서 유효성 검사를 한 후 처리를 하면 된다.


## Bearer 인증 토큰 사용 ##

비슷한 방식으로 Bearer 인증 토큰을 사용하는 방법에 대해 알아보자. `OpenApiSecurityAttribute(...)` 데코레이터를 아래와 같이 지정한다 (line #5).

* `Type`: `SecuritySchemeType.Http`
* `Scheme`: `OpenApiSecuritySchemeType.Bearer`
* `BearerFormat`: `JWT`

마찬가지로 여기서도 기본 기능을 사용하지 않기 때문에 `HttpTrigger` 바인딩의 AuthLevel 값을 `AuthorizationLevel.Anonymous`로 지정했다 (line #13).

https://gist.github.com/justinyoo/7a9dab5304ef36c8747a3c0cf8430b71?file=04-http-bearer-auth-flow-httptrigger.cs&highlights=6-9,13

이 상태에서 애저 펑션 앱을 실행하면 아래와 같은 Swagger UI 화면이 나타난다.

![Swagger UI - Bearer Auth][image-10]

위 그림에서 자물쇠 모양을 클릭하면 아래와 같이 Bearer 토큰 값을 입력하는 화면이 나오는데, 이 값은 향후 `Authorization` 헤더에 추가된다.

![Swagger UI - Bearer Auth - Details][image-11]

실제로 앱을 실행시키면 아래와 같이 요청 헤더를 통해 `Authorization` 헤더에 인증토큰 값이 JWT 형태로 보내진다.

![Swagger UI - Bearer Auth - Result][image-12]

이 JWT 형태의 토큰을 복호화해서 클레임을 찾아낸 후 이를 활용해서 유효성 처리를 하면 된다.


## OAuth2 암시적 인증 플로우 사용 ##

OAuth2 인증 플로우는 굉장히 다양한데, 이번에는 [OAuth2의 암시적 인증 플로우][az ad oauth2 implicit flow]를 구현해 보자. `OpenApiSecurityAttribute(...)` 데코레이터를 아래와 같이 지정한다 (line #6-8).

* `Type`: `SecuritySchemeType.OAuth2`
* `Flows`: `ImplicitAuthFlow`

마찬가지로 AuthLevel 값을 `Anonymous`로 지정했다 (line #12).

https://gist.github.com/justinyoo/7a9dab5304ef36c8747a3c0cf8430b71?file=05-oauth-implicit-auth-flow-httptrigger.cs&highlights=6-8,12

위의 암시적 OAuth2 인증 플로우에 적용한 `ImplicitAuthFlow` 클라스를 한 번 들여다 보자. 내부적으로 [애저 액티브 디렉토리][az ad]에 특화된 `AuthorizationUrl` 값과 `RefreshUrl`, `Scopes`를 지정해 줬는데, 여기서는 [싱글 테넌트 형식][az ad tenancy]으로 인증 플로우를 운영하기 때문에 테넌트 ID 값을 별도로 구성했다 (line #3-6, 10, 14-15). `Scopes` 값은 기본값으로 지정했다 (line #17).

https://gist.github.com/justinyoo/7a9dab5304ef36c8747a3c0cf8430b71?file=06-implicit-auth-flow.cs&highlights=3-6,10,14-15,17

이 상태에서 애저 펑션 앱을 실행하면 아래와 같은 Swagger UI 화면이 나타난다.

![Swagger UI - OAuth2 Implicit Auth][image-13]

위 그림에서 자물쇠 모양을 클릭하면 아래와 같이 Client ID 값을 입력하는 화면이 나오는데, 이 값을 이용해 애저 액티브 디렉토리에 로그인한 후 액세스 토큰을 발급받는다.

![Swagger UI - OAuth2 Implicit Auth - Details][image-14]

실제로 앱을 실행시키면 아래와 같이 요청 헤더를 통해 `Authorization` 헤더에 발급 받은 액세스 토큰 값이 JWT 형태로 보내진다.

![Swagger UI - OAuth2 Implicit Auth - Result][image-15]

이 JWT 형태의 토큰을 복호화해서 클레임을 찾아낸 후 이를 활용해서 유효성 처리를 하면 된다.


## OpenID Connect 인증 플로우 사용 ##

마지막으로 [OpenID Connect 인증 플로우][az ad oidc flow]를 이용해 보자. `OpenApiSecurityAttribute(...)` 데코레이터를 아래와 같이 지정한다 (line #6-9)

* `Type`: `SecuritySchemeType.OpenIdConnect`
* `OpenIdConnectUrl`: `https://login.microsoftonline.com/{tenant_id}/v2.0/.well-known/openid-configuration`
* `OpenIdConnectScopes`: `openid,profile`

물론 `{tenant_id}` 값은 실제 테넌트 ID 값으로 바꿔줘야 한다. 이렇게 하면 자동으로 OAuth2 인증 플로우를 찾아주게 된다. 마찬가지로 AuthLevel 값을 `Anonymous`로 지정했다 (line #12).

https://gist.github.com/justinyoo/7a9dab5304ef36c8747a3c0cf8430b71?file=07-openid-connect-auth-flow-httptrigger.cs&highlights=6-9,13

이 상태에서 애저 펑션 앱을 실행하면 아래와 같은 Swagger UI 화면이 나타난다.

![Swagger UI - OpenID Connect Auth][image-16]

위 그림에서 자물쇠 모양을 클릭하면 아래와 같은 화면이 나타나는데, 상황에 따라 둘 중 하나를 선택해서 인증하면 된다. 여기서는 아래의 암시적 인증 플로우를 선택해서 Client ID 값을 입력한 후 애저 액티브 디렉토리에 로그인해서 액세스 토큰을 발급받는다.

![Swagger UI - OpenID Connect Auth - Details][image-17]

실제로 앱을 실행시키면 아래와 같이 요청 헤더를 통해 `Authorization` 헤더에 발급 받은 액세스 토큰 값이 JWT 형태로 보내진다.

![Swagger UI - OpenID Connect Auth - Result][image-18]

이 JWT 형태의 토큰을 복호화해서 클레임을 찾아낸 후 이를 활용해서 유효성 처리를 하면 된다.

---

지금까지 애저 펑션으로 API를 사용할 때 OpenAPI와 연동해서 자주 사용하는 인증 방식에 대해 알아보았다. 이외에도 OAuth2 방식의 인증 플로우에 몇가지 더 있긴 하지만, 가장 자주 사용하는 방식이 바로 이 여섯 가지이니만큼 잘 알아두면 애저 펑션에서 기본적으로 제공하는 인증 이외에 추가적인 인증을 구현해서 연동시킬 수 있을 것이다.


[image-01]: https://sa0blogs.blob.core.windows.net/aliencube/2021/10/securing-azure-function-endpoints-via-openapi-auth-01.png
[image-02]: https://sa0blogs.blob.core.windows.net/aliencube/2021/10/securing-azure-function-endpoints-via-openapi-auth-02.png
[image-03]: https://sa0blogs.blob.core.windows.net/aliencube/2021/10/securing-azure-function-endpoints-via-openapi-auth-03.png
[image-04]: https://sa0blogs.blob.core.windows.net/aliencube/2021/10/securing-azure-function-endpoints-via-openapi-auth-04.png
[image-05]: https://sa0blogs.blob.core.windows.net/aliencube/2021/10/securing-azure-function-endpoints-via-openapi-auth-05.png
[image-06]: https://sa0blogs.blob.core.windows.net/aliencube/2021/10/securing-azure-function-endpoints-via-openapi-auth-06.png
[image-07]: https://sa0blogs.blob.core.windows.net/aliencube/2021/10/securing-azure-function-endpoints-via-openapi-auth-07.png
[image-08]: https://sa0blogs.blob.core.windows.net/aliencube/2021/10/securing-azure-function-endpoints-via-openapi-auth-08.png
[image-09]: https://sa0blogs.blob.core.windows.net/aliencube/2021/10/securing-azure-function-endpoints-via-openapi-auth-09.png
[image-10]: https://sa0blogs.blob.core.windows.net/aliencube/2021/10/securing-azure-function-endpoints-via-openapi-auth-10.png
[image-11]: https://sa0blogs.blob.core.windows.net/aliencube/2021/10/securing-azure-function-endpoints-via-openapi-auth-11.png
[image-12]: https://sa0blogs.blob.core.windows.net/aliencube/2021/10/securing-azure-function-endpoints-via-openapi-auth-12.png
[image-13]: https://sa0blogs.blob.core.windows.net/aliencube/2021/10/securing-azure-function-endpoints-via-openapi-auth-13.png
[image-14]: https://sa0blogs.blob.core.windows.net/aliencube/2021/10/securing-azure-function-endpoints-via-openapi-auth-14.png
[image-15]: https://sa0blogs.blob.core.windows.net/aliencube/2021/10/securing-azure-function-endpoints-via-openapi-auth-15.png
[image-16]: https://sa0blogs.blob.core.windows.net/aliencube/2021/10/securing-azure-function-endpoints-via-openapi-auth-16.png
[image-17]: https://sa0blogs.blob.core.windows.net/aliencube/2021/10/securing-azure-function-endpoints-via-openapi-auth-17.png
[image-18]: https://sa0blogs.blob.core.windows.net/aliencube/2021/10/securing-azure-function-endpoints-via-openapi-auth-18.png


[gh sample]: https://github.com/devkimchi/azure-functions-oauth-authentications-via-swagger-ui

[openapi spec security]: https://github.com/OAI/OpenAPI-Specification/blob/main/versions/3.0.1.md#securitySchemeObject

[az fncapp]: https://docs.microsoft.com/ko-kr/azure/azure-functions/functions-overview?WT.mc_id=dotnet-44465-juyoo
[az funcapp security baseline]: https://docs.microsoft.com/ko-kr/security/benchmark/azure/baselines/functions-security-baseline
[az fncapp extensions openapi]: https://github.com/azure/azure-functions-openapi-extension

[az ad]: https://docs.microsoft.com/ko-kr/azure/active-directory/fundamentals/active-directory-whatis?WT.mc_id=dotnet-44465-juyoo
[az ad tenancy]: https://docs.microsoft.com/ko-kr/azure/active-directory/develop/single-and-multi-tenant-apps?WT.mc_id=dotnet-44465-juyoo
[az ad oauth2 implicit flow]: https://docs.microsoft.com/ko-kr/azure/active-directory/develop/v2-oauth2-implicit-grant-flow?WT.mc_id=dotnet-44465-juyoo
[az ad oidc flow]: https://docs.microsoft.com/ko-kr/azure/active-directory/develop/v2-protocols-oidc?WT.mc_id=dotnet-44465-juyoo
