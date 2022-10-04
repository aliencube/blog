---
title: "GitHub 액션과 ARM 템플릿 검사도구를 이용한 Bicep 코드 품질 테스트"
slug: github-actions-and-arm-template-toolkit-to-test-bicep-codes
description: "이 포스트에서는 ARM 템플릿 검사도구를 이용해 Bicep이 자동 생성한 ARM 템플릿을 테스트 해 보고, 이를 GitHub Actions 워크플로우와 통합해 봅니다."
date: "2020-09-30"
author: Justin-Yoo
tags:
- arm-ttk
- bicep
- github-actions
- testing
cover: /2020/09/github-actions-and-arm-template-toolkit-to-test-bicep-codes-00.png
fullscreen: true
---

이 시리즈를 통해 프로젝트 Bicep과 ARM 템플릿 검사도구, 그리고 이를 적용한 GitHub 액션을 다뤄보도록 하자.

* [프로젝트 Bicep 맛보기][post 1]
* ***GitHub 액션과 ARM 템플릿 검사도구를 이용한 Bicep 코드 품질 테스트***
* [애저 Bicep 되짚어보기][post 3]

ARM 템플릿을 만들고 난 이후에는 이 템플릿이 제대로 만들어 진 것인지 혹은 실제로 작동을 할 것인지 등에 대해 검증을 해야 한다. 이 ARM 템플릿 유효성 검사를 위해 예전에 [Pester][pester]를 이용한 ARM 템플릿 테스팅 시리즈를 영문으로 포스팅했던 적이 있다([#1][post pester 1], [#2][post pester 2]). 하지만 이 방법은 실제 리소스를 프로비저닝하지는 않지만, 일단은 애저에 로그인을 해야 하는 단점이 있다. 만약 로그인을 하지 않고도 내가 작성한 ARM 템플릿의 유효성 검사를 할 수 있다면 어떨까?

[지난 포스트][post 1]에서는 [Bicep 프로젝트][gh bicep]와 이를 이용해 ARM 템플릿을 손쉽게 생성하는 방법에 대해 알아 보았다. 이번 포스트에서는 이 Bicep으로 자동 생성한 ARM 템플릿을 [ARM 템플릿 검사도구 (ARM Template Toolkit; ARM-TTK)][az arm ttk]를 이용해 유효성 검사를 진행하고, 이를 [GitHub 액션][gh actions]을 이용해 CI/CD 파이프라인에 적용시켜 보도록 한다.

> 이 포스트에 쓰인 Bicep 파일은 이곳 [GitHub 리포지토리][gh sample]에서 다운로드 받을 수 있다.


## ARM 템플릿 검사 도구 (ARM TTK) ##

[ARM 템플릿 검사 도구(ARM TTK)][az arm ttk]는 템플릿을 작성할 때 좀 더 가독성이 높고 쉽게 작성할 수 있게끔 일관적이고 표준적인 코딩 프랙티스를 제공한다. 여기에는 아래의 내용을 포함한다:

* 작성자의 의도를 검수한다. 사용하지 않는 파라미터나 변수는 제거하도록 요구한다.
* ARM 템플릿 작성시 의도하지 않은 보안 사고를 방지한다
* 자주 쓰이지만 하드 코딩해야 하는 값들을 환경 함수로 제공해서 오탈자를 방지한다.

[ARM TTK는 파워셸로 만들어졌고][gh arm ttk], 현재 버전 0.3이다. 파워셸 역시 크로스 플랫폼을 지원하므로 윈도우, 맥, 리눅스 상관 없이 ARM TTK를 실행시킬 수 있다.

![][image-01]

> ARM TTK를 사용하기 위해서는 공식 문서에 있는 다운로드 링크 보다는 가급적이면 GitHub 리포지토리를 클론 받아서 사용하는 것이 좀 더 최신의 코드를 사용할 수 있어서 좋다. 별도 컴파일이 필요 없기 때문이기도 하고, 아직 공식 릴리즈가 나오지 않았기 때문이기도 하다.


## ARM TTK 사용 ##

Bicep으로 작성한 ARM 템플릿을 테스트하기 위해서는 아래와 같이 하면 된다. 가장 먼저 bicep CLI를 이용해 ARM 템플릿을 빌드한다.

https://gist.github.com/justinyoo/0acf46566623e346553b5dabe5c1fe5b?file=01-bicep-build.ps1

다음으로는 파워셸 명령어를 실행시킨다. 여기서 주의할 점이 하나 있는데, 특정 디렉토리 안의 모든 파일을 검사하고 싶다면, 적어도 `azuredeploy.json` 혹은 `maintemplate.json` 파일이 하나 들어 있어야 한다. 그렇지 않으면 ARM TTK는 에러를 던지며 실행되지 않는다.

https://gist.github.com/justinyoo/0acf46566623e346553b5dabe5c1fe5b?file=02-run-arm-ttk.ps1

위와 같이 명령어를 실행시키고 난 후의 결과는 아래와 같다. 특정 리소스에 지정한 API 버전이 오래됐다는 에러메시지이다. 예를 들어 애저 저장소 계정에 지정한 API 버전은 `2017-10-01`인데, 2년 이상 오래된 버전이니 최신 버전인 `2019-06-01`을 사용하라고 권장한다.

![][image-02]

위와 같은 에러 메시지들을 정리하고 다시 실행 시켜보면 아래와 같이 모든 테스트를 통과했다.

![][image-03]


## Bicep CLI와 ARM TTK를 GitHub 액션 CI/CD 파이프라인에 적용하기 ##

이제 bicep CLI와 ARM TTK를 CI/CD 파이프라인에 적용시켜 볼 차례이다. 코드를 수정하고 리포지토리에 푸시하면 당연히 이 변경에 대한 검증을 CI/CD 파이프라인 상에서 진행시켜야 한다. 아래 두 GitHub 액션은 바로 이를 위한 것이다.

* [Bicep Build][gh actions bicep]
* [ARM TTK][gh actions arm ttk]

위 두 액션을 CI/CD 파이프라인에 적용시켜 보면 아래와 같다.

https://gist.github.com/justinyoo/0acf46566623e346553b5dabe5c1fe5b?file=03-github-actions.yaml&highlights=15-18,20-24,26-30

1. 먼저 Bicep Build 액션을 통해 `.bicep` 파일을 ARM 템플릿으로 변환시킨다 (line #15-18).
2. 그 다음으로 ARM TTK 액션을 이용해서 앞서 변환한 ARM 템플릿을 검증한다 (line #20-24).
3. 마지막 액션에서는 검증 결과를 출력한다. ARM TTK 액션은 검증 결과를 JSON 개체 문자열 형식으로 반환해 주기 때문에 이를 활용해서 리포트를 작성할 수도 있다 (line #26-30).

이 워크플로우를 따르면 Bicep으로 작성한 파일을 ARM TTK를 통해 아주 편하게 검증하고 이 모든 워크플로우를 통과한 ARM 템플릿만을 리소스 프로비저닝에 사용할 수 있게 된다.

단, 여기서 명심해야 할 것이 하나 더 있다. ARM TTK를 이용해 검증하기 전 ARM 템플릿 역시도 제대로 작동하는 것이었다. 즉, ARM TTK는 ARM 템플릿 자체가 제대로 작동을 하는가 아닌가를 검증한다기 보다는 ARM 템플릿 자체의 코드 품질에 대한 검증을 진행한다는 것이다. 다른 말로 ARM 템플릿 린팅(Linting)을 진행한다고 보는 것이 더 정확하다. 따라서, 이 템플릿이 제대로 동작하는지 아닌지의 여부는 여전히 앞서 언급한 Pester와 같은 도구와 함께 Azure CLI 등으로 실제 리소스에 대한 유효성 검증을 해야 한다.

---

지금까지 [Bicep CLI][gh actions bicep]와 [ARM TTK][gh actions arm ttk]를 GitHub 액션에 적용시켜 CI/CD 파이프라인을 구성해 보았다. 이 두 가지를 이용해서 앞으로는 ARM 템플릿을 작성할 때 코드 품질이 일관적으로, 그리고 가독성이 높아지는 형태로 작성해 보도록 하자.


[image-01]: /2020/09/github-actions-and-arm-template-toolkit-to-test-bicep-codes-01.png
[image-02]: /2020/09/github-actions-and-arm-template-toolkit-to-test-bicep-codes-02.png
[image-03]: /2020/09/github-actions-and-arm-template-toolkit-to-test-bicep-codes-03.png

[post 1]: /ko/2020/09/09/bicep-sneak-peek/
[post 3]: /ko/2021/04/21/bicep-refreshed/
[post pester 1]: https://devkimchi.com/2018/01/22/testing-arm-templates-with-pester/
[post pester 2]: https://devkimchi.com/2018/07/12/testing-arm-templates-with-pester-2/

[gh sample]: https://github.com/devkimchi/LiveStream-VM-Setup-Sample/blob/main/bicep/azuredeploy.bicep
[gh bicep]: https://github.com/Azure/bicep
[gh arm ttk]: https://github.com/Azure/arm-ttk
[gh actions]: https://github.com/features/actions
[gh actions bicep]: https://github.com/marketplace/actions/bicep-build
[gh actions arm ttk]: https://github.com/marketplace/actions/arm-template-toolkit-arm-ttk

[pester]: https://github.com/pester/Pester

[az arm ttk]: https://docs.microsoft.com/ko-kr/azure/azure-resource-manager/templates/test-toolkit?WT.mc_id=aliencubeorg-blog-juyoo
[az arm template]: https://docs.microsoft.com/ko-kr/azure/azure-resource-manager/templates/overview?WT.mc_id=aliencubeorg-blog-juyoo
[az arm template manual]: https://github.com/devkimchi/LiveStream-VM-Setup-Sample/blob/main/azuredeploy.json
[az arm template bicep]: https://github.com/devkimchi/LiveStream-VM-Setup-Sample/blob/main/bicep/azuredeploy.json
[az arm function providers]: https://docs.microsoft.com/ko-kr/azure/azure-resource-manager/templates/template-functions-resource?WT.mc_id=aliencubeorg-blog-juyoo#providers
[az arm validation providers]: https://docs.microsoft.com/ko-kr/azure/azure-resource-manager/templates/test-cases?WT.mc_id=aliencubeorg-blog-juyoo#use-hardcoded-api-version

[az bicep install]: https://github.com/Azure/bicep/blob/master/docs/installing.md
