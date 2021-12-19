---
title: "Oh My Azure Cloud Shell"
slug: oh-my-azure-cloud-shell
description: "이 포스트에서는 애저 클라우드 셸에서 oh-my-zsh, oh-my-posh 를 설치하고 사용하는 방법에 대해 다뤄봅니다."
date: "2021-12-22"
author: Justin-Yoo
tags:
- azure
- azure-cloud-shell
- oh-my-zsh
- oh-my-posh
cover: https://sa0blogs.blob.core.windows.net/aliencube/2021/12/oh-my-azure-cloud-shell-00.png
fullscreen: true
---

리눅스 또는 맥의 터미널 환경, 혹은 윈도우의 [WSL][wsl] 환경을 사용하다 보면 [oh-my-zsh][om zsh]을 들어봤거나 사용하고 있을 것이다. 또한 파워셸을 사용하고 있다면 [oh-my-posh][om posh]에 대해서도 들어봤거나 사용하고 있을 것이다. [애저][az]에서도 [애저 클라우드 셸][az csh] 환경을 제공하는데, 여기서는 bash 셸 환경과 파워셸 기본 환경을 제공한다. 따라서, 만약 oh-my-zsh 혹은 oh-my-posh 구성을 적용하고 싶다면 별도로 환경 구성을 해야 한다.

이 포스트에서는 이를 자동으로 구성해주는 bash 셸 스크립트와 파워셸 스크립트에 대해 알아보기로 한다.

> 이 포스트에 사용된 스크립트 소스 코드는 이 [깃헙 리포지토리][gh sample]에서 다운로드 받을 수 있다.


## oh-my-zsh 적용하기 ##

우선 [애저 클라우드 셸][az csh]에 [oh-my-zsh][om zsh] 환경을 구성해 보자. 먼저 bash 프롬프트인지 확인한다. 만약 bash 프롬프트가 아니라면 `bash` 명령어를 이용해 bash 프롬프트로 바꾼다.

1. 가장 먼저 oh-my-zsh 구성을 설치한다.

    https://gist.github.com/justinyoo/f5fa2a0e894a185828b76d64170de24e?file=01-install-oh-my-zsh.sh

2. 다음으로는 oh-my-zsh 용 플러그인을 설치한다. 다른 좋은 플러그인도 많지만, 여기서는 가장 널리 쓰이는 세가지 &ndash; [zsh-completions][om zsh plugins zsh-completions], [zsh-syntax-highlighting][om zsh plugins zsh-syntax-highlighting], [zsh-autosuggestions][om zsh plugins zsh-autosuggestions] &ndash; 정도만 설치한다. 추가적인 플러그인 설치가 더 필요하다면 아래와 같은 방법으로 설치하면 된다.

    https://gist.github.com/justinyoo/f5fa2a0e894a185828b76d64170de24e?file=02-install-oh-my-zsh-plugins.sh

3. 이어서 테마를 설치한다. 어떤 테마를 선택할지에 대해서는 취향에 따라 갈리겠지만, 여기서는 [Spaceship][om zsh themes spaceship] 테마 또는 [Powerlevel10k][om zsh themes p10k] 테마를 설치한다.

    https://gist.github.com/justinyoo/f5fa2a0e894a185828b76d64170de24e?file=03-install-oh-my-zsh-themes.sh

4. 만약 [Powerlevel10k][om zsh themes p10k] 테마를 설치했다면 아래 명령어를 통해 추가적인 환경 설정을 해 주어야 한다.

    https://gist.github.com/justinyoo/f5fa2a0e894a185828b76d64170de24e?file=04-configure-p10k.sh

5. [애저 클라우드 셸][az csh]은 bash 프롬프트를 기본값으로 사용한다. 그런데, 여기서는 `sudo` 명령어를 사용할 수 없기 때문에, zsh 프롬프트를 기본값이 되게끔 할 수 없다. 따라서, 아래와 같이 `.bashrc` 파일을 수정해서 zsh 프롬프트로 변경해 주어야 한다.

    https://gist.github.com/justinyoo/f5fa2a0e894a185828b76d64170de24e?file=05-update-bashrc.sh

6. 여기까지 설정이 끝났다면, [애저 클라우드 셸][az csh]을 종료하고 다시 시작한다. 혹은 `source .bashrc` 명령어를 실행시켜도 된다. 그러면 앞으로는 계속해서 oh-my-zsh 환경이 적용된 zsh 프롬프트를 사용할 수 있다.

7. 만약 이 모든 설정을 자동화하고 싶다면 아래 명령어를 실행시켜도 좋다.

    https://gist.github.com/justinyoo/f5fa2a0e894a185828b76d64170de24e?file=06-install-oh-my-azure-cloud-shell.sh

8. 만약 [Powerlevel10k][om zsh themes p10k] 테마를 설치하고 난 뒤, 현재 시간을 보여주고 싶다거나 감추고 싶다거나 하려면 아래 명령어를 실행시키면 된다.

    https://gist.github.com/justinyoo/f5fa2a0e894a185828b76d64170de24e?file=07-switch-p10k-clock.sh

지금까지 [애저 클라우드 셸][az csh]에 [oh-my-zsh][om zsh] 환경 구성을 하는 방법에 대해 알아보았다.


## oh-my-posh 적용하기 ##

이번에는 [애저 클라우드 셸][az csh]에 [oh-my-posh][om posh] 환경을 구성해 보자. 먼저 파워셸 프롬프트인지 확인한다. 만약 파워셸 프롬프트가 아니라면 `pwsh` 명령어를 이용해 파워셸 프롬프트로 바꾼다.

1. 가장 먼저 oh-my-posh 모듈을 설치한다.

    https://gist.github.com/justinyoo/f5fa2a0e894a185828b76d64170de24e?file=08-install-oh-my-posh.ps1

2. 이어서 테마를 선택한다. 앞서와 같이 어떤 테마를 선택할지에 대해서는 취향에 따라 갈리겠지만, 여기서는 [Spaceship][om posh themes spaceship] 테마 또는 [Powerlevel10k - Rainbow][om posh themes p10k] 테마를 선택한다.

    https://gist.github.com/justinyoo/f5fa2a0e894a185828b76d64170de24e?file=09-install-oh-my-posh-themes.ps1

3. 아래 명령어를 실행시켜 [터미널용 아이콘][om posh plugins terminal-icons] 모듈을 설치한다.

    https://gist.github.com/justinyoo/f5fa2a0e894a185828b76d64170de24e?file=10-install-oh-my-posh-plugins.ps1

4. `$PROFILE` 파일을 생성해서 파워셸 실행시 자동으로 모듈이 설치되게끔 한다.

    https://gist.github.com/justinyoo/f5fa2a0e894a185828b76d64170de24e?file=11-update-profile.ps1

5. 마지막으로 아래 명령어를 실행시켜 `$PROFILE` 파일에 저장된 내용을 반영한다.

    https://gist.github.com/justinyoo/f5fa2a0e894a185828b76d64170de24e?file=12-reload-profile.ps1

6. 만약 이 모든 설정을 자동화하고 싶다면 아래 명령어를 실행시켜도 좋다.

    https://gist.github.com/justinyoo/f5fa2a0e894a185828b76d64170de24e?file=13-install-oh-my-azure-cloud-shell.ps1

7. 만약 [Powerlevel10k - Rainbow][om posh themes p10k] 테마를 설치하고 난 뒤, 현재 시간을 보여주고 싶다거나 감추고 싶다거나 하려면 아래 명령어를 실행시키면 된다.

    https://gist.github.com/justinyoo/f5fa2a0e894a185828b76d64170de24e?file=14-switch-p10k-clock.ps1

지금까지 [애저 클라우드 셸][az csh]에 [oh-my-posh][om posh] 환경 구성을 하는 방법에 대해 알아보았다.

---

위와 같이 편의에 따라 [oh-my-zsh][om zsh] 혹은 [oh-my-posh][om posh], 혹은 둘 다 설치해서 사용한다면, 기존 로컬 개발환경의 터미널 경험을 그대로 [애저 클라우드 셸][az csh]에서도 사용할 수 있을 것이다.


[gh sample]: https://github.com/justinyoo/oh-my-azure-cloud-shell

[wsl]: https://docs.microsoft.com/ko-kr/windows/wsl/about?WT.mc_id=dotnet-52663-juyoo

[om zsh]: https://github.com/ohmyzsh/ohmyzsh
[om zsh plugins zsh-completions]: https://github.com/zsh-users/zsh-completions
[om zsh plugins zsh-syntax-highlighting]: https://github.com/zsh-users/zsh-syntax-highlighting
[om zsh plugins zsh-autosuggestions]: https://github.com/zsh-users/zsh-autosuggestions
[om zsh themes spaceship]: https://github.com/spaceship-prompt/spaceship-prompt
[om zsh themes p10k]: https://github.com/romkatv/powerlevel10k

[om posh]: https://ohmyposh.dev/
[om posh plugins terminal-icons]: https://github.com/devblackops/Terminal-Icons
[om posh themes spaceship]: https://ohmyposh.dev/docs/themes#spaceship
[om posh themes p10k]: https://ohmyposh.dev/docs/themes#powerlevel10k_rainbow

[az]: https://azure.microsoft.com/ko-kr/?WT.mc_id=dotnet-52663-juyoo
[az csh]: https://docs.microsoft.com/ko-kr/azure/cloud-shell/overview?WT.mc_id=dotnet-52663-juyoo
