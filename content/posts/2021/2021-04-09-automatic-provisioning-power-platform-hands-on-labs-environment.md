---
title: "파워 플랫폼 핸즈온랩 자동 환경 설정"
slug: automatic-provisioning-power-platform-hands-on-labs-environment
description: "이 포스트에서는 파워 플랫폼 핸즈온 랩을 위한 공통 환경 설정을 자동화하는 방법에 대해 알아봅니다."
date: "2021-04-09"
author: Justin-Yoo
tags:
- power-platform
- microsoft365
- provisioning
- powershell
cover: /2021/04/automatic-provisioning-power-platform-hands-on-labs-environment-00.png
fullscreen: true
---

[파워 플랫폼][pp] 핸즈온 실습을 진행해야 하는 당신! 실습을 위한 컨텐츠는 준비가 됐지만, 실습 환경에 대한 고민을 시작할 때가 됐다. 파워 플랫폼 핸즈온 실습을 위해서는 크게 세 가지 방법이 있다.

1. 실습자 본인의 기존 파워 플랫폼 환경을 그대로 사용하는 것
1. 실습자 각자 새롭게 환경을 구성하는 방법
1. 진행자가 일괄적으로 환경을 구성하는 방법

각각의 방법은 저마다 장단점이 있게 마련인데, 대략 아래와 같이 정리할 수 있을 것이다.

1. **첫번째 방법**은 진행자 입장에서는 가장 편리한 방법이다. 이미 실습자의 환경이 구성되어 있기 때문이다. 하지만, 실습자 개개인이 속한 테넌트마다 구성이 다를 수 있으므로 파워 플랫폼 실습에 적합한지 아닌지 알 수가 없다. 실습 동안 권한 문제라든가 다양한 문제를 겪을 가능성이 농후한 편이다.
2. **두번째 방법**은 진행자 입장에서는 편리할 수도 있고 아닐 수도 있다. 우선 환경 구성 자체를 실습자에게 위임하는 과정만 놓고 보자면 실습자는 편리할 수도 있겠지만, 그 환경 구성을 위한 가이드 문서를 꼼꼼하게 작성해서 전달해야 한다. 아무리 자세하게 가이드를 제공한다고 하더라도, 실습자 개개인의 컴퓨터 활용 능력에 따라 환경 구성이 손쉬울 수도 있고 아닐 수도 있다. 결국 가이드 문서대로 되었다고 가정할 수 없다.
3. **마지막 방법**은 진행자가 모두 사전에 구성을 해 놓는 방법인데, 실습자 입장에서는 실습에 집중할 수 있으니 아주 편리하다. 반면에 진행자 입장에서는 이를 매번 수동으로 설정한다고 생각하면 굉장히 끔찍한 일이다.

따라서, 이 포스트에서는 핸즈온랩 실습 진행자 관점에서 최초 수동으로 해야만 하는 부분을 제외하고는 모두 스크립트 한 번 실행으로 환경 설정을 최대한 자동화하는 방법에 대해 알아보기로 한다.

> 이 포스트에서 사용한 파워셸 스크립트는 [이 리포지토리][gh sample]에서 다운로드 받을 수 있다.


## 명령어 하나로 환경 설정하기 ##

편의상 여기서는 아래와 같은 내용으로 테넌트를 생성했다고 가정한다.

* 테넌트 이름: `powerplatformhandsonlab`
* 테넌트 URL: `powerplatformhandsonlab.onmicrosoft.com`
* 관리자 이메일: `admin@powerplatformhandsonlab.onmicrosoft.com`
* 관리자 패스워드: `Pa$$W0rd!@#$`

이 정보를 이용해서, 한 번에 환경 설정을 하려면 어떻게 하면 될까? 여기 전체 [파워셸 스크립트][gh sample code]가 있으니 이걸 이용해 아래와 같이 명령어를 실행시키면 된다.

https://gist.github.com/justinyoo/4cba9e122dfdb5684cd432c072b36b3d?file=15-set-environment.ps1

이게 된다고? 하나씩 살펴보기로 하자.


## Microsoft 365 테넌트 생성 ##

실습 진행자가 가장 먼저 해야 할 일은 [Microsoft 365][m365] 테넌트를 생성하는 것이다. Microsoft 365는 30일짜리 무료 평가판을 제공하는데, 관리자를 포함해 총 25개의 라이센스를 제공하므로, 실습용으로 사용하기 좋다. [http://aka.ms/Office365E5Trial][m365 trial e5] 링크를 클릭해서 아래와 같이 무료로 Miicrosoft 365 E5 평가판 테넌트를 생성한다.

![Microsoft 365 E5 평가판 가입 첫화면][image-01]

아래 화면에서 필요한 정보를 모두 입력하면 평가판 테넌트가 만들어진다.

![Microsoft 365 E5 평가판 가입 화면][image-02]

이제 테넌트가 만들어졌으니 실제로 실습 환경을 파워셸로 구성해 보자. **참고로 아래에서 진행한 모든 파워셸 명령어는 관리자 권한의 파워셸 콘솔 안에서 실행시켜야 한다.**


## 프로비저닝 순서 ##

환경 구성을 위한 순서가 딱히 정해져 있지는 않다. 하지만, 아래 순서대로 하는 것을 권장하는 편인데, 파워 앱 모듈과 애저 AD 모듈 사이에 호환성이 없어지는 부분이 하나 있기 때문이다.

1. Microsoft Dataverse 데이터베이스 초기화
1. 사용자 계정 추가
1. Microsoft 365 권한 부여
1. 라이센스 부여
1. 애저 권한 부여

만약에 첫번째 Microsoft Dataverse 데이터베이스 초기화를 가장 먼저 하지 않고 사용자 계정 추가 이후에 진행한다면 에러가 생긴다. 어떤 에러가 생기는 지는 아래에서 다시 다루도록 한다.

> **NOTE**: `AzureAD` 모듈을 사용하기 위해서는 Windows 환경의 파워셸 5.1 버전이 필요하다. 파워셸 6 이상의 버전은 지원하지 않는다. 사용 환경에 대해서는 [파워셸을 통해 Microsoft 365에 연결하기][m365 powershell connect] 페이지를 참조한다.


## AzureAD 파워셸 모듈 설치 ##

Microsoft 365 테넌트에 사용자 계정을 추가하기 위해서는 먼저 [`AzureAD` 모듈][psgallery azuread]을 설치해야 한다. 이 포스트를 쓰는 현재 최신 버전은 `2.0.2.130`이다. 아래와 같이 `Install-Module` 명령어를 사용하면 되는데, 맨 마지막의 `-Force -AllowClobber` 옵션을 주면 동일한 버전이 이미 설치가 되어 있을 경우에도 별다른 경고 메시지 없이 재설치한다.

https://gist.github.com/justinyoo/4cba9e122dfdb5684cd432c072b36b3d?file=01-install-azuread.ps1


## AzureAD 관리자 로그인 ##

파워셸 모듈 설치가 끝났다면, 이제 테넌트 관리자로 로그인 할 차례이다. 자동화 스크립트의 경우에는 가능한 한 현재 파워셸 콘솔 창에서 벗어나지 않고 모두 해결해야 하므로 아래와 같은 방식으로 로그인을 하면 효과적이다.

https://gist.github.com/justinyoo/4cba9e122dfdb5684cd432c072b36b3d?file=02-connect-azuread.ps1


## 사용자 추가 ##

관리자 로그인이 끝났다면, 이제 본격적으로 사용자를 추가할 차례이다. 평가판 테넌트에는 관리자를 포함해서 모두 25개의 라이센스를 사용할 수 있으므로, 추가 가능한 사용자 수는 최대 24명이다. 이와 관련한 자세한 내용은 [파워셸로 Microsoft 365 사용자 계정 만들기][m365 powershell account create] 페이지를 참조하고, 여기서는 아래 파워셸 스크립트를 실행시킨다.

* 사용자의 패스워드는 편의상 `UserPa$$W0rd!@#$`로 통일시켰으며, 임의로 바꿀 수 없게끔 설정한다.
* 사용자의 지역은 테넌트가 만들어진 지역으로 설정한다. 여기서는 `KR`이다.
* 모두 24개의 계정을 만들어야 하므로 `ForEach-Object` 루프 구문을 사용한다.
* 만들어진 계정은 나중에 다시 사용하기 위해 모두 `$users` 배열 개체에 저장한다.

https://gist.github.com/justinyoo/4cba9e122dfdb5684cd432c072b36b3d?file=03-new-azureaduser.ps1


## 사용자 계정 역할 및 권한 부여 ##

앞서 추가한 사용자에게 적합한 권한을 부여할 차례이다. 파워 플랫폼 실습을 위한 환경 설정이니만큼, 파워 플랫폼 관리자 권한을 부여하기로 한다. 그에 앞서 아래 명령어를 통해 파워 플랫폼 관리자 권한을 활성화 시킨다. 권한 할당과 관련한 자세한 내용은 이 [파워셸을 통해 Microsoft 365 계정에 관리자 권한 부여하기][m365 powershell role assign] 페이지를 참조한다.

https://gist.github.com/justinyoo/4cba9e122dfdb5684cd432c072b36b3d?file=04-get-azureaddirectoryrole.ps1

위와 같이 파워 플랫폼 관리자 권한을 `$role` 개체로 받아왔으니, 이를 앞서 생성한 계정에 모두 부여할 차례이다. 아래 명령어를 실행시킨다.

https://gist.github.com/justinyoo/4cba9e122dfdb5684cd432c072b36b3d?file=05-add-azureaddirectoryrolemember.ps1


## 사용자 계정 라이센스 부여 ##

이번에는 각 사용자에게 파워 플랫폼을 사용할 수 있는 라이센스를 부여할 차례이다. 기본적으로 평가판 테넌트에는 파워 플랫폼을 사용할 수 있는 라이센스를 계정 수 만큼 제공하고 있으니 이를 사용하면 된다. 이와 관련해서 좀 더 자세한 내용은 [파워셸을 통해 계정에 Microsoft 365 라이센스 부여하기][m365 powershell license assign] 페이지를 참조한다.

우선 라이센스 이름을 검색해 보자. 평가판 테넌트를 만든 후 추가적인 작업을 하지 않았으므로 라이센스는 하나만 나올 것이고 그 이름은 `ENTERPRISEPREMIUM`일 것이다.

https://gist.github.com/justinyoo/4cba9e122dfdb5684cd432c072b36b3d?file=06-get-azureadsubscribedsku.ps1

아래 명령어를 사용해서 모든 사용자 계정에 라이센스를 부여한다.

https://gist.github.com/justinyoo/4cba9e122dfdb5684cd432c072b36b3d?file=07-set-azureaduserlicense.ps1

여기까지 해서 핸즈온랩 실습에 필요한 모든 사용자 계정에 역할, 권한, 라이센스까지 자동으로 할당했다.


## 파워 플랫폼 기본 환경 Microsoft Dataverse 활성화 ##

파워 플랫폼 핸즈온랩 실습을 위해서 또 한가지 해야 할 것이 있다. 바로 [Microsoft Dataverse][pp dataverse] 라는 데이터베이스를 활성화 시키는 작업이다. 다양한 Microsoft 365 제품군과 협업을 하기 위해서 반드시 활성화 시켜야 하는 부분이므로, 아래와 같은 절차를 거쳐 활성화 시킨다. 파워셸을 이용한 파워 플랫폼 관리자 기능에 대한 좀 더 자세한 내용은 [관리자용 파워 앱 cmdlet][pp powershell dataverse create] 페이지를 참조한다.

가장 먼저 파워 앱 관련 파워셸 모듈인 [Microsoft.PowerApps.Administration.PowerShell][psgallery pa admin]와 [Microsoft.PowerApps.PowerShell][psgallery pa]을 설치한다. 앞서와 마찬가지로 `-Force -AllowClobber` 옵션을 통해 이미 설치가 되어 있을 경우 재설치한다.

https://gist.github.com/justinyoo/4cba9e122dfdb5684cd432c072b36b3d?file=08-install-powerapps.ps1

위에서 설정한 `$adminUpn`, `$adminPW` 값을 이용해서 파워 앱 환경에 관리자로 로그인한다.

https://gist.github.com/justinyoo/4cba9e122dfdb5684cd432c072b36b3d?file=09-add-powerappsaccount.ps1

> **NOTE**: 로그인 과정에서 아래와 같은 에러메시지를 내면서 파워 앱 환경에 로그인 할 수 없는 경우가 생긴다.
> 
> ![파워 앱 환경 로그인 불가능][image-03]
> 
> Microsoft 365 테넌트에 로그인하는 과정과 파워 앱에 로그인하는 경우가 내부적으로 달라 생기는 일이기 때문에, 당황하지 말고 새 파워셸 콘솔 창을 관리자 권한으로 열고 거기서 로그인한다.

* 기본 환경의 Dataverse 데이터베이스를 활성화 시킨다.
* 기본 환경의 통화 설정을 따라간다.
* 기본 환경의 언어 설정을 따라간다.

https://gist.github.com/justinyoo/4cba9e122dfdb5684cd432c072b36b3d?file=10-new-adminpowerappcdsdatabase.ps1


## 애저 구독 할당 ##

파워 플랫폼을 사용하다 보면 [커스텀 커넥터][pp cuscon]를 사용하기 위해 애저 리소스를 다뤄야 할 경우도 있다. 이 때 애저 구독이 필요한데, 평가판 테넌트를 생성하면, 평가판 애저 구독도 함께 활성화 시킬 수 있다. 다만, 이는 애저 포탈에서 직접 해야 한다. 관리자 계정으로 애저 포탈에 로그인하면 아래와 같은 화면이 나온다.

![애저 구독 평가판 가입 안내][image-04]

시작 버튼을 눌러 평가판 구독 절차를 진행한다.

![애저 구독 평가판 가입 절차][image-05]

위와 같이 관리자 계정에서 평가판 구독 절차가 끝났다면 아래 파워셸 명령어를 실행시켜 애저에 로그인한다. `$adminCredential` 개체는 앞서 AzureAD 로그인에 사용했던 것과 동일하다.

https://gist.github.com/justinyoo/4cba9e122dfdb5684cd432c072b36b3d?file=11-connect-azaccount.ps1

> **NOTE**: 파워셸로 애저 리소스를 관리하려면 관련 파워셸 모듈인 [Az][psgallery az]가 이미 설치되어 있어야 한다. 아래 명령어를 통해 설치한다.
> 
> https://gist.github.com/justinyoo/4cba9e122dfdb5684cd432c072b36b3d?file=12-install-az.ps1

체험판 구독의 경우 사용할 수 있는 애저 리소스의 종류가 제한적이므로 아래 명령어를 통해 파워 플랫폼 핸즈온 실습에 필요한 리소스를 활성화 시킨다. 여기서는 [애저 로직 앱][az logapp], [애저 저장소][az st], [애저 네트워크][az netwrk], [애저 API 매니지먼트][az apim], [애저 Cosmos DB][az cosdba] 정도의 리소스를 사용한다고 가정한다.

https://gist.github.com/justinyoo/4cba9e122dfdb5684cd432c072b36b3d?file=13-register-azresourceprovider.ps1

각 사용자 계정에 구독을 할당한다. 애저 구독과 권한 부여에 관련해서는 이 [애저 파워셸로 애저 리소스에 대한 권한 부여][az powershell role assign] 페이지를 참조하면 좋다.

> **NOTE**: 구독 전체를 할당하기 보다는 각 사용자별로 리소스 그룹을 생성하고 해당 리소스 그룹에만 기여자 권한을 부여해서 할당하는 것이 관리하기에 용이하다. 여기서는 지역을 `koreacentral`로 할당한다.

https://gist.github.com/justinyoo/4cba9e122dfdb5684cd432c072b36b3d?file=14-new-azroleassignment.ps1

각 실습자 계정별로 애저 평가판 구독과 실습에 필요한 리소스 그룹 할당이 끝났다.

---

지금까지 파워 플랫폼 핸즈온 실습을 위한 모든 환경 설정을 파워셸 스크립트로 구성했다. 이제 핸즈온 실습 일정이 잡힐 경우 이 파워셸 스크립트만 실행시키면 짧은 시간 안에 모든 구성을 끝마치고 준비할 수 있을 것이다.


[image-01]: /2021/04/automatic-provisioning-power-platform-hands-on-labs-environment-01-ko.png
[image-02]: /2021/04/automatic-provisioning-power-platform-hands-on-labs-environment-02-ko.png
[image-03]: /2021/04/automatic-provisioning-power-platform-hands-on-labs-environment-03-ko.png
[image-04]: /2021/04/automatic-provisioning-power-platform-hands-on-labs-environment-04-ko.png
[image-05]: /2021/04/automatic-provisioning-power-platform-hands-on-labs-environment-05-ko.png

[gh sample]: https://github.com/devkimchi/PowerPlatform-Hands-on-Lab-Environment-Automatic-Provsioning
[gh sample code]: https://github.com/devkimchi/PowerPlatform-Hands-on-Lab-Environment-Automatic-Provsioning/blob/main/AzureAD/Set-Environment.ps1

[pp]: https://powerplatform.microsoft.com/ko-kr/?WT.mc_id=power-23654-juyoo
[pp dataverse]: https://docs.microsoft.com/ko-kr/powerapps/maker/data-platform/data-platform-intro?WT.mc_id=power-23654-juyoo
[pp cuscon]: https://docs.microsoft.com/ko-kr/connectors/custom-connectors/?WT.mc_id=power-23654-juyoo

[pp powershell dataverse create]: https://docs.microsoft.com/ko-kr/power-platform/admin/powerapps-powershell?WT.mc_id=power-23654-juyoo#power-apps-cmdlets-for-administrators

[psgallery azuread]: https://www.powershellgallery.com/packages/AzureAD/
[psgallery pa]: https://www.powershellgallery.com/packages/Microsoft.PowerApps.PowerShell/
[psgallery pa admin]: https://www.powershellgallery.com/packages/Microsoft.PowerApps.Administration.PowerShell/
[psgallery az]: https://www.powershellgallery.com/packages/Az/

[m365]: https://www.microsoft.com/ko-kr/microsoft-365?WT.mc_id=power-23654-juyoo
[m365 trial e5]: http://aka.ms/Office365E5Trial

[m365 powershell connect]: https://docs.microsoft.com/ko-kr/microsoft-365/enterprise/connect-to-microsoft-365-powershell?WT.mc_id=power-23654-juyoo
[m365 powershell account create]: https://docs.microsoft.com/ko-kr/microsoft-365/enterprise/create-user-accounts-with-microsoft-365-powershell?WT.mc_id=power-23654-juyoo
[m365 powershell role assign]: https://docs.microsoft.com/ko-kr/microsoft-365/enterprise/assign-roles-to-user-accounts-with-microsoft-365-powershell?WT.mc_id=power-23654-juyoo
[m365 powershell license assign]: https://docs.microsoft.com/ko-kr/microsoft-365/enterprise/assign-licenses-to-user-accounts-with-microsoft-365-powershell?WT.mc_id=power-23654-juyoo

[az powershell role assign]: https://docs.microsoft.com/ko-kr/azure/role-based-access-control/role-assignments-powershell?WT.mc_id=power-23654-juyoo

[az logapp]: https://docs.microsoft.com/ko-kr/azure/logic-apps/logic-apps-overview?WT.mc_id=power-23654-juyoo
[az st]: https://docs.microsoft.com/ko-kr/azure/storage/?WT.mc_id=power-23654-juyoo
[az netwrk]: https://docs.microsoft.com/ko-kr/azure/virtual-network/virtual-networks-overview?WT.mc_id=power-23654-juyoo
[az apim]: https://docs.microsoft.com/ko-kr/azure/api-management/api-management-key-concepts?WT.mc_id=power-23654-juyoo
[az cosdba]: https://docs.microsoft.com/ko-kr/azure/cosmos-db/introduction?WT.mc_id=power-23654-juyoo
