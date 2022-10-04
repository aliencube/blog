---
title: "초콜라티로 애저 VM 위에 라이브 스트리밍을 위한 애플리케이션 자동 설치하기"
slug: app-provisioning-on-azure-vm-with-chocolatey-for-live-streaming
description: "이 포스트에서는 라이브 스트리밍을 위한 애플리케이션들을 초콜라티라는 앱 설치 도구를 이용해 애저 VM에 자동으로 설치하는 방법에 대해 알아봅니다."
date: "2020-08-26"
author: Justin-Yoo
tags:
- azure-vm
- chocolatey
- provisioning
- configuration
cover: /2020/08/app-provisioning-on-azure-vm-with-chocolatey-for-live-streaming-00.png
fullscreen: true
---

이 블로그 포스트 시리즈를 통해 [애저 VM][az vm] 인스턴스를 프로비저닝 할 때 자동으로 필요한 애플리케이션을 설치하는 방법과 더불어 이런 애저 리소스 프로비저닝을 [파워 플랫폼][pw platform]이 훌륭하게 수행할 수 있는 방법에 대해 알아 보고자 한다.

* ***초콜라티로 애저 VM 위에 라이브 스트리밍을 위한 애플리케이션 자동 설치하기***
* [파워 플랫폼으로 일회성 애저 리소스 프로비저닝하기][post next]

오프라인 밋업이나 컨퍼런스가 모두 사라진 요즘에는 이를 대신하기 위한 온라인 밋업과 컨퍼런스가 하루가 멀다하고 진행이 된다. 일반적으로 커뮤니티에서 주관하는 이벤트의 경우에는 유료로 방송 플랫폼을 구입하거나 자체적으로 솔루션을 만들어서 쓰곤 한다. 자체적으로 솔루션을 만들어서 쓴다고 하면 혼자 북치고 장구치고 다 하는 개인 방송의 경우에는 자기 컴퓨터에 모든 것을 설치해 놓고 쓰면 된다고 하지만, 만약 여러 사람이 동시다발적으로 접속해서 실시간으로 패널 토의를 한다든가 한다면 컴퓨터 사양이 어지간히 좋지 않는 이상 이를 감당하기 어려울 수 있다.

이럴 때 사용하면 좋은 대안이 몇 가지 있는데, 그 중 하나가 바로 클라우드에 올라간 가상머신(VM)을 이용하는 것이다. 가상머신은 언제든 필요할 때 마다 프로비저닝해서 인스턴스를 만들어 사용할 수 있고, 그렇게 사용한 인스턴스는 필요 없을 경우 곧바로 삭제하면 된다. 그런데, 이 방법 역시도 "라이브 스트리밍"이라는 관점에서 보면 단점이 한가지 있는데, VM 인스턴스를 생성할 때 마다 직접 라이브 스트리밍에 필요한 애플리케이션을 설치해야 한다. 어쩌다 한 번씩 VM 인스턴스를 만드는 거라면 크게 상관은 없겠지만, 이것도 꽤 손이 많이 가는 일이긴 하다. 따라서, 이 포스트를 통해 애저에 [윈도우용 VM][az vm]을 설치하고 동시에 [초콜라티 Chocolatey][chocolatey]라는 애플리케이션 설치 도구를 이용해서 라이브 스트리밍에 필요한 애플리케이션을 자동으로 설치하는 방법에 대해 알아보기로 한다.

> 이 포스트에 사용된 샘플 코드는 이곳 [깃헙 리포지토리][gh sample]에서 확인할 수 있다.


## 감사의 말 ##

이 포스트는 직장 동료인 [Henk Boelman][henk tw]의 [블로그 포스트][henk blog]와 [Frank Boucher][frank tw]의 [블로그 포스트][frank blog]를 참조로 해서 작성한 것이다. 땡큐베리감사!


## 라이브 스트리밍용 애플리케이션 설치 ##

라이브 스트리밍을 위해서는 윈도우즈 기준 대략 최소 아래와 같은 애플리케이션이 필요하다.

* [Microsoft Edge (크로미움 기반)][ms edge]: 이 글을 쓰는 현 시점에서 윈도우즈 10에 기본 설치되어 있는 Edge는 예전 버전이므로 새롭게 크로미움 버전으로 바뀐 애플리케이션으로 업데이트한다.
* [OBS Studio][obs]: 라이브 스트리밍을 위한 오픈 소스 애플리케이션이다.
* [OBS Studio NDI Plug-in][obs ndi]: OBS 자체적으로는 NDI 기능이 없다. 따라서, NDI 기능을 활용하기 위해서는 이 플러그인을 설치해야 한다.
* [Skype for Content Creators][skype]: 이 버전의 Skype는 [NDI][ndi] 기능을 활성화 시킬 수 있다. 이 기능이 활성화되면 화상통화 참여자 개개인의 화면과 공유 화면을 모두 따로 캡처할 수 있어 다양한 형태로 응용이 가능하다.

이 정도가 최소한으로 필요한 애플리케이션일텐데, 이를 초콜라티를 통해 설치하는 파워셸 스크립트를 아래와 같이 만들어 보자. 첫번째 명령어는 초콜라티 설치 스크립트를 다운로드 받아 실행시키는 것이다 (line #2). 그리고 그 다음에는 앞서 언급한 애플리케이션을 초콜라티를 통해 설치하는 명령어이다 (line #5-8). 리눅스의 apt, yum 또는 맥의 brew 같은 CLI 기반 애플리케이션 설치 도구를 사용해 본 경험이 있다면 이 개념을 쉽게 이해할 수 있을 것이다.

https://gist.github.com/justinyoo/acdfc3c854f21f4e10f56e3d9d75e4c7?file=01-install.ps1&highlights=2,5-8

이 스크립트를 애저 VM 인스턴스를 프로비저닝할 때 자동으로 실행만 시킬 수 있다면, 언제든 원하는 애플리케이션이 미리 설치된 새 VM을 언제 어디서든 사용할 수 있다.


## 애저 윈도우즈 VM 프로비저닝 ##

이제 애저에 윈도우용 VM 인스턴스를 생성해 보자. 이것도 애저 포탈에서 수동으로 설치하는 대신, [ARM 템플릿][az arm]을 이용해서 한번에 해결할 수 있다. ARM 템플릿 작성은 이 [퀵 스타트 템플릿][az quickstart]을 이용하면 쉽게 만들 수 있다. 이 템플릿을 바탕으로 라이브 스트리밍에 필요한 VM 사양으로 내용을 바꾸면 된다. 여기서는 [Deploy a simple Windows VM][az quickstart vm]을 기준으로 목적에 맞게 수정한 결과는 대략 아래와 같다. 편의상 불필요한 부분은 생략했다. 다만, VM 사양에 대해서는 설명할 필요가 있을 듯 해서 [VM 사이즈][az vm size] (line #43), [VM 이미지][az vm image] (line #48-51) 정보는 남겨뒀다.

https://gist.github.com/justinyoo/acdfc3c854f21f4e10f56e3d9d75e4c7?file=02-azure-vm.json&highlights=43,48-51

위에서는 편의상 자세한 정보는 생략했지만, 만약 좀 더 자세한 내용을 확인하고 싶다면 아래 링크를 클릭해 보도록 하자.

[ARM 템플릿 보기][gh sample arm]


### 커스텀 스트립트 확장 ###

애저 윈도우 VM을 프로비저닝할 때 커스텀 파워셸 스크립트를 실행시켜 추가적인 환경 설정을 하고자 한다면, 이 [확장 기능][az vm custom script]을 설치하면 된다. 이 페이지를 참조해서 ARM 템플릿에 선언한 내용은 아래와 같다.

다른 무엇 보다도 `property` 속성 안의 값들이 중요하다. 나머지 값들이야 그냥 정해진 것을 쓰면 된다지만, 커스텀 스크립트 파일의 위치를 나타내는 `fileUris` 값은 별도로 지정해 주어야 한다 (line #16). 이 때 이 URL은 이 ARM 템플릿에서 공개적으로 접근 가능해야 한다. 따라서 아래 스크립트와 같이 GitHub의 URL을 쓰거나 [애저 Blob 스토리지][az storage blob]의 URL을 사용하면 된다.

그리고, 이를 실행시킬 명령어 항목이 필요한데, 이는 `commandToExecute` 속성에 지정한다 (line #19). 이 때 외부에서 다운로드 받은 파워셸 스크립트를 실행시키는 것이므로 이 때만 권한을 살짝 풀어준다 (`-ExecutionPolicy Unrestricted`). 또한 맨 뒤의 `./install.ps1` 부분은 앞서 지정한 외부 URL에서 가져온 파일 이름이다.

https://gist.github.com/justinyoo/acdfc3c854f21f4e10f56e3d9d75e4c7?file=03-arm-vm-extension.json&highlights=16,19


### ARM 템플릿 실행 ###

위와 같이 ARM 템플릿을 작성한 후 실제로 이를 실행시킨다. 아래는 파워셸 명령어를 사용할 경우이다.

https://gist.github.com/justinyoo/acdfc3c854f21f4e10f56e3d9d75e4c7?file=04-arm-deploy.ps1

그리고 아래는 애저 CLI를 사용할 경우이다.

https://gist.github.com/justinyoo/acdfc3c854f21f4e10f56e3d9d75e4c7?file=05-arm-deploy.sh

위 명령어를 실행시키는 것이 번거롭다면, 아래 버튼을 눌러 곧바로 애저 포탈을 통해 프로비저닝을 할 수 있다.

[![Deploy to Azure](http://azuredeploy.net/deploybutton.png)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fdevkimchi%2FLiveStream-VM-Setup-Sample%2Fmain%2Fazuredeploy.json)

이렇게 해서 VM 인스턴스 프로비저닝이 끝난 후 [RDP][az vm rdp] 또는 [바스티온][az vm bastion]을 이용해 직접 접속해 보면 아래와 같이 지정된 애플리케이션이 자동으로 설치가 된 것을 확인할 수 있다.

![][image-01]

---

지금까지 [애저 VM][az vm]에 [초콜라티][chocolatey]를 이용해서 프로비저닝과 동시에 자동으로 라이브 스트리밍에 필요한 애플리케이션을 설치하는 방법에 대해 알아 보았다. 여러 가지 이유로 가상머신은 수시로 만들었다 지웠다 할텐데, 이 때 이 포스트에 언급한 ARM 템플릿을 이용하면 굉장히 손쉽게 한번에 VM 뿐만 아니라 애플리케이션 설치까지 할 수 있다. 앞으로 라이브 스트리밍을 고려하는 사람들에게 유용한 팁이 되길 희망한다.

[다음 포스트][post next]에서는 이런 애저 리소스 프로비저닝을 [파워 플랫폼][pw platform]을 통해 단순화시켜 볼 수 있는 방법에 대해 알아보기로한다.


[image-01]: /2020/08/app-provisioning-on-azure-vm-with-chocolatey-for-live-streaming-01.png

[post next]: /ko/2020/09/02/ad-hoc-azure-resource-provisioning-via-power-platform/

[gh sample]: https://github.com/devkimchi/LiveStream-VM-Setup-Sample
[gh sample arm]: https://github.com/devkimchi/LiveStream-VM-Setup-Sample/blob/main/azuredeploy.json#L347-L389

[chocolatey]: https://chocolatey.org/

[henk tw]: https://twitter.com/hboelman
[henk blog]: https://www.henkboelman.com/articles/online-meetups-with-obs-and-skype/
[frank tw]: https://twitter.com/fboucheros
[frank blog]: http://www.frankysnotes.com/2018/04/dont-install-your-software-yourself.html

[ms edge]: https://www.microsoft.com/ko-kr/edge?WT.mc_id=aliencubeorg-blog-juyoo
[skype]: https://www.skype.com/ko/content-creators/
[obs]: https://obsproject.com/
[obs ndi]: https://obsproject.com/forum/threads/obs-ndi-newtek-ndi%E2%84%A2-integration-into-obs-studio.69240/
[ndi]: https://www.ndi.tv/

[az arm]: https://docs.microsoft.com/ko-kr/azure/azure-resource-manager/templates/overview?WT.mc_id=aliencubeorg-blog-juyoo
[az quickstart]: https://azure.microsoft.com/ko-kr/resources/templates/?term=Deploy+a+simple+Windows+VM&WT.mc_id=aliencubeorg-blog-juyoo
[az quickstart vm]: https://azure.microsoft.com/ko-kr/resources/templates/101-vm-simple-windows/?WT.mc_id=aliencubeorg-blog-juyoo

[az vm]: https://docs.microsoft.com/ko-kr/azure/virtual-machines/windows/overview?WT.mc_id=aliencubeorg-blog-juyoo
[az vm size]: https://docs.microsoft.com/ko-kr/azure/virtual-machines/sizes?WT.mc_id=aliencubeorg-blog-juyoo
[az vm image]: https://docs.microsoft.com/ko-kr/azure/virtual-machines/windows/cli-ps-findimage?WT.mc_id=aliencubeorg-blog-juyoo
[az vm custom script]: https://docs.microsoft.com/ko-kr/azure/virtual-machines/extensions/custom-script-windows?WT.mc_id=aliencubeorg-blog-juyoo
[az vm rdp]: https://docs.microsoft.com/ko-kr/azure/virtual-machines/windows/connect-logon?WT.mc_id=aliencubeorg-blog-juyoo
[az vm bastion]: https://docs.microsoft.com/ko-kr/azure/bastion/bastion-connect-vm-rdp?WT.mc_id=aliencubeorg-blog-juyoo

[az storage blob]: https://docs.microsoft.com/ko-kr/azure/storage/blobs/storage-blobs-overview?WT.mc_id=aliencubeorg-blog-juyoo

[pw platform]: https://powerplatform.microsoft.com/ko-kr/?WT.mc_id=aliencubeorg-blog-juyoo
