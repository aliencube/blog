---
title: "서버리스 쿠킹 챌린지 제 5주차 - 새해 맞이 떡국"
slug: seasons-of-serverless-week-5
description: "이 포스트는 Seasons of Serverless 챌린지의 5주차에 해당하는 내용으로, 애저 듀어러블 펑션, 애저 로직 앱, 파워 오토메이트, 파워 앱을 이용해 한국 전통 요리중 하나인 떡국을 만들어 봅니다."
date: "2020-12-23"
author: Justin-Yoo
tags:
- azure-functions
- azure-logic-apps
- power-automate
- power-apps
cover: /2020/12/seasons-of-serverless-week-5-00.png
fullscreen: true
---

> 이 포스트는 [#SeasonsOfServerless][devto sos] 이벤트의 일환으로 5주차에 진행되는 내용을 담았습니다.
>
> 매주 전세계의 요리를 하나씩 소개하는데요, [마이크로소프트 클라우드 아드보캇][ms ca]과 [마이크로소프트 학생 앰배서더][ms lsa]가 함께 협업하여 도전 과제를 만들고, 이를 [애저 서버리스 서비스][az serverless]를 활용해서 구현해 봅니다.

* 1주차: [완벽한 칠면조 요리 - 북미 지역][devto sos week1] (영문)
* 2주차: [화사한 라두 - 인도 지역][devto sos week2] (영문)
* 3주차: [세상에서 가장 긴 케밥 - 유럽 지역 / 터키][devto sos week3] (영문)
* 4주차: [어마어마 바베큐 - 남미 지역 / 브라질][devto sos week4] (영문)
* 5주차: ***새해 맞이 떡국 - 아시아 지역 / 한국***
* 6주차: TBD (영문)
* 7주차: TBD (영문)

한국에서는 새해가 되면 모두가 떡국을 먹습니다. 떡국에 쓰이는 떡은 가래떡이라고 하는데요, 이 모양이 길다란 원통형이어서 떡국을 먹으며 장수를 기원한다고 합니다. 또한, 요리를 할 땐 이 가래떡을 얇게 써는데요, 이 얇게 썰린 모양이 마치 동전같다고 해서 떡국을 먹으며 경제적인 풍요를 기원하기도 합니다.


## 도전 과제 참여자 ##

* 유저스틴: [마이크로소프트 클라우드 아드보캇][author justin]
* 김유진: [마이크로소프트 학생 앰배서더][author youjin]
* 김홍민: [마이크로소프트 학생 앰배서더][author hongmin]
* 노아론: [마이크로소프트 학생 앰배서더][author aaron]


## 도전 내용 설명 ##

우리 MLSA 분들이 실제로 구현한 것을 인터뷰한 내용입니다. 비디오는 한국어로 되어 있지만, 한국어 자막과 영문 자막이 동시에 제공되니 한 번 보세요.

https://youtu.be/tq9Ntzy-ifM


## 떡국 재료들 ##

떡국을 만드는 방법은 꽤 간단한데요, 아래는 떡국 4인분을 만들기 위해 필요한 재료들입니다.

* 가래떡: 400g
* 쇠고기 볶음용: 100g
* 물: 10 컵
* 계란: 2 알
* 파: 1 단
* 다진 마늘: 1 숟갈
* 간장: 2 숟갈
* 참기름: 1 숟갈
* 식용유: 1 숟갈
* 소금/후추: 약간


## 떡국 레시피 ##

가장 간단한 떡국 만드는 방법은 아래와 같습니다.

1. 가래떡을 얇게 썰어 둡니다.
   * 가래떡 썰기 귀찮다면 이미 썰어놓은 것을 사도 됩니다.
   * 다만, 이 경우에는 요리 전에 약 30분 정도 충분히 물에 불려두는 것이 좋습니다.
2. 파를 얇게 썰어 둡니다.
3. 고온에서 쇠고기를 참기름과 식용유를 넣고 쇠고기 표면이 갈색이 될 때 까지 골고루 볶습니다.
4. 쇠고기를 볶아놓은 냄비에 물을 넣고 중불에서 약 30분 정도 끓입니다.
5. 끓이는 동안 거품이 발생하는데요, 거품이 넘치지 않게끔 제거해 줍니다.
6. 그동안 계란을 풀어 준비합니다.
7. 30분 정도 끓인 후에 다진 마늘과 간장을 넣고 다시 더 끓입니다. 이 때 간이 안 맞으면 소금을 좀 더 넣습니다.
8. 풀어둔 계란과 파를 넣고 마지막으로 조금만 더 끓입니다.
9. 취향에 따라 후추를 뿌려서 음식을 냅니다.

위의 레시피를 순서도로 구현하면 아래와 같습니다.

![Flow chart][image-01]


## 떡국 레시피 단계 구현 ##

각 단계별로 준비가 됐는지 확인하고 아직 준비가 되지 않았다면 준비가 될 때까지 좀 더 기다려야 하므로 기다리는 시간을 타이머로 구성했습니다. 이 때 이 시간이 일정하지 않기 때문에 [애저 듀어러블 펑션][az func durable]의 [타이머 기능][az func durable timer]을 이용하면 이런 상황에 유연하게 대응할 수 있습니다.

[학생 앰배서더][ms lsa]는 각자 세 개의 단계를 선택해서 선호하는 프로그래밍 언어로 애저 펑션에 구현했습니다. 결과적으로 [김유진][author youjin]님은 자바스크립트를 이용해 [4단계][gh sos step4], [6단계][gh sos step6], [8단계][gh sos step8]를, [김홍민][author hongmin]님은 C#을 이용해 [2단계][gh sos step2], [5단계][gh sos step5], [7단계][gh sos step7]를, [노아론][author aaron]님은 파이썬을 이용해 [1단계][gh sos step1], [3단계][gh sos step3], [9단계][gh sos step9] 펑션을 작성했습니다.


## 떡국 레시피 오케스트레이션 ##

각 단계는 모두 [애저 듀어러블 펑션][az func durable]으로 구현을 했지만, 이를 전체적으로 오케스트레이션 하기 위해 [애저 로직 앱][az logapp]을 사용했습니다. 앞서 언급한 순서도에 따르면 HTTP 요청으로 받아온 데이터를 [1단계 (떡 썰기)][gh sos step1], [2단계 (파 썰기)][gh sos step2], [3단계 (쇠고기 볶기)][gh sos step3], [6단계 (계란 풀기)][gh sos step6]으로 동시에 보내서 실행을 시킵니다. 그리고 그 결과를 각 단계로부터 받은 후에 다음 단계로 이동합니다. 이런 패턴을 [Fan-Out/Fan-In 또는 Scatter-Gather 패턴][fanout fanin]이라고 합니다.

![Logic App: Fan-out/Fan-in][image-02]

이후 [떡국을 끓이는 동안 (4단계)][gh sos step4], 거품이 랜덤하게 생기면 [거품을 제거해 줍니다 (5단계)][gh sos step5].

![Logic App: Boiling & De-bubbling][image-03]

그 뒤로 [간을 맞추고 (7단계)][gh sos step7], [계란과 파를 넣고 (8단계)][gh sos step8], [후추를 뿌려 완성된 떡국을 냅니다 (9단계)][gh sos step9]. 9단계에서는 완성된 떡국 이미지를 [애저 블롭 저장소][az st blob]에서 무작위로 가져오고, 이를 그 다음 액션을 통애 지정한 이메일 주소로 보내주게 됩니다.

![Logic App: Rest of Steps][image-04]

이 때 단계별로 각각 최소 2분 이상씩 걸리므로 [로직 앱의 2분 제한][az logapp limit]을 극복하기 위해 로직앱에서 [HTTP 액션][az logapp http] 대신, [HTTP 웹훅 액션][az logapp webhook]으로 구현했습니다. 이를 실제로 실행시켜 본 결과는 아래와 같습니다.

![Logic App: Run Result][image-05]

그리고 로직 앱 실행 후에는 이메일로 아래와 같은 이미지를 받게 됩니다.

![Email: Run Result][image-06]


## 떡국 레시피 자동 배포 ##

앞서 구현했던 [애저 듀어러블 펑션][az func durable] 앱과 [로직 앱 오케스트레이션][az logapp]은 모두 자동으로 빌드하고 배포할 수 있게끔 [깃헙 액션][gh actions]을 구현했습니다. 자세한 내용은 아래 링크를 확인해 보세요. 만약 [Bicep][az bicep]에 대해 좀 더 알고 싶다면 [이 링크][post bicep]를 확인해 보시기 바랍니다.

* [애저 리소스 템플릿 bicep 파일][gh bicep]
* [깃헙 액션 워크플로우][gh workflow]


## 파워 오토메이트 구현 ##

이렇게 만들어진 로직 앱을 [파워 플랫폼][pw platform]에서 실행시키기 위해 필요한 것이 바로 [파워 오토메이트][pw automate] 워크플로우입니다. 로직 앱을 그대로 옮겨와도 상관 없지만, 그 작업이 만만치 않기 때문에 파워 오토메이트 워크플로우를 별도로 만들고 거기서 로직 앱을 웹훅으로 호출하는 형태로 했습니다. [파워 오토메이트][pw automate]는 [파워 앱][pw apps]이 호출을 하고, 처리 결과를 다시 파워 앱으로 [푸시 알림][pw apps push]을 보내게끔 했습니다.

![Power Automate: Workflow][image-07]

이렇게 하기 위해서는 앞서 작성한 로직 앱을 살짝 수정해 줘야 합니다. 맨 마지막에 이메일을 보내는 것과 동시에 콜백 URL로 떡국 이미지 URL을 함께 보내주게끔 로직앱을 아래와 같이 변경했습니다.

![Logic App: Workflow Update][image-08]


## 파워 앱 구현 ##

[파워 오토메이트][pw automate]를 모바일 앱에서 호출하기 위해 여기서는 [파워 앱][pw apps]을 선택했습니다. 아래는 파워 앱 캔버스에 올라간 콘트롤들인데요, 아주 간단하게 구현할 수 있습니다.

![Power Apps: Canvas][image-09]

여기까지 구현한 후, [파워 앱][pw apps]을 발행하고 모바일 앱에서 실행시켜 보면 아래와 같습니다.

https://youtu.be/AJlCv2pAc8g

---

이렇게 해서 떡국을 요리하기 위한 레시피를 모두 서버리스로 구현해 보았습니다. 여기에 사용한 샘플 코드는 아래에서 확인하실 수 있습니다.

[솔루션 리포지토리][gh sample]

마찬가지로 샘플 [파워 앱][pw apps]과 [파워 오토메이트][pw automate] 워크플로우도 들어 있으니 여러분의 [파워 플랫폼][pw platform]에 업로드한 후 몇가지 API 주소만 바꿔서 실행시켜 보세요! 다른 레시피 챌린지도 하고 싶으신가요? [#SeasonsOfServerless][devto sos]를 확인해 보세요!


[image-01]: https://raw.githubusercontent.com/justinyoo/Seasons-of-Serverless/solution/solutions/2020-12-21/flowchart.png
[image-02]: /2020/12/seasons-of-serverless-week-5-02.png
[image-03]: /2020/12/seasons-of-serverless-week-5-03.png
[image-04]: /2020/12/seasons-of-serverless-week-5-04.png
[image-05]: /2020/12/seasons-of-serverless-week-5-05.png
[image-06]: /2020/12/seasons-of-serverless-week-5-06.jpg
[image-07]: /2020/12/seasons-of-serverless-week-5-07.png
[image-08]: /2020/12/seasons-of-serverless-week-5-08.png
[image-09]: /2020/12/seasons-of-serverless-week-5-09.png

[devto sos]: https://dev.to/azure/azure-advocates-seasons-of-serverless-join-our-virtual-festive-potluck-53m6
[devto sos week1]: https://dev.to/azure/seasonsofserverless-solution-1-developing-the-perfect-holiday-turkey-2p3f
[devto sos week2]: https://dev.to/azure/seasonsofserverless-solution-2-developing-lovely-ladoos-41h9
[devto sos week3]: https://dev.to/azure/seasonsofserverless-solution-3-the-longest-kebab-3pe1
[devto sos week4]: https://dev.to/azure/seasonsofserverless-solution-4-a-big-barbecue-4d5e
[devto sos week6]: https://dev.to/azure/week-6
[devto sos week7]: https://dev.to/azure/week-7

[post bicep]: /tag/bicep/

[author justin]: https://twitter.com/justinchronicle
[author youjin]: https://github.com/u0jin
[author hongmin]: https://github.com/hongman
[author aaron]: https://www.linkedin.com/in/aaronroh/

[gh sample]: https://github.com/justinyoo/Seasons-of-Serverless
[gh actions]: https://docs.github.com/en/free-pro-team@latest/actions
[gh bicep]: https://github.com/justinyoo/Seasons-of-Serverless/blob/solution/solutions/2020-12-21/Resources/azuredeploy.bicep
[gh workflow]: https://github.com/justinyoo/Seasons-of-Serverless/blob/solution/.github/workflows/main.yaml

[gh sos step1]: https://github.com/justinyoo/Seasons-of-Serverless/tree/solution/solutions/2020-12-21/Step-1
[gh sos step2]: https://github.com/justinyoo/Seasons-of-Serverless/tree/solution/solutions/2020-12-21/Step-2
[gh sos step3]: https://github.com/justinyoo/Seasons-of-Serverless/tree/solution/solutions/2020-12-21/Step-3
[gh sos step4]: https://github.com/justinyoo/Seasons-of-Serverless/tree/solution/solutions/2020-12-21/Step-4
[gh sos step5]: https://github.com/justinyoo/Seasons-of-Serverless/tree/solution/solutions/2020-12-21/Step-5
[gh sos step6]: https://github.com/justinyoo/Seasons-of-Serverless/tree/solution/solutions/2020-12-21/Step-6
[gh sos step7]: https://github.com/justinyoo/Seasons-of-Serverless/tree/solution/solutions/2020-12-21/Step-7
[gh sos step8]: https://github.com/justinyoo/Seasons-of-Serverless/tree/solution/solutions/2020-12-21/Step-8
[gh sos step9]: https://github.com/justinyoo/Seasons-of-Serverless/tree/solution/solutions/2020-12-21/Step-9

[ms ca]: https://developer.microsoft.com/ko-kr/advocates/?WT.mc_id=academic-10291-cxa
[ms lsa]: https://studentambassadors.microsoft.com/ko-KR/?WT.mc_id=academic-10291-cxa

[az bicep]: https://github.com/azure/bicep

[az serverless]: https://azure.microsoft.com/ko-kr/solutions/serverless/?WT.mc_id=academic-10291-cxa
[az st blob]: https://docs.microsoft.com/ko-kr/azure/storage/blobs/storage-blobs-introduction?WT.mc_id=academic-10291-cxa

[az func]: https://docs.microsoft.com/ko-kr/azure/azure-functions/functions-overview?WT.mc_id=academic-10291-cxa
[az func durable]: https://docs.microsoft.com/ko-kr/azure/azure-functions/durable/durable-functions-overview?WT.mc_id=academic-10291-cxa
[az func durable timer]: https://docs.microsoft.com/ko-kr/azure/azure-functions/durable/durable-functions-timers?WT.mc_id=academic-10291-cxa

[az logapp]: https://docs.microsoft.com/ko-kr/azure/logic-apps/logic-apps-overview?WT.mc_id=academic-10291-cxa
[az logapp limit]: https://docs.microsoft.com/ko-kr/azure/logic-apps/logic-apps-limits-and-config?WT.mc_id=academic-10291-cxa#http-limits
[az logapp http]: https://docs.microsoft.com/ko-kr/azure/logic-apps/logic-apps-workflow-actions-triggers?WT.mc_id=academic-10291-cxa#http-action
[az logapp webhook]: https://docs.microsoft.com/ko-kr/azure/logic-apps/logic-apps-workflow-actions-triggers?WT.mc_id=academic-10291-cxa#webhooks-and-subscriptions

[pw platform]: https://powerplatform.microsoft.com/ko-kr/?WT.mc_id=academic-10291-cxa
[pw automate]: https://flow.microsoft.com/ko-kr/?WT.mc_id=academic-10291-cxa
[pw apps]: https://powerapps.microsoft.com/ko-kr/?WT.mc_id=academic-10291-cxa
[pw apps push]: https://docs.microsoft.com/ko-kr/connectors/powerappsnotification/?WT.mc_id=academic-10291-cxa

[fanout fanin]: https://www.enterpriseintegrationpatterns.com/patterns/messaging/BroadcastAggregate.html
