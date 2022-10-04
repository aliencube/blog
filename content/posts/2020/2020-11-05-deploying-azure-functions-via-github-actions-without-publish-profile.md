---
title: "애저 펑션 배포 프로필 없이 깃헙 액션으로 배포하기"
slug: deploying-azure-functions-via-github-actions-without-publish-profile
description: "이 포스트에서는 깃헙 액션을 이용해서 애저 펑션을 배포할 때, 배포 프로필을 별도의 외부 입력에 의존하지 않고 직접 사용하는 방법에 대해 알아 봅니다."
date: "2020-11-05"
author: Justin-Yoo
tags:
- azure-functions
- github-actions
- azure
- publish-profile
cover: /2020/11/deploying-azure-functions-via-github-actions-without-publish-profile-00.png
fullscreen: true
---

이 시리즈에서는 애저 펑션 인스턴스를 운영하면서 사용자 지정 도메인을 연결하고, SSL 인증서를 붙이고, 수시로 변하는 공개 IP 주소를 관리하는 방법에 대해 알아본다.

* [애저 펑션에 APEX 도메인을 연결하는 세 가지 방법][post 1]
* [애저 펑션에 Let's Encrypt 인증서 자동으로 연동하기][post 2]
* [애저 펑션의 IP주소 변경시 깃헙 액션을 통해 DNS와 SSL 인증서를 자동으로 갱신하기][post 3]
* ***애저 펑션 배포 프로필 없이 깃헙 액션으로 배포하기***

[지난 포스트][post 3]에서는 [애저 펑션][az func] 인스턴스의 인바운드 IP가 변할 때 마다 [깃헙 액션][gh actions]을 이용해서 이를 자동으로 DNS에 반영하고, SSL 인증서를 갱신하는 방법에 대해 알아보았다. 이번 포스트는 이 시리즈의 마지막으로, 애저 펑션을 깃헙 액션을 통해 배포할 때 외부에서 배포 프로필을 입력받는 대신, 액션 워크플로우 안에서 자체적으로 해결할 수 있는 방법에 대해 알아보자.


## 애저 펑션 배포 액션 ##

[애저 펑션][az func]을 배포하기 위한 [공식 액션][gh actions azfunc]은 이미 깃헙 마켓플레이스에 올라와 있다. 공식 액션의 사용법은 대략 아래와 같다. `publish-profile` 파라미터를 통해 배포 프로필 값을 받아 애저 펑션 앱을 배포한다 (line #11).

https://gist.github.com/justinyoo/d806e47efde7749c6154b19f081f86d2?file=01-az-func-deploy.yaml&highlights=11


그런데 위에서 볼 수 있다시피, 이 배포 프로필은 리포지토리의 시크릿 환경 설정값을 통해 가져와야 한다. 만약 배포 프로필을 리셋한다거나 하면, 이 환경 설정값을 함께 수동으로 변경해 줘야 한다. 어찌 보면 번거로운 작업인데, 그렇다면 환경 설정값을 이용하는 대신 깃헙 액션 워크플로우 안에서 이를 처리할 수 있게 하면 어떨까? 충분히 가능하다.


## 파워셸 스크립트 ##

가장 먼저 [애저 로그인 프로필][gh actions az login]을 이용해서 [애저 파워셸][az pwsh]에 로그인한다. `enable-AzPSSession` 속성값을 `true`로 놓으면 애저 파워셸 모듈에 직접 접속할 수 있다 (line #9).

https://gist.github.com/justinyoo/d806e47efde7749c6154b19f081f86d2?file=02-az-login.yaml&highlights=9

이후 아래 파워셸 스크립트를 통해 애저 펑션 앱 인스턴스의 배포 프로필 값을 받아온다 (line #10-12). 그 다음 라인에서는 줄바꿈 문자열을 모두 없애는 명령어이고 (line #14), 마지막으로 배포 프로필을 반환값으로 지정한다 (line #16).

https://gist.github.com/justinyoo/d806e47efde7749c6154b19f081f86d2?file=03-get-publish-profile.yaml&highlights=10-12,14,16

이렇게 한 후 반환값을 깃헙 액션 워크플로우에 출력해 보자.

https://gist.github.com/justinyoo/d806e47efde7749c6154b19f081f86d2?file=04-show-publish-profile.yaml&highlights=12

그 결과는 아래와 같다. 배포 프로필을 성공적으로 가져오긴 했는데, 워크플로우 로그에서 패스워드 부분이 제대로 마스킹 처리가 되지 않았다. 따라서, 보안상 이 배포 프로필은 더이상 안전하지 않다고 가정해야 한다.

![][image-01]

배포 프로필 자체는 유효하지만, 한 번 배포에 사용하고 난 후에는 가급적이면 프로필을 리셋하는 것이 보안상 안전하다. 따라서, 아래와 같이 파워셸 스크립트를 이용해 배포 프로필을 리셋해 줘야 한다 (line #9-13).

https://gist.github.com/justinyoo/d806e47efde7749c6154b19f081f86d2?file=05-reset-publish-profile.yaml&highlights=9-13

이렇게 함으로써 애저 펑션 앱을 깃헙 액션을 이용해서 배포할 때 배포 프로필을 워크플로우 상에서 자동으로 추출하고 배포에 이용한 후 리셋하는 일련의 과정을 살펴봤다.


## 애저 앱 서비스 배포 프로필 액션 ##

그렇다면, 이 모든 과정을 그때그때 스크립트를 만들기 보다는 아예 깃헙 액션으로 만들어 놓으면 어떨까? 물론 이미 만들어 놓았다. [애저 앱 서비스 배포 프로필 액션][gh actions appsvc profile]을 이용하면 배포 프로필을 가져오는 것과 리셋하는 것을 한번에 손쉽게 진행할 수 있다. 액션이 실제로 하는 일은 위에 언급한 파워셸 스크립트와 동일하다. 따라서, 아래와 같이 깃헙 액션 워크플로우를 구성하면 좋다.

* 가장 먼저 배포 프로필을 가져온다 (line #12-19).
* 다음으로, 받아온 배포 프로필을 이용해서 애저 펑션 앱을 배포한다 (line #26).
* 마지막으로, 배포 프로필을 리셋한다 (line #28-35).

https://gist.github.com/justinyoo/d806e47efde7749c6154b19f081f86d2?file=06-workflow.yaml&highlights=12-19,26,28-35

이렇게 하면 배포 프로필을 굳이 알 필요도 없이 워크플로우 안에서 모두 해결이 가능하다. 거기에 더해서, 배포 프로필이 실수로 노출된다고 하더라도 마지막 액션에서 리셋해 버리기 때문에 더이상 노출된 배포 프로필은 유효하지 않다.

---

지금까지 [깃헙 액션][gh actions]을 이용해서 애저 펑션 앱을 배포할 때 배포 프로필을 워크플로우 안에서 자동으로 추출해서 사용하고 리셋하는 방법에 대해 알아 보았다. 이 포스트에서 소개한 깃헙 액션이 애저 펑션 앱을 조금이나마 손쉽게 배포하는 데 도움이 되길 바란다.


[image-01]: /2020/11/deploying-azure-functions-via-github-actions-without-publish-profile-01.png

[post 1]: /ko/2020/10/07/3-ways-mapping-apex-domains-to-azure-functions/
[post 2]: /ko/2020/10/14/lets-encrypt-ssl-certificate-on-azure-functions/
[post 3]: /ko/2020/10/28/updating-azure-dns-and-ssl-certificate-on-azure-functions-via-github-actions/
[post 4]: /ko/2020/11/05/deploying-azure-functions-via-github-actions-without-publish-profile/

[az func]: https://docs.microsoft.com/ko-kr/azure/azure-functions/functions-overview?WT.mc_id=devops-10694-juyoo

[az pwsh]: https://docs.microsoft.com/ko-kr/powershell/azure/new-azureps-module-az?WT.mc_id=devops-10694-juyoo

[gh actions]: https://docs.github.com/en/free-pro-team@latest/actions
[gh actions az login]: https://github.com/marketplace/actions/azure-login#configure-deployment-credentials
[gh actions azfunc]: https://github.com/marketplace/actions/azure-functions-action
[gh actions appsvc profile]: https://github.com/marketplace/actions/azure-app-service-publish-profile
