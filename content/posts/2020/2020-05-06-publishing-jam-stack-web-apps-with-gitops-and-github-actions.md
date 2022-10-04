---
title: "GitOps와 GitHub Actions를 이용해서 JAM 스택 웹사이트 자동 배포하기"
slug: publishing-jam-stack-web-apps-with-gitops-and-github-actions
description: "이 포스트에서는 GitHub Actions와 GitOps를 이용해서 JAM 스택 기반 웹사이트를 자동으로 배포하는 방법을 알아봅니다."
date: "2020-05-06"
author: Justin-Yoo
tags:
- gitops
- github-actions
- jam-stack
- azure-durable-functions
cover: /2020/05/publishing-jam-stack-web-apps-with-gitops-and-github-actions-00.png
fullscreen: true
---

[지난 포스트][post gitops schedule]에서는 [애저 Durable Functions][az func durable]와 [깃헙 액션][gh actions]를 이용해서 [JAM 스택][jam stack] 기반의 웹사이트를 예약 배포하는 방법에 대해 알아 보았다. 하지만, 예약 포스팅을 하는 부분은 자동화하지 못했는데, 이번 포스트에서는 이 예약 포스팅을 하는 부분까지도 자동화 시켜보도록 한다.


## JAM 스택이란? ##

JAM은 JavaScript, API 그리고 Mark-up을 통틀어 일컫는 약어이다. 좀 더 쉽게 얘기하자면, [Jekyll][jekyll], [Hugo][hugo], [Gatsby][gatsby], [VuePress][vuepress], [Gridsome][gridsome] 등과 같은 정적 웹사이트 생성기를 통해 만들어진 HTML 문서들을 프론트엔드로 하고, 백엔드 API와는 JavaScript로 통신하는 형태의 웹 앱이라고 보면 되겠다. 지금 여러분이 보고 있는 이 웹사이트 역시 [Gridsome][gridsome]으로 정적 웹사이트를 만들어서 운영하는 중이다.


## GitOps 자동화 ##

[지난 포스트][post gitops schedule]에서는 아래 그림에서 붉은색 화살표로 표시한 첫번째 단계를 제외한 모든 부분을 자동화하는 워크플로우를 작성했다.

![][image-01]

저 붉은 화살표가 하는 일은 PR을 생성하고 난 후 [애저 Durable Functions][az func durable]으로 예약 포스팅을 하는 API 요청을 보내면 되는 것인데, 이 부분을 자동화할 깔끔한 방법을 찾아내지 못하다가, PR 역시도 하나의 이벤트이고, 이벤트 페이로드에서 뭔가를 활용할 수 있겠다는 부분에 생각이 미쳤다.

따라서, 아래와 같이 PR을 생성할 때 본문에 예약 포스팅 타임스탬프를 적어두고 이를 이용하는 방식으로 접근해 봤다.

![][image-02]

이렇게 PR 요청 본문을 작성하고 PR에 대응하는 별도의 깃헙 액션 워크플로우를 아래와 같이 작성했다. 이 워크플로우는 PR이 처음 만들어질 때만 반응하게끔 한다 (line #8). 그리고 전체 이벤트 페이로드를 한 번 열어볼 수 있게끔 구성했다 (line #20).

https://gist.github.com/justinyoo/b1793b9fe678641de6308b956c3d77d1?file=01-pr-flow-1.yaml&highlights=8,20

이렇게 하고 나서 한 번 PR을 생성해 보면 대략 아래와 같은 이벤트 페이로드 구조를 볼 수 있다.

![][image-03]

따라서 이 `body` 필드의 값을 좀 더 정제해서 실제 타임스탬프를 가져와서 사용하면 된다. 이 부분은 깃헙 액션에서 아래와 같이 구성했다. 이벤트 페이로드에서 body 값을 추출한 후 이를 정규표현식을 이용해 타임스탬프를 추려냈다. 그리고 이렇게 추려낸 값을 `published`라는 반환값으로 지정했다 (line #5).

https://gist.github.com/justinyoo/b1793b9fe678641de6308b956c3d77d1?file=02-pr-flow-2.yaml&highlights=5

실제로 이렇게 추출한 `published`라는 값은 아래와 같은 액션으로 확인이 가능하다. 앞에 정의한 액션은 `prbody`라는 ID 값을 지정했고, 이를 통해 `steps.prbody.outputs.published`와 같은 형식으로 반환값에 접근이 가능하다.

https://gist.github.com/justinyoo/b1793b9fe678641de6308b956c3d77d1?file=03-pr-flow-3.yaml

이제 필요한 발행일 타임스탬프를 확보했으니 실제로 [애저 Durable Functions][az func durable] 엔드포인트로 예약 발행 API 요청을 보낼 차례이다. 이 역시도 깃헙 액션을 이용해 간단하게 구현했다. 아래는 `curl` 명령어를 이용해 애저 펑션으로 API 요청을 보내는 액션이다.

https://gist.github.com/justinyoo/b1793b9fe678641de6308b956c3d77d1?file=04-pr-flow-4.yaml

이렇게 함으로써 아래처럼 수동으로 API 요청을 보내는 대신

![][image-05]

PR 워크플로우가 자동으로 모든 것을 해결한다.

![][image-04]

이제는 PR만 생성하고 그 요청 본문에 타임스탬프만 넣어두면 자동으로 포스트 발행 예약이 진행되고 그에 맞춰 블로그 포스트가 발행된다.

---

지금까지 블로그 포스트 발행을 위한 모든 과정을 GitOps와 [애저 Durable Functions][az func durable], 그리고 [깃헙 액션][gh actions]을 이용해서 자동화해 봤다. 만약 정적 웹사이트를 운영하고 있고, 리포지토리가 깃헙에 있다면 이와 같은 방식으로 예약 포스팅을 자동화하는 것을 고려해 보면 좋을 것이다.


[image-01]: /2020/05/publishing-jam-stack-web-apps-with-gitops-and-github-actions-01.png
[image-02]: /2020/05/publishing-jam-stack-web-apps-with-gitops-and-github-actions-02.png
[image-03]: /2020/05/publishing-jam-stack-web-apps-with-gitops-and-github-actions-03.png
[image-04]: /2020/05/publishing-jam-stack-web-apps-with-gitops-and-github-actions-04.png
[image-05]: /2020/05/publishing-jam-stack-web-apps-with-gitops-and-github-actions-05.png

[post gitops schedule]: /ko/2020/03/25/scheduling-posts-with-gitops-durable-functions-and-github-actions/

[gh sample]: https://github.com/devkimchi/KeyVault-Reference-Sample
[gh actions]: https://github.com/features/actions

[jam stack]: https://jamstack.org/
[jekyll]: https://jekyllrb.com/
[hugo]: https://gohugo.io/
[gatsby]: https://www.gatsbyjs.org/
[vuepress]: https://vuepress.vuejs.org/
[gridsome]: https://gridsome.org/

[az func]: https://docs.microsoft.com/ko-kr/azure/azure-functions/functions-overview?WT.mc_id=aliencubeorg-blog-juyoo
[az func durable]: https://docs.microsoft.com/ko-kr/azure/azure-functions/durable/durable-functions-overview?tabs=csharp&WT.mc_id=aliencubeorg-blog-juyoo
