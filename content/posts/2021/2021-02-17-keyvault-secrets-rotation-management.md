---
title: "애저 키 저장소 시크릿 로테이션 관리"
slug: keyvault-secrets-rotation-management
description: "이 포스트에서는 애저 키 저장소의 시크릿 값을 로테이션할 때 애저 펑션을 이용해 일정 기간 이상 오래된 시크릿 값을 비활성화 시키는 방법에 대해 알아봅니다."
date: "2021-02-17"
author: Justin-Yoo
tags:
- azure-keyvault
- azure-functions
- azure-sdk
- secret-management
cover: /2021/02/keyvault-secrets-rotation-management-00.png
fullscreen: true
---


얼마전 [애저 키 저장소][az kv] 시크릿 값을 [애저 앱 서비스][az appsvc] 혹은 [애저 펑션][az fncapp]에서 참조할 때, 더이상 버전을 명시하지 않아도 된다는 [공지][az kv announcement]가 있었다. 따라서, [지난 포스트][post prev]에서 언급했던 [애저 키 저장소의 시크릿 값][az kv secrets]을 참조하는 방법들 중 두번째 방법이 이제는 가장 효과적인 접근 방식이 되었다.

https://gist.github.com/justinyoo/75c16e773d9e1c8b8a1d5d5efa37f9c9?file=01-keyvault-reference.txt

이럴 경우 가장 최신 버전의 시크릿 값을 자동으로 가져와서 보여주게 된다. 만약 최신 버전의 시크릿 값이 생성된지 아직 만 하루가 지나지 않았다면, 애저 앱 서비스 혹은 애저 펑션 내부적으로 작동하는 캐싱 메카니즘이 완전히 값을 받아오지 않았을 수도 있기 때문에, 이전 버전과 함께 [로테이션][az kv secrets rotation]을 시켜줘야 한다. 만약 이 로테이션에 더이상 쓰이지 않는 시크릿 버전이 있다면 모두 비활성화 시켜주는 것이 보안상 좋다.

* ***애저 키 저장소 시크릿 로테이션 관리***
* [이벤트 기반 애저 키 저장소 시크릿 로테이션 관리][post next]

[애저 키 저장소][az kv]에 저장할 수 있는 시크릿의 갯수는 딱히 제한된 것이 없다. 따라서 현업에서 사용하다 보면 굉장히 많은 수의 시크릿을 저장하게 되는데, 이럴 경우 로테이션에 더이상 쓰이지 않는 시크릿 버전을 일일이 찾아 비활성화 시켜주기에는 너무 많을 수 있다. 그렇다면, 이를 자동화할 수 있는 방법에는 무엇이 있을까? 이 포스트에서는 오래되었지만 여전히 활성화 상태로 남아있는 시크릿 버전들을 일괄적으로 비활성화시키는 방법을 애저 펑션으로 구현해 보기로 한다.

> 이 포스트에 사용한 샘플 코드는 이 [깃헙 리포지토리][gh sample]에서 다운로드 받을 수 있다.


## 애저 키 저장소 SDK ##

애저 키 저장소를 다루는 SDK는 현재 두 가지 버전이 있다.

* [Microsoft.Azure.KeyVault][nuget sdk kv old]
* [Azure.Security.KeyVault.Secrets][nuget sdk kv new]

이 중 전자는 이제 deprecated 된 버전이라서, 후자를 사용하면 된다. 이와 더불어 [Azure.Identity][nuget sdk identity] SDK도 함께 다운로드 받아 사용하도록 하자. 애저 펑션 프로젝트를 생성한 후 아래와 같이 두 NuGet 패키지를 설치한다.

https://gist.github.com/justinyoo/75c16e773d9e1c8b8a1d5d5efa37f9c9?file=02-add-nuget-packages.sh

또한, 키 저장소 패키지는 `IAsyncEnumerable` 인터페이스를 사용하므로 [System.Linq.Async][nuget linq async] 패키지도 다운로드 받는다.

https://gist.github.com/justinyoo/75c16e773d9e1c8b8a1d5d5efa37f9c9?file=03-add-nuget-packages.sh

> **NOTE**: 애저 펑션은 아직 .NET 5를 지원하지 않으므로 `System.Linq.Async` 5.0.0 버전의 패키지를 설치하지 않도록 한다.

이제 필요한 라이브러리 설치는 다 끝났고, 실제로 펑션 코드를 구현하기로 한다.


## 오래된 시크릿 버전 비활성화를 위한 애저 펑션 구현 ##

아래 명령어를 통해 애저 펑션 [HTTP 트리거][az fncapp trigger http]를 하나 만들자.

https://gist.github.com/justinyoo/75c16e773d9e1c8b8a1d5d5efa37f9c9?file=04-add-new-httptrigger.sh

기본 HTTP 트리거 템플릿으로 펑션이 하나 만들어 졌다. 이제 이 펑션 메소드의 `HttpTrigger` 바인딩을 아래와 같이 바꿔보자. HTTP 메소드는 `POST` 하나로 한정하고, 라우팅 URL을 `secrets/all/disable`로 두었다 (line #5).

https://gist.github.com/justinyoo/75c16e773d9e1c8b8a1d5d5efa37f9c9?file=05-secrets-httptrigger-01.cs&highlights=5

환경 변수를 통해 아래 두 값을 받아온다. 하나는 애저 키 저장소에 접근할 수 있는 URI이고, 다른 하나는 애저 키 저장소 인스턴스를 호스팅하는 테넌트의 ID값이다.

https://gist.github.com/justinyoo/75c16e773d9e1c8b8a1d5d5efa37f9c9?file=05-secrets-httptrigger-02.cs

다음으로는 애저 키 저장소에 접근할 수 있는 `SecretClient` 인스턴스를 생성한다. 이 때 인증 옵션을 `DefaultAzureCredentialOptions` 인스턴스를 통해 제공해야 하는데, 만약 개발하려는 로컬 컴퓨터에서 애저에 로그인한 계정이 여러 개의 테넌트 정보를 갖고 있다면, 아래와 같이 명시적으로 테넌트 ID 값을 지정해 줘야 한다. 그렇지 않으면 [인증 에러][nuget sdk identity error]가 발생한다 (line #4-6).

https://gist.github.com/justinyoo/75c16e773d9e1c8b8a1d5d5efa37f9c9?file=05-secrets-httptrigger-03.cs&highlights=4-6

이제 모든 시크릿을 가져와서 하나씩 처리를 해야 한다. 가장 먼저 할 일은 모든 시크릿을 가져오는 것이다 (line #2-4).

https://gist.github.com/justinyoo/75c16e773d9e1c8b8a1d5d5efa37f9c9?file=05-secrets-httptrigger-04.cs&highlights=2-4

이제 각각의 시크릿을 하나씩 돌면서 모든 버전을 가져온다. 단, 활성화 된 것만 가져오면 되므로 아래와 같이 `WhereAwait` 구문으로 필터링을 한다 (line #7). 또한 `OrderByDescendingAwait` 구문을 이용해 시간의 역순으로 정렬해서 가장 최근 것이 맨 앞으로 오게끔 한다 (line #8).

https://gist.github.com/justinyoo/75c16e773d9e1c8b8a1d5d5efa37f9c9?file=05-secrets-httptrigger-05.cs&highlights=7,8

만약 해당 시크릿에는 활성화된 버전이 없다면, 더이상 처리할 것이 없으므로 넘어간다.

https://gist.github.com/justinyoo/75c16e773d9e1c8b8a1d5d5efa37f9c9?file=05-secrets-httptrigger-06.cs

만약 해당 시크릿에는 활성화된 버전이 하나뿐이라면, 더이상 처리할 것이 없으므로 넘어간다.

https://gist.github.com/justinyoo/75c16e773d9e1c8b8a1d5d5efa37f9c9?file=05-secrets-httptrigger-07.cs

만약 해당 시크릿의 최신 버전이 생성된지 만 하루가 안 됐다면, 아직 로테이션이 필요하므로 넘어간다.

https://gist.github.com/justinyoo/75c16e773d9e1c8b8a1d5d5efa37f9c9?file=05-secrets-httptrigger-08.cs

이제 남은 시크릿 버전을 대상으로 비활성화 처리를 해야 한다. 가장 최신의 버전은 건너뛰고 그 다음부터 처리한다 (line #2). 그리고 `Enabled` 값을 `false`로 변경하고 (line #6), 업데이트한다 (line #8).

https://gist.github.com/justinyoo/75c16e773d9e1c8b8a1d5d5efa37f9c9?file=05-secrets-httptrigger-09.cs&highlights=2,6,8

마지막으로 처리 결과를 저장한 변수를 응답 개체에 실어 반환한다.

https://gist.github.com/justinyoo/75c16e773d9e1c8b8a1d5d5efa37f9c9?file=05-secrets-httptrigger-10.cs

이렇게 한 후 실제로 애저 펑션을 실행시켜 보면 가장 최신의 시크릿 버전을 제외한 모든 오래된 버전이 비활성화 된 것을 확인할 수 있다. 이 펑션앱에서 [HTTP 트리거][az fncapp trigger http] 대신 [타이머 트리거][az fncapp trigger timer]를 붙인다든가, 아니면 [애저 로직 앱][az logapp]을 연동시켜 스케줄링을 걸어 놓는다면 더이상 활성화 되어 있지만 더이상 사용하지 않는 애저 키 저장소의 시크릿 버전들에 대한 걱정을 덜 수 있을 것이다.

---

지금까지 [애저 키 저장소][az kv]의 시크릿 값을 [애저 앱 서비스][az appsvc] 혹은 [애저 펑션][az fncapp]에서 참조할 때 더이상 사용하지 않는 시크릿 버전을 자동으로 비활성화 시키는 방법에 대해 알아 보았다. 이를 이용해서 좀 더 관리 요소를 줄일 수 있기를 바란다. [다음 포스트][post next]에서는 특정 시크릿에 새 버전이 추가되는 이벤트를 이용하는 방법에 대해 알아보기로 하자.


[post prev]: /ko/2020/04/30/3-ways-referencing-azure-key-vault-from-azure-functions/
[post next]: /ko/2021/02/24/event-driven-keyvault-secrets-rotation-management

[gh sample]: https://github.com/devkimchi/KeyVault-Reference-Sample/tree/2021-02-17

[az logapp]: https://docs.microsoft.com/ko-kr/azure/logic-apps/logic-apps-overview?WT.mc_id=dotnet-16807-juyoo

[az appsvc]: https://docs.microsoft.com/ko-kr/azure/app-service/?WT.mc_id=dotnet-16807-juyoo

[az fncapp]: https://docs.microsoft.com/ko-kr/azure/azure-functions/functions-overview?WT.mc_id=dotnet-16807-juyoo
[az fncapp trigger http]: https://docs.microsoft.com/ko-kr/azure/azure-functions/functions-bindings-http-webhook-trigger?tabs=csharp&WT.mc_id=dotnet-16807-juyoo
[az fncapp trigger timer]: https://docs.microsoft.com/ko-kr/azure/azure-functions/functions-bindings-timer?tabs=csharp&WT.mc_id=dotnet-16807-juyoo

[az kv]: https://docs.microsoft.com/ko-kr/azure/key-vault/general/overview?WT.mc_id=dotnet-16807-juyoo
[az kv announcement]: https://azure.microsoft.com/ko-kr/updates/versions-no-longer-required-for-key-vault-references-in-app-service-and-azure-functions/?WT.mc_id=dotnet-16807-juyoo
[az kv secrets]: https://docs.microsoft.com/ko-kr/azure/key-vault/secrets/about-secrets?WT.mc_id=dotnet-16807-juyoo
[az kv secrets rotation]: https://docs.microsoft.com/ko-kr/azure/app-service/app-service-key-vault-references?WT.mc_id=dotnet-16807-juyoo#rotation

[nuget sdk kv old]: https://www.nuget.org/packages/Microsoft.Azure.KeyVault/
[nuget sdk kv new]: https://www.nuget.org/packages/Azure.Security.KeyVault.Secrets/
[nuget linq async]: https://www.nuget.org/packages/System.Linq.Async/
[nuget sdk identity]: https://www.nuget.org/packages/Azure.Identity/
[nuget sdk identity error]: https://github.com/Azure/azure-sdk-for-net/issues/11559#issuecomment-620233531
