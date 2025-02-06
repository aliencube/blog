---
title: "Azure EasyAuth에 페이지별 권한 부여하기"
slug: giving-granular-controls-to-azure-easyauth
description: "이 포스트에서는 Azure EasyAuth 토큰을 ClaimsPrincipal로 변환해서 ASP.NET Core 애플리케이션의 개별 엔드포인트에 접근권한을 부여하는 방법에 대해 알아봅니다."
date: "2025-02-07"
author: Justin-Yoo
tags:
- azure
- easyauth
- aspnet-core
- authentication
- authorization
cover: /2025/02/giving-granular-controls-to-azure-easyauth-00.png
fullscreen: true
---

웹 애플리케이션을 사용할 때 인증(Authentication)과 인가(권한부여; Authorisation)를 구현하는 것은 보안의 측면에서 꽤 중요한 작업이다. 전통적으로 유저네임/패스워드를 입력하는 방식부터 OAuth를 이용한 방식까지 다양한 인증 및 권한부여 구현법이 있다. 하지만, 이 인증/권한부여 관련 로직을 구현하는 것은 굉장히 복잡하고 어렵기도 한 작업이다.

다행히도 Azure에서는 이를 대신해 주는 기능이 따로 있는데, 이를 EasyAuth라고 한다. EasyAuth는 Azure PaaS 서비스 &ndash; [App Service][az easyauth appsvc], [Functions][az easyauth appsvc], [Container Apps][az easyauth aca], [Static Web Apps][az easyauth aswa] &ndash; 등에서 지원하며, 기존의 코드를 전혀 건드리지 않고도 구현할 수 있다는 장점이 있다.

그런데, 이 EasyAuth는 클라우드 아키텍처 패턴 중 Sidecar 패턴을 구현하다보니 웹 애플리케이션 전체를 보호할 수 있지만, 웹 애플리케이션의 특정한 영역만 보호한다거나 제외한다거나 하는 세부적인 조정은 어렵다.

이를 해결하기 위해 개발자 커뮤니티에서 다양한 시도를 해 왔다.

- [EasyAuth for App Service](https://github.com/MaximRouiller/MaximeRouiller.Azure.AppService.EasyAuth) by [Maxime Rouiller](https://github.com/MaximRouiller)
- [EasyAuth for Azure Container Apps](https://johnnyreilly.com/azure-container-apps-easy-auth-and-dotnet-authentication) by [John Reilly](https://bsky.app/profile/johnnyreilly.com)
- [EasyAuth for Azure Static Web Apps](https://github.com/anthonychu/blazor-auth-static-web-apps) by [Anthony Chu](https://bsky.app/profile/anthonychu.ca)

이 내용들은 여전히 지금도 유효하지만, 코드베이스 측면에서는 최신의 .NET 기능을 반영하기 위한 업데이트가 필요하다.

이번 포스트에서는 Azure App Service와 Azure Container Apps에서 어떻게 이 EasyAuth에서 생성한 인증 토큰을 Blazor를 포함한 ASP.NET Core 웹 애플리케이션에서 사용할 수 있게끔 변환시킬 수 있는지에 대해 알아보기로 한다.

> [이전 포스트](/ko/2021/09/15/accessing-msgraph-from-blazor-wasm-running-on-aswa/)를 참고하면 Azure Static Web Apps에서는 이 EasyAuth 기능을 어떻게 활용할 수 있는지 알 수 있다.

## NuGet 패키지 다운로드

현재 EasyAuth에서 생성하는 Client Principal 토큰을 ASP.NET Core 웹 앱에서 사용할 수 있게끔 [`ClaimsPrincipal`][dotnet claimsprincipal] 개체로 변환해주는 NuGet 패키지를 배포중이다. 현재 [Mcirosoft Entra ID][msft entra id]와 [GitHub OAuth][gh oauth app]를 지원한다.

| Package                                                                                                                   | Version | Downloads |
| ------------------------------------------------------------------------------------------------------------------------- | ------- | --------- |
| [Aliencube.Azure.Extensions.EasyAuth](https://www.nuget.org/packages/Aliencube.Azure.Extensions.EasyAuth)                 | [![NuGet Version](https://img.shields.io/nuget/v/Aliencube.Azure.Extensions.EasyAuth?logo=nuget)](https://www.nuget.org/packages/Aliencube.Azure.Extensions.EasyAuth)                 | [![NuGet Downloads](https://img.shields.io/nuget/dt/Aliencube.Azure.Extensions.EasyAuth?logo=nuget)](https://www.nuget.org/packages/Aliencube.Azure.Extensions.EasyAuth)                 |
| [Aliencube.Azure.Extensions.EasyAuth.EntraID](https://www.nuget.org/packages/Aliencube.Azure.Extensions.EasyAuth.EntraID) | [![NuGet Version](https://img.shields.io/nuget/v/Aliencube.Azure.Extensions.EasyAuth.EntraID?logo=nuget)](https://www.nuget.org/packages/Aliencube.Azure.Extensions.EasyAuth.EntraID) | [![NuGet Downloads](https://img.shields.io/nuget/dt/Aliencube.Azure.Extensions.EasyAuth.EntraID?logo=nuget)](https://www.nuget.org/packages/Aliencube.Azure.Extensions.EasyAuth.EntraID) |
| [Aliencube.Azure.Extensions.EasyAuth.GitHub](https://www.nuget.org/packages/Aliencube.Azure.Extensions.EasyAuth.GitHub)   | [![NuGet Version](https://img.shields.io/nuget/v/Aliencube.Azure.Extensions.EasyAuth.GitHub?logo=nuget)](https://www.nuget.org/packages/Aliencube.Azure.Extensions.EasyAuth.GitHub)   | [![NuGet Downloads](https://img.shields.io/nuget/dt/Aliencube.Azure.Extensions.EasyAuth.GitHub?logo=nuget)](https://www.nuget.org/packages/Aliencube.Azure.Extensions.EasyAuth.GitHub)   |

## NuGet 패키지 앱에 적용하기

> 여기서는 Blazor 웹 앱을 예시로 사용한다. 하지만 다른 ASP.NET Core 기반의 웹 애플리케이션에도 동일하게 적용시킬 수 있다.

### Microsoft Entra ID

1. 먼저 Blazor 애플리케이션에 NuGet 패키지를 추가한다.

    ```powershell
    dotnet add package Aliencube.Azure.Extensions.EasyAuth.EntraID
    ```

   > **NOTE**: 이 포스트를 쓰는 시점에서는 아직 프리뷰이기 때문에 `dotnet add package Aliencube.Azure.Extensions.EasyAuth.EntraID --prerelease`처럼 `--prerelease` 태그를 붙여서 설치한다.

1. `Program.cs` 파일을 열고 `var app = builder.Build();` 라인을 찾아 아래와 같이 인증 핸들러와 권한부여 인스턴스를 의존성 개체로 추가한다.

    ```csharp
    // 👇👇👇 인증/권한부여 의존성 개체 추가
    builder.Services.AddAuthentication(EasyAuthAuthenticationScheme.Name)
                    .AddAzureEasyAuthHandler<EntraIDEasyAuthAuthenticationHandler>();
    builder.Services.AddAuthorization();
    // 👆👆👆 인증/권한부여 의존성 개체 추가
    
    var app = builder.Build();
    ```

1. 같은 `Program.cs` 파일의 맨 아래에서 `app.Run();` 라인을 찾아 아래와 같이 인증/권한부여 기능을 활성화 시킨다.

    ```csharp
    // 👇👇👇 인증/권한부여 기능 추가
    app.UseAuthentication();
    app.UseAuthorization();
    // 👆👆👆 인증/권한부여 기능 추가
    
    app.Run();
    ```

1. 권한을 부여하고자 하는 특정 페이지 컴포넌트를 열고 아래와 같이 `Authorize` 속성을 추가한다.

    ```razor
    @page "/random-page-url"
    
    @* 👇👇👇 페이지 인증 및 권한부여 *@
    @using Aliencube.Azure.Extensions.EasyAuth
    @using Microsoft.AspNetCore.Authorization
    @attribute [Authorize(AuthenticationSchemes = EasyAuthAuthenticationScheme.Name)]
    @* 👆👆👆 페이지 인증 및 권한부여 *@
    ```

1. 저장 후 앱을 Azure Container Apps 또는 Azure App Service로 배포한다.
1. 배포가 끝난 후 아래와 같이 인증 관련 설정을 변경한다.

   ![Azure Container Apps 인증 설정 화면][image-01]
   ![Azure App Service 인증 설정 화면][image-02]

1. 웹 앱으로 이동해서 페이지가 잘 보이는지 확인한다. 그리고 `/random-page-url`로 이동해서 401 에러가 나오는지 확인한다.
1. 웹 앱의 `/.auth/login/aad` 링크를 통해 로그인한 후 다시 `/random-page-url`로 이동해서 페이지가 잘 나오는지 확인한다.

### GitHub OAuth

앞서 Microsoft Entra ID를 사용하는 과정과 거의 같지만, 다른 NuGet 패키지를 사용한다.

1. 먼저 Blazor 애플리케이션에 NuGet 패키지를 추가한다.

    ```powershell
    dotnet add package Aliencube.Azure.Extensions.EasyAuth.GitHub
    ```

   > **NOTE**: 이 포스트를 쓰는 시점에서는 아직 프리뷰이기 때문에 `dotnet add package Aliencube.Azure.Extensions.EasyAuth.GitHub --prerelease`처럼 `--prerelease` 태그를 붙여서 설치한다.

1. `Program.cs` 파일을 열고 `var app = builder.Build();` 라인을 찾아 아래와 같이 인증 핸들러와 권한부여 인스턴스를 의존성 개체로 추가한다.

    ```csharp
    // 👇👇👇 인증/권한부여 의존성 개체 추가
    builder.Services.AddAuthentication(EasyAuthAuthenticationScheme.Name)
                    .AddAzureEasyAuthHandler<GitHubEasyAuthAuthenticationHandler>();
    builder.Services.AddAuthorization();
    // 👆👆👆 인증/권한부여 의존성 개체 추가
    
    var app = builder.Build();
    ```

1. 이후 과정은 앞서와 동일하다.

## 무슨 일이 벌어진 걸까?

Azure EasyAuth 기능을 이용해 인증을 하면 `X-MS-CLIENT-PRINCIPAL` 이라는 요청 헤더가 생기고 그 값은 base-64 인코딩 된 문자열이다. 이를 디코딩해 보면 JSON 개체가 생긴다. 불필요한 정보는 다 생략하고 필요한 부분만 남겨뒀다. 아래는 Microsoft Entra ID로 인증할 때 보이는 내용이다.

```json
{
  "auth_typ": "aad",
  "name_typ": "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress",
  "role_typ": "http://schemas.microsoft.com/ws/2008/06/identity/claims/role",
  "claims": [
    {
      "typ": "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress",
      "val": "john.doe@contoso.com"
    },
    {
      "typ": "roles",
      "val": "User"
    }
  ]
}
```

그리고 아래는 GitHub OAuth 앱으로 인증할 때 보이는 내용이다.

```json
{
  "auth_typ": "github",
  "name_typ": "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name",
  "role_typ": "http://schemas.microsoft.com/ws/2008/06/identity/claims/role",
  "claims": [
    {
      "typ": "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name",
      "val": "John Doe"
    },
    {
      "typ": "urn:github:type",
      "val": "User"
    }
  ]
}
```

이 JSON 개체를 Blazor 앱에서 이해할 수 있는 `ClaimsPrincipal` 인스턴스로 변환해 줘야 개별 페이지마다 세세한 권한 부여가 가능하다.

그렇다면 이 JSON 개체를 어떻게 해석하는 것이 좋을까?

- `auth_typ`: 값은 현재 인증에 사용한 앱의 종류를 가리킨다. 예를 들어 Entra ID는 `aad`이고 GitHub OAuth는 `github`이다.
- `name_typ`: 인증 후 `HttpContext.User.Identity.Name`에서 볼 수 있는 로그인 한 사용자 이름이다.
- `role_typ`: 페이지별 권한 부여에 필요한 값이다.
- `claims`: 인증후 Microsoft Entra ID 또는 GitHub OAuth에서 생성하는 다양한 클레임의 묶음이다.

여기서 문제가 한가지 있다. 적어도 Microsoft Entra ID와 GitHub OAuth를 사용해서 인증을 해서 만들어지는 `X-MS-CLIENT-PRINCIPAL`의 토큰에는 해당 클레임이 없다. 위의 예시로 들어둔 JSON 개체만 봐도 Entra ID의 경우에는 `roles`가 있고 GitHub OAuth에서는 `urn:github:type` 클레임이 있을 뿐이어서, 이를 `ClaimsPrincipal` 인스턴스가 사용하는 OpenID Connect 표준 규격인 `role_typ` 키로 변환해 줘야 한다. 위에 사용한 NuGet 패키지는 이 변환 과정에 관여한다.

이렇게 바꿔두면 Blazor 페이지 컴포넌트에서 EasyAuth로 인증하거나 인증 후 특정 권한을 부여 받은 사람만 해당 페이지에 접근할 수 있게끔 설정할 수 있다. 아래는 `AuthenticationSchemes` 속성을 이용해서 EasyAuth로 인증한 모두가 접근하게끔 설정했다.

```razor
@page "/random-page-url"

@attribute [Authorize(AuthenticationSchemes = EasyAuthAuthenticationScheme.Name)]
```

그리고 아래와 같이 `Roles` 속성을 이용하면 특정 `User` 권한을 가진 사용자만 이 페이지에 접근할 수 있다.

```razor
@page "/random-page-url"

@attribute [Authorize(Roles = "User")]
```

## 향후 계획

Azure EasyAuth가 지원하는 인증서비스 제공자는 아래와 같다. 그 중에서 NuGet 패키지는 현재 Microsoft Entra ID와 GitHub만 우선적으로 지원하고 있다.

- ✅ Microsoft Entra ID
- ✅ GitHub
- ⏹️ OpenID Connect
- ⏹️ Google
- ⏹️ Facebook
- ⏹️ X
- ⏹️ Apple

관심 있는 사람들의 기여는 언제든 환영!

---

지금까지 Azure EasyAuth 인증 토큰을 ASP.NET Core 웹 애플리케이션에서 활용할 수 있는 `ClaimsPrincipal` 인스턴스로 변환하는 방법에 대해 알아 보았다. 이를 활용하면 개별 페이지마다 좀 더 세밀한 권한를 부여해서 애플리케이션의 보안 향상에 도움이 될 수 있을 것이다.

## 좀 더 알아보기

만약 Azure EasyAuth 관련해서 좀 더 알아보고 싶다면 아래 문서를 확인해 보도록 하자.

- [Azure App Service EasyAuth][az easyauth appsvc]
- [Azure Functions EasyAuth][az easyauth appsvc]
- [Azure Container Apps EasyAuth][az easyauth aca]
- [Azure Static Web Apps EasyAuth][az easyauth aswa]
- [Azure EasyAuth Extensions (Code)](https://github.com/aliencube/azure-easyauth-extensions)
- [Azure EasyAuth Samples (Code)](https://github.com/devkimchi/azure-easyauth-sample)

[image-01]: /2025/02/giving-granular-controls-to-azure-easyauth-01.png
[image-02]: /2025/02/giving-granular-controls-to-azure-easyauth-02.png

[az easyauth appsvc]: https://learn.microsoft.com/ko-kr/azure/app-service/overview-authentication-authorization
[az easyauth aca]: https://learn.microsoft.com/ko-kr/azure/container-apps/authentication
[az easyauth aswa]: https://learn.microsoft.com/ko-kr/azure/static-web-apps/authentication-authorization

[msft entra id]: https://learn.microsoft.com/ko-kr/entra/fundamentals/whatis

[gh oauth app]: https://docs.github.com/ko/apps/oauth-apps/building-oauth-apps/creating-an-oauth-app

[dotnet claimsprincipal]: https://learn.microsoft.com/ko-kr/dotnet/api/system.security.claims.claimsprincipal
