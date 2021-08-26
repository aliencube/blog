---
title: "애저 펑션 빌드 파이프라인 통합 테스트"
slug: azure-functions-integration-testing
description: "이 포스트에서는 애저 펑션을 빌드 파이프라인 안에서 백그라운드로 실행시킨 후 통합 테스트를 실행하는 방법에 대해 알아봅니다."
date: "2021-08-26"
author: Justin-Yoo
tags:
- azure
- azure-functions
- integration-testing
- github-actions
cover: https://sa0blogs.blob.core.windows.net/aliencube/2021/08/azure-functions-integration-testing-00.png
fullscreen: true
---

예전에 [Mountebank 라는 도구를 이용해서 통합 테스트를 수행하는 방법][post 1]에 대해 논의해 본 적이 있고, 이를 활용해 [종단간 테스트를 구현][post 2]해 보기도 했다. 종단간 테스트를 구현할 때는 실제로 앱을 배포한 후 시나리오에 맞춰 테스트를 진행한다. 그런데, 앱을 배포하기 전에 빌드 파이프라인 안에서 종단간 테스트와 동일한 형태로, 앱을 배포하지 않고 파이프라인 안에서 돌리면서 테스트 시나리오를 시뮬레이션 할 수 있다면 어떨까? 이렇게 할 수 있다면, 굳이 배포할 때까지 기다렸다가 테스트를 진행할 필요 없이 우선적으로 파이프라인 안에서 테스트 성공/실패 여부를 판단할 수 있을 것이다.

이 포스트에서는 [애저 펑션][az fncapp] 앱을 배포한 후 진행하는 종단간 테스트 대신, 빌드 파이프라인에서 어떤 식으로 동일하게 시뮬레이션 할 수 있는지 알아보기로 한다.

> 이 포스트에서 사용한 샘플 앱은 [이 리포지토리][gh sample]에서 다운로드 받을 수 있다.


## 애저 펑션 앱 만들기 ##

일단 아주 간단한 애저 펑션 앱을 하나 만들어 보자. 여기서는 [애저 펑션 OpenAPI 확장 기능][az fncapp openapi]을 구현하는 앱을 만들어 보자. 대략 아래와 같은 엔드포인트가 하나 있다고 가정한다.

https://gist.github.com/justinyoo/d00aefe95ff7a5292121fc06b604cfad?file=01-default-http-trigger.cs

이렇게 작성한 앱을 실행시켜 `http://localhost:7071/api/openapi/v3.json` 엔드포인트를 호출하면 아래와 같은 응답이 나와야 한다.

https://gist.github.com/justinyoo/d00aefe95ff7a5292121fc06b604cfad?file=02-openapi-v3.json

그렇다면, 이렇게 응답 개체가 제대로 뽑히는지 확인할 필요가 있다. 이 샘플 앱은 아주 간단해서 `components.schemas.greeting`의 개체 구조 역시도 단순하지만, 만약 이 개체 구조가 복잡하다면 실제로 그 복잡한 구조가 제대로 표현이 되는지를 테스트를 통해 확신할 수 있어야 한다. 이럴 때 이 샘플 펑션 앱을 배포해서 테스트를 하는 것이 과연 합리적인 선택일까? 굳이 배포하지 않고도 빌드 파이프라인 안에서 확인할 수 있다면 그게 좀 더 합리적이지 않을까?

이제 실제로 빌드 파이프라인에서 이 펑션 앱의 출력 결과를 테스트해 보도록 하자.


## 애저 펑션 앱을 백그라운드로 실행하기 ##

로컬에서 테스트를 하려면 위 애저 펑션 앱을 로컬에서 실행시켜야 한다. 애저 펑션 앱을 로컬에서 실행시키는 명령어는 [애저 펑션 CLI][az fncapp cli]를 이용하면 아주 간단하다.

https://gist.github.com/justinyoo/d00aefe95ff7a5292121fc06b604cfad?file=03-func-start.sh

그런데, 이 명령어의 단점이라면 단점인 것이, 백그라운드 프로세스로 실행시킬 수 없다는 점이다. 자체적으로 백그라운드 프로세스로 실행시킬 수 없기 때문에 아래와 같이 셸 명령어의 기능을 활용해야 한다. Bash 명령어는 아래와 같이 명령어를 사용한다. 먼저 `func start &` 커맨드를 실행시키고 이어서 `bg` 커맨드를 실행시킨다.

https://gist.github.com/justinyoo/d00aefe95ff7a5292121fc06b604cfad?file=04-func-start-bg.sh

파워셸은 `Start-Process` 커맨들릿을 `-NoNewWindow` 스위치와 함께 사용하면 백그라운드 프로세스로 실행시킬 수 있다.

https://gist.github.com/justinyoo/d00aefe95ff7a5292121fc06b604cfad?file=05-func-start.ps1

이렇게 한 후 Bash 셸에서는 `curl http://localhost:7071/api/openapi/v3.json` 명령어를 실행시켜 결과를 확인할 수 있다. 마찬가지로 파워셸에서는 `Invoke-RestMethod -Method Get -Uri http://localhost:7071/api/openapi/v3.json`와 같은 형태로 API 엔드포인트를 호출하면 된다.


## 애저 펑션 통합 테스트 코드 작성하기 ##

그렇다면, 이렇게 백그라운드 프로세스로 애저 펑션 앱을 실행시킬 수 있게 됐으니, 통합테스트 코드는 일단 앱이 돌아간다는 가정 아래 진행하면 된다. 아래 테스트 코드를 보자. `http://localhost:7071/api/openapi/v3.json` 엔드포인트를 호출해서 결과값을 받아온 후 이를 비직렬화한다. 그리고 페이로드 개체 정의가 기대한 결과값인지 확인한다.

https://gist.github.com/justinyoo/d00aefe95ff7a5292121fc06b604cfad?file=06-default-http-trigger-tests.cs

여기서 핵심은 실제로 펑션 앱이 백그라운드로 돌고 있고 그 앱에 연결하기 위해 `http://localhost:7071/api/openapi/v3.json` 이라는 실제 엔드포인트를 사용했다는 점이다. 어디에서도 목킹을 했다든가 하지 않고 실제로 동작하는 엔드포인트를 사용한 점이 중요하다.

이렇게 코드를 작성한 후 테스트 명령어를 아래와 같이 실행시킨다.

https://gist.github.com/justinyoo/d00aefe95ff7a5292121fc06b604cfad?file=07-dotnet-test.sh

그러면 테스트 결과값을 바로 확인 가능하다.


## 깃헙 액션 워크플로우에 적용시키기 ##

이제 로컬에서 백그라운드로 애저 펑션 앱을 실행시킬 수 있고 이를 이용해 통합 테스트를 수행할 수 있었다면, 실제로 이를 CI/CD 파이프라인에 적용시켜야 한다. 아래 [깃헙 액션][gha] 워크플로우를 하나씩 살펴보자. 본문과 크게 관련이 없는 액션은 편의상 생략했다.


### 빌드 서버 운영체제 ###

상황에 따라 다르겠지만, 여기서는 윈도우, 맥, 리눅스 모두 대응한다고 가정하고 그에 맞게 설정한다. 아래와 같이 `matrix` 속성을 사용하면 편리하다.

https://gist.github.com/justinyoo/d00aefe95ff7a5292121fc06b604cfad?file=08-build-1.yaml&highlights=5-6

깃헙 액션용 빌드 서버에는 기본적으로 [애저 펑션 CLI][az fncapp cli]가 설치되어 있지 않다. 따라서, 이를 설치해 줘야 한다.

https://gist.github.com/justinyoo/d00aefe95ff7a5292121fc06b604cfad?file=08-build-2.yaml&highlights=8

그리고 .NET Core 3.1 SDK도 함께 설치한다.

https://gist.github.com/justinyoo/d00aefe95ff7a5292121fc06b604cfad?file=08-build-3.yaml

지금까지 필요한 SDK 설치가 끝났다면, 이제 실제로 앱을 백그라운드에서 실행시키고 테스트를 실행시킬 차례이다.

아래는 윈도우를 제외한 맥, 리눅스용 액션이다. 가장 먼저 펑션 앱을 백그라운드 프로세스로 실행시킨다. 이 때 위에서는 `func start`라고 했는데, 여기서는 `func @("start","--verbose","false")` 라고 했다. 불필요한 로그 메시지를 최소화하기 위해서 `--verbose false` 옵션을 추가한 것이다.

https://gist.github.com/justinyoo/d00aefe95ff7a5292121fc06b604cfad?file=08-build-4.yaml&highlights=2,10

이번에는 윈도우 환경에서 사용할 액션이다. 위와 가장 큰 차이점은 `func` 명령어를 직접 사용하는 대신 `$func` 라는 변수로 만들어서 사용했다는 점이다.

만약 `func`를 그대로 사용한다면 `This command cannot be run due to the error: %1 is not a valid Win32 application.`와 같은 에러메시지를 발생시킨다. 이는 `func` 명령어가 `func.ps1` 이라는 파워셸 스크립트를 가리키는데, 윈도우 환경에서는 백그라운드 프로세스를 실행시킬 때 문제가 있으므로, 이를 `func.cmd` 파일로 대체하는 작업을 해줘야 하기 때문이다.

https://gist.github.com/justinyoo/d00aefe95ff7a5292121fc06b604cfad?file=08-build-5.yaml&highlights=2,8,11

이렇게 깃헙 액션 워크플로우를 작성한 후 깃헙 리포지토리에 푸시를 해 보면 빌드 파이프라인이 아래와 같이 잘 작동한 것을 확인할 수 있다.

![윈도우용 빌드 파이프라인 실행 결과][image-01]

![비윈도우용 빌드 파이프라인 실행 결과][image-02]

---

지금까지 [애저 펑션][az fncapp] 앱을 로컬에서 [애저 펑션 CLI][az fncapp cli]를 이용해 [깃헙 액션][gha] 워크플로우 안에서 백그라운드 프로세스로 실행시켜 통합 테스트, 종단간 테스트를 진행하는 방법에 대해 알아보았다. 이를 이용하면 굳이 종단간 테스트를 위해 애저에 인스턴스를 만들고 배포하기 전에 파이프라인 상에서 미리 테스트를 통해 결과값을 예상할 수 있을 것이다.


[image-01]: https://sa0blogs.blob.core.windows.net/aliencube/2021/08/azure-functions-integration-testing-01.png
[image-02]: https://sa0blogs.blob.core.windows.net/aliencube/2021/08/azure-functions-integration-testing-02.png


[post 1]: /ko/2019/08/07/azure-functions-integration-testing-with-mountebank/
[post 2]: /ko/2019/08/14/azure-functions-sre-on-azure-devops-the-first-cut/

[gh sample]: https://github.com/devkimchi/azure-functions-integration-testing

[az fncapp]: https://docs.microsoft.com/ko-kr/azure/azure-functions/functions-overview?WT.mc_id=dotnet-39275-juyoo
[az fncapp openapi]: https://github.com/azure/azure-functions-openapi-extension
[az fncapp cli]: https://docs.microsoft.com/ko-kr/azure/azure-functions/functions-run-local?WT.mc_id=dotnet-39275-juyoo

[gha]: https://github.com/features/actions
