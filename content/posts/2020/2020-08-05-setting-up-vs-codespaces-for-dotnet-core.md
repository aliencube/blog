---
title: ".NET Core 애플리케이션 개발을 위한 비주얼 스튜디오 Codespaces 환경 설정하기"
slug: setting-up-vs-codespaces-for-dotnet-core
description: "이 포스트에서는 비주얼 스튜디오 Codespaces 개발 환경을 설정하는 방법에 대해 알아봅니다."
date: "2020-08-05"
author: Justin-Yoo
tags:
- visual-studio-codespaces
- github-codespaces
- environment-setup
- dotnet-core
cover: /2020/08/setting-up-vs-codespaces-for-dotnet-core-00.png
fullscreen: true
---

> **업데이트**: 2020년 9월 4일 부터 [비주얼 스튜비오 Codespaces][vs cs]는 [깃헙 Codespaces][gh cs]로 통합된다. 기존 비주얼 스튜디오 Codespaces 사용자는 깃헙 Codespaces로 이전하라는 안내를 받게 되며, 2020년 11월 20일부터는 신규로 비주얼 스튜디오 Codespaces 인스턴스 생성을 할 수 없고, 2021년 2월 17일부터는 비주얼 스튜디오 Codespaces 서비스가 종료된다. 이 때 까지 이전하지 않은 인스턴스는 자동으로 삭제된다. 자세한 내용은 [공지 사항 블로그][vs cs consolidated]를 참조하기 바란다.

지난 2020년 4월부터 [비주얼 스튜디오 Codespaces 서비스가 정식으로 론칭됐다][vs cs]. 그 이후로 [깃헙 Codespaces][gh cs] 역시도 현재 프라이빗 프리뷰 기능으로 제공되고 있는 중인데, 이 둘은 사용법이 거의 동일하다. 이 둘의 차이점은 [이 포스트][devto post]를 통해 확인해 보도록 하자. 다만 여기서는 [닷넷 코어][dotnet core] 애플리케이션 개발을 위해 필요한 환경 설정에 대해 논의하기로 한다.

[비주얼 스튜디오 Codespaces (VS CS)][vs cs] 서비스는 온라인 기반 IDE로서 [애저 클라우드][az] 어딘가의 가상머신에서 돌아간다. 이 가상머신은 [VS CS][vs cs]를 시작할 때 자동으로 만들어지고 끝나면 자동으로 폐기되는 방식인데, 이 때 별다른 개발 환경 설정이 없다면 기본값으로 설정이 된다. 하지만, 이 기본값은 말 그대로 기본값이어서 예를 들어 닷넷 코어를 이용해 애플리케이션을 개발하려면 별도의 추가 환경 설정을 해 주어야 한다.

만약 이 때, 닷넷 코어 전용 개발 환경 설정이 미리 준비가 되어 있다면 어떨까? 이는 두 가지 방법으로 해결할 수 있다. 하나는 [프로젝트별 혹은 리포지토리별로 전체 개발 환경을 설정하는 방법][vs cs config]이 있고, 다른 하나는 [개발자 개인별로 개발 환경을 설정하는 방법][vs cs personal]이 있다. 프로젝트 팀별로 공통 환경 설정을 위해서는 전자를 선택하는 것이 좋고, 팀원 개개인의 선호하는 환경 설정을 위해서는 후자를 선택하면 좋다. 이 포스트는 우선 전자에 대한 얘기만 해 보도록 한다.


## 개발 환경 설정 ##

개발 환경은 도커 컨테이너 안에 정의할 수 있으므로 먼저 `Dockerfile`을 정의하면 좋다. 이미 [훌륭하게 정의된 환경][gh vs cs config]이 있기 때문에 이를 직접 사용해도 된다. 하지만, 여기서는 직접 만들어 보기로 하자. 우선 필요한 환경과 익스텐션들을 결정한다.

> 이 포스트에서 사용한 샘플 환경 설정은 [이 리포지토리][gh sample]에서 확인할 수 있다.


### `.devcontainer` 디렉토리 생성 ###

가장 먼저 해야 할 일은 리포지토리 안에 `.devcontainer` 디렉토리를 만드는 것이다. 이 안에 `Dockerfile`과 도커 컨테이너 실행시 자동으로 실행될 스크립트, 그리고 `devcontainer.json` 파일을 정의해야 한다.


### `Dockerfile` 설정 ###

닷넷 코어 SDK가 이미 빌드되어 있는 도커 이미지가 있으므로 이를 그대로 활용하기로 하자. 아래는 `Dockerfile`의 정의 문서이다. 우분투 18.04 LTS 이미지를 이용하기 때문에 [`3.1-bionic`][docker hub dotnet core] 이라는 태그를 사용했다 (line #1). 만약 다른 리눅스 환경을 사용하고 싶다면 다른 태그를 사용하면 된다.

https://gist.github.com/justinyoo/491cd606bd3b646f3fd5773d104e46f5?file=01-dockerfile.txt&highlights=1

이제 `setup.sh` 파일을 정의할 차례이다.


### `setup.sh` 설정 ###

아래와 같이 bash 스크립트를 정의한다. 아래 스크립트를 분석해 보자

1. 가장 먼저 할 일은 혹시 모를 기존의 패키지를 `apt-get` 명령어를 통해 업데이트한다. 만약 설치되지 않은 패키지라면 새롭게 설치한다 (line #1-9).
2. 그 다음에는 ASP.NET Core 관련 앱을 개발하다 보면 필요한 `node.js` 개발 환경을 위해 `nvm` 모듈을 설치한다 (line #12).
3. 세번째는 주석으로 막아두긴 했지만, 이 글을 쓰는 현재 도커 이미지에 기본적으로 설치된 파워셸 버전이 7.0.2 이기 때문에 혹시나 필요하다면 더욱 최신 버전으로 업데이트를 하고 싶을 경우 사용한다 (line #15).
4. 마지막으로는 bash 대신 zsh 를 사용하고 싶을 경우 좀 더 사용자 경험을 좋게 만들어주는 [oh my zsh][gh ohmyzsh] 셸 스크립트를 설치한다 (line #18-22).

https://gist.github.com/justinyoo/491cd606bd3b646f3fd5773d104e46f5?file=02-setup.sh&highlights=1-9,12,15,18-22

여기까지 해서 기본적인 도커 컨테이너 설정이 끝났다. 이제 [VS CS][vs cs]에 닷넷 코어 애플리케이션 개발을 위해 필요한 익스텐션들을 정의할 차례이다.


### 익스텐션 리스트 ###

`devcontainer.json` 파일은 [VS CS][vs cs]가 처음으로 실행될 때 참조하는 파일이다. 여기에 도커 컨테이너 정의 파일을 링크해 놓으면 위에 정의한 내용으로 개발 환경이 구성되는 것이다. 마찬가지로, 이 `devcontainer.json` 파일을 통해 필요한 익스텐션들을 한꺼번에 설치할 수 있다. 여기서는 아래 익스텐션들을 설치하기로 한다.

* [Azure Tools](https://marketplace.visualstudio.com/items?itemName=ms-vscode.vscode-node-azure-pack&WT.mc_id=aliencubeorg-blog-juyoo): 애저 사용을 위해 꼭 필요한 익스텐션 콜렉션
* [Bracket Pair Colorizer 2](https://marketplace.visualstudio.com/items?itemName=CoenraadS.bracket-pair-colorizer-2&WT.mc_id=aliencubeorg-blog-juyoo): 괄호 열고 닫을 때 헷갈리지 말라고 색깔 넣어주는 익스텐션
* [C#](https://marketplace.visualstudio.com/items?itemName=ms-dotnettools.csharp&WT.mc_id=aliencubeorg-blog-juyoo): C# 개발의 필수품
* [C# Extensions](https://marketplace.visualstudio.com/items?itemName=kreativ-software.csharpextensions&WT.mc_id=aliencubeorg-blog-juyoo): C# 익스텐션의 익스텐션
* [C# Sort Usings](https://marketplace.visualstudio.com/items?itemName=jongrant.csharpsortusings&WT.mc_id=aliencubeorg-blog-juyoo): C# 코딩시 최상단에 `using` 디렉티브를 순서대로 정렬시켜주는 익스텐션
* [C# XML Documentation Comments](https://marketplace.visualstudio.com/items?itemName=k--kato.docomment&WT.mc_id=aliencubeorg-blog-juyoo): C# 코딩을 하다보면 필요한 XML 형태의 문서 생성기 익스텐션
* [Docs Authoring Pack](https://marketplace.visualstudio.com/items?itemName=docsmsft.docs-authoring-pack&WT.mc_id=aliencubeorg-blog-juyoo): [docs.microsoft.com](https://docs.microsoft.com?WT.mc_id=aliencubeorg-blog-juyoo) 문서 생성/수정시 필요한 익스텐션 콜렉션
* [EditorConfig](https://marketplace.visualstudio.com/items?itemName=EditorConfig.EditorConfig&WT.mc_id=aliencubeorg-blog-juyoo): C# 코드 린팅을 위한 익스텐션
* [Git Graph](https://marketplace.visualstudio.com/items?itemName=mhutchie.git-graph&WT.mc_id=aliencubeorg-blog-juyoo): Git 그래프 익스텐션
* [Git History](https://marketplace.visualstudio.com/items?itemName=donjayamanne.githistory&WT.mc_id=aliencubeorg-blog-juyoo): Git 로그 익스텐션
* [GitHub Pull Requests and Issues](https://marketplace.visualstudio.com/items?itemName=github.vscode-pull-request-github&WT.mc_id=aliencubeorg-blog-juyoo): GitHub의 이슈와 PR을 보여주는 익스텐션
* [GitLens](https://marketplace.visualstudio.com/items?itemName=eamodio.gitlens&WT.mc_id=aliencubeorg-blog-juyoo): Git을 활용해서 소스코드를 관리할 때 꼭 필요한 Git Lens 익스텐션
* [IntelliCode](https://marketplace.visualstudio.com/items?itemName=visualstudioexptteam.vscodeintellicode&WT.mc_id=aliencubeorg-blog-juyoo): 인공지능 기반의 인텔리코드 익스텐션
* [Live Share](https://marketplace.visualstudio.com/items?itemName=ms-vsliveshare.vsliveshare&WT.mc_id=aliencubeorg-blog-juyoo): 라이브 코드 셰어 익스텐션
* [Markdown All in One](https://marketplace.visualstudio.com/items?itemName=yzhang.markdown-all-in-one&WT.mc_id=aliencubeorg-blog-juyoo): 마크다운 익스텐션
* [PowerShell](https://marketplace.visualstudio.com/items?itemName=ms-vscode.PowerShell&WT.mc_id=aliencubeorg-blog-juyoo): 파워셸 익스텐션
* [vscode-icons](https://marketplace.visualstudio.com/items?itemName=vscode-icons-team.vscode-icons&WT.mc_id=aliencubeorg-blog-juyoo): 아이콘 모음집 익스텐션

위 익스텐션들을 아래와 같은 형태로 추가하면 된다.

https://gist.github.com/justinyoo/491cd606bd3b646f3fd5773d104e46f5?file=03-devcontainer.json


### `devcontainer.json`의 다른 기능 ###

방금 익스텐션들을 `devcontainer.json` 파일 안에 정의했다. 그렇다면 이 파일은 그 이외에 어떤 다른 기능이 있을까? 이 `devcontainer.json` 파일은 [VS CS][vs cs]를 사용하는데 필요한 전반적인 환경을 설정하는 것이다. 대부분 기본값으로 놓고 사용해도 무리는 없지만, 그래도 세부적으로 조정을 해보고 싶다면 [이 문서][vs cs config]를 참조하기로 하고, 여기서는 필수적으로 고려해 볼만한 몇 가지만 다뤄본다.

* `dockerFile`: 앞서 정의한 `Dockerfile` 값을 여기에 지정한다.
* `forwardPorts`: [VS CS]에서 개발하는 애플리케이션이 특정 포트를 사용할 때, 이를 포워딩 시켜 외부로 공개한다. 예를 들어, ASP.NET Core 애플리케이션을 개발하다 보면 `5000`, `5001`번 포트를 기본값으로 사용하게 되는데, 이 값들을 여기에 배열로 넣어주면 된다. 마찬가지로 애저 펑션의 경우에는 기본적으로 `7071`번 포트를 사용하므로 함께 넣어주면 좋다.
* `settings`: [VS CS][vs cs]의 에디터 환경 설정을 위한 속성이다.


이렇게 모든 환경 설정이 끝났다.

## 깃헙 Codespaces 실행 ##

이제 이를 깃헙 리포지토리에 밀어넣고 아래와 같이 [깃헙 Codespaces][gh cs]를 실행시켜 보자. 만약 깃헙 Codespaces 프라이빗 프리뷰 프로그램에 들어가 있다면 아래와 같은 메뉴가 보일 것이다.

![][image-01]

클릭을 하고 들어가 보면 아래와 같이 [깃헙 Codespaces][gh cs] 환경에서 리포지토리의 모든 파일에 접근할 수 있다.

![][image-02]

또한, 파워셸 뿐만 아니라 zsh 셸 화면도 함께 볼 수 있다. 여기서 아래와 같이 명령어를 사용하면 곧바로 닷넷 코어 애플리케이션을 개발할 수 있다.

https://gist.github.com/justinyoo/491cd606bd3b646f3fd5773d104e46f5?file=04-dotnet-new.sh

위 명령어를 사용해서 콘솔 앱 프로젝트를 하나 열어보면 아래와 같다.

![][image-03]


## 비주얼 스튜디오 Codespaces 실행 ##

이번에는 동일한 리포지토리를 [VS CS][vs cs]에서 실행시켜 보자. 먼저 [https://online.visualstudio.com][vs cs vso]으로 이동한 후 로그인한다.

> [VS CS][vs cs]는 애저 구독이 있어야 하는데, 만약 없다면 [무료 구독 계정][az free]을 만들고 사용하면 된다.

로그인 후 만약 처음 Codespaces를 사용하는 상황이라면 새 과금 계획을 하나 생성해야 한다. [사용한 만큼 돈을 내는 구조][az vso pricing]이므로 사용후 더이상 필요없다고 판단한다면 곧바로 삭제를 하는 것이 좋다.

![][image-04]

과금 계획은 있지만 아직 Codespaces 인스턴스가 없을 경우에는 아래와 같이 새로 만들라는 화면이 보인다.

![][image-05]

아래와 같이 필요한 정보를 입력하고 인스턴스를 하나 생성하자. 여기서는 깃헙 리포지토리를 연결시켰는데, 이렇게 하면 리포지토리 전용 인스턴스가 되는 것이고, 리포지토리를 연결시키지 않는다면 범용으로 사용할 수 있는 인스턴스가 된다.

![][image-06]

이렇게 해서 생성된 Codespaces 인스턴스는 아래와 같이 보인다. 앞서 깃헙 Codespaces 에서 사용한 동일한 리포지토리를 사용했지만, 아직 닷넷 코어 애플리케이션 코드가 커밋되지 않았으므로 [VS CS][vs cs] 쪽에서는 그 부분에 대한 변경사항을 볼 수는 없다.

![][image-07]

---

지금까지 닷넷 코어 애플리케이션 개발을 위해 [VS CS][vs cs]에서 공통 환경 설정하는 부분에 대해 알아 보았다. 아무래도 이런 형태로 프로젝트별로 팀원들간 공통의 개발 환경을 공유하다보면 "내 컴퓨터에서는 돌아가는데?"와 같은 문제는 없어지지 않을까 생각한다.


[image-01]: /2020/08/setting-up-vs-codespaces-for-dotnet-core-01.png
[image-02]: /2020/08/setting-up-vs-codespaces-for-dotnet-core-02.png
[image-03]: /2020/08/setting-up-vs-codespaces-for-dotnet-core-03.png
[image-04]: /2020/08/setting-up-vs-codespaces-for-dotnet-core-04.png
[image-05]: /2020/08/setting-up-vs-codespaces-for-dotnet-core-05.png
[image-06]: /2020/08/setting-up-vs-codespaces-for-dotnet-core-06.png
[image-07]: /2020/08/setting-up-vs-codespaces-for-dotnet-core-07.png

[devto post]: https://dev.to/n3wt0n/visual-studio-github-codespaces-questions-answered-5ge7

[vs cs]: https://visualstudio.microsoft.com/ko/services/visual-studio-codespaces/?WT.mc_id=aliencubeorg-blog-juyoo
[vs cs blog]: https://devblogs.microsoft.com/visualstudio/introducing-visual-studio-codespaces/?WT.mc_id=aliencubeorg-blog-juyoo
[vs cs config]: https://docs.microsoft.com/ko-kr/visualstudio/codespaces/reference/configuring?WT.mc_id=aliencubeorg-blog-juyoo
[vs cs personal]: https://docs.microsoft.com/ko-kr/visualstudio/codespaces/reference/personalizing?WT.mc_id=aliencubeorg-blog-juyoo
[vs cs vso]: https://online.visualstudio.com/?WT.mc_id=aliencubeorg-blog-juyoo
[vs cs consolidated]: https://devblogs.microsoft.com/visualstudio/visual-studio-codespaces-is-consolidating-into-github-codespaces/?WT.mc_id=aliencubeorg-blog-juyoo

[gh cs]: https://github.com/features/codespaces/
[gh ohmyzsh]: https://github.com/ohmyzsh/ohmyzsh
[gh sample]: https://github.com/devkimchi/codespaces-dotnetcore
[gh vs cs config]: https://github.com/microsoft/vscode-dev-containers/tree/master/containers/dotnetcore

[dotnet core]: https://docs.microsoft.com/ko-kr/dotnet/?WT.mc_id=aliencubeorg-blog-juyoo

[az]: https://azure.microsoft.com/ko-kr/?WT.mc_id=aliencubeorg-blog-juyoo
[az free]: https://azure.microsoft.com/ko-kr/free/?WT.mc_id=aliencubeorg-blog-juyoo
[az vso pricing]: https://azure.microsoft.com/ko-kr/pricing/details/visual-studio-online/?WT.mc_id=aliencubeorg-blog-juyoo

[docker hub dotnet core]: https://hub.docker.com/_/microsoft-dotnet-core-sdk/
