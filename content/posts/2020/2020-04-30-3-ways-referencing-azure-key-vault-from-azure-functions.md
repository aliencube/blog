---
title: "애저 펑션에서 애저 키 저장소 값을 참조하는 세 가지 방법"
slug: 3-ways-referencing-azure-key-vault-from-azure-functions
description: "이 포스트에서는 애저 펑션에서 애저 키 저장소 값을 참조하는 세 가지 방법에 대해 논의해 보겠습니다."
date: "2020-04-30"
author: Justin-Yoo
tags:
- azure-functions
- azure-keyvault
- pro-tips
- local-dev
cover: /2020/04/3-ways-referencing-azure-key-vault-from-azure-functions-00.png
fullscreen: true
---

[애저 펑션][az func] 또는 [애저 앱 서비스][az appsvc]를 사용하다 보면 API 인증 키 라든가, 데이터베이스 커넥션 스트링이라든가 하는 것들을 반드시 사용하게 된다. [애저 키 저장소][az kv]를 이용해서 민감한 정보를 안전하게 보관하고 필요에 따라 그 값을 호출해서 사용하는 것도 한 가지 방법이다. 이 포스트에서는 [애저 펑션][az func]에서 애저 [키 저장소][az kv]에 저장된 값을 직접 액세스하는 대신 레퍼런스값을 참조하는 형식으로 해서 사용하는 트릭을 세 가지 정도 소개하고자 한다.

> 이 포스트에 쓰인 샘플 코드는 [KeyVault Reference Sample][gh sample]에서 다운로드 받을 수 있다.


## 키 저장소 레퍼런스 참조 원리 ##

먼저 [애저 키 저장소][az kv]의 값을 [애저 펑션][az func]에서 참조하는 원리에 대해 알아보자.

* [애저 펑션][az func] 앱 인스턴스는 일단 [관리 ID (Managed Identity)][az func mi]를 활성화 시켜, [애저 키 저장소][az kv]에서 직접 접근을 허락하게 한다. 이와 관련해서는 [이전 포스트][post azfunc mi]를 참고하면 좋다.
* [애저 펑션][az func]의 [App Settings][az func appsettings] 블레이드에서 [애저 키 저장소][az kv] 값을 참조한다. 참조를 위한 형식은 `@Microsoft.KeyVault(...)`와 같다.

![][image-01]

위 그림에서 볼 수 있다시피, 레퍼런스로 지정한 값은 `Source` 컬럼에 `Key Vault Reference`라고 되어 있는 것이 보인다.

그런데, [애저 펑션][az func] 앱은 이 [구성(Configuration) 값을 환경 변수로 인식한다][az appsvc appsettings]. 따라서 이 값들을 가져올 때 `Environment.GetEnvironmentVariable()` 메소드를 이용해서 가져온다.

> [내부적으로 이 값들을 비직렬화해서 가져올 수도 있는데][az appsvc envvar], 이는 여기서는 다루지 않는다.

그런데, [키 저장소][az kv] 레퍼런스 값은 환경 변수로 어떻게 바뀔까? 우리가 작성하는 앱에서는 단순히 환경 변수 값을 참조할 뿐이지 [키 저장소][az kv] 값을 참조하지 않는다. 이는 [앱 서비스][az appsvc] 인스턴스가 시작될 때 이를 내부적으로 참조하고 환경 변수로 치환하는 작업을 한다. 따라서, 우리는 굳이 자세한 내용을 알 필요는 없고, 그냥 참조값을 던져주고 쓰기만 하면 된다.


## 키 저장소 참조 방법 #1 ##

그렇다면 [공식 문서][az kv fncapp]에서 제공하는 방법으로는 아래와 같이 둘 중 하나의 형식으로 참조값을 설정하면 된다.

https://gist.github.com/justinyoo/a6557ce58fb28e526744d85941404e75?file=01-kv-reference-1.txt

https://gist.github.com/justinyoo/a6557ce58fb28e526744d85941404e75?file=02-kv-reference-2.txt

![][image-02]

펑션 앱 코드에서는 동일하게 `Environment.GetEnvironmentVariable("Hello")` 또는 `Environment.GetEnvironmentVariable("Hello2")`를 호출하면 된다.


## 키 저장소 참조 방법 #2 ##

그런데, 위의 방법이 큰 문제는 아니지만 조금 불편한 구석이 하나 있다. 시크릿 값이 바뀌면 버전도 따라서 같이 바뀌는데, 그 때 마다 매 번  `SecretUri` 값을 바꿔주거나 `SecretVersion` 값을 바꿔줘야 한다. 아직 프리뷰 단계이긴 하지만 [애저 이벤트 그리드][az evtgrd]를 이용하면 [키 저장소][az kv]에서 값이 바뀌었을 경우 그 [이벤트를 잡아 처리할 수도 있다][az kv evtgrd]. 그런데, 이렇게 하면 추가적인 코딩 작업이 필요하다. 그렇다면 별도의 코딩 작업 없이 새 시크릿 값을 곧바로 적용할 수 있는 방법이 있을까?

물론 있다. 아래의 `SecretUri` 값을 `<scret-version>` 부분만 빼고 넣는다. 여기서 명심해야 할 부분은 맨 마지막에 반드시 `/`로 끝나야 한다.

https://gist.github.com/justinyoo/a6557ce58fb28e526744d85941404e75?file=03-kv-reference-3.txt

![][image-03]

이렇게 하면 항상 최신의 시크릿 버전 값을 참조해서 그 결과를 보내준다. 그런데, 앞서 [애저 앱 서비스][az appsvc] 내부적으로 [키 저장소][az kv]값을 변환하고 환경 변수 값으로 치환하는 과정이 있다고 했는데, 이 과정에서 만약 시크릿 버전을 명시하지 않으면 환경 변수로 변환되는 도중에 살짝 레이턴시가 있다. 정확한 값은 모르지만 대략 10초 정도의 레이턴시를 경험했다. 따라서, 이 부분만 유념한다면 시크릿 버전을 굳이 명시하지 않아도 되니 편리하다.


## 키 저장소 참조 방법 #3 ##

위의 [키 저장소][az kv] 참조 문법인 `@Microsoft.KeyVault(...)`는 다 좋은데, 애저로 배포했을 경우에만 적용이 가능하다. 즉, 로컬 개발 환경에서는 적용할 수가 없다. 따라서, 로컬 개발 환경에서도 [키 저장소][az kv] 값을 참조하고 싶다면 약간의 코딩이 필요하다. 아래 코드를 살짝 보자. 먼저 정규표현식으로 [키 저장소][az kv] 참조 문법 패턴을 구성한다.

https://gist.github.com/justinyoo/a6557ce58fb28e526744d85941404e75?file=04-appsettings-handler-1.cs

그리고 `GetValueAsync(string key)` 메소드를 통해 환경 변수 값을 확인한다. 만약 환경 변수값이 [키 저장소][az kv] 참조 형식이 아니라면 그냥 값을 반환한다.

https://gist.github.com/justinyoo/a6557ce58fb28e526744d85941404e75?file=05-appsettings-handler-2.cs

만약 [키 저장소][az kv] 참조 형식이라면, `SecretUri` 값을 참조하는지 우선 확인해서 그 값을 보내 시크릿 값을 가져온다.

https://gist.github.com/justinyoo/a6557ce58fb28e526744d85941404e75?file=06-appsettings-handler-3.cs

만약 `VaultName` 값을 참조한다면 정규표현식을 통해 필요한 값들을 추출해서 [키 저장소][az kv]를 호출한 후 시크릿 값을 가져온다.

https://gist.github.com/justinyoo/a6557ce58fb28e526744d85941404e75?file=07-appsettings-handler-4.cs

만약 이도 저도 아니면 그냥 `null` 값을 반환한다.

https://gist.github.com/justinyoo/a6557ce58fb28e526744d85941404e75?file=08-appsettings-handler-5.cs

이렇게 하면 기존의 `Environment.GetEnvironmentVariable()` 메소드를 호출했던 부분을 이 `AppSettingsHandler.GetValueAsync()` 메소드로 바꿔줘야 한다. 그렇게 함으로써, 로컬 개발 환경에서도 [키 저장소][az kv]를 참조해서 호출할 수 있다.

이 방법의 가장 큰 단점이 하나 있다. 메소드 안에서는 완벽하게 작동하지만, 만약 [애저 펑션][az func]의 [트리거와 바인딩][az func bindings]에서는 작동하지 않는다. 트리거와 바인딩에서는 환경 변수를 직접 접근하지, 지금 위에서 작성한 핸들러를 통하지 않기 때문이다.

---

지금까지, [애저 펑션][az func]에서 [애저 키 저장소][az kv] 값을 참조하는 세 가지 방법에 대해 알아 보았다. 저마다 장점도 있고 단점도 있는지라, 어느 부분을 선택할지에 대해서는 전적으로 여러분의 몫이다.


[image-01]: /2020/04/3-ways-referencing-azure-key-vault-from-azure-functions-01.png
[image-02]: /2020/04/3-ways-referencing-azure-key-vault-from-azure-functions-02.png
[image-03]: /2020/04/3-ways-referencing-azure-key-vault-from-azure-functions-03.png

[post azfunc mi]: /ko/2019/01/03/accessing-key-vault-from-azure-functions-with-managed-identity/

[gh sample]: https://github.com/devkimchi/KeyVault-Reference-Sample

[az func]: https://docs.microsoft.com/ko-kr/azure/azure-functions/functions-overview?WT.mc_id=aliencubeorg-blog-juyoo
[az func mi]: https://docs.microsoft.com/ko-kr/azure/app-service/overview-managed-identity?tabs=dotnet&WT.mc_id=aliencubeorg-blog-juyoo
[az func appsettings]: https://docs.microsoft.com/ko-kr/azure/azure-functions/functions-how-to-use-azure-function-app-settings?WT.mc_id=aliencubeorg-blog-juyoo
[az func bindings]: https://docs.microsoft.com/ko-kr/azure/azure-functions/functions-triggers-bindings?WT.mc_id=aliencubeorg-blog-juyoo

[az appsvc]: https://docs.microsoft.com/ko-kr/azure/app-service/?WT.mc_id=aliencubeorg-blog-juyoo
[az appsvc appsettings]: https://docs.microsoft.com/ko-kr/azure/app-service/configure-common?WT.mc_id=aliencubeorg-blog-juyoo
[az appsvc envvar]: https://docs.microsoft.com/ko-kr/azure/app-service/containers/configure-language-dotnetcore?WT.mc_id=aliencubeorg-blog-juyoo#access-environment-variables

[az kv]: https://docs.microsoft.com/ko-kr/azure/key-vault/general/overview?WT.mc_id=aliencubeorg-blog-juyoo
[az kv fncapp]: https://docs.microsoft.com/ko-kr/azure/app-service/app-service-key-vault-references?WT.mc_id=aliencubeorg-blog-juyoo
[az kv evtgrd]: https://docs.microsoft.com/ko-kr/azure/key-vault/general/event-grid-overview?WT.mc_id=aliencubeorg-blog-juyoo

[az evtgrd]: https://docs.microsoft.com/ko-kr/azure/event-grid/overview?WT.mc_id=aliencubeorg-blog-juyoo
