---
title: "블레이저 웹 어셈블리로 헤드리스 CMS 만들어보기"
slug: building-headless-cms-with-blazor-wasm
description: "이 포스트에서는 워드프레스 사이트를 헤드리스 CMS로 이용해 블레이저 웹 어셈블리 앱에서 구현하고 애저 정적 웹 앱으로 호스팅해 봅니다."
date: "2022-05-11"
author: Justin-Yoo
tags:
- azure
- blazor-wasm
- staticwebapps
- headless-cms
cover: /2022/05/building-headless-cms-with-blazor-wasm-00.png
fullscreen: true
---

정적 웹사이트 구현 시나리오 중 하나는 바로 자신의 블로그 사이트를 운영하는 것이다. 이 때 만약 그동안 서비스형 워드프레스를 이용해 블로그 사이트를 운영해 왔다면, 이를 정적 웹사이트로 이전하는 작업 역시 만만치 않다. 그런데, 만약 기존의 워드프레스 사이트를 그대로 두고 껍데기만 정적 웹사이트로 구현 가능하다면 어떨까? 게다가 그 정적 웹사이트를 C#을 이용한 [블레이저 웹어셈블리][blazor wasm] 형식으로 구현할 수 있다면? 기존의 워드프레스 사이트는 데이터 저장소 용도로 사용하고 나머지는 내 맘대로 작성할 수 있다면 무척이나 편리할 것이다.

이 포스트에서는 워드프레스를 데이터 저장소이자 헤드리스 CMS로 두고, 블레이저 웹어셈블리를 이용해 정적 웹사이트를 만든 후 이를 [애저 정적 웹 앱][az swa] 인스턴스에 배포하는 것 까지 한 번 해 보기로 한다.

> 이 포스트에서 사용한 샘플 애플리케이션 코드는 [깃헙 리포지토리][gh sample]를 통해 다운로드 받을 수 있다.


## 서비스형 워드프레스 사이트 ##

예전에 운영하다가 더이상 운영하지 않는 서비스형 워드프레스 사이트가 하나 있다.

![Old Wordpress Blog Site][image-01]

이 사이트를 정적 웹사이트로 이전하기 위해서는 우선 이 사이트의 컨텐츠에 접근할 수 있어야 한다. 워드프레스 사이트는 HTTP API를 제공하기 때문에 이를 이용해서 데이터를 가져올 수 있다. 아래 명령어를 터미널에서 입력해 보자.

```bash
curl -X GET https://public-api.wordpress.com/rest/v1.1/sites/<site-name>.wordpress.com/posts/
```

혹은 포스트맨을 통해 실행시켜 보면 아래와 같이 컨텐츠를 받아올 수 있다.

![Result from Wordpress API][image-02]

이제 이 데이터를 블레이저 웹어셈블리 앱에 출력시키기만 하면 된다. 이 때 두 가지 방법이 있다.

1. 블레이저 웹 어셈블리 앱에서 직접 API를 호출하는 방법
2. 블레이저 웹 어셈블리 앱에서 프록시 API를 거쳐 API를 호출하는 방법

여기서는 두 번째 방법으로 시도해 보기로 하자.


## 프록시 API 앱 ##

애저 정적 웹 앱을 호스팅하기 위해 프록시 API 앱은 굳이 필요하지는 않다. 다만, CORS 이슈가 있다거나, 다양한 종류의 API를 호출한다거나, 보안 이슈가 있을 경우에는 프록시 API를 사용하는 것이 좋다. [애저 정적 웹 앱][az swa]은 자체적으로 이 프록시 API 기능을 제공하고 있으므로 여기서도 그 기능을 이용하기로 한다. 우선 아래와 같이 [애저 펑션][az fncapp]앱의 [HTTP 트리거][az fncapp triggers http]를 생성한다.

워드프레스 API를 호출하기 위한 엔드포인트 URL을 `GetPosts`로 두고 `HttpClient` 인스턴스를 생성한다.

```csharp
public static class PostHttpTrigger
{
    private const string GetPosts = "https://public-api.wordpress.com/rest/v1.1/sites/{0}/posts";
    private static HttpClient http = new HttpClient();
```    

아래와 같이 [애저 펑션의 OpenAPI 확장 기능][az fncapp extensions openapi]을 추가한다. 이를 이용하면 블레이저 웹어셈블리 앱에서 좀 더 쉽게 이 프록시 API에 접근할 수 있다. 이 부분은 다시 아래에서 언급할 예정이다.

```csharp
    [FunctionName("PostHttpTrigger")]
    [OpenApiOperation(operationId: "posts.get", tags: new[] { "posts" })]
    [OpenApiResponseWithBody(statusCode: HttpStatusCode.OK, contentType: "application/json", bodyType: typeof(PostCollection), Description = "The OK response")]
    public static async Task<IActionResult> GetPostsAsync(
        [HttpTrigger(AuthorizationLevel.Anonymous, "GET", Route = "posts")] HttpRequest req,
        ILogger log)
    {
```    

환경 변수 값을 통해 실제 워드프레스 사이트 이름을 가져온다.

```csharp
        var requestUri = new Uri(string.Format(GetPosts, Environment.GetEnvironmentVariable("SITE__NAME")));
```    

마지막으로 이 워드프레스 API 엔드포인트를 호출한 후 값을 받아와서 `PostCollection` 이라는 개체로 비직렬화 시킨다.

```csharp
        var json = await http.GetStringAsync(requestUri).ConfigureAwait(false);
        var posts = JsonConvert.DeserializeObject<PostCollection>(json);

        return new OkObjectResult(posts);
    }
}
```

`PostCollection` 개체와 그 부속 개체는 대략 아래와 같이 생겼다. 앞서 스크린샷과 같이 굉장히 방대한 데이터가 있지만, 여기서는 딱 필요한 부분만 정의하기로 한다.

```csharp
public class PostCollection
{
    public virtual int Found { get; set; }

    public virtual List<PostItem> Posts { get; set; }

    [JsonProperty("meta")]
    public virtual Metadata Metadata { get; set; }
}

public class PostItem
{
    [JsonProperty("ID")]
    public virtual int PostId { get; set; }

    public virtual Author Author { get; set; }

    [JsonProperty("date")]
    public virtual DateTimeOffset DatePublished { get; set; }

    public virtual string Title { get; set; }

    [JsonProperty("URL")]
    public virtual string Url { get; set; }

    public virtual string Excerpt { get; set; }

    public virtual string Content { get; set; }
}

public class Author
{
    [JsonProperty("ID")]
    public virtual int AuthorId { get; set; }

    [JsonProperty("first_name")]
    public virtual string FirstName { get; set; }

    [JsonProperty("last_name")]
    public virtual string Surname { get; set; }

    public virtual string Name { get; set; }
}

public class Metadata
{
    public virtual Dictionary<string, string> Links { get; set; }

    [JsonProperty("next_page")]
    public virtual string NextPage { get; set; }

    [JsonProperty("wpcom")]
    public virtual bool IsWordpressCom { get; set; }
}
```

마지막으로 `local.settings.json` 파일을 아래와 같이 설정한다. 위 코드에서 `SITE__NAME` 값을 참조하기 때문에 환경 변수를 설정했고, 앞으로 작성할 블레이저 웹어셈블리 앱과 연결시키기 위해 CORS 설정을 한다.

```json
{
  "IsEncrypted": false,
  "Values": {
    "AzureWebJobsStorage": "UseDevelopmentStorage=true",
    "FUNCTIONS_WORKER_RUNTIME": "dotnet",

    "SITE__NAME": "<site-name>.wordpress.com"
  },
  "Host": {
    "CORS": "*"
  }
}
```

위와 같이 정리한 후 프록시 API 앱을 실행시켜 보면 대략 아래와 같이 데이터를 확인할 수 있다.

![Result from Proxy API][image-03]

이제 프록시 API는 준비가 끝났다. 프록시 API에 접근하기 위한 OpenAPI 문서 URL은 아래의 셋 중 하나를 선택하면 된다.

* `http://localhost:7071/api/swagger.json` &ndash; OpenAPI V2
* `http://localhost:7071/api/openapi/v2.json` &ndash; OpenAPI V2
* `http://localhost:7071/api/openapi/v3.json` &ndash; OpenAPI V3


## 블레이저 웹어셈블리 앱 ##

우선 [비주얼 스튜디오][vs]를 이용해서 블레이저 웹 어셈블리 앱 프로젝트를 생성한다.

![Blazor WASM Project][image-04]

프로젝트 이름을 `WebApp`으로 하고 나머지는 모두 기본값으로 둔 후 앱을 생성한다. 앱이 생성된 후에 곧바로 F5 키를 눌러 실행시켜 보면 아래와 같이 보일 것이다.

![Blazor WASM App][image-05]

이제 여기에 앞서 생성한 프록시 API를 연결할 차례이다. 우선 백그라운드로 애저 펑션 앱이 현재 작동중인 것을 확인한 후 아래와 같이 "**Connected Services**" 메뉴를 클릭한다.

![Connected Service Menu][image-06]

그리고 나오는 화면에서 "➕" 버튼을 클릭한다.

![Add Reference][image-07]

서비스 레퍼런스를 선택할 수 있는 화면이 나오는데, 여기서 "**OpenAPI**"를 선택한다.

![Choose OpenAPI][image-08]

그러면 나오는 화면에서 OpenAPI URL을 입력하고, 네임스페이스를 `WebApp.Proxies`, 클라스 이름을 `ProxyClient`로 하고 마무리한다.

![Enter OpenAPI Details][image-09]

그러면 아래와 같이 프록시 API가 추가된 것을 볼 수 있다.

![OpenAPI Reference Added][image-10]

이 때, 오른쪽의 점 세 개 버튼을 클릭해서 보이는 "**View generated code**" 메뉴를 클릭하면 OpenAPI 문서를 읽어서 자동으로 코드를 생성해 준 것을 확인할 수 있다.

![Auto-generated Code from OpenAPI][image-11]

우리는 이제 이 코드를 사용하기만 하면 되므로, 아래와 같이 `Program.cs` 파일을 통해 `ProxyClient`에 대한 의존성을 정의한다.

> 로컬 개발일 경우에는 기본 설정된 Base URL 값이 `http://localhost:7071/api`이기 때문에 상관 없지만, 애저로 배포할 경우에는 실제 호스트 주소를 사용하라는 내용도 포함되어 있다 (line #9-13).

```csharp
builder.Services.AddScoped(sp => new HttpClient { BaseAddress = new Uri(builder.HostEnvironment.BaseAddress) });

// Add these lines to inject the dependency of ProxyClient
builder.Services.AddScoped(sp =>
{
    var client = sp.GetService<HttpClient>();
    var api = new ProxyClient(client);
    if (!builder.HostEnvironment.IsDevelopment())
    {
        var baseUrl = $"{builder.HostEnvironment.BaseAddress.TrimEnd('/')}/api";
        api.BaseUrl = baseUrl;
    }

    return api;
});
```

이제 블레이저 웹어셈블리 앱은 프록시 API를 자유롭게 호출할 수 있게 되었으니, 실제로 페이지상에서 이를 호출해 보기로 하자. 여기서는 `index.razor` 페이지를 이용해서 블로그 포스트 리스트만 보여주는 과정을 구현한다.

우선 아래와 같이 앞서 `Program.cs`에 추가했던 `ProxyClient` 의존성 개체를 주입한다.

```csharp
@page "/"

@* Inject ProxyClient dependency *@
@using WebApp.Proxies
@inject ProxyClient Api
```

아래과 같이 `<SurveyPrompt>` 컴포넌트 밑에 테이블로 포스트 리스트를 받아올 수 있게 처리한다.

```csharp
...
<SurveyPrompt Title="How is Blazor working for you?" />

@if (posts == null)
{
    <p><em>Loading...</em></p>
}
else
{
    <table class="table">
        <thead>
            <tr>
                <th>Date</th>
                <th>Author</th>
                <th>Title</th>
                <th>Excerpt</th>
            </tr>
        </thead>
        <tbody>
            @foreach (var post in posts)
            {
                <tr>
                    <td>@post.Date.ToString()</td>
                    <td>@post.Author.Name</td>
                    <td><a href="@post.URL">@post.Title</a></td>
                    <td>@((MarkupString)post.Excerpt)</td>
                </tr>
            }
        </tbody>
    </table>
}
```

아래와 같이 프록시 API를 호출해서 포스트 리스트를 가져온다.

```csharp
@code {
    private List<PostItem> posts;

    protected override async Task OnInitializedAsync()
    {
        var collection = await Api.Posts_getAsync().ConfigureAwait(false);
        posts = collection?.Posts.ToList();
    }
}
```

이제 다시 블레이저 앱을 실행시켜보자. 그러면 아래와 같이 첫 페이지에서 블로그 포스트 리스트를 확인할 수 있다.

![List of Blog Posts][image-12]

여기까지 한 후 프록시 API 앱과 블레이저 웹어셈블리 앱을 깃헙 리포지토리에 푸시한다.


## 애저 정적 웹 앱에 호스팅하기 ##

위와 같이 모든 앱 코딩이 끝났다면, 이제 [애저 정적 웹 앱][az swa]에 배포할 차례이다. 이 부분은 예전에 이미 다뤄봤기 때문에 여기서는 별도의 언급 없이 이전 포스트 링크로 대체하기로 하자.

* [애저 정적 웹 앱에 블레이저 웹 어셈블리 앱 호스팅하기][post-1]
* [깃헙 액션 재활용 기능을 활용해서 애저 정적 웹 앱 CI/CD 파이프라인 리팩토링하기][post-2]

이렇게 애저 정적 웹 앱 호스팅이 끝났다면, 실제로 웹사이트에 접속하면 아래와 같이 보인다.

![List of Blog Posts on Azure][image-13]

---

지금까지 서비스형 워드프레스를 헤드리스 CMS로 이용해서 [애저 정적 웹 앱][az swa]에 [블레이저 웹어셈블리][blazor wasm]와 [애저 펑션][az fncapp] 프록시 API를 통해 웹사이트를 구현하는 방법에 대해 알아보았다. 이 포스트에서는 간단한 컨셉 정도만을 구현했지만, 기본적인 큰 틀은 다 구현을 해봤기 때문에 적절한 API 호출과 UI 레이아웃만 구성한다면, 기존의 서비스형 워드프레스를 그대로 사용한 채로 나만의 멋진 블로그 사이트를 운영할 수도 있을 것이다.


[image-01]: /2022/05/building-headless-cms-with-blazor-wasm-01.png
[image-02]: /2022/05/building-headless-cms-with-blazor-wasm-02.png
[image-03]: /2022/05/building-headless-cms-with-blazor-wasm-03.png
[image-04]: /2022/05/building-headless-cms-with-blazor-wasm-04.png
[image-05]: /2022/05/building-headless-cms-with-blazor-wasm-05.png
[image-06]: /2022/05/building-headless-cms-with-blazor-wasm-06.png
[image-07]: /2022/05/building-headless-cms-with-blazor-wasm-07.png
[image-08]: /2022/05/building-headless-cms-with-blazor-wasm-08.png
[image-09]: /2022/05/building-headless-cms-with-blazor-wasm-09.png
[image-10]: /2022/05/building-headless-cms-with-blazor-wasm-10.png
[image-11]: /2022/05/building-headless-cms-with-blazor-wasm-11.png
[image-12]: /2022/05/building-headless-cms-with-blazor-wasm-12.png
[image-13]: /2022/05/building-headless-cms-with-blazor-wasm-13.png


[post-1]: /ko/2020/06/17/hosting-blazor-web-assembly-app-on-azure-static-webapp/
[post-2]: /ko/2021/12/01/refactoring-aswa-github-actions-workflow/

[gh sample]: https://github.com/justinyoo/blazor-wasm-azfunc-aswa

[gh actions]: https://github.com/features/actions

[az fncapp]: https://docs.microsoft.com/ko-kr/azure/azure-functions/functions-overview?WT.mc_id=dotnet-66600-juyoo
[az fncapp triggers http]: https://docs.microsoft.com/ko-kr/azure/azure-functions/functions-bindings-http-webhook-trigger?tabs=in-process%2Cfunctionsv2&pivots=programming-language-csharp&WT.mc_id=dotnet-66600-juyoo
[az fncapp extensions openapi]: https://aka.ms/azfunc-openapi

[az swa]: https://docs.microsoft.com/ko-kr/azure/static-web-apps/overview?WT.mc_id=dotnet-66600-juyoo

[vs]: https://visualstudio.microsoft.com/vs/?WT.mc_id=dotnet-66600-juyoo

[blazor]: https://dotnet.microsoft.com/apps/aspnet/web-apps/blazor?WT.mc_id=dotnet-66600-juyoo
[blazor wasm]: https://docs.microsoft.com/ko-kr/aspnet/core/blazor/host-and-deploy/webassembly?view=aspnetcore-6.0&WT.mc_id=dotnet-66600-juyoo
