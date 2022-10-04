---
title: "애저 펑션에 APEX 도메인을 연결하는 세 가지 방법"
slug: 3-ways-mapping-apex-domains-to-azure-functions
description: "이 포스트에서는 애저 펑션 인스턴스에 APEX 도메인을 연결하는 세 가지 방법에 대해 논의해 봅니다."
date: "2020-10-07"
author: Justin-Yoo
tags:
- azure-functions
- apex-domain
- azure-cli
- arm-template
cover: /2020/10/3-ways-mapping-apex-domains-to-azure-functions-00.png
fullscreen: true
---

이 시리즈에서는 애저 펑션 인스턴스를 운영하면서 사용자 지정 도메인을 연결하고, SSL 인증서를 붙이고, 수시로 변하는 공개 IP 주소를 관리하는 방법에 대해 알아본다.

* ***애저 펑션에 APEX 도메인을 연결하는 세 가지 방법***
* [애저 펑션에 Let's Encrypt 인증서 자동으로 연동하기][post 2]
* [애저 펑션의 IP주소 변경시 깃헙 액션을 통해 DNS와 SSL 인증서를 자동으로 갱신하기][post 3]
* [애저 펑션 배포 프로필 없이 깃헙 액션으로 배포하기][post 4]

[애저 펑션][az func] 인스턴스가 하나 있다. 당신의 고객은 이 애저 펑션 인스턴스에 사용자 지정 도메인을 연결하고 싶어한다. 만약 사용자 지정 도메인이 `api.contoso.com`이라면 DNS의 CNAME 레코드를 변경하면 되는지라 큰 문제가 없다. 그런데, APEX 도메인을 연결하고 싶어한다면 어떻게 할까?

> APEX 도메인, 루트 도메인 등은 모두 같은 표현으로 `contoso.com`과 같은 도메인을 가리킬 때 사용한다.

![][image-01]

[애저 포탈][az portal]을 통해 APEX 도메인을 연결하려고 하면 위와 같은 에러 메시지가 나오면서 연결 자체를 할 수가 없다. APEX 도메인을 연결하기 위해서는 우선 IP 주소를 할당하는 A 레코드가 있어야 하는데, 이를 애저 포탈에서는 지원하지 않기 때문이다.

그렇다면, 우리는 고객사의 요청을 수행할 수 없는가? 꼭 그렇지만은 않다.

![][image-02]

항상 그렇듯, 언제나 방법은 있다. 이 포스트에서는 세 가지 다른 방법으로 애저 펑션 인스턴스에 APEX 도메인을 연결시키는 방법에 대해 알아보자.


## 도메인 소유권 확인 ##

애저 포탈에서 애저 펑션 인스턴스의 `사용자 지정 도메인 블레이드`로 들어가서 도메인 소유권을 먼저 확인해야 한다.

![][image-03]

* 위 그림에서 보이는 `사용자 지정 도메인 확인 ID` 값을 복사한다.
* DNS 서버의 TXT 레코드에 아래와 같이 등록한다.
* A 레코드에 IP 주소를 등록한다.

일단 여기까지 애저 포탈을 통해 진행하면 된다.

> 만약 위에 언급한 `사용자 지정 도메인 확인 ID` 값을 포탈이 아닌 [애저 CLI][az cli]를 통해 확인하고 싶다면 이 [링크][gh azcli]를 참조하면 좋다.

![][image-04]

이제 APEX 도메인에 대한 소유권 확인은 끝났다. 하지만, 아직 APEX 도메인을 연결시키지는 못한 상태이다.


## 애저 파워셸로 등록하기 ##

[애저 포탈][az portal]을 통해 등록할 수 없다면, [애저 파워셸][az pwsh]을 통해 등록하면 된다. 아래와 같이 `Set-AzWebApp` cmdlet을 사용해서 APEX 도메인을 등록한다.

https://gist.github.com/justinyoo/8f4f33645adf7426969855b29171e91a?file=01-set-azwebapp.ps1&highlights=7

여기서 명심해야 할 부분이 하나 있다. 위에 입력한 명령어의 `-HostNames` 파라미터에는 사용자 지정 도메인 뿐만 아니라 기존에 기본값으로 주어지는 도메인까지 모두 포함시켜야 한다 (line #7). 그렇지 않으면 기존에 이미 설정되어 있던 모든 도메인 네임 설정이 사라지고, 기본값으로 주어지는 도메인을 삭제할 수 없다는 경고 메시지를 받게 된다.

![][image-05]

모든 설정이 잘 끝났다면, 아래와 같은 화면을 볼 수 있다.

![][image-06]


## 애저 CLI로 등록하기 ##

만약 [애저 CLI][az cli]를 선호한다면 아래와 같이 `az functionapp config hostname add` 명령어를 사용하면 된다.

https://gist.github.com/justinyoo/8f4f33645adf7426969855b29171e91a?file=02-az-functionapp.sh

[애저 파워셸][az pwsh]을 사용할 때와 다르게 이번에는 한 번에 하나씩 도메인을 추가할 수 있다. 그리고, 기존에 추가했던 사용자 지정 도메인은 사라지지 않고 그대로 남아있다.


## ARM 템플릿으로 등록하기 ##

위와 같은 커맨드라인 명령어 방식 보다 ARM 템플릿을 선호할 경우 이를 이용해서 등록할 수도 있다. 아래는 간단하게 [bicep 형식][gh bicep]으로 표현한 것이다 (line #5-8)

https://gist.github.com/justinyoo/8f4f33645adf7426969855b29171e91a?file=03-add-hostname.bicep&highlights=6-9

이를 ARM 템플릿으로 컴파일한 결과는 아래와 같다.

https://gist.github.com/justinyoo/8f4f33645adf7426969855b29171e91a?file=04-add-hostname.json

---

지금까지, 세 가지 방법을 이용해서 APEX 도메인을 [애저 펑션][az func] 인스턴스에 연결하는 방법에 대해 알아 보았다. 일반적인 사용자 케이스로만 보면 애저 펑션은 사용자 지정 도메인을 연결할 일이 없거나, 있어도 루트 도메인으로 연결할 일이 거의 없다고 판단했는지, 애저 포탈에서는 루트 도메인 등록 자체를 할 수 없게 막아두었다. 하지만, [애저 파워셸][az pwsh] 또는 [애저 CLI][az cli]를 사용한다거나 ARM 템플릿을 사용하는 방식을 이용하면 루트 도메인을 등록할 수 있다. 만약 여러분의 고객 중 하나가 동일한 요구사항이 있다면 이 포스트가 도움이 되기를 바란다.

[다음 포스트][post 2]에서는 이 사용자 지정 도메인에 [Let's Encrypt][letsencrypt]로 생성한 SSL 인증서를 연결하는 방법에 대해 알아보기로 하자.


[image-01]: /2020/10/3-ways-mapping-apex-domains-to-azure-functions-01-ko.png
[image-02]: /2020/10/3-ways-mapping-apex-domains-to-azure-functions-02-ko.jpg
[image-03]: /2020/10/3-ways-mapping-apex-domains-to-azure-functions-03-ko.png
[image-04]: /2020/10/3-ways-mapping-apex-domains-to-azure-functions-04-ko.png
[image-05]: /2020/10/3-ways-mapping-apex-domains-to-azure-functions-05-ko.png
[image-06]: /2020/10/3-ways-mapping-apex-domains-to-azure-functions-06-ko.png

[post 1]: /ko/2020/10/07/3-ways-mapping-apex-domains-to-azure-functions/
[post 2]: /ko/2020/10/14/lets-encrypt-ssl-certificate-on-azure-functions/
[post 3]: /ko/2020/10/28/updating-azure-dns-and-ssl-certificate-on-azure-functions-via-github-actions/
[post 4]: /ko/2020/11/05/deploying-azure-functions-via-github-actions-without-publish-profile/

[az func]: https://docs.microsoft.com/ko-kr/azure/azure-functions/functions-overview?WT.mc_id=aliencubeorg-blog-juyoo
[az portal]: https://azure.microsoft.com/ko-kr/features/azure-portal/?WT.mc_id=aliencubeorg-blog-juyoo
[az cli]: https://docs.microsoft.com/ko-kr/cli/azure/what-is-azure-cli?WT.mc_id=aliencubeorg-blog-juyoo
[az pwsh]: https://docs.microsoft.com/ko-kr/powershell/azure/new-azureps-module-az?WT.mc_id=aliencubeorg-blog-juyoo

[gh azcli]: https://github.com/Azure/azure-cli/issues/14142#issuecomment-676539150
[gh bicep]: https://github.com/azure/bicep

[letsencrypt]: https://letsencrypt.org/
