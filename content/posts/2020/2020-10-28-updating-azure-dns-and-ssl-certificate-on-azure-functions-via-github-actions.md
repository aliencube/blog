---
title: "애저 펑션의 IP주소 변경시 깃헙 액션을 통해 DNS와 SSL 인증서를 자동으로 갱신하기"
slug: updating-azure-dns-and-ssl-certificate-on-azure-functions-via-github-actions
description: "이 포스트에서는 깃헙 액션을 이용해서 애저 펑션 인스턴스에 할당된 공개 IP 주소가 바뀔 때 마다 DNS의 A 레코드를 자동으로 갱신하고 그에 따른 인증서를 함께 갱신하는 방법에 대해 논의해 봅니다."
date: "2020-10-28"
author: Justin-Yoo
tags:
- azure-functions
- azure-dns
- ssl-certificate
- github-actions
cover: /2020/10/updating-azure-dns-and-ssl-certificate-on-azure-functions-via-github-actions-00.png
fullscreen: true
---

이 시리즈에서는 애저 펑션 인스턴스를 운영하면서 사용자 지정 도메인을 연결하고, SSL 인증서를 붙이고, 수시로 변하는 공개 IP 주소를 관리하는 방법에 대해 알아본다.

* [애저 펑션에 APEX 도메인을 연결하는 세 가지 방법][post 1]
* [애저 펑션에 Let's Encrypt 인증서 자동으로 연동하기][post 2]
* ***애저 펑션의 IP주소 변경시 깃헙 액션을 통해 DNS와 SSL 인증서를 자동으로 갱신하기***
* [애저 펑션 배포 프로필 없이 깃헙 액션으로 배포하기][post 4]

[지난 포스트][post 2]에서는 [Let's Encrypt][letsencrypt]에서 발행하는 SSL 인증서를 사용자 지정 APEX 도메인에 연결했다면, 이번에는 [애저 펑션][az func] 인스턴스의 인바운드 IP가 변할 때 마다 [깃헙 액션][gh actions]을 이용해서 이를 자동으로 DNS에 반영하고, SSL 인증서를 갱신하는 방법에 대해 알아보기로 한다.

> 이 포스트에 쓰인 모든 깃헙 액션 관련 소스 코드는 [이 리포지토리][gh sample]에서 다운로드 받을 수 있다.


## 애저 펑션 인바운드 IP ##

애저 펑션은 기본적으로 인바운드 IP 주소가 고정되어 있지 않다. 즉, 언제든 IP 주소가 바뀔 수 있는 가능성이 있다는 말과 동일하다. 아래 그림과 같이 할당 가능한 IP 주소가 하나 이상이다.

![][image-01]

따라서, 만약 사용자 지정 APEX 도메인을 애저 펑션 인스턴스에 연결시켜 놓았다면, IP 주소를 이용하는 A 레코드를 IP 주소가 바뀔 때 마다 갱신해 줘야 한다.


## 애저 DNS의 A 레코드 갱신하기 ##

우선 [애저 파워셸][az pwsh]을 이용해서 기존 애저 펑션 앱 인스턴스의 인바운드 IP 주소를 확인한다.

https://gist.github.com/justinyoo/20017fe33f1d70fb8a6531e71c3e615c?file=01-get-inbound-ip-address.ps1

그다음에는 DNS에 설정되어 있는 A 레코드 IP 주소를 확인한다. 여기서는 [애저 DNS][az dns] 서비스를 사용한다고 가정한다. A 레코드 값은 하나 이상 설정할 수 있으므로, 여기서는 가장 첫번째 IP 주소라고 하자.

https://gist.github.com/justinyoo/20017fe33f1d70fb8a6531e71c3e615c?file=02-get-arecord.ps1

이제 이 두 개의 IP 주소를 비교해서 두 값이 다르다면 애저 DNS의 A 레코드 값을 갱신한다.

https://gist.github.com/justinyoo/20017fe33f1d70fb8a6531e71c3e615c?file=03-compare-ip-address.ps1

여기까지 해서 애저 DNS에 등록된 A 레코드 값을 검사한 후 갱신하는 작업을 해 보았다.


## SSL 인증서 갱신하기 ##

만약 A 레코드 값이 갱신되었다면 기존에 사용하던 인증서는 더이상 유효하지 않다. 따라서, SSL 인증서도 갱신해 주어야 한다. [지난 포스트][post 2]에서 사용했던 [SSL 인증서 업데이트 도구][gh acmebot keyvault]는 [HTTP API 엔드포인트도 함께 제공][gh acmebot keyvault httpapi]하므로, 이를 이용해서 파워셸 스크립트를 작성해 보면 아래와 같다.

https://gist.github.com/justinyoo/20017fe33f1d70fb8a6531e71c3e615c?file=04-update-ssl-certificate.ps1

이제 SSL 인증서 역시 갱신된 IP 주소를 반영하여 새롭게 갱신되었다.

![][image-02]

![][image-03]



## SSL 인증서 동기화하기 ##

SSL 인증서가 갱신되었지만, 아직 애저 펑션 앱 인스턴스에는 그 결과가 반영되지 않았다.

![][image-04]

갱신된 SSL 인증서는 최대 [48시간 이내][az kv certificate import]에 애저 펑션 인스턴스에 동기화 되어 반영된다. 하지만, 48시간이 너무 길다 생각한다면, 아래 파워셸 스크립트를 이용해서 수동으로 동기화할 수 있다. 먼저 액세스 토큰을 가져온다. 여기서는 [서비스 계정(Service Principal)][az pwsh spn]을 이용한다고 가정하고 서비스 계정의 Client ID 값을 이용해서 액세스 토큰을 받아온다.

https://gist.github.com/justinyoo/20017fe33f1d70fb8a6531e71c3e615c?file=05-get-access-token.ps1

다음으로는 유닉스 타임스탬프 값을 밀리세컨드 단위로 받아온다.

https://gist.github.com/justinyoo/20017fe33f1d70fb8a6531e71c3e615c?file=06-get-epoch.ps1

애저 HTTP API 엔드포인트를 아래와 같이 구성한다. 이미 서비스 계정으로 로그인 했기 때문에 `$subscriptionId` 값은 알고 있다고 가정한다.

https://gist.github.com/justinyoo/20017fe33f1d70fb8a6531e71c3e615c?file=07-set-endpoint.ps1

기존에 등록되어 있던 인증서 값을 HTTP API 호출을 통해 받아온다.

https://gist.github.com/justinyoo/20017fe33f1d70fb8a6531e71c3e615c?file=08-get-ssl-certificate.ps1

이렇게 받아온 인증서 값을 `PUT` 요청으로 보내면 새 인증서가 동기화된다.

https://gist.github.com/justinyoo/20017fe33f1d70fb8a6531e71c3e615c?file=09-sync-ssl-certificate.ps1

동기화된 결과는 `$result` 개체에서 확인할 수 있는데, `$result.properties.thumbprint` 값과 `$cert.properties.thumbprint` 값이 달라야만 새 인증서로 동기화가 완료된 것이다. 만약 동일하다면 갱신된 인증서가 아직 반영되지 않았다는 것을 의미한다. 동기화가 끝나면 아래와 같이 애저 포털에서 확인 가능하다.

![][image-05]


## 깃헙 액션 워크플로우 만들기 ##

위의 세 작업을 각각의 깃헙 액션으로 만들어 보자. 그런데, 굳이 깃헙 액션으로 만든 이유가 있을까?

* 깃헙 액션 역시도 다른 서버리스 서비스와 비슷하게 이벤트 기반으로 작동한다.
* 다른 서버리스 서비스들과 달리 최소한의 인스턴스 설정 조차도 필요 없는데, 리포지토리만 있으면 깃헙 액션은 곧바로 작동하기 때문이다.
* 오픈 소스 프로젝트의 경우 깃헙 액션을 무료로 사용할 수 있다.

모든 깃헙 액션은 애저 파워셸을 기반으로 하기 때문에 아래와 같이 공통의 `Dockerfile`을 사용한다.

https://gist.github.com/justinyoo/20017fe33f1d70fb8a6531e71c3e615c?file=10-az-powershell.dockerfile

각 액션의 `entrypoint.ps1` 파일은 위에 언급한 로직을 통해 아래와 같이 작성한다.


### A 레코드 갱신 ###

아래는 A 레코드를 갱신하는 깃헙 액션이다. A 레코드가 갱신되지 않으면 SSL 인증서 갱신을 할 필요가 없기 때문에 A 레코드 갱신 여부를 반환한다 (line #27, 38).

https://gist.github.com/justinyoo/20017fe33f1d70fb8a6531e71c3e615c?file=11-update-arecord.ps1&highlights=27,38


### SSL 인증서 갱신 ###

아래는 SSL 인증서를 갱신하는 깃헙 액션이다. 인증서 갱신에 실패하면 다음 액션인 인증서 동기화를 할 필요가 없으므로 인증서 갱신 여부를 반환한다 (line #14).

https://gist.github.com/justinyoo/20017fe33f1d70fb8a6531e71c3e615c?file=12-update-ssl-certificate.ps1&highlights=14


### SSL 인증서 동기화 ###

아래는 갱신된 SSL 인증서를 동기화하는 깃헙 액션이다. 인증서 동기화 여부를 반환한다 (line #44).

https://gist.github.com/justinyoo/20017fe33f1d70fb8a6531e71c3e615c?file=13-sync-ssl-certificate.ps1&highlights=44


### 깃헙 액션 워크플로우 ###

가장 이상적인 형태는 애저 펑션 앱 인스턴스의 인바운드 IP 주소가 변경될 때 이벤트를 발생시키고 이 이벤트가 깃헙 액션을 실행시키는 것이겠지만, 이 글을 쓰는 현재 관련 이벤트는 존재하지 않으므로, 스케줄러를 통해 A 레코드 갱신 여부를 확인하는 워크플로우를 작성한다.

* 액션 트리거는 스케줄러이므로 아래와 같이 CRON 스케줄러를 설정한다 (line #4-5). 여기서는 30분마다 스케줄러를 실행시킨다.
* 여기에 사용한 액션들은 마켓플레이스에 공개적으로 등록한 것들이 아니고 소스코드에 포함되어 있으므로 소스코드 체크아웃을 한다 (line #14-15).
* DNS의 A 레코드를 갱신한다.
* A 레코드 갱신 여부에 따라 (line #29) SSL 인증서를 갱신한다.
* SSL 인증서 갱신 여부에 따라 (line #37) SSL 인증서를 애저 펑션 인스턴스와 동기화한다.
* SSL 인증서 동기화 여부에 따라 (line #47) 관리자에게 알림 이메일을 보낸다.

https://gist.github.com/justinyoo/20017fe33f1d70fb8a6531e71c3e615c?file=14-github-action-workflow.yaml&highlights=4-5,14-15,29,37,47

여기까지 한 후 실제로 워크플로우가 돌아가면 아래와 같이 SSL 인증서 동기화까지 끝난 결과가 보인다.

![][image-06]

그리고, 실제 이메일 알림은 아래와 같다.

![][image-07]

만약, DNS의 현재 A 레코드가 최신이라면, 아래와 같이 깃헙 액션 워크플로우 안에서 나머지 액션을 실행하지 않는다.

![][image-08]

---

지금까지 [깃헙 액션][gh actions]을 이용해서 주기적으로 [애저 펑션][az func]의 인바운드 IP 주소를 확인한 후, IP 주소 변경 사항을 [애저 DNS][az dns]의 A 레코드에 반영시킨 후, SSL 인증서를 갱신하고 마지막으로 애저 펑션의 인증서와 동기화하는 작업에 대해 알아보았다. [다음 포스트][post 4]에서는 애저 펑션 앱을 깃헙 액션을 통해 배포할 때 배포 프로필 없이 하는 방법에 대해 알아보기로 한다.


[image-01]: /2020/10/updating-azure-dns-and-ssl-certificate-on-azure-functions-via-github-actions-01.png
[image-02]: /2020/10/updating-azure-dns-and-ssl-certificate-on-azure-functions-via-github-actions-02.png
[image-03]: /2020/10/updating-azure-dns-and-ssl-certificate-on-azure-functions-via-github-actions-03.png
[image-04]: /2020/10/updating-azure-dns-and-ssl-certificate-on-azure-functions-via-github-actions-04.png
[image-05]: /2020/10/updating-azure-dns-and-ssl-certificate-on-azure-functions-via-github-actions-05.png
[image-06]: /2020/10/updating-azure-dns-and-ssl-certificate-on-azure-functions-via-github-actions-06.png
[image-07]: /2020/10/updating-azure-dns-and-ssl-certificate-on-azure-functions-via-github-actions-07.png
[image-08]: /2020/10/updating-azure-dns-and-ssl-certificate-on-azure-functions-via-github-actions-08.png

[post 1]: /ko/2020/10/07/3-ways-mapping-apex-domains-to-azure-functions/
[post 2]: /ko/2020/10/14/lets-encrypt-ssl-certificate-on-azure-functions/
[post 3]: /ko/2020/10/28/updating-azure-dns-and-ssl-certificate-on-azure-functions-via-github-actions/
[post 4]: /ko/2020/11/05/deploying-azure-functions-via-github-actions-without-publish-profile/

[az func]: https://docs.microsoft.com/ko-kr/azure/azure-functions/functions-overview?WT.mc_id=devops-10318-juyoo

[az kv certificate import]: https://docs.microsoft.com/ko-kr/azure/app-service/configure-ssl-certificate?WT.mc_id=devops-10318-juyoo#import-a-certificate-from-key-vault
[az dns]: https://docs.microsoft.com/ko-kr/azure/dns/dns-overview?WT.mc_id=devops-10318-juyoo
[az pwsh]: https://docs.microsoft.com/ko-kr/powershell/azure/new-azureps-module-az?WT.mc_id=devops-10318-juyoo
[az pwsh spn]: https://docs.microsoft.com/ko-kr/powershell/azure/create-azure-service-principal-azureps?WT.mc_id=devops-10318-juyoo

[gh sample]: https://github.com/devrel-kr/dvrl-kr-dns-update
[gh actions]: https://docs.github.com/en/free-pro-team@latest/actions
[gh acmebot keyvault]: https://github.com/shibayan/keyvault-acmebot
[gh acmebot keyvault httpapi]: https://github.com/shibayan/keyvault-acmebot/wiki/REST-API

[letsencrypt]: https://letsencrypt.org/
