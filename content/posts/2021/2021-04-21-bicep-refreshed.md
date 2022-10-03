---
title: "애저 Bicep 되짚어보기"
slug: bicep-refreshed
description: "이 포스트에서는 애저 Bicep 프로젝트 관련 새롭게 추가된 내용에 대해 알아봅니다."
date: "2021-04-21"
author: Justin-Yoo
tags:
- azure
- bicep
- arm
- update
cover: /2021/04/bicep-refreshed-00.png
fullscreen: true
---

[지난 포스트][post 1]에서는 극초기의 [Bicep 프로젝트][bicep]에 대해 소개를 했었다. 포스팅 당시에는 `0.1.x` 버전이었으나, 지금은 `0.3.x` 버전으로 실제 프로덕션에서 사용할 수 있을 만큼 안정화가 되었으며 추가적인 기능도 많이 생겼다. 이 포스트에서는 신규로 추가된 bicep의 기능에 대해 알아보기로 한다.

* [프로젝트 Bicep 맛보기][post 1]
* [GitHub 액션과 ARM 템플릿 검사도구를 이용한 Bicep 코드 품질 테스트][post 2]
* ***애저 Bicep 되짚어보기***


## 애저 CLI 통합 ##

Bicep CLI는 단독으로도 사용할 수 있지만, 만약 [애저 CLI][az cli]를 사용하고 있다면 [v2.20.0 버전][az cli release v2.20.0] 이후부터는 bicep CLI와 통합되었다. 따라서 아래와 같은 명령어가 가능하다.

https://gist.github.com/justinyoo/e27d3ddc8d868a1c16293f8286b3ff67?file=01-bicep-build.sh

> **참고**: `v0.2.x` 버전까지는 여러 bicep 파일을 한번에 컴파일하는 것이 가능했지만, `v0.3.x` 버전부터는 한 번에 bicep 파일 하나만 처리할 수 있다. 따라서, 한 번에 여러 파일을 빌드하고 싶다면 별도의 작업을 해 주어야 한다. 아래는 파워셸을 예로 들어 스크립트를 작성해 봤다.
> 
> https://gist.github.com/justinyoo/e27d3ddc8d868a1c16293f8286b3ff67?file=02-bicep-build.ps1

이렇게 애저 CLI와 통합된 덕에 bicep 파일을 그대로 애저 CLI를 통해 실행시킬 수 있다. 따라서 아래와 같은 명령어가 가능하다.

https://gist.github.com/justinyoo/e27d3ddc8d868a1c16293f8286b3ff67?file=03-bicep-deploy.sh


## Bicep 디컴파일 ##

`v0.1.x` 버전에서는 오로지 `.bicep` 파일을 `.json` 파일로 컴파일하는 기능만 가능했으니, [`v0.2.59` 버전][bicep release v0.2.59] 이후로는 기존의 ARM 템플릿을 .bicep 파일로 디컴파일 하는 것도 가능하다. 이는 기존에 사용하던 ARM 템플릿을 유지보수하기 위해 bicep 파일로 변환시키는 데 굉장히 유용하다. 아래와 같은 명령어를 사용하면 된다.

https://gist.github.com/justinyoo/e27d3ddc8d868a1c16293f8286b3ff67?file=04-bicep-decompile.sh

> **NOTE**: 이 디컴파일 기능은 ARM 템플릿 안에 `copy` 속성이 있을 경우 아직 제대로 처리하지 못한다. 좀 더 버전업이 되면 가능해질 것으로 예상한다.


## 파라미터 데코레이터 ##

파라미터 작성이 `v0.1.x` 버전에 비해 훨씬 간결해졌다. 파라미터의 속성을 데코레이터로 지정할 수 있게끔 개선되었다. 아래 코드를 보자. 저장소 어카운트의 SKU 값은 정해져 있으므로, 아래와 같이 `@allowed` 데코레이터를 사용하면 훨씬 더 가독성이 높아진다.

https://gist.github.com/justinyoo/e27d3ddc8d868a1c16293f8286b3ff67?file=05-bicep-decorators.bicep


## 조건부 리소스 선언 ##

삼항연산을 사용해서 조건에 따라 속성값을 다르게 부여할 수도 있지만, 리소스 자체를 조건부로 선언할 수도 있다. 아래와 같은 형태로 구성해 보자. 리소스 그룹의 프로비저닝 지역이 **한국 중부 (Korea Central)**일 때에만 애저 앱 서비스 인스턴스를 프로비저닝하게끔 조건을 걸 수 있다.

https://gist.github.com/justinyoo/e27d3ddc8d868a1c16293f8286b3ff67?file=06-bicep-conditions.bicep


## 순환문 선언 ##

ARM 템플릿에서는 `copy` 속성과 `copyIndex()` 함수를 이용해서 동일한 리소스를 반복해서 프로비저닝했다면, bicep 에서는 이제 `for...in` 루프를 사용해서 선언할 수 있다. 아래 코드를 보자. 애저 앱 서비스 인스턴스를 배열 파라미터를 통해 한 번에 프로비저닝하게끔 했다.

https://gist.github.com/justinyoo/e27d3ddc8d868a1c16293f8286b3ff67?file=07-bicep-loops-1.bicep

아래 리소스 선언과 같이 배열과 인덱스를 동시에 활용할 수도 있다.

https://gist.github.com/justinyoo/e27d3ddc8d868a1c16293f8286b3ff67?file=08-bicep-loops-2.bicep

또한 배열과 상관없이 `range()` 함수를 사용해서 루프를 돌려 리소스를 선언할 수도 있다.

https://gist.github.com/justinyoo/e27d3ddc8d868a1c16293f8286b3ff67?file=09-bicep-loops-3.bicep

여기서 한 가지 명심해야 하는 부분이 있다. `for...in` 루프를 사용하면 결과적으로 리소스 배열이 생기게 되므로 이 `for...in` 루프의 바로 바깥에 배열 표시(`[...]`)를 해야 한다. 그러면 나머지는 bicep이 알아서 처리한다.


## 모듈화 ##

ARM 템플릿에서는 연결 템플릿 기능을 이용해 리소스별 모듈화를 시도했다면, bicep에서는 `module` 이라는 키워드를 이용해서 모듈화를 선언한다. 아래와 같이 애저 펑션 인스턴스를 선언하기 위해서는 최소 저장소 어카운트, 컨섬션 플랜, 애저 펑션의 세 가지 리소스가 필요한데, 이를 각각 모듈로 구성해서 하나로 통합할 수 있다. 개별 모듈은 물론 독립적으로 사용할 수 있어야 한다.


### 저장소 어카운트 모듈 ###

https://gist.github.com/justinyoo/e27d3ddc8d868a1c16293f8286b3ff67?file=10-bicep-storage-account.bicep


### 컨섬션 플랜 모듈 ###

https://gist.github.com/justinyoo/e27d3ddc8d868a1c16293f8286b3ff67?file=11-bicep-consumption-plan.bicep


### 애저 펑션 모듈 ###

https://gist.github.com/justinyoo/e27d3ddc8d868a1c16293f8286b3ff67?file=12-bicep-function-app.bicep


### 모듈 오케스트레이션 ###

위와 같이 개별 리소스마다 모듈화를 해 놓았다면, 이를 하나로 통합해서 활용할 수 있는 오케스트레이션 파일을 아래와 같이 생성하면 된다.

https://gist.github.com/justinyoo/e27d3ddc8d868a1c16293f8286b3ff67?file=13-bicep-azuredeploy.bicep

> **NOTE**: 이 글을 쓰는 시점에서는 ARM 템플릿과 달리 아직 모듈 참조를 위해 외부 URL을 사용할 수 없다.

---

지금까지 애저 bicep의 새로운 기능에 대해 알아 보았다. 계속해서 추가 기능이 들어가고 있는 만큼 굉장히 빠르게 업데이트 되고 있으니, 꾸준히 사용해 보면 좋을 것이다.


[post 1]: /ko/2020/09/09/bicep-sneak-peek/
[post 2]: /ko/2020/09/30/github-actions-and-arm-template-toolkit-to-test-bicep-codes/

[bicep]: https://github.com/Azure/bicep/
[bicep release v0.2.59]: https://github.com/Azure/bicep/releases/tag/v0.2.59

[az cli]: https://docs.microsoft.com/ko-kr/cli/azure/what-is-azure-cli?WT.mc_id=devops-25381-juyoo
[az cli release v2.20.0]: https://docs.microsoft.com/ko-kr/cli/azure/release-notes-azure-cli?WT.mc_id=devops-25381-juyoo#march-02-2021

[az arm template linked]: https://docs.microsoft.com/ko-kr/azure/azure-resource-manager/templates/linked-templates?WT.mc_id=devops-25381-juyoo
