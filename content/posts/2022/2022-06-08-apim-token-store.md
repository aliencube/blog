---
title: "애저 API 관리자의 효과적인 OAuth 인증 관리"
slug: apim-token-store
description: "이 포스트에서는 애저 API 관리자에서 프리뷰 기능으로 제공하는 써드파티 OAuth 앱의 인증키를 자동으로 관리해 주는 기능에 대해 알아봅니다."
date: "2022-06-08"
author: Justin-Yoo
tags:
- azure
- api-management
- oauth-token-store
- authorizations
cover: /2022/06/apim-token-store-00.png
fullscreen: true
---

애플리케이션을 개발하다 보면 많은 경우 다른 서비스와 메시지를 주고 받기 위해 API를 사용한다. 이 때 공개 API가 아닌 이상은 반드시 어떤 형태로든 인증 과정을 거치게 되는데, 이 때 가장 널리 쓰이는 방식이 1) API 인증 키를 사용하는 방법 혹은 2) OAuth 인증을 통해 액세스 토큰을 받아 사용하는 방법 등이 있다.

만약 API 인증 키를 이용하려고 하면 해당 인증 키를 [애저 키 저장소][az kv]와 같은 안전한 곳에 저장해 두고 필요에 따라 불러서 쓰면 되기 때문에 상대적으로 간편하다. 하지만 OAuth 인증을 사용하려면 좀 더 복잡한데, 대략 아래와 같은 흐름을 거쳐서 API를 호출하게 된다.

1. 인증 코드를 요청한다
2. 인증 코드를 받는다.
3. 인증 코드를 통해 액세스 토큰을 요청한다.
4. 액세스 토큰과 리프레시 토큰을 받는다.
5. 액세스 토큰을 통해 API를 호출한다.
6. API 호출 결과를 받는다.
7. 액세스 토큰 유효기간이 지났다면 리프레시 토큰을 통해 액세스 토큰을 요청한다.
8. 새 액세스 토큰과 새 리프레시 토큰을 받는다.
9. 새 액세스 토큰을 통해 API를 호출한다.
10. API 호출 결과를 받는다.

![Conceptual OAuth Flow][image-01]

> OAuth 인증 플로우와 관련한 좀 더 자세한 내용은 [이 문서][oauth flow concept]를 참고하면 좋다.

따라서, 위와 같은 일련의 과정을 개발자가 직접 애플리케이션을 개발할 때 구현해야 한다. 그나마 SDK가 있는 서비스일 경우에는 상대적으로 간단해지긴 하겠지만, 그렇지 않을 경우에는 굉장히 복잡한 구현이 필요할 수도 있다. 그런데, 만약 이 모든 과정을 누가 대신해 준다면 어떨까? 보안상 안전한 곳에서 이 모든 과정을 누군가 대신해 주고 개발자에게는 액세스 토큰만 바로바로 발급해 준다면 애플리케이션 개발에 있어서 아주아주 편리한 기능이 될 수 있을 것이다. 마침 [애저 API 관리자][az apim]에서 이 [OAuth 인증 과정을 단축시켜 주는 기능을 프리뷰로 선보였는데][az apim tokenstore], 이 포스트에서는 이 과정에 대해 한 번 알아보기로 한다.

> 여기서 쓰인 예제 애플리케이션 코드는 [이곳][gh sample]에서 다운로드 받을 수 있다.

위의 샘플 예제 앱에는 두가지 앱이 있다. 하나는 [블레이저 웹 어셈블리][blazor wasm] 기반 앱이고 다른 하나는 리액트 기반의 앱이다. 여기서는 블레이저 웹 어셈블리 앱 기반으로 설명한다.

> 만약 리액트 기반의 앱을 구성했는지 보고 싶다면 직장 동료인 [Aaron Powell][tw aaron]이 작성한 [블로그 포스트][post 1]를 참조한다 (영문).


## 블레이저 웹 어셈블리 앱 ##

블레이저 웹 어셈블리 앱을 생성하는 절차는 [이 문서][blazor tutorial]를 참조하고, 여기서는 이 인증 코드를 호출하는 컴포넌트만 살펴본다. 이 컴포넌트는 사용자의 정보를 입력 받아 드롭박스에 .csv 형태로 파일을 저장한다. 아래 Razor 코드를 보자. 딱히 특별한 내용은 없고, 사용자의 입력을 받아 처리하는 형태로 구성되어 있다. 사용자가 데이터를 입력한 후 버튼을 눌러 제출하면 `OnFormSubmittedAsync` 이벤트가 작동하는 구조이다 (line #3).

```razor
<div class="container-sm" style="max-width: 540px;">
  <h1>Blazor Lead Capture</h1>
  <form class="clearfix" @onsubmit="OnFormSubmittedAsync">
    <fieldset>
      <div>
        <label for="firstName" class="form-label">First name</label>
        <input type="text" class="form-control" id="firstName" name="firstName" placeholder="Justin" value="@userInfo.FirstName" @onchange="@(e => OnFieldChanged(e, "firstName"))" />
      </div>
      <div>
        <label for="lastName" class="form-label">Last name</label>
        <input type="text" class="form-control" id="lastName" name="lastName" placeholder="Yoo" value="@userInfo.LastName" @onchange="@(e => OnFieldChanged(e, "lastName"))" />
      </div>
    </fieldset>

    <fieldset>
      <div>
        <label htmlFor="email" class="form-label">Email</label>
        <input type="email" class="form-control" id="email" name="email" placeholder="bar@email.com" value="@userInfo.Email" @onchange="@(e => OnFieldChanged(e, "email"))" />
      </div>
      <div>
        <label htmlFor="phone" class="form-label">Phone</label>
        <input type="phone" class="form-control" id="phone" name="phone" placeholder="555-555-555" value="@userInfo.Phone" @onchange="@(e => OnFieldChanged(e, "phone"))" />
      </div>
    </fieldset>

    <fieldset>
      <button type="submit" class="btn btn-@componentUIInfo.ButtonColour" disabled="@(componentUIInfo.Submitting || string.IsNullOrWhiteSpace(userInfo.FirstName) || string.IsNullOrWhiteSpace(userInfo.LastName) || string.IsNullOrWhiteSpace(userInfo.Email) || string.IsNullOrWhiteSpace(userInfo.Phone))">
        <span>Submit</span>
        <span class="spinner-border spinner-border-sm" style="display:@componentUIInfo.DisplaySpinner;" role="status" aria-hidden="true"></span>
      </button>
    </fieldset>
  </form>

  <div class="alert alert-@componentUIInfo.AlertResult" style="display:@componentUIInfo.DisplayResult;">
    <h2>@componentUIInfo.MessageResult</h2>
    <button type="reset" class="btn btn-dark" @onclick="ResetFields">
      <span>Start Over?</span>
    </button>
  </div>
</div>
```

이번에는 이 UI를 제어하는 C# 코드 부분을 살펴보자. 편의상 다른 부분은 생략했고, 가장 중요한 메소드 두 개만 남겨뒀다. 위의 폼에서 버튼을 클릭하면 아래와 같이 `OnFormSubmittedAsync` 이벤트 핸들러가 작동하고, 이는 `SaveToDropboxAsync` 메소드를 호출하게 되는데, 이 메소드가 실질적으로 드롭박스에 데이터를 저장하는 역할을 한다.

```csharp
@code {
    ...
    protected async Task OnFormSubmittedAsync(EventArgs e)
    {
        ...
        await SaveToDropboxAsync().ConfigureAwait(false);
    }
```

아래 메소드를 보면, 가장 먼저 환경 변수로 선언한 `APIM_Endpoint` 값을 가져온다 (line #4). 블레이저 웹 어셈블리에서는 이 환경 변수값을 `appsettings.json` 파일에 저장하는데, 이는 아래에서 다시 다뤄보기로 한다. 이렇게 받아온 API 관리자 엔드포인트를 호출하면 액세스 토큰이 딸려오고 (line #7), 이를 이용해서 드롭박스에 파일을 생성하고 저장하는 일을 한다.

```csharp
    private async Task SaveToDropboxAsync()
    {
        // Gets the APIM endpoint from appsettings.json
        var requestUrl = Configuration.GetValue<string>("APIM_Endpoint");
        
        // Gets the auth token from APIM
        var token = await Http.GetStringAsync(requestUrl).ConfigureAwait(false);

        // Builds contents.
        var path = $"/submissions/{DateTimeOffset.UtcNow.ToString("yyyyMMddHHmmss")}.csv";
        var contents = $"{userInfo.FirstName},{userInfo.LastName},{userInfo.Email},{userInfo.Phone}";
        var bytes = UTF8Encoding.UTF8.GetBytes(contents);

        // Uploads the contents.
        var result = default(FileMetadata);
        using(var dropbox = new DropboxClient(token))
        using(var stream = new MemoryStream(bytes))
        {
            result = await dropbox.Files.UploadAsync(path, WriteMode.Overwrite.Instance, body: stream).ConfigureAwait(false);
        }

        ...
    }
}
```

그렇다면, `APIM_Endpoint` 환경 변수가 저장되어 있는 `appsettings.json` 파일을 살펴보자. 아래와 같이 API 관리자에 정의되어 있는 드롭박스 액세스 토큰을 호출하는 API 엔드포인트가 된다.

```json
{
  "APIM_Endpoint": "https://<APIM_NAME>.azure-api.net/dropbox-demo/token?subscription-key=<APIM_SUBSCRIPTION_KEY>"
}
```

이렇게 한 후 애플리케이션을 실제로 실행시켜보자.

```powershell
dotnet watch run
```

브라우저를 열어 `https://localhost:5001`로 접속하면 아래와 같은 화면이 보일 것이다.

![Blazor WASM Landing Page][image-02]

여기에 필요한 정보를 모두 입력한 후 "**Submit**" 버튼을 클릭하면 드롭박스에 내용이 저장된다. 이 때 웹 브라우저의 개발자도구를 열어 보면 API 관리자의 액세스 토큰을 요청하는 URL을 호출한 것이 보인다.

![Request Access Token #1][image-03]

이 요청을 통해 반환되는 값이 바로 드롭박스에 접근할 수 있는 액세스 토큰이다.

![Request Access Token #2][image-04]

이 액세스 토큰을 이용해 드롭박스에 파일을 생성하고 저장하게 되는데, 그 결과는 아래와 같다.

![Dropbox Upload Result][image-05]

위의 블레이저 웹 어셈블리 앱 실행 과정에서 어디에더 드롭박스 접근을 위해 액세스 토큰을 얻기 위한 OAuth 인증 관련 코드가 없다. 오히려 API 관리자를 통한 최초 API 호출을 통해 직접 액세스 토큰을 받아오는 것부터 시작하는데, 액세스 토큰 발급 이전의 수많은 단계를 어떻게 생략할 수 있었을까? 이 기능이 바로 이 포스트에서 언급하고자 하는 API 관리자의 OAuth 인증 관리 기능이다. API 관리자에서 내부적으로 인증 과정을 대신해 주고 액세스 토큰을 받아주는 기능이라고 할 수 있다.


## 애저 API 관리자 인스턴스 ##

그렇다면, 애저 API 관리자에서는 이를 어떻게 관리할까? 우선 아래와 같이 애저 API 관리자 인스턴스를 bicep을 통해 선언한다. 이 때 이 APIM 인스턴스는 반드시 [관리 ID][az apim mi]를 활성화시켜야 한다 (line #13-15). 이 부분은 아래에서 다시 설명하기로 한다. 편의상 인스턴스 이름은 `token-store-demo-apim`, 지역은 `West Central US`로 지정한다. 이 때 이 인스턴스가 만들어지는 리소스 그룹은 `rg-token-store-demo`라고 가정한다.

```javascript
// APIM instance
resource apim 'Microsoft.ApiManagement/service@2021-08-01' = {
  name: 'token-store-demo-apim'
  location: 'westcentralus'
  sku: {
    name: 'Developer'
    capacity: 1
  }
  properties: {
    publisherName: 'John Doe'
    publisherEmail: 'john.doe@nomail.com'
  }
  identity: {
    type: 'SystemAssigned'
  }
}
```

다음으로는 이 APIM 인스턴스의 도메인 주소가 달라도 블레이저 웹 어셈블리 앱 안에서 작동할 수 있게끔 CORS 정책을 설정한다. `inbound` 노드 안에 `cors` 노드를 추가하고 그 밑에 `allowed-origins` 설정에서 `*` 값을 추가한다. 하지만, 여기서는 편의상 이렇게 설정할 뿐이지 실제로는 필요한 애저 정적 웹 앱의 URL만 추가하는 것이 보안 측면에서 가장 적절하다.

```javascript
// Service Policy
resource apim_policy 'Microsoft.ApiManagement/service/policies@2021-08-01' = {
  parent: apim
  name: 'policy'
  properties: {
    value: service_policy
    format: 'xml'
  }
}

// Service Policy Definition
var service_policy = '''
<policies>
    <inbound>
        <cors allow-credentials="false">
            <allowed-origins>
                <origin>*</origin>
            </allowed-origins>
            <allowed-methods>
                <method>GET</method>
                <method>POST</method>
            </allowed-methods>
        </cors>
    </inbound>
    <backend>
        <forward-request />
    </backend>
    <outbound />
    <on-error />
</policies>'''
```

APIM 인스턴스가 위와 같이 만들어졌다면, 그 안에 API를 추가해야 한다. 아래와 같이 API와 엔드포인트를 추가한다. API 관리자가 호출하는 드롭박스 API의 호스트를 `serviceUrl` 속성에 할당하고, 이 API를 통해 액세스 토큰을 받아오는 엔드포인트를 `/token`으로 선언한다. 따라서, 이 bicep 선언에 따르면 전체적인 API 엔드포인트 URL의 구조는 `https://token-store-demo-apim.azure-api.net/dropbox-demo/token`가 된다.

```javascript
// API
resource api 'Microsoft.ApiManagement/service/apis@2021-08-01' = {
  name: 'dropbox-demo'
  parent: apim
  properties: {
    serviceUrl:'https://api.dropboxapi.com'
    path: 'dropbox-demo'
    displayName:'dropbox-demo'
    protocols:[
      'https'
    ]
  }
}

// Operation
resource api_gettoken 'Microsoft.ApiManagement/service/apis/operations@2021-08-01' = {
  name: 'gettoken'
  parent: api
  properties: {
    method: 'GET'
    urlTemplate: '/token'
    displayName: 'gettoken'
  }
}
```

이제 이 엔드포인트를 통해 액세스 토큰을 직접 받아와야 하는데, 이를 위한 엔드포인트 정책을 아래와 같이 선언한다.

```javascript
// Operation Policy
resource api_gettoken_policy 'Microsoft.ApiManagement/service/apis/operations/policies@2021-08-01' = {
  parent: api_gettoken
  name: 'policy'
  properties: {
    value: operation_token_policy
    format: 'xml'
  }
}
```

아래 XML 문서의 내용이 액세스 토큰을 받아오는 정책에 대해 정의한 부분이다. `get-authorization-context` 노드의 속성값을 아래와 같이 정의한다.

* `provider-id`: `dropbox-demo`
* `authorization-id`: `auth`
* `context-variable-name`: `auth-context`
* `identity-type`: `managed`

앞서 API 관리자 인스턴스를 생성할 때 관리 ID 기능을 활성화 시켰는데, 그 부분을 바로 `identity-type`에서 활용한다. `context-variable-name` 값으로 지정한 `auth-context`는 그 아래 `return-response` 노드에서 인증 컨텍스트를 추출할 때 사용한다. 마지막으로 `provider-id`, `authorization-id` 값은 아래에서 다시 설명하겠지만, API 관리자의 OAuth 인증 대행 기능을 위해 사용한다.

```javascript
// Operation Token Policy Definition
var operation_token_policy = '''
<policies>
    <inbound>
        <base />
        <get-authorization-context provider-id="dropbox-demo" authorization-id="auth" context-variable-name="auth-context" ignore-error="false" identity-type="managed" />
        <return-response>
            <set-body>@(((Authorization)context.Variables.GetValueOrDefault(&quot;auth-context&quot;))?.AccessToken)</set-body>
        </return-response>
    </inbound>
    <backend>
        <base />
    </backend>
    <outbound>
        <base />
    </outbound>
    <on-error>
        <base />
    </on-error>
</policies>'''
```

위와 같이 APIM 인스턴스를 생성했다면, 이제는 앞서 작성한 블레이저 웹 어셈블리 앱을 애저 정적 웹 앱으로 배포할 차례이다.


## 애저 정적 웹 앱 인스턴스 생성 ##

블레이저 웹 어셈블리 앱을 호스팅하기 위해서는 [애저 정적 웹 앱][az swa] 인스턴스를 하나 생성하면 된다. 아래 bicep 코드를 통해 생성해 보기로 하자. 정적 웹 앱 인스턴스 이름은 `token-store-demo-blazor-swa`, 지역은 `Central US`로 선언한다. 이 때 이 인스턴스가 만들어지는 리소스 그룹은 `rg-token-store-demo`라고 가정한다.

```javascript
// SWA instance
resource sttapp 'Microsoft.Web/staticSites@2021-02-01' = {
  name: 'token-store-demo-blazor-swa'
  location: 'centralus'
  sku: {
    name: 'Free'
  }
  properties: {
    allowConfigFileUpdates: true
    stagingEnvironmentPolicy: 'Enabled'
  }
}
```


## 애저 정적 웹 앱에 블레이저 웹 어셈블리 앱 배포 ##

위와 같이 애저 정적 웹 앱 인스턴스를 생성했다면, 이제 여기에 블레이저 웹 어셈블리 앱을 배포해 보기로 하자. 현재 블레이저 앱은 `src/frontend/blazor` 디렉토리에 있고, 가장 먼저 할 일은 블레이저 웹 어셈블리 앱이 위에서 생성한 APIM 인스턴스에 접속할 수 있는 엔드포인트에 접근할 수 있게끔 `appsettings.json` 파일을 수정하는 것이다. 아래 명령어를 통해 APIM 엔드포인트를 받아온다.

```powershell
# Get APIM gateway URL
rg_name=rg-token-store-demo
apim_name=token-store-demo-apim

gateway_url=$(az apim show -g $rg_name -n $apim_name --query "gatewayUrl" -o tsv)

# Get APIM subscription key
subscription_id=$(az account show --query "id" -o tsv)
apim_secret_uri=/subscriptions/$subscription_id/resourceGroups/$rg_name/providers/Microsoft.ApiManagement/service/$apim_name/subscriptions/master/listSecrets
api_version=2021-08-01

subscription_key=$(az rest --method post --uri $apim_secret_uri\?api-version=$api_version | jq '.primaryKey' -r)

# Build APIM endpoint
apim_endpoint=$gateway_url/dropbox-demo/token\?subscription-key=$subscription_key
```

위의 과정을 거치고 난 후 `apim_endpoint` 변수에 할당된 엔드포인트 URL을 사용하면 된다. 아래와 같이 `src/frontend/blazor/wwwroot` 아래에 있는 `appsettings.sample.json` 파일의 이름을 `appsettings.json`으로 수정하고 엔드포인트를 수정한다.

```json
{
  "APIM_Endpoint": "<apim_endpoint>"
}
```

아래 명령어를 이용해서 아티팩트를 빌드한다.

```powershell
dotnet restore ./src/frontend/blazor
dotnet build ./src/frontend/blazor
dotnet publish ./src/frontend/blazor -c Release -o ./src/frontend/blazor/bin
```

위 명령어를 통해 아티팩트를 만들었다면, 애저 정적 웹 앱에 배포하기 위한 배포 키를 아래 명령어를 통해 받는다.

```powershell
swa_key=$(az staticwebapp secrets list \
    -g rg-token-store-demo \
    -n token-store-demo-blazor-swa \
    --query "properties.apiKey" -o tsv)
```

마지막으로, 아래 명령어를 통해 배포한다.

```powershell
swa deploy -a ./src/frontend/blazor/bin/wwwroot -d $swa_key --env default
```

그런데, 블레이저 웹 어셈블리 앱을 배포하고 난 후 애저 정적 웹 앱에 접속해서 데이터를 입력하고 나면 아래와 같은 에러가 발생한다.

![Internal Server Error][image-06]

앞서 로컬에서 작동시킬 때에 사용한 APIM 엔드포인트는 이미 모든 구성이 다 끝난 상태였으므로 사용하는 것에 아무 문제가 없었다. 하지만, 이번에는 새로운 APIM 엔드포인트를 사용하면서 아직 마지막 환경 구성을 끝내지 않았기 때문에 다음에 오는 마지막 작업을 해 줘야 한다.


## 드롭박스 앱 최초 인증 ##

아직까지는 APIM 인스턴스가 어떻게 드롭박스 앱에 접근할 것인지에 대한 최초 인증을 하지 않은 상태이므로, 드롭박스 앱을 인증해 줘야 한다. 이를 위해서는 가장 먼저 드롭박스 앱이 필요한데, 자세한 내용은 [이 문서][dropbox appconsole]를 참조한다. 드롭박스 앱을 만들고 나면 `App key`, `App secret` 값이 만들어진다.

![Dropbox App][image-07]

이 값을 이용해 APIM 인스턴스에서 최초 인증을 한다. 아래 그림과 같이 왼쪽의 블레이드에서 "**Authorizations (preview)**" 메뉴를 클릭한다.

![Authorizations Preview][image-08]

그러면 아래와 같이 현재는 아무런 인증 앱이 없다고 나온다. "**Create**" 버튼을 클릭한다.

![Authorizations Preview Pane][image-09]

앞서 APIM 인스턴스를 생성할 때 API 정책을 정할 때 `get-authorization-context` 노드에 적용했던 여러 가지 속성들을 여기서 사용할 차례이다.

* "**Provider name**" 필드에 `dropbox-demo`를 입력한다.
* "**Identity provider**" 필드는 `DropBox`를 선택한다.
* "**Client id**" 필드는 드롭박스의 App key 값을 입력한다.
* "**Client secret**" 필드는 드롭박스의 App secret 값을 입력한다.
* "**Scopes**" 필드에는 `files.metadata.write files.content.write files.content.read`을 입력한다.
* "**Authorization name**" 필드에는 `auth`를 입력한다.

![Create Authorization][image-10]

위와 같이 입력한 후 "**Create**" 버튼을 클릭하면 아래와 같이 리디렉션 URL이 생성되는데, 이 값을 복사한다.

![Redirection URL][image-11]

복사한 리디렉션 URL 값을 드롭박스 앱에 추가한다.

![DropBox App Update][image-12]

다시 APIM 인스턴스로 돌아와서 아래와 같이 드롭박스 앱에 로그인한다. 별도의 팝업창이 나타나면 "**Continue**", "**Allow**", "**Allow access**" 버튼을 차례로 클릭해서 진행한다.

![DropBox App Login on APIM][image-13]

그러면 아래와 같이 드롭박스 앱에 대한 인증을 성공적으로 마쳤다는 메시지가 나타난다.

![DropBox App Authorized][image-14]

APIM 인스턴스가 드롭박스 앱을 사용하기 위한 인증이 끝났다면, 이제는 드롭박스 앱에 APIM 인스턴스가 접근하기 위한 인증을 할 차례이다. 처음 APIM 인스턴스를 생성할 때, 관리 ID 기능을 활성화시켜뒀으므로, 이를 이용한다. 아래와 같이 "**Managed identity**"를 선택하고 "**Add members**" 버튼을 클릭한다.

![APIM Managed Identity][image-15]

그리고 아래 그림과 같이 현재 APIM 인스턴스를 찾아 추가한다.

![Add APIM Managed Identity][image-16]

그러면 이제 드롭박스 인증이 끝났고, APIM 인스턴스도 자유롭게 드롭박스 앱에 접근할 수 있게 되었다.

![APIM Managed Identity Added][image-17]

이제 아래와 같이 드롭박스 인증이 추가된 것을 볼 수 있다.

![DropBox Authorization Created][image-18]

그렇다면, 실제로 이 인증 관리 기능이 제대로 작동하는지 확인해 보기로 하자. 아래 그림과 같이 왼쪽의 "**APIs**" 블레이드 ➡️ "**dropbox-demo**" API ➡️ "**gettoken**" 엔드포인트를 거쳐 "**Send**" 버튼을 클릭한다.

![APIM Test][image-19]

그러면 성공적으로 드롭박스의 액세스 토큰이 발급된 것이 보인다.

![APIM Test Success][image-20]

다시 애저 정적 웹 앱으로 돌아가서 데이터를 입력해 보자. 이번에는 에러 없이 제대로 API를 호출한 것이 보인다.

![Static Web App to APIM Request Success][image-21]

그리고 드롭박스에도 제대로 데이터를 업로드한 것이 보인다.

![DropBox File Created][image-22]

---

지금까지 [애저 API 관리자][az apim]의 새로운 기능인 OAuth 인증 관리에 대해 살펴보았다. OAuth 인증을 통해 궁극적으로 우리가 원하는 것은 액세스 토큰이고, 이 액세스 토큰을 받아오기 위해 거쳐야 하는 과정을 APIM 인스턴스에서 대행해 준다. 따라서 애플리케이션 개발 과정에서는 이 인증 기능 구현을 위한 노력을 획기적으로 줄일 수 있으므로 개발 속도가 훨씬 빨라질 수 있게 되는 것을 알 수 있다. 현재는 프리뷰 기능이기 때문에 지원하는 OAuth 앱 제공자가 한정적이지만 계속해서 기능이 추가될 예정이니 시간이 지나면 좀 더 편리하게 사용할 수 있을 것이다.

만약 이 APIM OAuth 인증 관리 기능에 대해 좀 더 자세히 알고 싶다면 아래 문서를 참조한다.

* [애저 API 관리자에서 인증 기능 관리][az apim tokenstore]



[image-01]: /2022/06/apim-token-store-01.png
[image-02]: /2022/06/apim-token-store-02.png
[image-03]: /2022/06/apim-token-store-03.png
[image-04]: /2022/06/apim-token-store-04.png
[image-05]: /2022/06/apim-token-store-05.png
[image-06]: /2022/06/apim-token-store-06.png
[image-07]: /2022/06/apim-token-store-07.png
[image-08]: /2022/06/apim-token-store-08.png
[image-09]: /2022/06/apim-token-store-09.png
[image-10]: /2022/06/apim-token-store-10.png
[image-11]: /2022/06/apim-token-store-11.png
[image-12]: /2022/06/apim-token-store-12.png
[image-13]: /2022/06/apim-token-store-13.png
[image-14]: /2022/06/apim-token-store-14.png
[image-15]: /2022/06/apim-token-store-15.png
[image-16]: /2022/06/apim-token-store-16.png
[image-17]: /2022/06/apim-token-store-17.png
[image-18]: /2022/06/apim-token-store-18.png
[image-19]: /2022/06/apim-token-store-19.png
[image-20]: /2022/06/apim-token-store-20.png
[image-21]: /2022/06/apim-token-store-21.png
[image-22]: /2022/06/apim-token-store-22.png


[post 1]: https://techcommunity.microsoft.com/t5/apps-on-azure-blog/implementing-a-token-store-with-apim-authorizations/ba-p/3453516?WT.mc_id=dotnet-57408-juyoo

[gh sample]: https://github.com/aaronpowell/token-store-demo

[tw aaron]: https://twitter.com/slace

[blazor]: https://dotnet.microsoft.com/apps/aspnet/web-apps/blazor?WT.mc_id=dotnet-57408-juyoo
[blazor wasm]: https://docs.microsoft.com/ko-kr/aspnet/core/blazor/host-and-deploy/webassembly?view=aspnetcore-6.0&WT.mc_id=dotnet-57408-juyoo
[blazor tutorial]: https://dotnet.microsoft.com/apps/aspnet/web-apps/blazor?WT.mc_id=dotnet-57408-juyoo

[az swa]: https://docs.microsoft.com/ko-kr/azure/static-web-apps/overview?WT.mc_id=dotnet-57408-juyoo

[az apim]: https://docs.microsoft.com/ko-kr/azure/api-management/api-management-key-concepts?WT.mc_id=dotnet-57408-juyoo
[az apim mi]: https://docs.microsoft.com/ko-kr/azure/api-management/api-management-howto-use-managed-service-identity?WT.mc_id=dotnet-57408-juyoo
[az apim tokenstore]: https://docs.microsoft.com/ko-kr/azure/api-management/authorizations-overview?WT.mc_id=dotnet-57408-juyoo

[az kv]: https://docs.microsoft.com/ko-kr/azure/key-vault/general/basic-concepts?WT.mc_id=dotnet-57408-juyoo

[oauth flow concept]: https://docs.microsoft.com/ko-kr/azure/active-directory/develop/v2-oauth2-auth-code-flow?WT.mc_id=dotnet-57408-juyoo

[dropbox appconsole]: https://www.dropbox.com/developers/reference/getting-started#app%20console
