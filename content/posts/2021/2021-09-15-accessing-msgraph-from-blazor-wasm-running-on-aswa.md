---
title: "애저 정적 웹 앱에 배포한 블레이저 웹어셈블리 앱에 Microsoft Graph를 통해 사용자 데이터 출력하기"
slug: accessing-msgraph-from-blazor-wasm-running-on-aswa
description: "이 포스트에서는 Microsoft Graph API를 이용해 애저 정적 웹 앱 인스턴스에 배포한 블레이저 웹어셈블리 앱에 사용자 데이터를 출력하는 방법에 대해 알아봅니다."
date: "2021-09-15"
author: Justin-Yoo
tags:
- azure-static-web-apps
- blazor-wasm
- msal
- msgraph
cover: https://sa0blogs.blob.core.windows.net/aliencube/2021/09/accessing-msgraph-from-blazor-wasm-running-on-aswa-00.png
fullscreen: true
---

[애저 정적 웹 앱][az swa]은 굉장히 쉽고 간단한 [인증 기능][az swa authn]을 제공하고 있다. 이를 통하면 별도의 복잡한 인증 절차를 거칠 필요가 없이 쉽게 애저 정적 웹 앱 인스턴스에 로그인할 수 있다. 그런데, 이 인증 관련 정보는 로그인을 했다 아니다 정도만 알 수 있을 뿐, 좀 더 자세한 정보를 알기 위해서는 별도의 작업을 더 해줘야 한다. 이 포스트에서는 [애저 정적 웹 앱][az swa]에 배포한 [블레이저 웹어셈블리][blazor wasm] 앱에서 [Microsoft Graph API][ms graph]를 이용해 사용자 프로필 데이터에 접근하는 방법에 대해 알아보기로 한다.

> 이 포스트에서 사용한 예제 애플리케이션은 이 [깃헙 리포지토리][gh sample]를 참고한다.


## 애저 정적 웹 앱 로그인 사용자 데이터 출력 ##

[블레이저 웹어셈블리][blazor wasm]로 앱을 만들어서 [애저 정적 웹 앱][az swa]으로 배포한 후, 로그인 전의 페이지 상태는 대략 아래와 같은 모양일 것이다.

![로그인 이전][image-01]

블레이저 웹어셈블리 앱을 이용해 애저 정적 웹 앱에 로그인할 때, [애저 액티브 디렉토리][az ad]를 주 인증 공급자로 활용하려고 한다면 위 그림의 ***로그인*** 링크에 아래와 같이 연결하면 된다.

https://gist.github.com/justinyoo/38ae8fdd9ac6161551a9aee0d15b76e7?file=01-auth-login-aad.txt

로그인 후에는 블레이저 안에서 아래와 같은 코드를 이용해 로그인 데이터에 접근할 수 있다. 가독성을 위해 불필요한 코드는 제거했다.

https://gist.github.com/justinyoo/38ae8fdd9ac6161551a9aee0d15b76e7?file=02-auth-me.cs

이렇게 해서 받은 데이터는 아래와 같이 생겼다.

https://gist.github.com/justinyoo/38ae8fdd9ac6161551a9aee0d15b76e7?file=03-auth-details.json

즉, 애저 정적 웹 앱에서 제공하는 로그인 정보는 위와 같이 제한적인 내용이 전부이다. 따라서, 로그인 사용자의 이름이라든가 다른 정보를 확인하려면 추가적인 작업을 더 해줘야 한다.


## Microsoft Graph를 이용한 사용자 데이터 접근 ##

위의 로그인 정보로 알 수 있는 사용자의 개인 정보는 로그인에 사용한 이메일 주소가 전부이다. 여기서 알 수 있는 사실은 아래와 같다:

* 내 계정이 속한 테넌트로 로그인이 되었다.
* 로그인 정보는 내가 로그인 한 이메일 주소를 말해준다.
* 로그인 정보는 내가 로그인 한 테넌트의 정보를 **말해주지 않는다**.
* 로그인 정보는 애저 정적 웹 앱 인스턴스가 호스팅되고 있는 테넌트의 정보를 **말해 주지 않는다**.
* 로그인 정보는 내가 접근하고자 하는 테넌트의 정보를 **말해주지 않는다**.

즉, 내가 로그인 한 테넌트, 정적 웹 앱 인스턴스가 호스팅되고 있는 테넌트, 그리고 사용자 데이터가 저장되어 있는 테넌트가 모두 다를 수 있다는 의미이다. 내가 알고 있는 정보는:

1. 일단 어딘가의 테넌트에 로그인이 되어 있고,
2. 그 로그인 관련 정보 중에 내가 아는 것은 이메일 주소 뿐이다.

그렇다면, 어떻게 내가 접근하고자 하는 테넌트의 사용자 데이터를 알 수 있을까?

가장 먼저 해야 할 일은 바로 리소스에 접근할 수 있도로 인증하는 일이다. 현재 정적 웹 앱에 로그인은 했지만, 이 로그인 내용만으로는 리소스 접근할 수 없기 때문이다. [애저 정적 웹 앱][az swa]은 백엔드 API로 [애저 펑션][az swa api]을 제공하고 있으므로 애저 펑션 앱을 이용해서 리소스에 접근하도록 하자.

애저 정적 웹 앱의 블레이저 웹어셈블리 쪽에서 API를 호출할 때 항상 로그인 정보를 요청 헤더 `x-ms-client-principal`를 통해 보낸다. 헤더에 담겨 있는 정보는 Base64 인코딩된 문자열인데 대략 아래와 비슷하게 생겼다.

https://gist.github.com/justinyoo/38ae8fdd9ac6161551a9aee0d15b76e7?file=04-client-principal.txt

따라서 이렇게 넘어온 데이터를 디코딩하고 비직렬화해서 로그인에 사용한 이메일을 찾아야 한다. 우선 비직렬화에 필요한 개체를 아래와 같이 정의한다.

https://gist.github.com/justinyoo/38ae8fdd9ac6161551a9aee0d15b76e7?file=05-client-principal.cs

그리고 애저 펑션의 엔드포인트 안에서 헤더를 아래와 같이 비직렬화 한 후 로그인에 사용한 이메일 주소를 알아낸다.

https://gist.github.com/justinyoo/38ae8fdd9ac6161551a9aee0d15b76e7?file=06-get-client-principal.cs

이제 사용자 데이터를 조회하기 위한 기초 작업은 끝났다. 다음 단계로 넘어가 보자.


## 애저 액티브 디렉토리 접근을 위한 앱 설정 ##

우선 애저 액티브 디렉토리 접근을 위해서는 [앱을 하나 등록][ms al apprego]해야 한다. 애저 포탈을 통해 금방 만들 수 있다. 이 포스트에서는 앱을 등록하는 과정에 대해서는 설명하지 않는 대신 [이 문서][ms al apprego]를 참고하면 좋다. 또한 앱을 등록한 후에는 [적절한 권한][ms al apprego roles]을 부여해야 한다. 여기서는 위임 권한 대신 [애플리케이션 권한][ms al apprego roles]을 사용한다. 권한의 범위는 `User.Read.All` 정도면 충분하다.

위와 같이 앱을 설정한 후에는 고유의 `TenantID`, `ClientID`, `ClientSecret` 값이 부여된다.


## Microsoft Authentication Library (MSAL) for .NET ##

우선 애저 액티브 디렉토리에 저장된 사용자 데이터를 조회하기 위해서는 인증을 해야 한다. 다양한 방법으로 로그인 할 수 있지만, 여기서는 사용자의 로그인 개입 없이 API가 직접 로그인할 수 있도록 하는 [클라이언트 자격증명 방법][ms al clientcredential]을 사용하기로 한다. 이를 위해서는 아래 NuGet 패키지가 필요하다.

* [Microsoft.Identity.Client][nuget msal]: `dotnet add package Microsoft.Identity.Client`

위 패키지를 설치하고 난 후에 `local.settings.json` 파일에 아래와 같은 환경 변수를 추가한다. 불필요한 부분은 제외하고 인증에 필요한 부분만 남겨뒀다.

https://gist.github.com/justinyoo/38ae8fdd9ac6161551a9aee0d15b76e7?file=07-local-settings.json

이후 아래와 같은 코드를 작성한다. 아래는 별도의 사용자 상호작용 없이 ClientID 값과 ClientSecret 값만으로 액세스 토큰을 받는 방법이다. [`ConfidentialClientApplicationBuilder`][ms al clientcredential builder] 클라스를 이용하면 쉽게 액세스 토큰을 받을 수 있다 (line #16-20).

https://gist.github.com/justinyoo/38ae8fdd9ac6161551a9aee0d15b76e7?file=08-get-access-token.cs&highlights=16-20

이렇게 받아온 액세스 토큰을 이용하면 그 다음부터는 자유롭게 [Microsoft Graph][ms graph] API를 사용할 수 있다.


## Microsoft Graph API for .NET ##

Microsoft Graph API 사용을 위해서는 아래 NuGet 패키지가 필요하다.

* [Microsoft.Graph][nuget msgraph]: `dotnet add package Microsoft.Graph`

그리고 아래와 같이 Graph API에 접근하기 위한 코드를 작성한다. 위에 작성했던 액세스 토큰을 받는 메소드인 `GetAccessTokenAsync()`를 여기서 호출한다 (line #4-8).

https://gist.github.com/justinyoo/38ae8fdd9ac6161551a9aee0d15b76e7?file=09-get-graph-client.cs&highlights=4-8

마지막으로 애저 펑션 메소드 안에서 위에 작성한 Microsoft Graph API 클라이언트 생성 메소드인 `GetGraphClientAsync()`를 호출한 후 (line #1), 처음에 만들어 놓은 `ClientPrincipal` 개체의 이메일 주소를 이용해 사용자 정보를 조회한다 (line #4). 만약 해당 이메일 주소로 조회가 안 될 경우에는 해당 로그인 사용자는 조회하고자 하는 테넌트에 게스트 혹은 외부 사용자로 추가되지 않았기 때문에 `404 Not Found` 결과를 반환한다 (line #7).

https://gist.github.com/justinyoo/38ae8fdd9ac6161551a9aee0d15b76e7?file=10-get-user.cs&highlights=1,4,7

사용자 정보를 받아왔다면 이는 아래와 같이 굉장히 방대한 양을 가지고 있다.

https://gist.github.com/justinyoo/38ae8fdd9ac6161551a9aee0d15b76e7?file=11-user.json

하지만, 굳이 반환 개체에 이 수많은 정보를 노출시킬 필요는 없으므로 아래와 같이 내가 필요한 정보만 담아두는 개체를 생성한다.

https://gist.github.com/justinyoo/38ae8fdd9ac6161551a9aee0d15b76e7?file=12-loggedin-user.cs

그리고, 이를 이용해서 반환에 필요한 `LoggedInUser` 개체로 변환한 후 반환한다.

https://gist.github.com/justinyoo/38ae8fdd9ac6161551a9aee0d15b76e7?file=13-get-loggedin-user.cs

이렇게 애저 펑션으로 사용자 정보를 조회하고 반환하는 API를 작성했다.


## 애저 정적 웹 앱 사용자 정보 조회 ##

이제 블레이저 웹어셈블리 앱에서 위 API를 호출해서 사용자 정보를 조회한다. 아래는 `try { ... } catch { ... }` 블록으로 감쌌는데, 만약 API 호출시 에러가 난다면 별다른 조치 없이 조용하게 `null` 값을 반환하게 하기 위함이다. 실제로는 좀 더 정교하게 에러 처리를 해야겠지만, 여기서는 편의상 이렇게 처리했다.

https://gist.github.com/justinyoo/38ae8fdd9ac6161551a9aee0d15b76e7?file=14-get-loggedin-user.cs

위의 메소드를 블레이저 컴포넌트에서 호출하는 로직을 아래와 같이 작성해 보자. 편의상 불필요한 코드는 삭제했다 (line #6, 18).

https://gist.github.com/justinyoo/38ae8fdd9ac6161551a9aee0d15b76e7?file=15-get-loggedin-user-ko.razor&highlights=6,18

만약 로그인 한 사용자의 정보가 조회하고자 하는 테넌트에 있다면 아래와 같이 나올 것이다.

![로그인 후 화면 - 사용자 정보 있음][image-02]

만약 로그인 한 사용자의 정보가 조회하고자 하는 테넌트에 없다면 아래와 같이 나올 것이다.

![로그인 후 화면 - 사용자 정보 없음][image-03]

이렇게 해서 사용자 정보를 조회하고 화면에 출력하는 코드를 작성해 봤다.

---

지금까지 [애저 정적 웹 앱][az swa] 인스턴스에 [블레이저 웹어셈블리][blazor wasm] 앱을 호스팅하고 [애저 펑션][az fncapp] API를 통해 [MSAL][ms al]로 [애저 액티브 디렉토리][az ad]에 인증하고, [Microsoft Graph][ms graph] API를 이용해 사용자 정보를 가져오는 방법에 대해 알아보았다. Microsoft Graph API는 [Microsoft 365][m365]의 거의 모든 리소스에 접근이 가능한 만큼 이를 이용하면 [셰어포인트][m365 spo], [팀즈][m365 teams] 같은 다른 서비스들도 쉽게 이용할 수 있을 것이다.


[image-01]: https://sa0blogs.blob.core.windows.net/aliencube/2021/09/accessing-msgraph-from-blazor-wasm-running-on-aswa-01-ko.png
[image-02]: https://sa0blogs.blob.core.windows.net/aliencube/2021/09/accessing-msgraph-from-blazor-wasm-running-on-aswa-02-ko.png
[image-03]: https://sa0blogs.blob.core.windows.net/aliencube/2021/09/accessing-msgraph-from-blazor-wasm-running-on-aswa-03-ko.png


[gh sample]: https://github.com/fusiondevkr/fusiondevkr

[az swa]: https://docs.microsoft.com/ko-kr/azure/static-web-apps/overview?WT.mc_id=dotnet-42714-juyoo
[az swa authn]: https://docs.microsoft.com/ko-kr/azure/static-web-apps/authentication-authorization?WT.mc_id=dotnet-42714-juyoo
[az swa api]: https://docs.microsoft.com/ko-kr/azure/static-web-apps/apis?WT.mc_id=dotnet-42714-juyoo

[az fncapp]: https://docs.microsoft.com/ko-kr/azure/azure-functions/functions-overview?WT.mc_id=dotnet-42714-juyoo

[blazor wasm]: https://docs.microsoft.com/ko-kr/aspnet/core/blazor/?WT.mc_id=dotnet-42714-juyoo

[az ad]: https://docs.microsoft.com/ko-kr/azure/active-directory/fundamentals/active-directory-whatis?WT.mc_id=dotnet-42714-juyoo

[ms al]: https://docs.microsoft.com/ko-kr/azure/active-directory/develop/msal-overview?WT.mc_id=dotnet-42714-juyoo
[ms al clientcredential]: https://docs.microsoft.com/ko-kr/azure/active-directory/develop/v2-oauth2-client-creds-grant-flow?WT.mc_id=dotnet-42714-juyoo#get-a-token
[ms al clientcredential builder]: https://docs.microsoft.com/ko-kr/dotnet/api/microsoft.identity.client.confidentialclientapplicationbuilder?WT.mc_id=dotnet-42714-juyoo
[ms al apprego]: https://docs.microsoft.com/ko-kr/azure/active-directory/develop/quickstart-register-app?WT.mc_id=dotnet-42714-juyoo
[ms al apprego roles]: https://docs.microsoft.com/ko-kr/azure/active-directory/develop/quickstart-configure-app-access-web-apis?WT.mc_id=dotnet-42714-juyoo#application-permission-to-microsoft-graph

[ms graph]: https://docs.microsoft.com/ko-kr/graph/overview?WT.mc_id=dotnet-42714-juyoo

[nuget msal]: https://www.nuget.org/packages/Microsoft.Identity.Client/
[nuget msgraph]: https://www.nuget.org/packages/Microsoft.Graph/

[m365]: https://www.microsoft.com/ko-kr/microsoft-365?WT.mc_id=dotnet-42714-juyoo
[m365 spo]: https://www.microsoft.com/ko-kr/microsoft-365/sharepoint/collaboration?WT.mc_id=dotnet-42714-juyoo
[m365 teams]: https://www.microsoft.com/ko-kr/microsoft-teams/group-chat-software?WT.mc_id=dotnet-42714-juyoo
