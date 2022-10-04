---
title: "프로젝트 Bicep 맛보기"
slug: bicep-sneak-peek
description: "이 포스트에서는 ARM 템플릿의 DSL로 최근 공개한 Bicep을 한 번 이용해서 ARM 템플릿을 빌드해 봅니다."
date: "2020-09-09"
author: Justin-Yoo
tags:
- bicep
- azure-resource-manager
- arm-template
- sneak-peek
cover: /2020/09/bicep-sneak-peek-00.png
fullscreen: true
---

이 시리즈를 통해 프로젝트 Bicep과 ARM 템플릿 검사도구, 그리고 이를 적용한 GitHub 액션을 다뤄보도록 하자.

* ***프로젝트 Bicep 맛보기***
* [GitHub 액션과 ARM 템플릿 검사도구를 이용한 Bicep 코드 품질 테스트][post 2]
* [애저 Bicep 되짚어보기][post 3]

애저에 리소스를 프로비저닝하기 위해서는 몇 가지 방법이 있다. 그 중 하나가 바로 [ARM 템플릿][az arm template]을 사용하는 것이다. 그런데, 이 ARM 템플릿은 약간의 러닝 커브가 있고, 템플릿의 구조 자체가 복잡해서 최초 작성시에 시간이 오래 걸리는 편이다. 이를 위해 다양한 베스트 프랙티스가 생겼지만, 여전히 최초 작성시 시간이 많이 걸리는 것은 사실이다.

이 문제를 해결하기 위해 마이크로소프트에서는 이 ARM 템플릿을 쉽게 작성하기 위한 DSL을 공개했다. 이 DSL의 이름은 [`Bicep`][gh bicep]인데, 이 글을 쓰는 시점에서는 현재 0.1 버전으로 극초기 프리뷰 단계이다. 이 포스트에서는 이 Bicep을 이용해서 ARM 템플릿을 빌드해 보고, 기존의 ARM 템플릿에 비해 어떤 장점이 있는지 알아보기로 한다.

> 이 포스트에 쓰인 Bicep 파일은 이곳 [GitHub 리포지토리][gh sample]에서 다운로드 받을 수 있다.


## ARM 템플릿 기본 형태 ##

ARM 템플릿의 기본 골격은 대략 아래와 같다.

https://gist.github.com/justinyoo/f3605100c5bfe7f7c7bd32f2d5fd1eb2?file=01-arm-template.json

여기서 `parameters`, `variables`, `resources`, `outputs` 속성은 거의 필수적으로 쓰이다시피 하기 때문에, Bicep 에서도 이를 우선적으로 지원한다.


## Bicep 파라미터 Parameters ##

Bicep 에서 파라미터는 아래와 같은 식으로 정의한다.

https://gist.github.com/justinyoo/f3605100c5bfe7f7c7bd32f2d5fd1eb2?file=02-parameter-1.bicep

가장 간단한 형태로는 아래와 같다. 파라미터의 설명이 들어있는 `description` 부분을 아예 주석으로 빼냈다 (line #1). 그리고, 기본값을 지정하는 방식은 마치 프로그래밍 언어에서 변수를 지정하듯 할 수도 있다 (line #7). 물론 기본값을 지정하지 않고 파라미터만 정의할 수도 있다 (line #8).

https://gist.github.com/justinyoo/f3605100c5bfe7f7c7bd32f2d5fd1eb2?file=03-parameter-2.bicep&highlights=1,7,8

하지만 `secure`, `allowed` 등과 같은 속성을 쓰려면 아래와 같이 개체 형식으로 써야 한다 (line #3-5).

https://gist.github.com/justinyoo/f3605100c5bfe7f7c7bd32f2d5fd1eb2?file=04-parameter-3.bicep&highlights=3-5


## Bicep 변수 Variables ##

파라미터가 ARM 템플릿의 외부로부터 입력 받는 값이라면, 변수는 템플릿의 내부에서 만드는 값이다. Bicep 에서 변수의 기본 문법은 아래와 같다. 문자열의 경우 인터폴레이션과 삼항연산식 등을 사용하면 간단하게 쓸 수도 있다 (line #2-3).

https://gist.github.com/justinyoo/f3605100c5bfe7f7c7bd32f2d5fd1eb2?file=05-variable.bicep&highlights=2-3

파라미터는 `=` 기호를 사용하지 않는 반면에, 변수에서는 `=` 기호를 사용한다. 그 이유는 대략 아래와 같다.

* 파라미터는 구조를 선언하는 것이고,
* 변수는 값을 할당하는 것이다.

> 만약 파라미터에 기본값을 할당하는 경우라면 위의 예제 코드와 마찬가지로 `=` 기호를 사용해야 한다.


## Bicep 리소스 Resources ##

실제 애저 리소스를 정의하는 부분의 영역은 아래와 같다.

https://gist.github.com/justinyoo/f3605100c5bfe7f7c7bd32f2d5fd1eb2?file=06-resource.bicep&highlights=1,19

몇 가지 눈에 띄는 점이 있다.

* 리소스 타입과 네임스페이스, 그리고 API 버전을 `네임스페이스/타입@버전`의 형태로 선언한다 (line #1). 개인적으로 API 버전을 선언할 때 [`providers()` 함수][az arm function providers]를 사용하는 것을 선호하지만, [권장하지 않는 방법이라고 한다][az arm validation providers]. 아래의 예제처럼 명시적으로 버전을 지정하는 것을 권장한다. 만약 리소스별 API 버전을 확인하고 싶다면 아래와 같이 파워셸 명령어를 쓰면 된다.

https://gist.github.com/justinyoo/f3605100c5bfe7f7c7bd32f2d5fd1eb2?file=07-get-provider.ps1

* ARM 템플릿에서 각 리소스별 의존성을 정의하기 위해서는 `dependsOn` 속성을 직접 지정해 줘야 하지만, Bicep 에서는 리소스 레퍼런스만 지정하면 알아서 자동으로 의존성을 해소한다. 예를 들어, 애저 저장소 계정을 선언하고 이후 가상 머신을 선언할 때 저장소 계정 리소스에 대한 레퍼런스를 적어주는 식이다 (line #19).

여기서 한 가지 재미있는 사실을 발견할 수 있다. ARM 템플릿은 `parameters` 섹션, `variables` 섹션, `resources` 섹션 등이 구분되어 있어 그 안에서만 파라미터, 변수, 리소스를 정의해야 하지만, Bicep 에서는 그 부분이 굉장히 자유롭다. 따라서, 파일 안의 어디에서든 `param`, `var`, `resource`를 선언하고 자유롭게 레퍼런스를 사용하면 된다. 그러면 컴파일을 하면서 자동으로 ARM 템플릿에 반영이 된다.


## Bicep 설치 및 사용 ##

앞서 언급했다시피 이 글을 쓰는 현재 Bicep은 0.1 버전이라서 아직 많은 부분이 개선되어야 한다. 따라서, 설치 방법이 조금 까다롭긴 하지만, [링크][az bicep install]의 설치 문서를 따라하면 어렵지 않게 설치할 수 있다. 설치가 끝난 후에는 아래 명령어를 통해 앞서 작성한 Bicep 파일을 컴파일 하면 된다.

https://gist.github.com/justinyoo/f3605100c5bfe7f7c7bd32f2d5fd1eb2?file=08-build-bicep.sh


## ARM 템플릿 파일 비교 ##

그렇다면, [직접 ARM 템플릿을 작성했을 때][az arm template manual]와 [Bicep을 이용해 생성한 ARM 템플릿][az arm template bicep]을 비교해 보도록 하자. 두 템플릿은 동일한 내용이다. 직접 작성했을 때에는 415 줄의 템플릿이 만들어졌지만, Bicep을 이용해 자동으로 생성했을 땐 306 줄로 줄어들었다. 또한 Bicep 원본 파일 자체는 288 줄이다. 아마도 변수를 좀 더 단순화시키는 쪽으로 리팩토링을 한다면 Bicep 파일의 크기는 더 줄어들 수 있을 것이다.


---

지금까지 ARM 템플릿의 DSL인 [Bicep][gh bicep]의 초기 프리뷰 버전에 대해 살펴보았다. Bicep을 다뤄본 첫 인상은 꽤 사용하기 쉬워졌고, 템플릿 작성 경험이 좋아지겠다는 사실이다. 이를 통해 이 글을 읽는 여러분들 역시 애저 ARM 템플릿의 개발 경험이 좀 더 나아지길 바란다. [다음 포스트][post 2]에서는 이렇게 만들어진 Bicep 파일을 [ARM 템플릿 검사도구(ARM TTK)][az arm ttk]를 이용해 ARM 템플릿 자체에 대한 유효성 검사를 진행해 보도록 한다.

[post 2]: /ko/2020/09/30/github-actions-and-arm-template-toolkit-to-test-bicep-codes/
[post 3]: /ko/2021/04/21/bicep-refreshed/

[gh sample]: https://github.com/devkimchi/LiveStream-VM-Setup-Sample/blob/main/bicep/azuredeploy.bicep
[gh bicep]: https://github.com/Azure/bicep

[az arm ttk]: https://docs.microsoft.com/ko-kr/azure/azure-resource-manager/templates/test-toolkit?WT.mc_id=aliencubeorg-blog-juyoo
[az arm template]: https://docs.microsoft.com/ko-kr/azure/azure-resource-manager/templates/overview?WT.mc_id=aliencubeorg-blog-juyoo
[az arm template manual]: https://github.com/devkimchi/LiveStream-VM-Setup-Sample/blob/main/azuredeploy.json
[az arm template bicep]: https://github.com/devkimchi/LiveStream-VM-Setup-Sample/blob/main/bicep/azuredeploy.json
[az arm function providers]: https://docs.microsoft.com/ko-kr/azure/azure-resource-manager/templates/template-functions-resource?WT.mc_id=aliencubeorg-blog-juyoo#providers
[az arm validation providers]: https://docs.microsoft.com/ko-kr/azure/azure-resource-manager/templates/test-cases?WT.mc_id=aliencubeorg-blog-juyoo#use-hardcoded-api-version

[az bicep install]: https://github.com/Azure/bicep/blob/master/docs/installing.md
