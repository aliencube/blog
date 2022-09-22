---
title: "애저 AD B2C를 이용해 블레이저 웹어셈블리 앱 인증 모듈 연동하기"
slug: authn-ing-blazor-wasm-with-azure-ad-b2c
description: "이 포스트에서는 블레이저 웹어셈블리 앱에 애저 AD B2C 기반의 인증 기능을 연동하는 방법에 대해 알아봅니다."
date: "2022-09-23"
author: Justin-Yoo
tags:
- dotnet
- blazor-wasm
- azure-ad-b2c
- authn
cover: https://sa0blogs.blob.core.windows.net/aliencube/2022/09/authn-ing-blazor-wasm-with-azure-ad-b2c-00-ko.png
fullscreen: true
---

[블레이저 웹어셈블리][blazor wasm] 앱에 인증을 구현하려면 꽤 다양한 방법으로 접근이 가능하다. 만약 [애저 정적 웹 앱(ASWA)][az swa]에 배포하는 경우라면 [지난 블로그 포스트][post 1]에서 언급한 것과 같이 자체적으로 제공하는 기능을 쓰면 된다. 하지만, 일반적인 경우에는 인증 과정을 직접 구현해야 하는데, 이 역시도 [애저 AD B2C][az ad b2c]와 같은 인증 서비스를 활용하면 굉장히 쉽게 구현할 수 있다. 이 포스트에서는 애저 AD B2C 서비스를 이용해 블레이저 웹어셈블리 앱에 손쉽게 인증 기능을 연동시키는 방법에 대해 알아보기로 한다.

> 이 포스트에 사용한 샘플 애플리케이션은 [이곳][gh sample]에서 다운로드 받을 수 있다.


## 애저 AD와 애저 AD B2C의 차이 ##

[애저 AD][az ad]와 [애저 AD B2C][az ad b2c]는 이름이 비슷한 만큼 하는 일도 비슷하다. 기본적으로 인증 관련 서비스를 제공한다는 목적에는 둘 다 동일하다고 보면 된다.

그렇다면 어떤 차이가 있을까? 애저 AD의 경우에는 좀 더 조직 내 자원에 대한 접근을 설정할 수 있다. 예를 들어 셰어포인트라든가, 팀즈라든가 하는 것들에 대해 어느 정도까지 접근을 허용할 것인지 아닌지 등을 관리할 수 있다고 보면 좋다. 반면에 애저 AD B2C의 경우는 동일하게 인증 관련 역할을 하긴 하지만, 일반 사용자가 특정 애플리케이션에 로그인하는 기능을 제공한다. 애저 AD B2C는 독립적으로 작동하기 때문에 애저 AD B2C를 통해 인증에 성공했다고 해서 조직 내의 다른 애플리케이션이라든가 자원에 접근할 수는 없다.


## 애저 AD B2C 준비 ##

애저 포탈에서 애저 AD B2C 인스턴스를 하나 생성한다. 여기서는 테넌트 이름을 `fitabilitydevkr`라고 가정하기로 하자. 그러면 아래와 같이 애저 AD B2C 인스턴스가 만들어진다.

![Azure AD B2C][image-01]

"Open B2C Tenant" 링크를 클릭하면 애저 AD B2C 어드민 화면으로 들어간다. 거기서 왼쪽의 "앱 등록" 메뉴를 클릭한다. 이후 상단의 "새 등록" 메뉴를 클릭한다.

![Azure AD B2C - New app][image-02]

애플리케이션 등록 화면에서 아래와 같이 입력한다.

* 이름: 애플리케이션 이름 입력. 여기서는 `my-fitability-app` 사용.
* 계정 유형: "모든 ID 공급자 또는 조직 디렉터리의 계정(사용자 흐름에서 사용자를 인증하는 용도)" 선택
* 리디렉션 URI(권장): SPA(단일 페이지 애플리케이션) 선택 후, `https://localhost/authentication/login-callback` 입력
* 권한: "openid 및 offline_access 권한에 대한 관리자 동의 허용" 체크

![Azure AD B2C - App registration][image-03]

등록이 끝난 후에 다시 한 번 아래 사항을 확인한다.

* 단일 페이지 애플리케이션 &ndash; 리디렉션 URI
* 암시적 하이브리드 흐름 &ndash; 액세스 토큰, ID 토큰
* 지원되는 계정 유형 &ndash; 모든 ID 공급자 또는 조직

![Azure AD B2C - Authentication][image-04]

애플리케이션 등록이 끝나면 아래와 같은 화면이 보인다. 애플리케이션(클라이언트) ID 값을 복사해 둔다.

![Azure AD B2C - Client ID][image-05]

이번에는 로그인 프로세스를 정의해야 한다. 애저 AD B2C 인스턴스 화면에서 "사용자 흐름" 메뉴를 클릭한 후 "새 사용자 흐름" 메뉴를 선택한다.

![Azure AD B2C - User flow][image-06]

다양한 사용자 흐름 형식이 있는데, 그 중에서 "등록 및 로그인" 흐름을 선택한다. 이후 "권장" 버전을 선택한다.

![Azure AD B2C - Sign up and sign in #1][image-07]

아래와 같이 입력한 후 "만들기" 버튼을 클릭한다.

* 이름: `SignUpSignIn`
* ID 공급자: Email Signup

> 다양한 ID 공급자를 추가할 수 있는데, 이 포스트에서는 이메일 주소만 입력 받는 것으로 한다.

![Azure AD B2C - Sign up and sign in #2][image-08]

만약 토큰에 좀 더 많은 정보를 추가하고 싶다면 아래 내용을 입력하면 좋다.

![Azure AD B2C - Sign up and sign in #3][image-09]

애저 AD B2C 쪽에서 블레이저 웹어셈블리 앱이 사용할 수 있는 인증 정보를 구성했다.


## 블레이저 웹 어셈블리 앱 준비 ##

[블레이저 웹어셈블리][blazor wasm] 앱에 [애저 AD B2C][az ad b2c] 인증 기능을 연동하기 위해서는 우선 블레이저 웹 어셈블리 앱 프로젝트를 만들어야 한다. 이 포스트를 쓰는 현재 [비주얼 스튜디오 2022][vs 2022]의 가장 최신 버전은 17.3.4인데, 비주얼 스튜디오에서는 이 프로젝트를 생성할 수 없다.

![Blazor WASM in VS2022][image-10]

따라서, dotnet CLI를 이용해서 애저 AD B2C를 지원하는 블레이저 웹어셈블리 프로젝트를 만들어야 한다. 아래와 같이 터미널에서 입력한다. 여기서 아래의 옵션에 주목해 보자.

* `--framework net6.0`: 이 옵션을 주지 않을 경우 현재 컴퓨터에 설치된 가장 최신 버전의 .NET 프레임워크 버전을 사용한다. 여기서는 명시적으로 `net6.0` 옵션을 줘서 .NET 6 버전으로 고정시켰다.
* `--hosted false`: 블레이저 웹어셈블리 앱에 프록시 API를 붙여 사용하지 않는 독립실행형(스탠드얼론) 앱으로 구성할 때 사용한다. 만약 프록시 API를 붙인 호스팅형 블레이저 웹어셈블리 앱을 만들고 싶다면 `true` 값을 주면 된다.
* `--auth IndividualB2C`: 애저 AD B2C 서비스를 이용하고 싶을 때 사용하는 옵션이다.
* `--aad-b2c-instance "https://<TENANT_NAME>.b2clogin.com/"`: 애저 AD B2C 인스턴스의 로그인 주소를 가리킨다. 이 포스트에서는 테넌트 이름을 `fitabilitydevkr` 이라고 가정한다.
* `--domain "<TENANT_NAME>.onmicrosoft.com"`: 애저 AD B2C 인스턴스의 도메인 주소를 가리킨다. 이 포스트에서는 테넌트 이름을 `fitabilitydevkr` 이라고 가정한다.
* `--client-id`: 앞서 만들어 둔 OAuth 앱의 Client ID 값을 입력한다.
* `--susi-policy-id "B2C_1_SignUpSignIn"`: 앞서 만들어 둔 등록/로그인 정책의 이름을 입력한다. 앞서 정책 이름을 `SignUpSignIn`으로 정했다.

```powershell
dotnet new blazorwasm \
    --output "MyBlazorWasmApp" \
    --framework net6.0 \
    --hosted false \
    --auth IndividualB2C \
    --aad-b2c-instance "https://<TENANT_NAME>.b2clogin.com/" \
    --domain "<TENANT_NAME>.onmicrosoft.com" \
    --client-id "<CLIENT_ID>" \
    --susi-policy-id "B2C_1_SignUpSignIn"
```

이렇게 해서 블레이저 웹어셈블리 앱 프로젝트가 만들어졌다. 이제 `Program.cs` 파일을 수정할 차례이다. 아래와 같이 `AddMsalAuthentication(...)` 메서드를 추가한다.

```csharp
builder.Services.AddScoped(sp => new HttpClient
{
    BaseAddress = new Uri(builder.HostEnvironment.BaseAddress)
});

// ⬇️⬇️⬇️ Add these lines below ⬇️⬇️⬇️
builder.Services.AddMsalAuthentication(options =>
{
    options.ProviderOptions.DefaultAccessTokenScopes.Add("openid");
    options.ProviderOptions.DefaultAccessTokenScopes.Add("offline_access");

    builder.Configuration.Bind("AzureAdB2C", options.ProviderOptions.Authentication);
});
/// ⬆️⬆️⬆️ Add these lines above ⬆️⬆️⬆️

await builder.Build().RunAsync();
```

이제 `wwwroot` 디렉토리에 있는 `appsettings.json` 파일을 아래와 같이 수정한다.

```json
{
  "AzureAdB2C": {
    "Authority": "https://<TENANT_NAME>.b2clogin.com/<TENANT_NAME>.onmicrosoft.com/B2C_1_SignUpSignIn",
    "ClientId": "<CLIENT_ID>",
    "ValidateAuthority": false
  }
}
```

여기까지 설정한 후 블레이저 웹어셈블리 앱을 빌드하고 로컬에서 실행시킨다.

![Blazor WASM landing page][image-11]

이후 오른쪽 상단의 "Log in" 버튼을 클릭하면 아래와 같이 로그인 화면이 나타난다.

![Blazor WASM login page][image-12]

여기서 이메일 주소와 패스워드를 넣고 로그인을 해도 되고, 그 아래 "Sign up now" 버튼을 클릭해서 새 계정을 등록시켜도 된다. 로그인 후에는 아래와 같은 화면을 볼 수 있다.

![Blazor WASM logged in][image-13]

로그인 사용자의 이름이 보인다.

---

지금까지 블레이저 웹어셈블리 앱에 애저 AD B2C 인증 서비스를 추가해 보았다. 기본적으로 이메일 주소와 패스워드만 있으면 별다른 코드 작성 없이도 인증 기능을 굉장히 손쉽게 구성할 수 있다. 다음 포스트에서는 애저 AD B2C 서비스를 이용해서 이메일 주소 뿐만 아니라 다른 소셜 로그인 기능을 통합해 보기로 한다.


## 블레이저 앱에 대해 더 알고 싶다면? ##

만약 블레이저 앱에 대해 좀 더 알고 싶다면 아래 튜토리얼을 참조해 보면 좋다.

* [블레이저][blazor] (영문)
* [블레이저 튜토리얼][blazor tutorial] (영문)
* [블레이저 Learn][blazor learn] (한국어)


## 블레이저 앱과 애저 AD B2C 연동 방법에 대해 더 자세하게 알고 싶다면? ##

* [스탠드얼론 블레이저 웹어셈블리앱에 애저 AD B2C 연동하기][blazor wasm standalone az ad b2c]
* [호스트형 블레이저 웹어셈블리앱에 애저 AD B2C 연동하기][blazor wasm hosted az ad b2c]
* [블레이저 웹어셈블리 보안 추가 시나리오][blazor wasm addtional scenarios]
* [애저 AD B2C 보안 사용자 흐름 정책][az ad b2c userflow]


[image-01]: https://sa0blogs.blob.core.windows.net/aliencube/2022/09/authn-ing-blazor-wasm-with-azure-ad-b2c-01-ko.png
[image-02]: https://sa0blogs.blob.core.windows.net/aliencube/2022/09/authn-ing-blazor-wasm-with-azure-ad-b2c-02-ko.png
[image-03]: https://sa0blogs.blob.core.windows.net/aliencube/2022/09/authn-ing-blazor-wasm-with-azure-ad-b2c-03-ko.png
[image-04]: https://sa0blogs.blob.core.windows.net/aliencube/2022/09/authn-ing-blazor-wasm-with-azure-ad-b2c-04-ko.png
[image-05]: https://sa0blogs.blob.core.windows.net/aliencube/2022/09/authn-ing-blazor-wasm-with-azure-ad-b2c-05-ko.png
[image-06]: https://sa0blogs.blob.core.windows.net/aliencube/2022/09/authn-ing-blazor-wasm-with-azure-ad-b2c-06-ko.png
[image-07]: https://sa0blogs.blob.core.windows.net/aliencube/2022/09/authn-ing-blazor-wasm-with-azure-ad-b2c-07-ko.png
[image-08]: https://sa0blogs.blob.core.windows.net/aliencube/2022/09/authn-ing-blazor-wasm-with-azure-ad-b2c-08-ko.png
[image-09]: https://sa0blogs.blob.core.windows.net/aliencube/2022/09/authn-ing-blazor-wasm-with-azure-ad-b2c-09-ko.png
[image-10]: https://sa0blogs.blob.core.windows.net/aliencube/2022/09/authn-ing-blazor-wasm-with-azure-ad-b2c-10-ko.png
[image-11]: https://sa0blogs.blob.core.windows.net/aliencube/2022/09/authn-ing-blazor-wasm-with-azure-ad-b2c-11-ko.png
[image-12]: https://sa0blogs.blob.core.windows.net/aliencube/2022/09/authn-ing-blazor-wasm-with-azure-ad-b2c-12-ko.png
[image-13]: https://sa0blogs.blob.core.windows.net/aliencube/2022/09/authn-ing-blazor-wasm-with-azure-ad-b2c-13-ko.png


[post 1]: /ko/2021/09/15/accessing-msgraph-from-blazor-wasm-running-on-aswa/

[gh sample]: https://github.com/fitability/fitability-app

[vs 2022]: https://visualstudio.microsoft.com/vs/?WT.mc_id=dotnet-70466-juyoo

[blazor]: https://dotnet.microsoft.com/apps/aspnet/web-apps/blazor?WT.mc_id=dotnet-70466-juyoo
[blazor tutorial]: https://dotnet.microsoft.com/learn/aspnet/blazor-tutorial/intro?WT.mc_id=dotnet-70466-juyoo
[blazor learn]: https://learn.microsoft.com/ko-kr/training/paths/build-web-apps-with-blazor/?WT.mc_id=dotnet-70466-juyoo
[blazor wasm]: https://learn.microsoft.com/ko-kr/aspnet/core/blazor/?WT.mc_id=dotnet-70466-juyoo#blazor-webassembly
[blazor wasm standalone az ad b2c]: https://learn.microsoft.com/ko-kr/aspnet/core/blazor/security/webassembly/standalone-with-azure-active-directory-b2c?WT.mc_id=dotnet-70466-juyoo
[blazor wasm hosted az ad b2c]: https://learn.microsoft.com/ko-kr/aspnet/core/blazor/security/webassembly/hosted-with-azure-active-directory-b2c?WT.mc_id=dotnet-70466-juyoo
[blazor wasm addtional scenarios]: https://learn.microsoft.com/ko-kr/aspnet/core/blazor/security/webassembly/additional-scenarios?WT.mc_id=dotnet-70466-juyoo

[az swa]: https://learn.microsoft.com/ko-kr/azure/static-web-apps/overview?WT.mc_id=dotnet-70466-juyoo
[az ad]: https://learn.microsoft.com/ko-kr/azure/active-directory/fundamentals/active-directory-whatis?WT.mc_id=dotnet-70466-juyoo
[az ad b2c]: https://learn.microsoft.com/ko-kr/azure/active-directory-b2c/overview?WT.mc_id=dotnet-70466-juyoo
[az ad b2c userflow]: https://learn.microsoft.com/ko-kr/azure/active-directory-b2c/tutorial-create-user-flows?pivots=b2c-user-flow&WT.mc_id=dotnet-70466-juyoo
