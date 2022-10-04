---
title: "GitOps, 애저 Durable Functions, GitHub Actions을 이용한 블로그 예약 포스팅 구현"
slug: scheduling-posts-with-gitops-durable-functions-and-github-actions
description: "이 포스트에서는 정적 웹사이트를 이용한 블로그를 사용할 때, 애저 Durable Functions과 GitHub Actions를 이용해서 예약 포스팅을 하는 방법에 대해 알아봅니다."
date: "2020-03-25"
author: Justin-Yoo
tags:
- azure-durable-functions
- github-actions
- gitops
- event-scheduling
cover: /2020/03/scheduling-posts-with-gitops-durable-functions-and-github-actions-00.png
fullscreen: true
---

[Gridsome][gridsome]과 같은 정적 웹사이트 생성기를 이용해 블로그를 쓰다보면 가장 불편한 점이 예약 포스팅이다. [워드프레스][wordpress]와 같은 전문 블로그 플랫폼을 쓴다면 예약 포스팅 기능은 기본으로 갖춰져 있기 때문에 큰 문제가 없다지만, 정적 웹사이트 생성기에는 그런 기능이 없다. 따라서, 예약 포스팅 기능 같은 경우는 깔끔하게 포기하고 실제 포스트하고자 하는 날에 맞춰 포스트를 올리거나 해야 한다. 그렇다면 예약 포스팅 기능을 한 번 만들어서 사용해 보면 어떨까? 이 포스트에서 [애저 Durable Functions][az func durable] 기능과 [GitHub Actions][gh actions] 기능을 활용해서 예약 포스팅을 하는 방법에 대해 논의해 보자.

> 1. 이 포스트에서는 [애저 Durable Functions][az func durable]을 사용했는데, 이는 직장 동료인 [Todd][todd]가 개발한 [PublishTo.Dev][todd publishtodev]에서 아이디어를 가져왔다.
> 2. 이 포스트에서 사용한 예제 코드는 [https://github.com/devkimchi/GitHub-Repository-Event-Scheduler][gh sample]에서 다운로드 받을 수 있다.


## Durable Functions 이란? ##

[서버리스][post serverless] 아키텍처의 특징 중 하나는 Stateless라는 점이다. 이 부분은 일견 맞는 얘기기도 하고 아니기도 한데, 서버리스 형태의 API 애플리케이션을 개발한다면 Stateless 라는 부분은 옳다. 하지만, 일반적으로 서버리스 애플리케이션의 특징 중 하나인 이벤트 기반 아키텍처라는 개념으로 넓혀본다면 대부분의 이벤트는 Stateful하다. 예를 들어 매 시간마다 작동하는 타이머 펑션이 하나 있다고 가정해 보자. 이 때 "매 시간"은 어디서 알 수 있는 걸까? 분명히 어딘가에 정보가 있어서 그 값을 읽어들여야 할 것이다. 바로 이 "어딘가에 있는 정보"는 State(상태)를 의미하는 것이고, 이 State를 참조하는 애플리케이션을 가리켜 우리는 Stateful하다고 얘기한다.

애저에서 대표적인 Staeful 서버리스 애플리케이션에는 [로직 앱][az logapp]이 있다. [로직 앱][az logapp]의 작동 원리를 간단하게 짚어보자면 전체 워크플로우를 관리하기 위해서 각각의 액션마다 State를 저장하고 이를 바탕으로 다음 액션을 실행하는 방식이다. 그렇다면, [애저 펑션][az func]에서도 [로직 앱][az logapp]과 같은 워크플로우를 관리할 수 있을까? 그렇게 워크플로우를 관리하기 위해서는 State를 저장해야 하는데, [애저 펑션][az func]에서는 이를 어떤 식으로 구현할까? [Durable Functions][az func durable]이 바로 이 Stateful한 [애저 펑션][az func]의 구현체이다. 그렇다면 이 포스트에서 의도한 "[Durable Functions][az func durable]을 이용해서 예약 포스팅을 하기" 위해서는 어떤 형태의 Stateful한 워크플로우가 필요할까?


## 워크플로우 설계 ##

[Durable Functions][az func durable]에서 구현하는 대략의 워크플로우는 아래와 같다.

![][image-01]

가장 먼저 스케줄을 펑션 엔드포인트로 보낸다. 이 펑션은 특별한 기능이 있다기 보다는 받은 페이로드를 [Durable Functions][az func durable]로 보내기 위한 게이트웨이 역할을 한다. 실제 워크플로우 [오케스트레이션][az func durable orchestrations]은 두번째 펑션에서 관장한다. 오케스트레이션 펑션은 먼저 스케줄을 체크하고 타이머를 호출하면서 [애저 큐 저장소][az storage queue]로 메시지를 하나 보낸다. 동시에 오케스트레이션 State를 [애저 테이블 저장소][az storage table]에 저장한다. 그러면 타이머를 통해 [큐 저장소][az storage queue]에 저장시킨 메시지가 스케줄에 맞춰 실행이 되면 [테이블 저장소][az storage table]에 저장된 상태값을 바탕으로 깃헙으로 이벤트를 보내는 마지막 펑션을 실행시킨다. 첫번째 펑션은 이벤트를 받아주는 곳에 불과한지라 크게 복잡한 부분은 없고, 두번째 펑션에서 워크플로우 로직을, 마지막 펑션에서 비지니스 로직을 구현하면 된다.


## 워크플로우 구현 ##

### 엔드포인트 펑션 ###

이 펑션은 실제 외부로 열린 엔드포인트의 역할을 함으로써 이벤트 페이로드를 받아서 [오케스트레이션 펑션][az func durable orchestrations]으로 보내는 역할을 한다. 아래는 실제 페이로드의 모습이다.

https://gist.github.com/justinyoo/0516447045d0ef3c606d7e84f0ecd872?file=01-payload-ac.json

이제 아래 코드를 보자. 가장 먼저 페이로드를 받아온다 (line #8). 그리고 [오케스트레이션 펑션][az func durable orchestrations]을 실행시키면서 페이로드를 보낸다 (line #9). 마지막으로 현재 실행 상태를 확인할 수 있는 메타데이터를 반환한다 (line #13).

https://gist.github.com/justinyoo/0516447045d0ef3c606d7e84f0ecd872?file=02-entrypoint.cs&highlights=8-9,13


### 오케스트레이션 펑션 ###

실제로 [오케스트레이션 펑션][az func durable orchestrations]은 아래와 같이 구성했다. 가장 먼저 페이로드를 받아온다 (line #6). 그 다음에는 스케줄의 최대 길이를 체크한다 (line #9-13). [애저 Durable Functions][az func durable]에서 [타이머는 최장 7일 까지만 가능][az func durable timer limitations]한데 이는 [큐 저장소][az storage queue]의 [최대 저장 주기가 7일][az storage queue lifespan]이기 때문이다. 스케줄링의 최대 길이는 설정이 가능하며, 여기서는 이 값을 6.5일로 제한했다.

> 물론 7일보다 긴 스케줄링이 가능하기도 하지만, 이 포스트에서는 다루지 않는다.

그리고 실제 입력받은 스케줄을 검토해서 앱에서 지정한 최대 저장 주기보다 짧은지 확인한다 (line #25-28). 그런 다음에 타이머를 실행시켜 예약을 걸어둔다 (line #30). 이 시점에서 이 오케스트레이션 펑션은 잠시 동작을 멈추고 타이머가 종료되는 시점까지 대기한다. 그리고 실제 타이머가 종료되는 시점에 다시 활성화가 된 후 세번째 펑션을 실행시킨다 (line #32).

https://gist.github.com/justinyoo/0516447045d0ef3c606d7e84f0ecd872?file=03-orchestrator.cs&highlights=6,9-13,25-28,30,32


### 액티비티 펑션 ###

이 펑션에서는 실제로 [깃헙 API][gh api]를 호출해서 이벤트를 보낸다. 아래 코드를 보자. 깃헙 API 문서에서 정의하는 바와 같이 [`repository_dispatch` API][gh api repository dispatch]를 호출한다. 사실 이 부분은 [Octokit][octokit]을 쓰면 굉장히 손쉽게 해결할 수 있는데, 현재 [구현이 아직 안 되어 있어서][octokit issue] 직접 API를 호출하는 식으로 구현했다 (line #18-19).

https://gist.github.com/justinyoo/0516447045d0ef3c606d7e84f0ecd872?file=04-activity.cs&highlights=16,18-19

맨 처음 [Durable Functions][az func durable] 엔드포인트를 호출했을 때 받아온 페이로드는 한 번 더 [`repository_dispatch` API][gh api repository dispatch]를 위해 감싸준다 (line #16). 이 액티비티 펑션이 [GitHub API][gh api]를 호출하면 실제로 [GitHub Actions 워크플로우가 실행][gh actions repository dispatch]된다.


### 웹훅 펑션 ###

이 펑션은 기본적으로 위에 언급한 액티비티 펑션과 동일하다. 단지, 이벤트 타입을 `publish`로 다르게 줄 뿐이다 (line #18). 이 웹훅 펑션의 용도는 아래에서 설명하기로 하자.

https://gist.github.com/justinyoo/0516447045d0ef3c606d7e84f0ecd872?file=05-webhook.cs&highlights=18



## GitHub Actions 설계 ##

그렇다면, [GitHub Actions][gh actions] 워크플로우는 어떻게 만들 수 있을까? 아래 그림을 통해 간단하게 설명을 해 보자. 아래 그림은 전체 워크플로우를 설명한 것이다. 가장 먼저 새 포스트를 작성하면 그에 따른 PR을 준비한다. PR 번호가 생기면 이 번호와 예정 출판 날짜를 정한 후 [애저 Durable Functions][az func durable]를 호출한다.

![][image-02]

[애저 펑션][az func] 부분은 바로 위에서 설명을 했고, 두번째 [GitHub Actions][gh actions] 부분은 [이전 포스트][post prev]에서 설명했다. 이 포스트에서 설명할 부분은 첫번째 [`repository dispatch` 이벤트][gh actions repository dispatch]로 실행되는 [GitHub Actions][gh actions]이다. 아래 워크플로우 정의 문서를 보자. 이 워크플로우는 오직 [`repository_dispatch` 이벤트][gh actions repository dispatch]에 의해서만 실행된다 (line #3). 또한, `if` 조건절에 따라 오직 이벤트 타입이 `merge-pr`일 경우에만 실행된다 (line #8). 이 워크플로우가 하는 일은 무척이나 간단하다. 앞서 PR을 생성했고, 이 워크플로우에서는 [`github-pr-merge-action` 액션][gh actions merge]을 이용해서 이 PR을 머지한다 (line #14).

https://gist.github.com/justinyoo/0516447045d0ef3c606d7e84f0ecd872?file=06-workflow.yaml&highlights=3,8,14,24-27

> **NOTE**: [GitHub PR Merge 액션][gh actions merge]은 내가 만들어서 배포한 것이다. 🙈 써보고 좋으면 별표좀... 굽신

여기서 중요한 것이 하나 있다. 예상대로라면 머지가 끝난 후에 자동으로 그다음 워크플로우가 실행이 되어야 하지만, 실제로는 그렇지 않다. 따라서 그다음 배포 워크플로우를 수동으로 직접 실행시켜야 한다. 그런데, 배포 워크플로우를 수동으로 실행시킬 방법은 현재는 제공되지 않는다. 대신 앞서 언급한 [`repository_dispatch` 이벤트][gh actions repository dispatch]를 이용해서 외부에서 이벤트를 걸어주는 방식으로 실행시키면 된다 (line #24-27). 여기서는 앞서 언급한 [애저 펑션][az func]의 웹훅 엔드포인트를 호출해서 그 펑션 안에서 `publish` 이벤트를 발생시키게 했다.

여기까지 한 후 이 [Durable Functions][az func durable] 앱을 애저로 배포한 후, 실행시켜 보자. 그러면 정해진 날짜에 정확하게 실행이 되어 포스트를 발행하게 된다. 이 포스트 역시 이 절차에 따라 발행이 된 것이다.


## GitOps 적용 ##

[Weaveworks][weaveworks]에서 소개한 [GitOps][weaveworks gitops] 모델의 아이디어는 대략 "PR을 기반으로 변화를 감지해서 애플리케이션을 배포하는 것"이다. 이 포스트에서 소개한 워크플로우도 정확하게 GitOps 모델과 일치하지는 않지만 어느 정도는 PR을 기반으로 해서 배포를 진행하는 방식이 GitOps와 유사하다.

이 포스트를 예로 들어보자.

1. 새 포스트가 준비되었다.
2. 이 포스트를 발행하기 위해서 새로 PR을 생성한다.
3. [애저 Durable Functions][az func durable]을 통해 배포 스케줄을 지정한다.
4. [Durable Functions][az func durable]은 스케줄에 따라 지정한 시점에 GitHub으로 이벤트를 발생시킨다.
5. 이 이벤트에 따라 [GitHub Actions][gh actions]을 통해 PR을 머지한다.
6. PR이 머지된 후 자동으로 웹사이트 배포를 진행한다.

굉장히 GitOps 프로세스와 유사하지 않은가?

---

지금까지 [애저 Durable Functions][az func durable]와 [GitHub Actions][gh actions]를 이용해 정적 블로그 포스트를 예약 발행하는 방법에 대해 알아보았다. 이 전체적인 워크플로우가 한 번 익숙해지면 [애저 Durable Functions][az func durable]과 [GitHub Actions][gh actions]가 작동하는 방식에 대해 좀 더 포괄적인 이해가 가능할 것이다. 정적 웹사이트를 GitHub 리포지토리를 통해 호스팅하고 있다면 이 방법으로 예약 포스팅을 해 보는 것은 어떨까?


[image-01]: /2020/03/scheduling-posts-with-gitops-durable-functions-and-github-actions-01.png
[image-02]: /2020/03/scheduling-posts-with-gitops-durable-functions-and-github-actions-02.png

[post serverless]: https://blog.aliencube.org/ko/2016/06/23/serverless-architectures/
[post prev]: https://blog.aliencube.org/ko/2020/01/03/migrating-wordpress-to-gridsome-on-netlify-through-github-actions/

[todd]: https://twitter.com/toddanglin
[todd publishtodev]: https://www.publishto.dev/

[gh sample]: https://github.com/devkimchi/GitHub-Repository-Event-Scheduler
[gh actions]: https://github.com/features/actions
[gh actions repository dispatch]: https://help.github.com/en/actions/reference/events-that-trigger-workflows#external-events-repository_dispatch
[gh actions merge]: https://github.com/marketplace/actions/github-pr-merge-generic
[gh api]: https://developer.github.com/v3/
[gh api repository dispatch]: https://developer.github.com/v3/repos/#create-a-repository-dispatch-event

[az logapp]: https://docs.microsoft.com/ko-kr/azure/logic-apps/logic-apps-overview?WT.mc_id=aliencubeorg-blog-juyoo
[az func]: https://docs.microsoft.com/ko-kr/azure/azure-functions/functions-overview?WT.mc_id=aliencubeorg-blog-juyoo
[az func durable]: https://docs.microsoft.com/ko-kr/azure/azure-functions/durable/durable-functions-overview?tabs=csharp&WT.mc_id=aliencubeorg-blog-juyoo
[az func durable orchestrations]: https://docs.microsoft.com/ko-kr/azure/azure-functions/durable/durable-functions-orchestrations?tabs=csharp&WT.mc_id=aliencubeorg-blog-juyoo
[az func durable timer limitations]: https://docs.microsoft.com/ko-kr/azure/azure-functions/durable/durable-functions-timers?tabs=csharp&WT.mc_id=aliencubeorg-blog-juyoo#timer-limitations

[az storage table]: https://docs.microsoft.com/ko-kr/azure/storage/tables/table-storage-overview?WT.mc_id=aliencubeorg-blog-juyoo
[az storage queue]: https://docs.microsoft.com/ko-kr/azure/storage/queues/storage-queues-introduction?WT.mc_id=aliencubeorg-blog-juyoo
[az storage queue lifespan]: https://github.com/Azure/azure-functions-durable-extension/issues/14

[octokit]: https://github.com/octokit/octokit.net
[octokit issue]: https://github.com/octokit/octokit.net/issues/2100

[gridsome]: https://gridsome.org/
[wordpress]: https://wordpress.org/
[devto]: https://dev.to/

[weaveworks]: https://www.weave.works/
[weaveworks gitops]: https://www.weave.works/blog/gitops-operations-by-pull-request
