---
title: "깃헙 액션과 Microsoft 365, 파워 플랫폼을 이용해서 혼자서 해커톤 운영하기"
slug: running-hackathon-by-yourself-with-gha-m365-and-pp
description: "이 포스트에서는 깃헙 액션과 Microsoft 365, 파워 플랫폼을 이용해 해커톤 전반적인 운영을 자동화하는 방법에 대해 알아봅니다."
date: "2021-08-20"
author: Justin-Yoo
tags:
- azure
- github-actions
- power-platform
- azure-functions
cover: https://sa0blogs.blob.core.windows.net/aliencube/2021/08/running-hackathon-by-yourself-with-gha-m365-and-pp-00-ko.png
fullscreen: true
---

일반적으로 해커톤은 오프라인으로 한날 한시에 한 곳에 모여 2박 3일 정도의 기간으로 진행한다. 하지만, 요즘과 같은 상황에서는 모든 행사들이 온라인으로 전환이 됐고, 해커톤 역시도 온라인 기반으로 진행하는 경우가 대부분이다. 기존의 오프라인 경험을 그대로 살리기 위해 다양한 온라인 협업 도구들을 사용해서 진행하기도 하지만, 이럴 경우에는 오프라인 형태와 비슷하게 운영 인력들이 꽤 많이 필요하다. 만약 제한된 예산과 제한된 리소스만으로 온라인 해커톤을 운영하고자 한다면 어떻게 해야 할까?

이 포스트는 지난 8월 2일부터 15일까지 [HackaLearn][hackalearn]이라는 100% 온라인 기반의 해커톤 행사를 직접 기획하고 운영했던 기록이다. 혹시나 누군가 이런 비슷한 형태의 온라인 해커톤을 기획하고 있다면 참고가 될 수 있기를 바란다.


## 배경 ##

지난 2021년 5월 [//Build][build2021] 이벤트에서 정식 서비스 론칭을 하게 된 [애저 정적 웹 앱][az swa]은 타사의 정적 웹 앱 서비스에 비해 후발주자인지라 인지도를 높이기 위한 다양한 시도를 하고 있던 중이었다. 그러다가 팀에서 이 [애저 정적 웹 앱][az swa]을 이용한 해커톤을 진행해 보면 어떨까 하는 아이디어가 나왔고, 단순히 해커톤만 하는 것이 아니라 아직 정적 웹 앱 이라는 개념 자체가 생소한 개발자들을 위해 [Microsoft Learn 모듈][az swa learn]로 먼저 정적 웹 앱 자체에 대해 학습하고 이를 이용해서 애플리케이션을 개발하는 해커톤을 진행해 보자 라는 쪽으로 방향을 잡았다.

![HackaLearn 배너][image-01]

실제로 이스라엘에서 이 행사를 가장 먼저 진행했었고, 유럽내 다른 지역에서도 비슷한 컨셉으로 진행 중인 것으로 알고 있는데 일단 한국에서도 HackaLearn 아이디어를 한국 상황에 맞게 좀 더 확장해서 진행해 보는 것으로 했다. 마침 대학생들 방학 기간이기도 하고 해서 [Microsoft Learn 스튜던트 앰배서더(MLSA)][mlsa]들과 [깃헙 캠퍼스 익스퍼트(GCE)][gce]와 함께 코드 리뷰를 하고, 현업의 멘토들과 함께 실무적인 온라인 멘토링을 준비했다.


## 문제점 발견 ##

앞서 언급했다시피 해커톤을 운영하기 위해서는 절대적인 시간과 인력 그리고 예산이 필요하다. 하지만 그 어느 하나도 자유롭지 않은 상태에서 아주 제한된 시간과 인력 그리고 예산만으로 해커톤 이벤트를 진행하기에는 한계가 있었다. 특히나 시스템 운영을 위한 인력은 나 혼자 뿐이고, [MLSA][mlsa]와 [GCE][gce]는 코드 리뷰를 해야 하고, 현장 멘토들은 각자 분야에 맞춰 온라인 멘토링을 담당하기 때문에 최대한 모든 것들을 자동화 시켜놓아야 내 입장에서는 해커톤과 다른 업무를 동시에 진행할 수 있었다.

> 어떻게 하면 거의 모든 업무를 자동화할 수 있을까?

이 질문에 대한 해결책을 찾아 나가는 과정이 바로 이번 해커톤 행사에서 가장 집중한 부분이었다.


### 현재 제약사항 ###

* **행사 진행을 위한 웹사이트가 없음**

  👉 해커톤을 위한 웹사이트가 없었다. 굳이 이 행사만을 위해 일회성 웹사이트를 만드는 것이 비효율적이라 판단하고 대신 깃헙 리포지토리를 그대로 사용하기로 했다.

* **행사 참가자 정보 입력 방법이 없음**

  👉 행사 참가자들이 신청을 위해 자신의 정보를 입력할 수 있는 방법이 없었다. 따라서, 이를 위해 [Microsoft 폼][spo forms] 기능을 사용하기로 했다.

* **행사 참가자 정보 저장용 데이터베이스가 없음**

  👉 행사 참가자들의 정보를 기록하고 진행상황을 추적하기 위한 데이터베이스가 없었다. 따라서, 굳이 이를 위해 데이터베이스를 만들기 보다는 [셰어포인트][spo]의 기능 중 하나인 [Microsoft 리스트][spo lists] 기능을 사용하기로 했다.

* **행사 참가자 및 팀의 진행 상황을 추적하기 위한 팀별 대시보드가 없음**

  👉 행사 참가자 및 팀의 각 챌린지별 진행 상황을 한눈에 볼 수 있는 페이지가 없었다. 이는 참가자 등록시 팀 페이지를 자동을 생성하고 이를 꾸준히 각 팀원들이 PR로 업데이트하는 쪽으로 방향을 정했다.

해커톤 진행에 필요한 전형적인 업무 프로세스 자체는 아래에 언급할 시퀀스 다이어그램과 같이 이미 정의가 된 상태이므로, 이를 구현하기 위해 위에 언급한 제약사항을 중심으로 풀어가야 했다. 생각보다 많은 부분을 별다른 코딩 혹은 개발 노력 없이 자동화 시킬 수 있다고 판단하고 최대한 업무 프로세스를 자동화시키는 쪽으로 방향을 잡았다.


## 프로세스 자동화 구현 계획 ##

앞서 언급한 제약사항이 오히려 새로운 자동화 프로세스를 도입할 수 있는 기회가 됐다.

* 깃헙 리포지토리를 기반으로 HackaLearn 행사를 진행중이니 PR과 이슈들은 모두 [깃헙 액션][gha]을 이용해서 처리하기로 했다.
* [Microsoft 폼][spo forms]라든가, [리스트][spo lists]라든가 하는 다양한 [Microsoft 365][m365]의 서비스를 이용해서 데이터 입력 및 저장 처리를 하기로 했다.
* [파워 플랫폼][pp]의 핵심 기증 중 하나인 [파워 오토메이트][pau]를 활용해서 업무 처리 프로세스를 자동화 시키기로 했다.

이렇게 큰 그림을 그려놓고 깃헙 리포지토리와 Microsoft 365 서비스를 파워 오토메이트로 통합을 시켜놓으면 별도의 애플리케이션 개발에 드는 시간과 비용을 최소화해서 금방 업무 처리 프로세스를 자동화 시킬 수 있었다.


## 프로세스 자동화 구현 결과 &ndash; 참가자 등록 ##

가장 먼저 참가자가 참가 신청 양식을 통해 등록하면 이 데이터를 데이터베이스에 저장시키는 프로세스이다. [Microsoft 폼][spo forms]를 통해 참가 신청을 하게 되면 해당 데이터가 [파워 오토메이트][pau]를 통해 [Microsoft 리스트][spo lists]에 저장되고, 동시에 깃헙 리포지토리에 팀 페이지를 생성한다.

![참가자 등록 프로세스][image-02]

위 그림의 전체 업무 프로세스는 크게 두 부분으로 나뉜다. 하나는 참가자의 정보를 파워 오토메이트에서 처리하는 부분이고, 다른 하나는 깃헙 액션으로 처리하는 부분이다.


### 파워 오토메이트 워크플로우 ###

우선 파워 오토메이트 워크플로우 부분부터 살펴보자. [Microsoft 폼][spo forms]를 통해 참가신청 정보가 들어오면 자동으로 파워 오토메이트 워크플로우가 실행되고, 가장 먼저 이메일 주소를 기준으로 이미 참가신청을 한 사람인지 아닌지 확인한다. 만약 신규 참가자라면 [Microsoft 리스트][spo lists]에 참가자 정보를 저장한다.

![참가자 등록 플로우 1][image-05]

그리고 팀 페이지를 생성한다. 파워 오토메이트에서 직접 생성하는 대신 여기서는 필요한 정보를 바탕으로 템플릿을 만든 후에 이를 깃헙 액션을 호출해서 생성하게끔 한다. 이 때, 깃헙 액션을 호출하는 이벤트는 [`workflow_dispatch`][gha events workflowdispatch]이다.

![참가자 등록 플로우 2][image-06]

마지막으로 등록한 참가자에게 확인 이메일을 보낸다. 이 때 참가자가 영문으로 이름을 등록했을 수도 있고, 한국어로 등록했을 수도 있어서 거기에 맞춰 영문으로 이름을 등록했을 경우에는 `[이름] [성]` 순서(예: Justin Yoo)로, 한국어로 등록했을 경우에는 `[성][이름]` 순서(예: 유저스틴)로 이메일에 들어갈 이름을 맞췄다. 빨간 사각형 안에 들어있는 액션들이 바로 이 작업을 한 것들이다. 이 부분은 향후 별도의 애저 펑션 API와 커스텀 커넥터를 통해 단순화 시킬 수도 있다.

![참가자 등록 플로우 3][image-07]


### 깃헙 액션 워크플로우 ###

위의 파워 오토메이트 워크플로우에서 깃헙 액션을 호출해서 팀 페이지를 생성한다고 언급했다. 그렇다면 이 깃헙 액션은 어떻게 생겼는지 한 번 살펴보자. `workflow_dispatch` 이벤트를 통해 활성화 되며, 입력값으로 `teamName`, `content` 값을 받아온다.

https://gist.github.com/justinyoo/29b678fecea0c2acdbe829644a55cbb5?file=01-on-team-page-requested-1.yaml&highlights=4,6,10

[마켓플레이스][gha marketplace]에 있는 여러 가지 깃헙 액션을 사용해서 팀 페이지를 만들고 다시 리포지토리로 푸시한다.

https://gist.github.com/justinyoo/29b678fecea0c2acdbe829644a55cbb5?file=01-on-team-page-requested-2.yaml&highlights=2,11-14,17

이렇게 함으로써 참가자 등록과 관련한 워크플로우는 모두 자동화가 되었다. 제일 중요한 것들 중 하나가 끝난 셈인데, 이제 다음 프로세스로 넘어가 보자.


## 프로세스 자동화 구현 결과 &ndash; 챌린지 진행상황 업데이트 ##

이번 HackaLearn 이벤트에서는 개별 참가자는 총 여섯 가지의 챌린지를 수행해야 하는데 하나를 끝낼 때마다 진행상황을 자신의 팀 페이지를 수정하는 PR을 통해 업데이트한다. 개별 챌린지마다 큰 차이는 없으니 여기서는 간단하게 SNS 인증샷 챌린지를 예로 들어보기로 하자.

![SNS 인증샷 챌린지][image-03]

1. 가장 먼저 참가자가 자신의 팀 페이지에 SNS 인증샷 포스트 링크를 업데이트하고 PR을 생성하면 아래와 같이 깃헙 액션이 돌면서 `review-required`라고 라벨링을 한다.
2. 리뷰어가 해당 링크를 직접 들어가서 요구한 해시태그 `#hackalearn`, `#hackalearnkorea`가 들어있는지 확인하고 인증샷 내용이 알맞는지 확인한다.
3. 리뷰어는 확인이 끝난 후 `review-completed`라고 라벨링을 한다. 그러면 깃헙 액션이 알아서 `review-required`라는 라벨을 삭제한다.
4. 리뷰어는 최종적으로 `/socialsignoff`라는 코멘트를 통해 다음 액션을 실행시킬 수 있다. 이 코멘트는 다시 깃헙 액션을 실행시키고, 깃헙 액션은 파워 오토메이트를 실행시켜 [Microsoft 리스트][spo lists]에 저장되어 있는 참가자의 진행상황 정보를 업데이트한다.
5. 업데이트가 끝나면 파워 오토메이트는 다시 깃헙 액션을 호출해서 `record-updated`, `completed-social` 라벨을 추가하고, `review-completed` 라벨을 삭제한다.
6. 만약 이 때 참가자의 깃헙 ID가 확인되지 않는다면, 다시 돌아와서 `review-required`라고 다시 라벨링을 하고 추가적인 리뷰를 요구한다.


### 깃헙 액션 워크플로우 ###

여기서 이와 관련한 깃헙 액션 워크플로우를 한 번 살펴보자. 총 다섯 개의 깃헙 액션 워크플로우를 사용했다.


#### 챌린지 등록 PR ####

참가자가 챌린지 완료 결과를 PR을 통해 등록하면 맨 처음에는 아래와 같은 깃헙 액션 워크플로우를 실행한다. 참가자가 포크한 리포지토리로부터 PR이 들어오기 때문에 [`pull_request_target`][gha events pullrequesttarget]이라는 이벤트 트리거를 사용했고, `teams` 디렉토리 아래의 파일들을 업데이트 할 때만 작동하는 것으로 조건을 걸어뒀다.

https://gist.github.com/justinyoo/29b678fecea0c2acdbe829644a55cbb5?file=02-on-challenge-submitted-1.yaml&highlights=4,10

만약 PR 생성 시각이 최종 제출 마감 시각보다 늦다면, 해당 PR은 더이상 받지 않아야 한다. 따라서, 이를 자동으로 체크하기 위한 로직을 파워셸 스크립트를 이용해서 구현했다. PR 이벤트에서 가져오는 `created_at` 속성값은 UTC 값이므로 마감 시각과 비교를 위해서는 한국 시각으로 변환을 시켜야 하는데, 그 작업 역시 해당 스크립트 안에 포함시켰다.

https://gist.github.com/justinyoo/29b678fecea0c2acdbe829644a55cbb5?file=02-on-challenge-submitted-2.yaml&highlights=12-16

위와 같이 제출 시각을 검토한 후, 만약 최종 제출 기한을 넘겼다면 더이상 받아줄 수 없으므로 곧바로 해당 PR을 닫는다.

https://gist.github.com/justinyoo/29b678fecea0c2acdbe829644a55cbb5?file=02-on-challenge-submitted-3.yaml&highlights=2,10,25

아직 제출 기한 전이라면 아래와 같이 라벨링을 하고, 코멘트를 남긴 후에 리뷰어를 랜덤으로 할당한다.

https://gist.github.com/justinyoo/29b678fecea0c2acdbe829644a55cbb5?file=02-on-challenge-submitted-4.yaml&highlights=2,9,24


#### 챌린지 리뷰 완료 ####

리뷰어가 해당 PR에 할당되면 이제 리뷰어는 챌린지에 대한 리뷰를 하고 그 결과에 따라 라벨링을 한다. 이 라벨링은 새로운 깃헙 액션을 트리거한다.

https://gist.github.com/justinyoo/29b678fecea0c2acdbe829644a55cbb5?file=03-on-challenge-labelled-1.yaml&highlights=6,7


#### 챌린지 리뷰 승인 ####

리뷰를 완료한 후 위와 같이 라벨링이 끝나면 다음으로는 `/signoff` 코멘트를 남겨서 자동으로 다음 깃헙 액션이 실행되게끔 한다. 아래 깃헙 액션은 [`issue_comment`][gha events issuecomment] 이벤트를 받아 실행된다.

https://gist.github.com/justinyoo/29b678fecea0c2acdbe829644a55cbb5?file=04-on-challenge-review-commented-1.yaml&highlights=4-6

워크플로우의 첫번째 단계는 할당된 리뷰어가 승인을 했는지, 어떤 챌린지인지를 검증하는 작업이다. 파워셸 스크립트로 아래와 같이 구현한다. 승인 전에 `review-completed`라는 라벨이 반드시 존재해야 하고, 코멘트를 남긴 사람이 현재 할당된 리뷰어여야 하며, 허락된 리뷰어 리스트(`secrets.PR_REVIEWERS`)에 있어야만 한다.

https://gist.github.com/justinyoo/29b678fecea0c2acdbe829644a55cbb5?file=04-on-challenge-review-commented-2.yaml&highlights=16-18

위의 조건을 만족시켰다면 챌린지의 종류에 따라 아래중 하나의 액션을 실행한다. 아래 액션은 파워 오토메이트를 호출하는 액션이다. 파워 오토메이트 워크플로우는 깃헙 액션으로부터 넘어온 데이터를 처리하고 확인 이메일을 보낸 뒤, 다시 다음 깃헙 액션을 호출한다.

https://gist.github.com/justinyoo/29b678fecea0c2acdbe829644a55cbb5?file=04-on-challenge-review-commented-3.yaml&highlights=2,9,16,23,30,37


#### 챌린지 완료 혹은 추가 작업 요구 ####

앞서 파워 오토메이트에서 처리할 것들을 다 하고난 후에 다시 깃헙 액션을 호출해서 나머지 단계를 수행하게 된다. 이 때 깃헙 액션은 [`workflow_dispatch`][gha events workflowdispatch] 이벤트를 받아 실행된다. 파워 오토메이트로부터 받는 입력값은 `prId`, `labelsToAdd`, `labelsToRemove`, `isMergeable`이다.

https://gist.github.com/justinyoo/29b678fecea0c2acdbe829644a55cbb5?file=05-on-challenge-completed-1.yaml&highlights=4,6,10,14,18

이 워크플로우의 첫번째 액션은 위에서 받은 입력값을 이용해 라벨을 추가하고 삭제하는 것이다. 이미 파워 오토메이트에서 데이터베이스 업데이트도 이뤄졌고, 이메일도 보냈기 때문에 해당 과정을 반영한 라벨를 새롭게 추가하고 기존 라벨을 삭제한다.

https://gist.github.com/justinyoo/29b678fecea0c2acdbe829644a55cbb5?file=05-on-challenge-completed-2.yaml&highlights=8

마지막으로 해당 챌린지 진행 상황이 들어간 PR을 병합하는 액션이다. 만약 파워 오토메이트 워크플로우에서 오류가 생겼다면 `isMeargeable` 값이 `false`일 것이므로 아래 액션은 실행되지 않는다.

https://gist.github.com/justinyoo/29b678fecea0c2acdbe829644a55cbb5?file=05-on-challenge-completed-3.yaml&highlights=8-9


### 파워 오토메이트 워크플로우 ###

앞서 깃헙 액션을 통해 챌린지 진행상황을 등록하는 파워 오토메이트 워크플로우를 호출한다고 했는데, 아래 파워 오토메이트는 이를 처리하는 플로우를 나타낸다. 가장 먼저 어떤 챌린지인지 확인한다. 어느 챌린지에도 해당하지 않는다면 처리하지 않는다.

![챌린지 등록 워크플로우 1][image-08]

이후 등록된 깃헙 ID인지를 확인해서 참가자가 맞다면 [Microsoft 리스트][spo lists]의 데이터를 업데이트하고 그렇지 않으면 처리하지 않는다.

![챌린지 등록 워크플로우 2][image-09]

마지막으로 모든 챌린지를 완료했을 경우와 아닌 경우를 나눠 확인 이메일을 보낸다.

![챌린지 등록 워크플로우 3][image-10]


## 기타 파워 오토메이트 워크플로우 ##

여기서 사용한 파워 오토메이트 워크플로우는 크게 깃헙 액션을 통해서 실행시키는 것과 깃헙 액션과 상관 없이 관리 목적을 위해 수동으로 실행시키는 두가지 형태로 구성한다. 두 가지 형태 모두 [Microsoft 리스트][spo lists]를 이용해 데이터를 조회하고 이 데이터를 바탕으로 이메일을 보낸다든가, 깃헙 리포지토리에 파일을 업데이트한다든가 하는 일련의 작업들인데 위의 과정과 비슷하기 때문에 여기서는 특별히 따로 언급하지 않는다. 대신 아래 스크린샷을 통해 사용했던 파워 오토메이트 워크플로우를 보여주고자 한다. 총 15개의 워크플로우를 사용했다.

![파워 오토메이트 리스트][image-04]

위와 같이 프로세스를 자동화 해 놓으니, 운영하는 입장에서는 게시판에 질문이 올라올 때와 PR을 리뷰할 때 빼놓고는 수동으로 처리할 일이 하나도 없었다. 참가자들 대부분이 아직 학생이고 깃과 깃헙 자체에 익숙하지 않은 사람들이 많아서 리뷰 시간의 대부분은 PR 병합시 발생하는 충돌사항을 처리하는 것들이었다.


## 결과 ##

이렇게 해서 HackaLearn 이벤트는 모두 성황리에 끝났다. 참고로 아래 숫자들은 이번 HackaLearn 이벤트와 관련한 숫자들이다.

* `14`: HackaLearn 기간
* `171`: 총 등록자 수
* `62`: 클라우드 스킬 챌린지 완료자 수
* `21`: SNS 인증샷 챌린지 완료 팀 수
* `17`: 애저 정적 웹 앱 URL 챌린지 완료 팀 수
* `16`: 깃헙 리포지토리 링크 챌린지 완료 팀 수
* `20`: 블로그 후기 챌린지 완료 팀 수
* `13`: 모든 여섯 가지 챌린지 완료 팀 수


## 배운 것들 ##

이렇게 자동화에 포커스를 맞추면서 행사를 운영하면서 여러 가지 고려사항들을 배웠다. 다음에 또 이런 형태로 이벤트를 진행한다면 이를 보완해야 할 것이다.

* **참가자들이 보내는 PR 형식이 기대치와 다름을 인정해야 한다**
  
  👉 따라서, 리뷰 프로세스 자동화를 구성할 때 최대한 유연하게 동작하도록 해야 한다.

* **리뷰 프로세스는 최대한 간결하게 만들어야 한다**

  👉 이번에는 살짝 복잡했는데, 이것 때문에 리뷰어들이 혼란스러워 했다. 리뷰어는 PR 리뷰에만 신경쓰게 하고 나머지는 최대한 간결하게 만들어야 한다.

* **리뷰어는 PR마다 무작위로 할당하는 대신 팀 단위로 할당해야 한다**

  👉 리뷰어를 무작위로 할당하되 팀 단위로 할당한다면 PR 병합시 컨플릭트가 생기는 문제를 엄청나게 해소할 수 있다.

* **파워 오토메이트 워크플로우를 최대한 모듈화해서 재사용한다**

  👉 파워 오토메이트 워크플로우에서 비슷한 시퀀스로 여러 단계를 묶어서 사용하는 경우가 많다.
  👉 따라서, 이런 것들은 블록 단위로 묶어서 하위 워크플로우로 만들거나 애저 펑션 API 같은 것으로 처리하게 하고 이를 커스텀 커넥터로 묶어 워크플로우를 간결하게 해야 한다.


## 부대행사 ##

HackaLearn 이벤트 도중에 [깃헙 액션][gha]과 [애저 정적 웹 앱][az swa] 서비스를 활용해서 실제로 서비스를 만들어 보는 핸즈온 워크샵을 세 번 정도 진행했다.

* 깃헙 액션 핸즈온 워크샵

  https://youtu.be/e_elLW6uNSc

  <br>

* 애저 정적 웹 앱 핸즈온 워크샵

  https://youtu.be/Hxkv6AjAisY

  <br>

* 헤드리스 CMS를 활용해 정적 웹 앱 만들기 핸즈온 워크샵

  https://youtu.be/x3j3mDblqMY

  <br>

---

지금까지 온라인으로 해커톤을 혼자서 진행할 때 [깃헙 액션][gha]과 [Microsoft 365][m365], 그리고 [파워 오토메이트][pau]를 이용해서 모든 운영 프로세스를 자동화 하는 방법에 대해 간략하게 적어봤다. 물론, 앞으로 개선의 여지는 충분히 있지만, 앞으로도 충분히 혼자서 기획하고 운영할 수 있지 않을까 생각한다.

이 자리를 빌어 최전방에서 참가자들의 PR 리뷰에 참여해 준 우리 [MLSA][mlsa]와 [GCE][gce] 스탭들, 그리고 현업 멘토님들께 다시 한 번 감사를 드리고 싶다. 이분들이 없었더라면 아무리 운영 자동화를 시켜놨다 하더라도 절대로 안정적으로 행사가 진행되지 않았을 것이다.


[image-01]: https://sa0blogs.blob.core.windows.net/aliencube/2021/08/running-hackathon-by-yourself-with-gha-m365-and-pp-01-ko.png
[image-02]: https://sa0blogs.blob.core.windows.net/aliencube/2021/08/registration-ko.png
[image-03]: https://sa0blogs.blob.core.windows.net/aliencube/2021/08/challenge-social-ko.png
[image-04]: https://sa0blogs.blob.core.windows.net/aliencube/2021/08/running-hackathon-by-yourself-with-gha-m365-and-pp-04.png
[image-05]: https://sa0blogs.blob.core.windows.net/aliencube/2021/08/running-hackathon-by-yourself-with-gha-m365-and-pp-05.png
[image-06]: https://sa0blogs.blob.core.windows.net/aliencube/2021/08/running-hackathon-by-yourself-with-gha-m365-and-pp-06.png
[image-07]: https://sa0blogs.blob.core.windows.net/aliencube/2021/08/running-hackathon-by-yourself-with-gha-m365-and-pp-07.png
[image-08]: https://sa0blogs.blob.core.windows.net/aliencube/2021/08/running-hackathon-by-yourself-with-gha-m365-and-pp-08.png
[image-09]: https://sa0blogs.blob.core.windows.net/aliencube/2021/08/running-hackathon-by-yourself-with-gha-m365-and-pp-09.png
[image-10]: https://sa0blogs.blob.core.windows.net/aliencube/2021/08/running-hackathon-by-yourself-with-gha-m365-and-pp-10.png


[hackalearn]: https://aka.ms/hackalearn/korea

[build2021]: https://mybuild.microsoft.com/home?WT.mc_id=power-39037-juyoo

[az swa]: https://docs.microsoft.com/ko-kr/azure/static-web-apps/overview?WT.mc_id=power-39037-juyoo
[az swa learn]: https://docs.microsoft.com/ko-kr/learn/paths/azure-static-web-apps/?WT.mc_id=power-39037-juyoo

[mlsa]: https://studentambassadors.microsoft.com/ko-kr/?WT.mc_id=power-39037-juyoo
[gce]: https://githubcampus.expert

[m365]: https://www.microsoft.com/ko-kr/microsoft-365?WT.mc_id=power-39037-juyoo

[spo]: https://www.microsoft.com/ko-kr/microsoft-365/sharepoint/collaboration?WT.mc_id=power-39037-juyoo
[spo lists]: https://www.microsoft.com/ko-kr/microsoft-365/microsoft-lists?WT.mc_id=power-39037-juyoo
[spo forms]: https://www.microsoft.com/ko-kr/microsoft-365/online-surveys-polls-quizzes?WT.mc_id=power-39037-juyoo

[gha]: https://github.com/features/actions
[gha marketplace]: https://github.com/marketplace?type=actions
[gha events workflowdispatch]: https://docs.github.com/en/github-ae@latest/actions/reference/events-that-trigger-workflows#workflow_dispatch
[gha events pullrequesttarget]: https://docs.github.com/en/github-ae@latest/actions/reference/events-that-trigger-workflows#pull_request_target
[gha events issuecomment]: https://docs.github.com/en/github-ae@latest/actions/reference/events-that-trigger-workflows#issue_comment

[pp]: https://powerplatform.microsoft.com/ko-kr/?WT.mc_id=power-39037-juyoo
[pau]: https://flow.microsoft.com/ko-kr/?WT.mc_id=power-39037-juyoo
