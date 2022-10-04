---
title: "애저 펑션에 Let's Encrypt 인증서 자동으로 연동하기"
slug: lets-encrypt-ssl-certificate-on-azure-functions
description: "이 포스트에서는 애저 펑션 인스턴스에 Let's Encrypt로 손쉽게 SSL 인증서를 생성하고 이를 연결하는 방법에 대해 논의해 봅니다."
date: "2020-10-14"
author: Justin-Yoo
tags:
- azure-functions
- apex-domain
- lets-encrypt
- ssl-certificate
cover: /2020/10/lets-encrypt-ssl-certificate-on-azure-functions-00.png
fullscreen: true
---

이 시리즈에서는 애저 펑션 인스턴스를 운영하면서 사용자 지정 도메인을 연결하고, SSL 인증서를 붙이고, 수시로 변하는 공개 IP 주소를 관리하는 방법에 대해 알아본다.

* [애저 펑션에 APEX 도메인을 연결하는 세 가지 방법][post 1]
* ***애저 펑션에 Let's Encrypt 인증서 자동으로 연동하기***
* [애저 펑션의 IP주소 변경시 깃헙 액션을 통해 DNS와 SSL 인증서를 자동으로 갱신하기][post 3]
* [애저 펑션 배포 프로필 없이 깃헙 액션으로 배포하기][post 4]

[지난 포스트][post 1]에서는 [애저 펑션][az func] 인스턴스에 루트 도메인 혹은 APEX 도메인을 연결하는 방법에 대해 알아봤다면, 이번에는 [Let's Encrypt][letsencrypt]에서 발행하는 SSL 인증서를 연동해서 사용자 지정 도메인에 대해 HTTPS 커넥션을 가능하게끔 해 보기로 한다.


## Let's Encrypt ##

[Let's Encrypt][letsencrypt]는 무료로 SSL 인증서를 발급해 주는 비영리 단체이다. 무료라고 해서 인증서 품질이 나쁘다거나 하지는 않고, 굉장히 많은 기업에서 스폰서를 해 주고 있다. 여기서 발행하는 SSL 인증서는 100% 무료이며, 동시에 3개월의 유효기간을 갖고 있다. 따라서, 매 3개월마다 인증서를 갱신해야 하는 불편함이 있다. 하지만, 이런 불편함은 자동화로 극복할 수 있기 때문에 처음 설정만 제대로 하면 크게 문제가 없다.


## 애저 앱 서비스 사이트 확장 기능 ##

애저 앱 서비스는 써드파티 확장 기능을 제공하고 있다. 그 중 하나가 바로 이 [Let's Encrypt 확장 기능][gh site extension]이다. [애저 WebJob][az webjob]의 형태로 되어 있어서 3개월마다 자동으로 연장도 해주고 해서 꽤 유용한 기능이다.

![][image-01]

그런데, 이 확장 기능의 치명적인 단점이 몇 가지가 있다.

* 리눅스 기반 앱 서비스(애저 펑션 포함)에는 적용할 수 없다.
* 앱 서비스 인스턴스에 강력한 의존성을 갖고 있다. 앱 서비스 인스턴스를 생성할 때 마다 새로 확장 기능도 설치해야 한다.
* 앱 배포시 삭제후 재배포 옵션을 선택하면 확장 기능도 함께 삭제된다.

인증서를 설치하기 위해 매 번 확장 기능을 설치하는 것도 번거롭고 그다지 안정적인 방법으로는 보이지 않는다.


## 인증서 관리 전용 애저 펑션 앱 ##

다행히도 애저 MVP인 [Shibayan][tw shibayan]이 굉장히 훌륭한 [애저 펑션 앱][gh acmebot keyvault]을 만들어서 배포중이다. 이를 이용하면 굉장히 손쉽게 여러 사이트에 대한 인증서를 생성해서 [애저 키 저장소][az kv]에 저장한다. 이렇게 저장된 인증서는 곧바로 애저 펑션 인스턴스에 바인딩해서 사용할 수 있다.

가장 먼저 아래 ARM 템플릿을 실행시켜 애저 펑션 앱과 키 저장소 인스턴스를 생성한다. 이는 미리 만들어 둔 템플릿을 사용하는 것이고, 직접 애저 리소스를 프로비저닝하고 싶다면, 직접 해도 상관은 없다.

[![](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fshibayan%2Fkeyvault-acmebot%2Fmaster%2Fazuredeploy.json)

위 링크를 통해 애저 리소소를 프로비저닝했다면, 애저 펑션과 애저 키 저장소 인스턴스가 하나 만들어진다. 그리고, 애저 펑션은 [관리 ID 기능][az func mi]이 활성화 되어 있어서 직접 키 저장소에 접근이 가능하다. 리소스 프로비저닝이 끝난 후에는 아래 절차를 따라한다.

> 인증서 관리용 애저 펑션 앱의 주소는 `https://ssl-management.azurewebsites.net` 이라고 가정하자.


### 인증 및 권한 부여 ###

인증서 발급용 애저 펑션 인스턴스는 관리 UI를 포함하고 있고, 이는 인증을 통해 접근 가능하다. 따라서 아래와 같이 [인증 및 권한 부여][az func auth] 기능을 활성화 시켜야 한다.

![][image-02]

그리고, 위 그림에서와 같이 애저 액티브 디렉토리를 설정한다. 애저 포탈에 접속하는 계정을 사용해서 로그인할 수 있게끔 하기 위함이다. 아래와 같이 관리 모드를 `기본`으로 설정하고 `Azure AD 앱` 값을 설정한다. 기본적으로 애저 펑션 앱의 이름으로 할 수 있게 되어 있으므로, 별다른 수정 없이 그대로 사용하기로 한다.

![][image-03]

이렇게 해서 인증서 발급용 애저 펑션 인스턴스에 대한 설정이 끝났다.


### 애저 DNS 설정 ###

여기서는 [애저 DNS 서비스][az dns]를 사용해서 도메인을 관리하는 것으로 가정한다. 애저 DNS 서비스가 들어 있는 리소스 그룹의 `액세스 제어 (IAM)` 블레이드를 선택하고 아래와 같이 역할을 할당한다.

![][image-04]

* `역할`: DNS 영역 참가자
* `액세스 할당`: 함수 앱
* `구성원`: 인증서 발급용 애저 펑션 앱


## 인증서 생성 ##

이제 준비가 끝났으니, 인증서 관리용 애저 펑션의 엔드포인트로 웹 브라우저를 열어 접속한다. 엔드포인트 주소는 `https://ssl-management.azurewebsites.net/add-certificate`이다. 최초 접속시에는 아래와 같이 로그인 화면이 나온다.

![][image-05]

로그인 후에는 아래와 같이 인증서를 생성하기 위한 화면이 나온다. 여기서 APEX 도메인의 경우에는 `Record name` 필드에 아무것도 넣지 않고 `Add` 버튼을 클릭하고, 서브도메인을 사용하려면 서브도메인 이름을 넣고 추가한다. 여기서는 `cnts.com`, `dev.cnts.com` 두 개의 도메인을 위한 인증서를 생성한다.

![][image-06]

> 위의 방법을 따라 하면, 하나의 인증서로 `cnts.com`, `dev.cnts.com` 이렇게 두 도메인을 사용할 수 있다. 만약 별도의 인증서가 필요하다면 한 번에 하나씩 생성해야 한다.

인증서 생성이 끝났다면 아래와 같은 팝업이 나타난다.

![][image-07]

이제 애저 키 저장소로 가서 인증서 생성 여부를 확인해 보면 인증서가 아래와 같이 생성된 것을 볼 수 있다.

![][image-08]


## 애저 펑션에 인증서 적용하기 ##

이제 [이전 포스트][post 1]에서 지정한 사용자 지정 도메인에 인증서를 연결할 차례이다. 내가 실제로 서비스 할 애저 펑션 인스턴스의 TLS/SSL 설정 블레이드로 들어간다. `프라이빗 키 인증서 (.pfx)` 탭을 선택하고 `Key Vault 인증서 가져오기` 버튼을 클릭해서, 애저 키 저장소에서 방금 생성한 인증서를 가져온다.

![][image-09]

인증서 가져오기가 끝나면 아래와 같은 화면을 볼 수 있다. 앞서 `cnts.com`, `dev.cnts.com` 두 도메인을 관장하는 인증서를 만들었던 것을 기억한다면 아래와 같은 화면이 보이는 것이 정상이다.

![][image-10]

이제 `사용자 지정 도메인` 블레이드로 가 보면 아직 SSL 바인딩이 적용되지 않은 것을 볼 수 있다. `바인딩 추가`를 클릭한다. `사용자 지정 도메인` 필드에서 `cnts.com`을 선택하고, `프라이빗 인증서 지문 필드에서` `cnts.com,dev.cnts.com`을 선택한다. 마지막으로 `TLS/SSL 유형` 필드에서 `SNI SSL`을 선택한다.

![][image-11]

사용자 지정 도메인에 SSL 인증서가 제대로 바인딩이 된 것을 알 수 있다.

![][image-12]

---

지금까지 [애저 펑션][az func]에 사용자 지정 도메인을 설정하고 [Let's Encrypt][letsencrypt]로 생성한 SSL 인증서를 연결해 보았다. [다음 포스트][post 3]에서는 애저 펑션의 IP 주소가 변경됐을 때 자동으로 이를 [애저 DNS][az dns]의 A 레코드에 반영시켜주는 방법에 대해 알아보기로 한다.


[image-01]: /2020/10/lets-encrypt-ssl-certificate-on-azure-functions-01-ko.png
[image-02]: /2020/10/lets-encrypt-ssl-certificate-on-azure-functions-02-ko.png
[image-03]: /2020/10/lets-encrypt-ssl-certificate-on-azure-functions-03-ko.png
[image-04]: /2020/10/lets-encrypt-ssl-certificate-on-azure-functions-04-ko.png
[image-05]: /2020/10/lets-encrypt-ssl-certificate-on-azure-functions-05-ko.png
[image-06]: /2020/10/lets-encrypt-ssl-certificate-on-azure-functions-06-ko.png
[image-07]: /2020/10/lets-encrypt-ssl-certificate-on-azure-functions-07-ko.png
[image-08]: /2020/10/lets-encrypt-ssl-certificate-on-azure-functions-08-ko.png
[image-09]: /2020/10/lets-encrypt-ssl-certificate-on-azure-functions-09-ko.png
[image-10]: /2020/10/lets-encrypt-ssl-certificate-on-azure-functions-10-ko.png
[image-11]: /2020/10/lets-encrypt-ssl-certificate-on-azure-functions-11-ko.png
[image-12]: /2020/10/lets-encrypt-ssl-certificate-on-azure-functions-12-ko.png

[post 1]: /ko/2020/10/07/3-ways-mapping-apex-domains-to-azure-functions/
[post 2]: /ko/2020/10/14/lets-encrypt-ssl-certificate-on-azure-functions/
[post 3]: /ko/2020/10/28/updating-azure-dns-and-ssl-certificate-on-azure-functions-via-github-actions/
[post 4]: /ko/2020/11/05/deploying-azure-functions-via-github-actions-without-publish-profile/

[az func]: https://docs.microsoft.com/ko-kr/azure/azure-functions/functions-overview?WT.mc_id=aliencubeorg-blog-juyoo
[az func mi]: https://docs.microsoft.com/ko-kr/azure/app-service/overview-managed-identity?tabs=dotnet&WT.mc_id=aliencubeorg-blog-juyoo
[az func auth]: https://docs.microsoft.com/ko-kr/azure/app-service/overview-authentication-authorization?WT.mc_id=aliencubeorg-blog-juyoo

[az webjob]: https://docs.microsoft.com/ko-kr/azure/app-service/webjobs-create?WT.mc_id=aliencubeorg-blog-juyoo
[az kv]: https://docs.microsoft.com/ko-kr/azure/key-vault/general/overview?WT.mc_id=aliencubeorg-blog-juyoo
[az dns]: https://docs.microsoft.com/ko-kr/azure/dns/dns-overview?WT.mc_id=aliencubeorg-blog-juyoo

[gh site extension]: https://github.com/sjkp/letsencrypt-siteextension
[gh acmebot keyvault]: https://github.com/shibayan/keyvault-acmebot

[letsencrypt]: https://letsencrypt.org/

[tw shibayan]: https://twitter.com/shibayan
