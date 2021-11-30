---
title: "깃헙 액션 재활용 기능을 활용해서 애저 정적 웹 앱 CI/CD 파이프라인 리팩토링하기"
slug: refactoring-aswa-github-actions-workflow
description: "이 포스트에서는 깃헙 액션의 워크플로우 재활용 기능을 이용해서 애저 정적 웹 앱의 CI/CD 파이프라인을 리팩토링해 봅니다."
date: "2021-12-01"
author: Justin-Yoo
tags:
- azure
- azure-static-webapps
- github-actions
- refactoring
cover: https://sa0blogs.blob.core.windows.net/aliencube/2021/12/refactoring-aswa-github-actions-workflow-00.png
fullscreen: true
---


예전에 [애저 DevOps 파이프라인을 효과적으로 리팩토링하는 전략][post azdo refactoring]에 대해 논의해 본 적이 있었다. [깃헙 액션][gh actions]도 꽤 효율적이긴 한데, [애저 DevOps][az do]만큼 효과적으로 구현하려면 고민해야 할 지점이 상당히 많은 편이다. 그 중에서 최근에 출시한 ["재활용 워크플로우"][gh actions reusable] 기능을 이용하면 이런 고민들을 상당히 많이 덜어낼 수 있다. 이 포스트에서는 이 "재활용 워크플로우" 기능을 이용해서 [애저 정적 웹 앱][az swa]에서 자동으로 생성해 주는 CI/CD 파이프라인을 리팩토링해 보기로 한다.


## 애저 정적 웹 앱 워크플로우 ##

[애저 정적 웹 앱][az swa] 인스턴스를 프로비저닝할 때 별다른 설정을 하지 않으면 자동으로 깃헙 액션 워크플로우가 생성된다. 아래는 자동으로 생성되는 워크플로우 파일의 예이다.

https://gist.github.com/justinyoo/3f8de0ebaff5bdd7e41c961ed37b5b53?file=01-original-workflow.yaml

문제는 한 번 생성된 워크플로우 파일은 파일 이름을 변경할 수가 없기 때문에 워크플로우를 수정하려면 항상 이 파일 안에서만 수정해야 한다.

어떤 상황을 한 번 가정해 보자. 회사에서 애저 정적 웹 앱으로 배포하는 코드가 있고, 개발용, 테스트용, 라이브용 인스턴스가 따로 있다고 가정을 해 보자. 굳이 [모노리포 방식][monorepo]까지 언급을 하지 않더라도 충분히 나올 수 있는 상황이다. 이런 경우에는 인스턴스 수 만큼 자동으로 깃헙 액션 워크플로우가 생성된다. 하지만, 그 안의 내용은 변수값을 제외하고는 차이가 없다. 동일한 코드를 굳이 여러 파일에서 나눠서 관리하기 보다는 리팩토링을 해서 하나로 합친 후 이를 각자 호출하면 훨씬 더 간결해 질 것이다.

얼마전까지만 해도 이 리팩토링을 위해 [`workflow_dispatch`][gh actions workflow_dispath] 이벤트를 활용해야 했다. 이 이벤트를 자동으로 호출하기 위해서는 [웹훅 이벤트][gh api workflow_dispatch]까지 동원해야 한다. 한 번 구성을 해 놓으면 웹훅을 위한 액세스 토큰이 유효한 이상 계속 사용할 수 있다. 하지만, 어느 순간 액세스 토큰 유효기간이 지났다거나, 노출됐다거나 하는 등의 이유로 토큰을 사용하지 못하는 상황이 생긴다면? 이런 부분을 좀 해결할 수 있는 방법이 없을까?


## 재활용 워크플로우 ##

이번에 새롭게 소개된 [`workflow_call`][gh actions workflow_call] 이벤트를 활용하면 워크플로우를 job 수준에서 좀 더 편리하게 재활용 할 수 있게 된다. 위 코드를 살짝 리팩토링해 보자. 우선 위에 정의해 놓은 두 `build_and_deploy_job`, `close_pull_request_job`을 그대로 복사해 온다. 이 워크플로우를 "재활용 워크플로우"라고 하자.

https://gist.github.com/justinyoo/3f8de0ebaff5bdd7e41c961ed37b5b53?file=02-reusable-workflow-1.yaml

위 워크플로우에 보면 이벤트 개체에서 받아오는 값과 시크릿 변수값, 웹 앱 소스 위치값이 있는데, 이를 나열해 보면 아래와 같다. 이 값은 변경 가능한 변수값이다.

* 이벤트 개체
  * 이벤트 이름: `github.event_name`
  * 이벤트 액션: `github.event.action`
* 시크릿
  * 애저 정적 웹 앱 배포 API 토큰: `secrets.AZURE_STATIC_WEB_APPS_API_TOKEN_XXXX_XXXX_XXXX`
  * 깃헙 토큰: `secrets.GITHUB_TOKEN`
* 애저 정적 웹 앱 위치
  * 웹 앱 위치: `app_location`
  * API 앱 위치: `api_location`
  * 웹 앱 아티팩트 위치: `output_location`

하지만, 재활용 워크플로우에서는 이 값들에 직접 접근할 수 없으므로 변수 처리를 해서 항상 "호출 워크플로우"를 통해 받아와야 한다. 재활용 워크플로우에 있던 여러 값들을 아래와 같이 변수형태로 바꿔보자 *(line #7-8, 20-21, 23-24, 30-31, 33-34, 36-37, 43-44, 54-55)*.

https://gist.github.com/justinyoo/3f8de0ebaff5bdd7e41c961ed37b5b53?file=03-reusable-workflow-2.yaml&highlights=7-8,20-21,23-24,30-31,33-34,36-37,43-44,54-55

위와 같이 변수 처리를 했다면, 이제 이 변수들에 대해 정의를 해야 한다. 재활용 워크플로우에서 변수 정의는 아래와 같이 `workflow_call` 이벤트에 붙여서 하면 된다. 일반 변수는 `inputs` 어트리뷰트 아래에 *(line #6-21)*, 시크릿 변수는 `secrets` 어트리뷰트 아래에 *(line #23-27)* 정의한다.

https://gist.github.com/justinyoo/3f8de0ebaff5bdd7e41c961ed37b5b53?file=04-reusable-workflow-3.yaml&highlights=6-21,23-27

이렇게 재활용 워크플로우에 대한 리팩토링이 끝났다.


## 호출 워크플로우 ##

이제 애저 정적 웹 앱 워크플로우를 수정해 보자. 기존 `jobs` 노드 아래에 있던 내용은 더이상 필요없으므로 지우고 아래와 같이 "재활용 워크플로우"를 설정한다.

* 재활용 워크플로우 *(line #14-15)*: `<org_name>/<repo_name>/.github/workflows/<reusable_workflow_filename>@<branch_or_tag>`
* 변수 *(line #17-22)*: `with` 속성
* 시크릿 *(line #24-26)*: `secrets` 속성

https://gist.github.com/justinyoo/3f8de0ebaff5bdd7e41c961ed37b5b53?file=05-caller-workflow.yaml&highlights=14-15,17-22,24-26

이제 "호출 워크플로우"가 몇개가 됐든, 위와 같이 리팩토링을 해 놓으면 더이상 "호출 워크플로우"는 수정할 필요 없이 "재활용 워크플로우"만 수정하면 모든 "호출 워크플로우"로 한 번에 변경사항을 반영할 수 있어 무척이나 편리해진다.

---

지금까지 [애저 정적 웹 앱][az swa] 인스턴스 배포를 위한 [깃헙 액션 워크플로우][gh actions]를 [재활용 워크플로우][gh actions reusable] 기능을 이용해 리팩토링해 봤다. 이 아이디어를 기반으로 좀 더 다양한 빌드/배포 시나리오에서 활용할 수 있을 것이다.


[post azdo refactoring]: /ko/2019/09/04/azure-devops-pipelines-refactoring-technics/

[az do]: https://docs.microsoft.com/ko-kr/azure/devops/user-guide/what-is-azure-devops?view=azure-devops&WT.mc_id=dotnet-51099-juyooo&ocid=AID3035186

[az swa]: https://docs.microsoft.com/ko-kr/azure/static-web-apps/overview?WT.mc_id=dotnet-51099-juyooo&ocid=AID3035186

[gh actions]: https://docs.github.com/en/actions
[gh actions reusable]: https://docs.github.com/en/actions/learn-github-actions/reusing-workflows
[gh actions workflow_call]: https://docs.github.com/en/actions/learn-github-actions/events-that-trigger-workflows#workflow_call
[gh actions workflow_dispath]: https://docs.github.com/en/actions/learn-github-actions/events-that-trigger-workflows#workflow_dispatch
[gh api workflow_dispatch]: https://docs.github.com/en/rest/reference/actions#create-a-workflow-dispatch-event

[monorepo]: https://en.wikipedia.org/wiki/Monorepo
