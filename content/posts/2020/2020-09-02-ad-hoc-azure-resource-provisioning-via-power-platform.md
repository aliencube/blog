---
title: "파워 플랫폼으로 일회성 애저 리소스 프로비저닝하기"
slug: ad-hoc-azure-resource-provisioning-via-power-platform
description: "이 포스트에서는 현업에서 어쩌다 한 번씩 필요할 때 애저 리소스를 생성하곤 하는 경우에 파워 플랫폼을 이용해서 프로비저닝하는 방법에 대해 알아봅니다."
date: "2020-09-02"
author: Justin-Yoo
tags:
- power-platform
- azure-resource-manager
- provisioning
- workflow
cover: /2020/09/ad-hoc-azure-resource-provisioning-via-power-platform-00.png
fullscreen: true
---

이 블로그 포스트 시리즈를 통해 현업에서 어쩌다 한 번 생성하고 마는 [애저 VM][az vm]과 같이 일회성으로 필요한 작업들에 대해 파워 플랫폼이 훌륭하게 대응할 수 있는 방법에 대해 알아 보고자 한다.

* [초콜라티로 애저 VM 위에 라이브 스트리밍을 위한 애플리케이션 자동 설치하기][post prev]
* ***파워 플랫폼으로 일회성 애저 리소스 프로비저닝하기***

[지난 포스트][post prev]에서는 라이브 스트리밍용 애저 VM을 프로비저닝하면서 동시에 필요한 애플리케이션을 설치하는 방법에 대해 알아보았다. 그런데, 이런 형태의 VM은 수시로 만들고 지우고 한다기 보다는 비정기적으로 어쩌다 한 번씩 만들어서 사용할 뿐이다. 따라서, 한 번 [ARM 템플릿][az arm template]을 만들어 두긴 하지만, 이걸 또 언제 쓸 지는 알 수 없다. 한참 후에 다시 같은 사양으로 VM을 하나 프로비저닝하게 됐는데, 마침 컴퓨터에 접속해서 프로비저닝 명령어를 보낼 수 없는 상황이 올 수 있다. 또는 컴퓨터 대신 모바일 기기에서 손쉽게 프로비저닝 하고 싶을 수도 있다.

이런 일회성 작업을 모바일에서 손쉽게 하기에 [파워 앱][pw apps]만한 것이 없다. 이 포스트에서는 [파워 앱][pw apps]과 [파워 오토메이트][pw automate]를 통해 일회성 애저 리소스 프로비저닝을 하는 방법에 대해 알아보기로 한다.


## 파워 오토메이트 워크플로우 &ndash; 애저 리소스 프로비저닝 ##

우선 아래와 같은 트릭을 하나 소개한다. 파워 앱으로부터 넘어오는 파라미터들의 갯수가 많거나 가변적인 경우에 사용하면 좋은 팁이다. 파워 앱으로부터는 단 하나의 파라미터만을 넘겨 받는데, 이를 아래와 같이 [`Compose` 액션][pw automate compose]을 이용해서 받고 그 때 변수명은 자동으로 `ParametersInJson_Inputs`라고 정해진다.

![][image-01]

그리고난 후 `json(triggerBody()?['ParametersInJson_Inputs'])` 라는 함수식을 `Inputs` 필드에 입력하면 아래와 같다.

![][image-02]

> 파워 앱에서 넘어오는 변수를 이런 방식으로 처리하면 좋은 점 중 하나는 변수 처리를 유연하게 할 수 있다는 점이다. 변수의 갯수가 변한다거나, 이름이 바뀐다거나 하면 항상 파워 오토메이트를 수정하고 업데이트한 후, 이를 다시 파워 앱에 반영하는 작업을 반복해야 하는데, 이 방식으로 하면 변수의 갯수는 항상 하나이고, 그 안에서 속성값이 바뀌는 거라 손이 덜 가게 된다.

이제 이렇게 받아온 파라미터를 가지고 아래와 같이 ARM 템플릿을 실행시키면 된다. 파워 오토메이트에는 [애저 리소스 매니저 관련 액션][pw automate arm deploy]이 이미 제공되기 때문에 곧바로 사용할 수 있다. 아래 그림의 변수들은 앞에서 받아 Compose 액션으로 변환시킨 JSON 개체의 값들을 `outputs('ParametersInJson')?['resourceGroupName']`, `outputs('ParametersInJson')?['deploymentName']`, `outputs('ParametersInJson')?['vmName']`, `outputs('ParametersInJson')?['vmAdminUsername']`, `outputs('ParametersInJson')?['vmAdminPassword']` 등으로 받아서 적용시킨 값들이다.

![][image-03]

다음 액션은 앞서 실행시킨 애저 리소스 매니저 액션의 결과값을 다시 파워 앱으로 돌려주는 [Response 액션][pw automate response]이다. JSON 스키마를 추가해서 파워 앱에서 바로 속성 값들을 참조할 수 있게 처리했다.

![][image-04]

여기까지 하면 굉장히 쉽게 애저 리소스 생성과 관련한 파워 오토메이트 워크플로우를 만들 수 있다.

그런데, 여기서 한 가지 문제가 있다. 리소스 생성에 걸리는 시간은 리소스의 종류와 갯수에 따라 천차만별인데, 짧게는 30초에서 길게는 40분도 걸린다. 이렇게 오래 걸리는 워크플로우를 파워 앱 쪽에서 기다릴 수 없으므로 이 워크플로우는 반드시 비동기식으로 진행할 수 밖에 없다. 따라서, 리소스를 생성하는 워크플로우는 여기서 마무리 짓기로 하고, 다른 워크플로우를 통해 리소스 생성이 완료되었는지 확인하기로 한다.


## 파워 오토메이트 워크플로우 &ndash; 애저 리소스 프로비저닝 상태 확인 ##

이번에는 애저 리소스 프로비저닝 상태를 확인하는 워크플로우를 만들어 보자. 여기서는 단순히 현재 진행중인 리소스 프로비저닝이 진행중인지 혹은 끝났는지만 확인하는 부분만 있다. 앞서와 같이 파라미터를 정리하는 [`Compose` 액션][pw automate compose]을 이용한다.

![][image-02]

이번에는 [현재 프로비저닝 상태를 체크하는 액션][pw automate arm status]을 사용한다. 변수는 `outputs('ParametersInJson')?['resourceGroupName']`, `outputs('ParametersInJson')?['deploymentName']`을 사용했다.

![][image-05]

그리고난 후 그 결과값을 역시 [Response 액션][pw automate response]에 담아 파워 앱으로 보낸다.

![][image-04]

이렇게 두 애저 리소스 프로비저닝 관련 워크플로우를 작성했다. 이제 파워 앱을 만들어 보자.


## 파워 앱 &ndash; 일회성 애저 리소스 프로비저닝 ##

실제로 현업 담당자가 사용할 파워 앱의 화면은 대략 이렇게 생겼을 것이다. 앞서 작성한 파워 오토메이트 워크플로우에서 활용하는 다섯 개의 파라미터를 입력 받는다. 그리고 마지막의 버튼 두 개를 이용해서 리소스 프로비저닝을 하고 리소스 프로비저닝 상태를 체크한다.

![][image-06]

아래와 같이 `Provision!` 버튼에 파워 오토메이트 워크플로우를 연결한다. 워크플로우로 보내는 파라미터는 [`Set()` 펑션][pw apps set]으로 만들었다.

![][image-07]

아래와 같이 `Set()` 펑션으로 `request`라는 임시 변수에 JSON 개체를 만들어 저장하고, 이 `request` 변수를 파워 오토메이트 워크플로우에 실어 보냈다. 그리고 그 결과값은 [`ClearCollect()` 펑션][pw apps clearcollect]을 통해 `result` 라는 콜렉션에 저장시켰다.

https://gist.github.com/justinyoo/90c6e7a93fc0f957aa35751a8fee32f4?file=01-create-resource.txt

위에서도 언급했다시피 `Provision!` 버튼을 눌렀다고 해서 곧바로 리소스가 만들어지는 것은 아니라 길게는 40분 이상도 걸리기 때문에 그 옆의 `Status` 버튼을 이용해 수시로 상태를 체크하게끔 했다.

![][image-08]

위와 마찬가지 방법으로 `Set()` 펑션을 이용해서 `status`라는 임시 변수에 JSON 개체를 담아서 이를 파워 오토메이트 워크플로우에 보낸다. 그리고 그 결과값을 다시 `result` 라는 콜렉션에 담는다.

https://gist.github.com/justinyoo/90c6e7a93fc0f957aa35751a8fee32f4?file=02-check-provisioning-status.txt

> 여기서는 간단하게 만들려고 `Status` 버튼을 사용했지만, [`Timer` 콘트롤][pw apps timer]을 이용하면 훨씬 더 쉽게 자동으로 상태를 체크할 수도 있다. 이 포스트에서는 다루지 않는다.

아래 상태 결과를 나타내는 라벨 콘트롤에는 `First(result).properties.provisioningState`와 같이 [`First()` 함수][pw apps first]를 이용해 `result` 콜렉션의 값을 참조한다.

![][image-09]

파워 앱도 모두 완성했다! 이제 실제로 값을 넣고 실행시켜 보자. 맨 마지막 Status 값이 계속해서 바뀌는 것이 보일 것이다.

![][image-10]

![][image-11]

![][image-12]

![][image-13]

---

지금까지 일회성으로 애저 리소스를 프로비저닝하기 위해 간단하게 파워 앱을 만들고 실행시켜 봤다. 여기서는 굉장히 간단하게 한다고 한 가지 사용자 케이스만 다루었지만, 실제 현업에서는 이런 일회성 작업이 꽤 많이 있을 것이다. 그 중에서 빈도가 상대적으로 높으면서 조금은 번거로운 작업들은 이렇게 파워 플랫폼을 이용해서 어느 정도 정형화 및 자동화를 시켜 놓으면 업무 생산성 향상에 꽤 많은 도움이 될 것이다.


[image-01]: /2020/09/ad-hoc-azure-resource-provisioning-via-power-platform-01.png
[image-02]: /2020/09/ad-hoc-azure-resource-provisioning-via-power-platform-02.png
[image-03]: /2020/09/ad-hoc-azure-resource-provisioning-via-power-platform-03.png
[image-04]: /2020/09/ad-hoc-azure-resource-provisioning-via-power-platform-04.png
[image-05]: /2020/09/ad-hoc-azure-resource-provisioning-via-power-platform-05.png
[image-06]: /2020/09/ad-hoc-azure-resource-provisioning-via-power-platform-06.png
[image-07]: /2020/09/ad-hoc-azure-resource-provisioning-via-power-platform-07.png
[image-08]: /2020/09/ad-hoc-azure-resource-provisioning-via-power-platform-08.png
[image-09]: /2020/09/ad-hoc-azure-resource-provisioning-via-power-platform-09.png
[image-10]: /2020/09/ad-hoc-azure-resource-provisioning-via-power-platform-10.png
[image-11]: /2020/09/ad-hoc-azure-resource-provisioning-via-power-platform-11.png
[image-12]: /2020/09/ad-hoc-azure-resource-provisioning-via-power-platform-12.png
[image-13]: /2020/09/ad-hoc-azure-resource-provisioning-via-power-platform-13.png

[post prev]: /ko/2020/08/26/app-provisioning-on-azure-vm-with-chocolatey-for-live-streaming/

[az vm]: https://docs.microsoft.com/ko-kr/azure/virtual-machines/windows/overview?WT.mc_id=aliencubeorg-blog-juyoo

[az arm template]: https://docs.microsoft.com/ko-kr/azure/azure-resource-manager/templates/overview?WT.mc_id=aliencubeorg-blog-juyoo

[pw platform]: https://powerplatform.microsoft.com/ko-kr/?WT.mc_id=aliencubeorg-blog-juyoo

[pw apps]: https://powerapps.microsoft.com/ko-kr/?WT.mc_id=aliencubeorg-blog-juyoo
[pw apps set]: https://docs.microsoft.com/ko-kr/powerapps/maker/canvas-apps/functions/function-set?WT.mc_id=aliencubeorg-blog-juyoo
[pw apps clearcollect]: https://docs.microsoft.com/ko-kr/powerapps/maker/canvas-apps/functions/function-clear-collect-clearcollect?WT.mc_id=aliencubeorg-blog-juyoo#clearcollect
[pw apps first]: https://docs.microsoft.com/ko-kr/powerapps/maker/canvas-apps/functions/function-first-last?WT.mc_id=aliencubeorg-blog-juyoo
[pw apps timer]: https://docs.microsoft.com/ko-kr/powerapps/maker/canvas-apps/controls/control-timer?WT.mc_id=aliencubeorg-blog-juyoo

[pw automate]: https://flow.microsoft.com/ko-kr/?WT.mc_id=aliencubeorg-blog-juyoo
[pw automate compose]: https://docs.microsoft.com/ko-kr/power-automate/data-operations?WT.mc_id=aliencubeorg-blog-juyoo#use-the-compose-action
[pw automate arm deploy]: https://docs.microsoft.com/ko-kr/connectors/arm/?WT.mc_id=aliencubeorg-blog-juyoo#create-or-update-a-template-deployment
[pw automate arm status]: https://docs.microsoft.com/ko-kr/connectors/arm/?WT.mc_id=aliencubeorg-blog-juyoo#read-a-template-deployment
[pw automate response]: https://docs.microsoft.com/ko-kr/azure/connectors/connectors-native-reqres?WT.mc_id=aliencubeorg-blog-juyoo#add-a-response-action
