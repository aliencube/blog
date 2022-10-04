---
title: "애저 Durable Functions을 이용해서 RequestBin 앱 만들기"
slug: building-requestbin-with-durable-functions
description: "이 포스트에서는 애저 Durable Functions의 Stateful 속성을 이용해서 RequestBin 앱을 만들어 보겠습니다."
date: "2020-04-23"
author: Justin-Yoo
tags:
- azure-durable-functions
- requestbin
- stateful-api
- event-sourcing
cover: /2020/04/building-requestbin-with-durable-functions-00.png
fullscreen: true
---

[이전 포스트][post gitops]에서 [애저 Durable Functions][az func durable]를 통해 워크플로우 오케스트레이션에 대해 다뤄보았다. 이런 오케스트레이션은 이 [Durable Functions][az func durable]의 Stateful한 특성 때문인데, 이 특성을 이용하면 훨씬 더 다양한 용도로 사용할 수 있다. 이번 포스트에서는 바로 이 [Durable Functions][az func durable]의 Stateful 속성을 이용해서 [RequestBin 애플리케이션][requestbin] 기능을 구현해 보기로 한다.

> 이 포스트에 쓰인 샘플 코드는 [Durable RequestBin Sample][gh sample]에서 다운로드 받을 수 있다.

웹훅을 테스트 하기 위해 사용했던 추억의 RequestBin 앱 첫화면을 다들 기억할 것이다.

![][image-01]

현재는 정식으로 서비스 되고 있지 않고 다만 소스 코드와 함께 [샘플 형태][requestbin herokuapp]로 운영되고 있다. 따라서, 내가 직접 이 서비스를 운영하고 싶다면 도커 컨테이너를 이용해서 어딘가에 호스팅을 해야 한다. 이와 관련해서 [애저 컨테이너 인스턴스][az aci]를 활용해서 [올리는 방법][gh sample aci]과 [애저 앱 서비스][az appsvc]를 활용해서 [올리는 방법][gh sample appsvc]을 예전에 소개한 적이 있다. 오리지널 RequestBin 앱은 애플리케이션과 Redis 캐시로 구성이 되어 있는데, 캐시의 특성상 언제든 데이터가 소실될 수 있기 때문에 가끔은 오래된 웹훅 히스토리를 확인하고 싶을 때에는 난감할 수 있다.

마침 애저 [Durable Functions][az func durable]는 자체적으로 테이블 저장소 기능을 이용해서 [이벤트 소싱 패턴][event sourcing pattern]을 구현해 놓았다. 또한 이를 통해 데이터를 Stateful하게 저장할 수 있기 때문에 코드와 데이터를 동시에 다뤄야 하는 RequestBin 애플리케이션을 처음부터 만들어 보기에 아주 적절한 예시가 될 수 있다.


## 상태 저장 엔티티 ##

[Durable Functions][az func durable]의 오케스트레이션 기능은 `IDurableOrchestrationClient`를 통해 State (상태)를 암시적으로 저장하는 반면에, 상태 저장 엔티티를 사용하면 `IDurableClient`를 통해 이 State를 명시적으로 저장하고 호출한다. 따라서, 대략 아래와 같은 모양이 될 것이다. 오케스트레이션을 구현하는 대신 State에 직접 접근하는 `IDurableClient` 인스턴스가 보일 것이다 (line #4).

https://gist.github.com/justinyoo/01426032d1ee6886796d9cb72e048dd9?file=01-create-bin.cs&highlights=4

이제 이를 바탕으로 상태를 정의하는 엔티티를 생성한다. `binId`는 유일한 값이라면 뭐가 되든 상관 없다. 여기서는 GUID를 사용한다 (line #6). `EntityId`가 바로 상태를 관장하는 값이다 (line #7).

https://gist.github.com/justinyoo/01426032d1ee6886796d9cb72e048dd9?file=02-create-bin.cs&highlights=6,7

위 코드에서 보면 `"Bin"`이라는 값은 이 상태를 명시적으로 저장하고 삭제하는 엔티티의 이름이다. 이 엔티티는 액터 모델의 구현을 따라간다. 엔티티의 상태와 더불어 어떤 식으로 엔티티의 상태를 변경시킬 수 있는지에 대한 액션도 정의되어 있다. 대략 아래와 같은 모습이다. 먼저 `IBin` 인터페이스를 통해 상태 변경과 관련한 액션을 정의한다. 여기서는 상태를 추가하고 리셋하는 역할만 한다.

https://gist.github.com/justinyoo/01426032d1ee6886796d9cb72e048dd9?file=03-ibin.cs

그리고 아래와 같이 `Bin` 클라스로 인터페이스 구현을 하는데, 상태 저장을 위한 `History`라는 속성을 정의한다 (line #5). 이 때 클라스 데코레이터로 직렬화 옵션을 `MemberSerialization.OptIn`라고 주면 (line #1) 명시적으로 `JsonProperty` 데코레이터를 선언한 속성에 대해서만 직렬화를 시도한다 (line #4). 마지막 줄에 보면 `Run()`라는 이름의 정적 메소드가 있는데 (line #23), 이를 통해 이벤트를 발생시켜 테이블 저장소에 상태를 저장하게 된다.

https://gist.github.com/justinyoo/01426032d1ee6886796d9cb72e048dd9?file=04-bin.cs&highlights=1,4,5,23


## Bin 생성 ##

그렇다면, 이제 이 엔티티에 어떻게 상태를 저장할까? `SignalEntityAsync()` 메소드를 통해 엔티티에 구현한 메소드를 호출한다 (line #8). 여기서는 비어있는 Bin 객체만 반환시킬 예정이므로 `null` 값을 보내게 된다. 이렇게 해서 비어있는 Bin이 하나 만들어졌다.

https://gist.github.com/justinyoo/01426032d1ee6886796d9cb72e048dd9?file=05-create-bin.cs&highlights=8

여기까지 해서 펑션 앱을 실제로 돌려보면 테이블 저장소에 아래와 같은 형태로 레코드가 생성된 것이 보일 것이다. `history` 필드에 비어있는 배열만 보이는가? 현재 `Bin`만 만들어졌기 때문이다.

![][image-02]


## 웹훅 요청 저장 ##

이제 웹훅 요청 히스토리를 하나씩 저장시켜 보자. 앞서 만든 엔드포인트와 거의 비슷하다. 다만 이번에는 요청 데이터를 넣어줘야 한다. 타임스탬프, 요청 메소드, 헤더, 쿼리스트링, 페이로드를 모두 캡쳐해서 저장한다 (line #10-14).

https://gist.github.com/justinyoo/01426032d1ee6886796d9cb72e048dd9?file=06-add-history.cs&highlights=10-14

그리고, 앞서와 같이 `bin`을 만들어 `SignalEntityAsync()` 메소드를 통해 히스토리를 추가한다 (line #11).

https://gist.github.com/justinyoo/01426032d1ee6886796d9cb72e048dd9?file=07-add-history.cs&highlights=11

이렇게 한 후 실제로 웹훅 요청을 날려보면 테이블 저장소의 데이터가 아래와 같이 변경된 것이 보인다. 실제로 요청 데이터가 저장된 것이다.

![][image-03]


## 웹훅 히스토리 조회 ##

그렇다면, 지금까지 저장해 놓은 웹훅 히스토리를 열어봐야 할 필요도 있을 것이다. 이 경우는 아래와 같이 먼저 Bin 레퍼런스를 생성한다 (line #7).

https://gist.github.com/justinyoo/01426032d1ee6886796d9cb72e048dd9?file=08-get-history.cs&highlights=7

그리고 난 뒤, `ReadEntityStateAsync()` 메소드를 통해 현재 상태를 가져와서 응답 객체로 반환한다 (line #9).

https://gist.github.com/justinyoo/01426032d1ee6886796d9cb72e048dd9?file=09-get-history.cs&highlights=9

이렇게 하면 아래와 같이 저장된 웹훅 요청 데이터에 대한 히스토리를 볼 수 있다.

![][image-04]


## 웹훅 히스토리 삭제 ##

이번에는 Bin 안에 저장된 모든 히스토리를 삭제해보자. 먼저 Bin 레퍼런스를 생성한다 (line #7).

https://gist.github.com/justinyoo/01426032d1ee6886796d9cb72e048dd9?file=10-reset-history.cs&highlights=7

그리고 난 후 이번에는 `SignalEntityAsync()` 메소드를 통해 `Bin` 액터의 `Reset()` 메소드를 호출한다 (line #9).

https://gist.github.com/justinyoo/01426032d1ee6886796d9cb72e048dd9?file=11-reset-history.cs&highlights=9

이후 테이블 저장소를 조회해 보면 모든 웹훅 히스토리가 사라진 것이 보인다.

![][image-05]


## Bin 삭제 ##

이제 마지막으로 이 Bin이 더이상 필요없을 때 삭제하는 엔드포인트를 만들어 보자. 먼저 Bin 레퍼런스를 생성한다 (line #7).

https://gist.github.com/justinyoo/01426032d1ee6886796d9cb72e048dd9?file=12-purge-bin.cs&highlights=7

그리고, `PurgeInstanceHistoryAsync()` 메소드를 통해 엔티티 자체를 테이블 저장소에서 삭제한다 (line #9).

https://gist.github.com/justinyoo/01426032d1ee6886796d9cb72e048dd9?file=13-purge-bin.cs&highlights=9

실제로 이 엔드포인트를 호출하면 아래와 같이 테이블 저장소에서 엔티티가 완전히 사라진 것을 확인할 수 있다.

![][image-06]

---

이렇게 RequestBin 앱을 구현해 봤다. 이 코드만 가지고서는 간단한 웹훅 확인 용도로 사용하는데에는 큰 문제가 없다. 여기에 더해 조금 더 UI를 붙여준다거나 하면 좀 더 완성도가 높은 앱이 될 것이다. 이 실습의 포인트는 [Durable Functions][az func durable]의 Stateful한 특성을 오케스트레이션 용도 뿐만 아니라 직접 액세스를 통해 다양한 활용도를 실험해 볼 수 있다는 데 있다. 앞으로 이 Durable Functions을 통해 좀 더 다양한 워크플로우 관리를 할 수 있기를 기대한다.


[image-01]: /2020/04/building-requestbin-with-durable-functions-01.png
[image-02]: /2020/04/building-requestbin-with-durable-functions-02.png
[image-03]: /2020/04/building-requestbin-with-durable-functions-03.png
[image-04]: /2020/04/building-requestbin-with-durable-functions-04.png
[image-05]: /2020/04/building-requestbin-with-durable-functions-05.png
[image-06]: /2020/04/building-requestbin-with-durable-functions-06.png

[post gitops]: /ko/2020/03/25/scheduling-posts-with-gitops-durable-functions-and-github-actions/

[gh sample]: https://github.com/devkimchi/RequestBin-Sample
[gh sample aci]: https://github.com/aliencube/RequestBin-on-ACI
[gh sample appsvc]: https://github.com/aliencube/RequestBin-on-Azure-App-Service

[az func]: https://docs.microsoft.com/ko-kr/azure/azure-functions/functions-overview?WT.mc_id=aliencubeorg-blog-juyoo
[az func durable]: https://docs.microsoft.com/ko-kr/azure/azure-functions/durable/durable-functions-overview?tabs=csharp&WT.mc_id=aliencubeorg-blog-juyoo
[az func durable entity]: https://docs.microsoft.com/ko-kr/azure/azure-functions/durable/durable-functions-entities?tabs=csharp&WT.mc_id=aliencubeorg-blog-juyoo

[requestbin]: https://github.com/Runscope/requestbin
[requestbin herokuapp]: https://requestbin.herokuapp.com/

[az aci]: https://docs.microsoft.com/ko-kr/azure/container-instances/container-instances-overview?WT.mc_id=aliencubeorg-blog-juyoo
[az appsvc]: https://docs.microsoft.com/ko-kr/azure/app-service/?WT.mc_id=aliencubeorg-blog-juyoo

[event sourcing pattern]: https://docs.microsoft.com/ko-kr/azure/architecture/patterns/event-sourcing?WT.mc_id=aliencubeorg-blog-juyoo
