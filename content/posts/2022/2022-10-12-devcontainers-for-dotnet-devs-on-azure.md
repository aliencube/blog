---
title: "닷넷 개발자를 위한 개발 컨테이너 설정"
slug: devcontainers-for-dotnet-devs-on-azure
description: "이 포스트에서는 DevContainer에 대해서 개괄적으로 알아보며, 동시에 애저에서 .NET 앱을 개발하기 위해 유용할만한 것들을 DevContainer로 구성해보는 방법에 대해 알아봅니다."
date: "2022-10-12"
author: Justin-Yoo
tags:
- azure
- dotnet
- devcontainers
- gh-codespaces
cover: /2022/10/devcontainers-for-dotnet-devs-on-azure-00.png
fullscreen: true
---

컨테이너 기술은 정말로 여기저기서 쓰이고 있다. 단순히 개발 환경과 운영 환경을 동일하게 구성하는 것 뿐만 아니라 CI/CD 파이프라인을 위한 컨테이너, 테스트 자동화를 위한 컨테이너 등에서도 쓰이고 있다. 이렇게 컨테이너들을 많이 만들다보면 저마다의 방식대로 만들 것이고, 결국에는 컨테이너를 만드는 것 자체가 하나의 유지보수 활동이 된다.

유지보수 활동이 된다는 것은 유지보수가 편하게끔 만들어주는 뭔가가 있다면 굉장히 간편해지기도 하는데, 그렇다면 이런 컨테이너 환경을 구성하기 위한 표준 스펙이 있다면 어떨까? 표준에 맞춰 컨테이너 환경을 구성한다면 이를 이용해서 다양한 시나리오에서 빠르게 구성하고 활용할 수 있을 것이다. 이 표준을 가리켜 [개발 컨테이너(Development Container 또는 DevContainer)][container dev]라고 한다. 이 표준을 이용해서 개발 환경을 공통의 컨테이너 스펙에 맞춰 미리 정해둔다면 어떨까? 이 개발 컨테이너 스펙을 리포지토리에 저장해둔 후 [깃헙 코드스페이스][gh codespaces]와 같은 형식으로 구동시켜 사용할 수 있다면, 적어도 동일한 리포지토리를 사용하는 프로젝트 안에서는 동일한 개발 환경을 이용해 모두가 작업할 수 있을 것이다. 이 포스트에서는 이 개발 컨테이너 스펙을 이용해 애저 클라우드에서 .NET 앱을 개발하기 위한 환경 설정을 하는 방법에 대해 알아보기로 한다.

> 이 포스트에 사용한 컨테이너 템플릿은 [이곳][gh sample]에서 다운로드 받을 수 있다.


## 개발 컨테이너 구성 요소 ##

개발 컨테이너는 기본적으로 두 개의 파일이 필요하다.

* `devcontainer.json`: 개발 컨테이너 구성을 위한 각종 메타데이터 설정이 들어있다.
* `Dockerfile`: 컨테이너 이미지를 구성한다. `devcontainer.json`을 통해 호출한다.

이와는 별개로 개발 컨테이너의 생애주기에 따라 다양한 시점에서 셸 스크립트를 호출할 수 있다. `devcontainer.json` 파일의 `postCreateCommand` 속성을 이용하는 방법이 가장 널리 쓰인다. 이 포스트에서는 이 때 `post-create.sh` 파일을 실행시키는 것으로 가정한다.

> 이와는 별개로 `Dockerfile` 안에서 `RUN` 명령어를 통해 셸 스크립트를 직접 실행시킬 수 있게끔 할 수도 있다. 다만 이 때는 `root` 권한으로 스크립트를 실행시키는 것이고, `postCreateCommand`를 통해서는 `devcontainer.json`에서 지정한 사용자의 계정 권한으로 실행시키는 차이가 있다.


## `devcontainer.json` 구조 ##

`devcontainer.json` 파일 안에 개발 컨테이너 구성을 위한 다양한 메타데이터를 정의하고 있다. 여기서 다 다루지는 않고 필요한 것들만 짚어보자.

* `build`: 도커 컨테이너 빌드를 위한 내용이 들어간다. `Dockerfile`의 위치를 지정하고, 빌드에 필요한 파라미터를 전달한다.
* `forwardPorts`: 개발 컨테이너에서 앱 개발시 사용할 포트를 지정한다. 애저 펑션의 경우 `7071`번 포트, 애저 정적 웹 앱의 경우 `5000`, `5001` 포트를 주로 사용하므로 이를 지정하면 개발 컨테이너 안에서 디버깅하기 편리하다.
* `features`: 컨테이너 기본 이미지 안에 모든 것들 다 넣고 빌드할 수도 있지만, 이미 표준화 시켜놓은 각종 [기능][container dev features]들을 추가할 수도 있다. 이 기능들의 리스트는 [이곳][container dev features list]에서 확인할 수 있다. 예를 들어 공통 유틸리티, 애저 CLI, GitHub CLI, 테라폼 등와 같은 도구부터 시작해서 베이스 이미지에 포함되지 않은 node.js, 자바, 파이썬 등의 각종 프로그래밍 언어들도 추가시킬 수 있다.
* `customizations`: 개발 컨테이너 안에서 사용하려는 다양한 도구들을 입맛에 맞게 설정할 수 있다. 대표적으로 `vscode` 속성을 이용해서 확장 기능(`extensions`)이라든가 에디터 환경(`settings`) 등을 설정할 수 있다.
* `remoteUser`: 개발 컨테이너 안에서 사용할 계정이다. 설정하지 않는다면 `root` 계정의 권한으로 사용할 수 있다. 이 포스트에서는 `vscode`라는 이름의 일반 사용자 계정으로 지정한다.
* `postCreateCommand`: 개발 컨테이너가 만들어지고 난 후 일반 사용자 계정 권한으로 실행시킬 명령어를 정의한다. 이 안에서 스크립트를 실행시킬 수도 있다.

> 이 외에도 더 많은 내용들이 있으므로 좀 더 필요하다면 [이 페이지][container dev spec]를 확인한다.


## 개발 컨테이너 구성 순서 ##

개발 컨테이너가 만들어지는 순서는 아래와 같다.

1. 도커 컨테이너를 빌드한다. 만약 이 때 별도의 스크립트를 연동시켰다면 컨테이너 빌드 시점에서 함께 실행된다.
2. 도커 컨테이너를 빌드하는 과정에서 `devcontainer.json` 안의 `features`에 정의해 놓은 기능들을 추가한다.
3. 도커 컨테이너가 다 만들어졌으면 `devcontainer.json` 안의 `postCreateCommand` 속정에 정의한 명령어를 실행시킨다.
4. 만약 개인화를 위한 [dotfiles][container dev dotfiles]가 있다면, `postCreateCommand` 이후 dotfiles를 적용시킨다.
5. 개발 컨테이너를 실행시키는 과정에서 `devcontainer.json` 안에 정의해 놓은 확장 기능과 환경 설정 값을 적용시킨다.

그럼 이 과정을 통해 실제로 애저에서 .NET 애플리케이션을 개발하기 위한 개발 컨테이너 구성을 해 보기로 하자.


## 기본 컨테이너 이미지 선정 ##

[개발 컨테이너 이미지 리포지토리][container dev images]에는 미리 설정해 둔 다양한 [개발 컨테이너용 기본 이미지 리스트][container dev images list]가 있다. 이 중에서 우리는 .NET용 이미지를 선택한다. 다양한 버전의 도커 이미지를 사용할 수 있지만, 우분투 기반의 .NET 6를 활용한다고 가정하면 `6.0-jammy` (우분투 22.04) 또는 `6.0-focal` (우분투 20.04) 태그를 사용하면 된다. 여기서는 기본값으로 우분투 22.04 이미지를 사용한다고 가정하자.

```dockerfile
# [Choice] .NET version: 6.0-jammy, 6.0-focal
ARG VARIANT="6.0-jammy"
FROM mcr.microsoft.com/dotnet/sdk:${VARIANT}
```


## `devcontainer.json` 구성 ##

이번에는 모든 메타데이터가 들어있는 `devcontainer.json` 파일을 구성해 보자.


### `build` ###

아래와 같이 `Dockerfile`을 호출하고 `VARIANT` 파라미터에 `6.0-jammy` 값을 보낸다.

```jsonc
"build": {
  "dockerfile": "./Dockerfile",
  "context": ".",
  "args": {
    "VARIANT": "6.0-jammy"
    // Use this only if you need Razor support, until OmniSharp supports .NET 6 properly
    // "VARIANT": "6.0-focal"
  }
}
```

> 단, 이 포스트를 쓰는 현재 블레이저 앱과 같이 Razor 문법을 사용해야 할 경우라면 22.04(jammy) 버전 대신 20.04(focal) 버전을 선택해야 한다.


### `forwardPorts` ###

애저 펑션 앱을 개발하다 보면 로컬에서 `7071`번 포트를 사용한다든가 해서 앱 별로 특정 포트를 사용하는 경우가 있다. 이런 경우 해당 포트를 지정해 주면 웹브라우저에서 포트 포워딩이 가능해진다. 아래는 애저 펑션(`7071`), ASP.NET Core 앱(`5000`, `5001`), 애저 정적 웹 앱 CLI(`4280`)에서 사용하는 포트들을 지정한 것이다.

```jsonc
"forwardPorts": [
  // Azure Functions
  7071,
  // ASP.NET Core Web/API App, Blazor App
  5000, 5001,
  // Azure Static Web App CLI
  4280
]
```


### `features` ###

기본 도커 컨테이너 이미지에 도구들이나 언어들을 추가하고 싶다면 아래와 같이 `features`에 추가하면 된다.

* 공통 유틸리티 기능 추가: 이 기능을 통해 zsh, oh-my-zsh 등을 추가할 수 있다.

    ```jsonc
    "features": {
      // Install common utilities
      "ghcr.io/devcontainers/features/common-utils:1": {
        "installZsh": true,
        "installOhMyZsh": true,
        "upgradePackages": true,
        "username": "vscode",
        "uid": "1000",
        "gid": "1000"
      }
    }
    ```

* 애저 CLI 기능 추가: 이 기능을 통해 최신 버전의 애저 CLI를 설치할 수 있다.

    ```jsonc
    "features": {
      // Uncomment the below to install Azure CLI
      "ghcr.io/devcontainers/features/azure-cli:1": {
        "version": "latest"
      }
    }
    ```

* 깃헙 CLI 기능 추가: 이 기능을 통해 최신 버전의 깃헙 CLI를 설치할 수 있다.

    ```jsonc
    "features": {
      // Uncomment the below to install GitHub CLI
      "ghcr.io/devcontainers/features/github-cli:1": {
        "version": "latest"
      }
    }
    ```

* node.js 기능 추가: 이 기능을 통해 최신 버전의 node.js LTS 버전을 설치할 수 있다.

    ```jsonc
    "features": {
      // Uncomment the below to install node.js
      "ghcr.io/devcontainers/features/node:1": {
        "version": "lts",
        "nodeGypDependencies": true,
        "nvmInstallPath": "/usr/local/share/nvm"
      }
    }
    ```

만약 추가하고 싶은 기능이 아직 [이곳][container dev features list]에 보이지 않는다면, 이는 아래 언급할 `postCreateCommand` 명령어를 통해 실행시킬 `post-create.sh` 파일을 통해 추가하면 된다.

만약 위 기능들 중에서 반드시 순서를 지켜 추가해야 하는 것들이 있다면 아래와 같이 `overrideFeatureInstallOrder` 속성에 순서대로 추가시키면 된다. 여기서는 공통 유틸리티를 가장 먼저 설치하고 나머지는 아무래도 좋기 때문에 아래와 같이 설정한다.

```jsonc
"overrideFeatureInstallOrder": [
  "ghcr.io/devcontainers/features/common-utils"
]
```


### `customizations.vscode.extensions` ###

개발 컨테이너를 실행하고 나면 자동으로 추가시키고 싶은 확장 기능들이 있다. 이 확장 기능들은 `customizations.vscode.extensions` 섹션에서 정의하면 된다. 아래 지정해 둔 익스텐션들은 애저에서 .NET 애플리케이션을 개발할 때 필요한 것들이다. 취향에 따라 추가하거나 뺄 수 있다. 만약 더 추가하고 싶은 익스텐션이 있다면, [Visual Studio Code Marketplace][vscode marketplace]를 통해 검색한 후 익스텐션의 ID 값을 추가해 주면 된다. 예를 들어 [C# 확장 기능][vscode extensions csharp]의 익스텐션 ID 값은 `ms-dotnettools.csharp`이다.

```jsonc
"customizations": {
  "vscode": {
    "extensions": [
      // Recommended extensions - GitHub
      "cschleiden.vscode-github-actions",
      "GitHub.vscode-pull-request-github",

      // Recommended extensions - Azure
      "ms-azuretools.vscode-bicep",

      // Recommended extensions - Collaboration
      "eamodio.gitlens",
      "EditorConfig.EditorConfig",
      "MS-vsliveshare.vsliveshare-pack",
      "streetsidesoftware.code-spell-checker",

      // Recommended extensions - .NET
      "Fudge.auto-using",
      "jongrant.csharpsortusings",
      "kreativ-software.csharpextensions",

      // Recommended extensions - Power Platform
      "microsoft-IsvExpTools.powerplatform-vscode",

      // Recommended extensions - Markdown
      "bierner.github-markdown-preview",
      "DavidAnson.vscode-markdownlint",
      "docsmsft.docs-linting",
      "johnpapa.read-time",
      "yzhang.markdown-all-in-one",

      // Required extensions
      "GitHub.copilot",
      "ms-dotnettools.csharp",
      "ms-vscode.PowerShell",
      "ms-vscode.vscode-node-azure-pack",
      "VisualStudioExptTeam.vscodeintellicode"
    ]
  }
}
```


### `customizations.vscode.settings` ###

개발 컨테이너를 실행하고 나면 에디터의 환경을 수정하고 싶은 것들이 있다. 이 수정사항들은 `customizations.vscode.settings` 섹션에서 정의하면 된다. 아래 지정해 둔 설정사항들은 사람마다 취향을 따라가므로 적당히 추가하거나 뺄 수 있다. 만약 더 수정하고 싶은 설정사항이 있다면, [사용자 및 워크스페이스 설정][vscode settings] 페이지를 참고한다.

* 터미널 셸 실행 환경: 별다른 설정이 없다면 `bash` 셸을 기본으로 한다. 하지만, `zsh` 셸을 기본으로 설정하고 싶다면 아래와 같이 수정한다.

    ```jsonc
    "customizations": {
      "vscode": {
        "settings": {
          // Uncomment if you want to use zsh as the default shell
          "terminal.integrated.defaultProfile.linux": "zsh",
          "terminal.integrated.profiles.linux": {
            "zsh": {
              "path": "/usr/bin/zsh"
            }
          }
        }
      }
    }
    ```

* 터미널 폰트를 수정하고 싶다면 아래와 같이 지정한다. 이는 특히 oh-my-zsh 또는 oh-my-posh 를 적용시킬 때 유용하다.

    ```jsonc
    "customizations": {
      "vscode": {
        "settings": {
          // Uncomment if you want to use CaskaydiaCove Nerd Font as the default terminal font
          "terminal.integrated.fontFamily": "CaskaydiaCove Nerd Font"
        }
      }
    }
    ```

* 화면 오른쪽에 보이는 미니맵을 감추고 싶다면 아래와 같이 지정한다.

    ```jsonc
    "customizations": {
      "vscode": {
        "settings": {
          // Uncomment if you want to disable the minimap view
          "editor.minimap.enabled": false
        }
      }
    }
    ```

* 화면 왼쪽의 탐색기 설정을 바꾸고 싶다면 아래와 같이 설정한다. 아래 내용은, 파일을 확장자별로 정렬하되 관련 있는 것들을 묶어서 함께 보여주게끔 설정한 것이다.

    ```jsonc
    "customizations": {
      "vscode": {
        "settings": {
          // Recommended settings for the explorer pane
          "explorer.sortOrder": "type",
          "explorer.fileNesting.enabled": true,
          "explorer.fileNesting.patterns": {
            "*.bicep": "${capture}.json",
            "*.razor": "${capture}.razor.css",
            "*.js": "${capture}.js.map"
          }
        }
      }
    }
    ```


### `postCreateCommand` ###

개발 컨테이너가 만들어지고 난 후 추가적으로 뭔가를 해야 하는 경우가 있다. 예를 들자면 `features` 속성을 통해 추가할 수 없는 기능이라든가 하는 것들인데, 이 때 `postCreateCommand` 속성값으로 명령어를 지정해 두면 해당 명령어를 실행시킨다. 이 때, 여기에 셸 스크립트를 실행하도록 해 놓는다면, 추가적인 셸 스크립트를 실행시킬 수도 있다.

아래는 `post-create.sh` 셸 스크립트를 `bash` 명령어를 통해 실행시키는 것이다.

```jsonc
"postCreateCommand": "/bin/bash ./.devcontainer/post-create.sh > ~/post-create.log",
```

아래는 동일한 `post-create.sh` 셸 스크립트를 `zsh` 명령어를 통해 실행시키는 것이다.

```jsonc
"postCreateCommand": "/usr/bin/zsh ./.devcontainer/post-create.sh > ~/post-create.log",
```

그렇다면, 이 `post-create.sh` 파일 안에서는 어떤 일이 벌어지는지 살펴보자.


## `post-create.sh` ##

이번에는 개발 컨테이너가 만들어지고 난 후 실행시키는 셸 스크립트에 대해 알아보자.


### CaskaydiaCove Nerd Font ###

oh-my-zsh 또는 oh-my-posh를 사용하고 있다면 기본 폰트 대신 사용할 폰트를 추가하는 것이 좋다. 아래 폰트는 Cascadia Code 폰트에 다양한 아이콘을 추가한 폰트인 CaskaydiaCove Nerd Font를 추가하는 명령어이다.

```powershell
## CaskaydiaCove Nerd Font
# Uncomment the below to install the CaskaydiaCove Nerd Font
mkdir $HOME/.local
mkdir $HOME/.local/share
mkdir $HOME/.local/share/fonts
wget https://github.com/ryanoasis/nerd-fonts/releases/latest/download/CascadiaCode.zip
unzip CascadiaCode.zip -d $HOME/.local/share/fonts
rm CascadiaCode.zip
```


### 애저 CLI 확장 기능 ###

아래 명령어는 `devcontainer.json` 파일의 `features` 섹션에서 애저 CLI를 추가했을 경우 활성화 시키면 된다. 애저 CLI의 모든 확장 기능을 추가하는 명령어인데, 모든 확장 기능을 한번에 추가할 경우 네트워크 상황에 따라 30분에서 한시간 정도 걸리기 때문에 활성화시킬 때 주의해야 한다.

```powershell
## AZURE CLI EXTENSIONS ##
# Uncomment the below to install Azure CLI extensions
extensions=$(az extension list-available --query "[].name" | jq -c -r '.[]')
for extension in $extensions;
do
    az extension add --name $extension
done
```


### 애저 Bicep CLI ###

만약 애저 Bicep 파일을 자주 다룬다면 아래 명령어를 실행시켜 Bicep CLI를 설치하는 것이 좋다.

```powershell
## AZURE BICEP CLI ##
# Uncomment the below to install Azure Bicep CLI
az bicep install
```


### 애저 펑션 핵심 도구 ###

로컬에서 애저 펑션 앱을 개발해야 한다면 아래 명령어를 실행시켜 설치한다. npm 명령어를 사용하기 때문에 사전에 `devcontainer.json` 파일의 `features` 섹션을 통해 node.js 기능을 설치했어야 한다.

```powershell
## AZURE FUNCTIONS CORE TOOLS ##
# Uncomment the below to install Azure Functions Core Tools
npm i -g azure-functions-core-tools@4 --unsafe-perm true
```


### 애저 정적 웹 앱 CLI ###

애저 정적 웹 앱을 통해 블레이저 앱을 배포한다든가 하려면 아래 섹션을 활성화시키는 것이 좋다. 마찬가지로 npm 명령어를 사용하기 때문에 사전에 `devcontainer.json` 파일의 `features` 섹션을 통해 node.js 기능을 설치했어야 한다.

```powershell
## AZURE STATIC WEB APPS CLI ##
# Uncomment the below to install Azure Static Web Apps CLI
npm install -g @azure/static-web-apps-cli
```


### 애저 개발 CLI ###

애저 개발 CLI를 사용하고 싶다면 아래 섹션을 활성화시킨다. 이 때 반드시 사전에 `devcontainer.json` 파일의 `features` 섹션을 통해 애저 CLI와 깃헙 CLI 기능을 설치했어야 한다.

```powershell
## AZURE DEV CLI ##
# Uncomment the below to install Azure Dev CLI
curl -fsSL https://aka.ms/install-azd.sh | bash
```


### oh-my-zsh 플러그인 및 테마 설치 ###

oh-my-zsh 기능을 좀 더 잘 사용하고 싶다면 아래 섹션을 활성화하여 플러그인과 테마를 설치하면 좋다. 여기서는 powerlevel10k 테마를 설치하는 것으로 가정한다.

```powershell
## OH-MY-ZSH PLUGINS & THEMES (POWERLEVEL10K) ##
# Uncomment the below to install oh-my-zsh plugins and themes (powerlevel10k) without dotfiles integration
git clone https://github.com/zsh-users/zsh-completions.git $HOME/.oh-my-zsh/custom/plugins/zsh-completions
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git $HOME/.oh-my-zsh/custom/plugins/zsh-syntax-highlighting
git clone https://github.com/zsh-users/zsh-autosuggestions.git $HOME/.oh-my-zsh/custom/plugins/zsh-autosuggestions

git clone https://github.com/romkatv/powerlevel10k.git $HOME/.oh-my-zsh/custom/themes/powerlevel10k --depth=1
ln -s $HOME/.oh-my-zsh/custom/themes/powerlevel10k/powerlevel10k.zsh-theme $HOME/.oh-my-zsh/custom/themes/powerlevel10k.zsh-theme
```


### oh-my-zsh powerlevel10k 테마 설정 ###

powerlevel10k 테마는 별도의 설정 파일을 갖고 있다. 아래 섹션을 활성화시켜 사전에 구성된 설정 파일을 복사한다. 만약 이와 같은 구성이 자신의 `dotfiles` 리포지토리에 설정되어 있다면 여기서 활성화시킬 필요는 없다.

```powershell
## OH-MY-ZSH - POWERLEVEL10K SETTINGS ##
# Uncomment the below to update the oh-my-zsh settings without dotfiles integration
curl https://raw.githubusercontent.com/justinyoo/devcontainers-dotnet/main/oh-my-zsh/.p10k-with-clock.zsh > $HOME/.p10k-with-clock.zsh
curl https://raw.githubusercontent.com/justinyoo/devcontainers-dotnet/main/oh-my-zsh/.p10k-without-clock.zsh > $HOME/.p10k-without-clock.zsh
curl https://raw.githubusercontent.com/justinyoo/devcontainers-dotnet/main/oh-my-zsh/switch-p10k-clock.sh > $HOME/switch-p10k-clock.sh
chmod +x ~/switch-p10k-clock.sh

cp $HOME/.p10k-with-clock.zsh $HOME/.p10k.zsh
cp $HOME/.zshrc $HOME/.zshrc.bak

echo "$(cat $HOME/.zshrc)" | awk '{gsub(/ZSH_THEME=\"codespaces\"/, "ZSH_THEME=\"powerlevel10k\"")}1' > $HOME/.zshrc.replaced && mv $HOME/.zshrc.replaced $HOME/.zshrc
echo "$(cat $HOME/.zshrc)" | awk '{gsub(/plugins=\(git\)/, "plugins=(git zsh-completions zsh-syntax-highlighting zsh-autosuggestions)")}1' > $HOME/.zshrc.replaced && mv $HOME/.zshrc.replaced $HOME/.zshrc
echo "
# To customize prompt, run 'p10k configure' or edit ~/.p10k.zsh.
[[ ! -f ~/.p10k.zsh ]] || source ~/.p10k.zsh
" >> $HOME/.zshrc
```


### oh-my-posh 설치 ###

파워셸도 많이 사용하는 편이라면 oh-my-posh 라는 도구를 설정하면 oh-my-zsh 같은 느낌을 줄 수 있다. 아래 섹션을 활성화시켜 설치한다.

```powershell
## OH-MY-POSH ##
# Uncomment the below to install oh-my-posh
sudo wget https://github.com/JanDeDobbeleer/oh-my-posh/releases/latest/download/posh-linux-amd64 -O /usr/local/bin/oh-my-posh
sudo chmod +x /usr/local/bin/oh-my-posh
```


### oh-my-posh 환경 구성 ###

oh-my-posh는 별도의 powerlevel10k 구성파일이 있으므로 아래 섹션을 활성화시켜 구성 파일을 복사한다. 만약 이와 같은 구성이 자신의 `dotfiles` 리포지토리에 설정되어 있다면 여기서 활성화시킬 필요는 없다.

```powershell
## OH-MY-POSH - POWERLEVEL10K SETTINGS ##
# Uncomment the below to update the oh-my-posh settings without dotfiles integration
curl https://raw.githubusercontent.com/justinyoo/devcontainers-dotnet/main/oh-my-posh/p10k-with-clock.omp.json > $HOME/p10k-with-clock.omp.json
curl https://raw.githubusercontent.com/justinyoo/devcontainers-dotnet/main/oh-my-posh/p10k-without-clock.omp.json > $HOME/p10k-without-clock.omp.json
curl https://raw.githubusercontent.com/justinyoo/devcontainers-dotnet/main/oh-my-posh/switch-p10k-clock.ps1 > $HOME/switch-p10k-clock.ps1

mkdir $HOME/.config/powershell
curl https://raw.githubusercontent.com/justinyoo/devcontainers-dotnet/main/oh-my-posh/Microsoft.PowerShell_profile.ps1 > $HOME/.config/powershell/Microsoft.PowerShell_profile.ps1

cp $HOME/p10k-with-clock.omp.json $HOME/p10k.omp.json
```

위와 같이 설정을 한 후 깃헙 코드스페이스를 실행시켜보면 아래와 같이 터미널 환경이 적용된 것을 확인할 수 있다. 아래 스크린샷에서 왼쪽은 oh-my-zsh를 적용한 zsh 터미널이고, 오른쪽은 oh-my-posh를 적용한 파워셸 터미널이다.

![GH Codespaces][image-01]

---

지금까지 [깃헙 코드스페이스][gh codespaces]를 실행시키기 위해 [개발 컨테이너][container dev] 환경을 애저와 .NET 개발 환경에 맞춰 구성하는 방법에 대해 알아보았다. 여기서 다룬 내용은 개발 컨테이너의 방대한 내용 중 거의 일부에 불과한지라 자신의 개발 환경 혹은 팀의 개발 환경을 구성하기 위해서는 좀 더 연구가 필요할 것이다. 다만, 이 포스트에서 제공하는 개발 컨테이너 템플릿은 최소한의 방향성만을 제시한다고 알고 있으면 좋다.


## 개발 컨테이너에 대해 더 알고 싶다면? ##

만약 개발 컨테이너에 대해 좀 더 알고 싶다면 아래 튜토리얼을 참조해 보면 좋다.

* [초급자 시리즈: 개발 컨테이너][container dev learn] (영문)
* [개발 컨테이너 구성하기][container dev create] (영문)
* [깃헙 코드스페이스][gh codespaces] (영문)
* [깃헙 코드스페이스 구성하기][container dev codespaces] (영문)


[image-01]: /2022/10/devcontainers-for-dotnet-devs-on-azure-01.png


[gh sample]: https://github.com/justinyoo/devcontainers-dotnet
[gh codespaces]: https://docs.github.com/codespaces/overview

[container dev]: https://containers.dev/
[container dev spec]: https://github.com/devcontainers/spec
[container dev images]: https://github.com/devcontainers/images
[container dev images list]: https://github.com/devcontainers/images/tree/main/src
[container dev features]: https://github.com/devcontainers/features
[container dev features list]: https://github.com/devcontainers/features/tree/main/src

[container dev create]: https://code.visualstudio.com/docs/remote/create-dev-container?WT.mc_id=dotnet-78728-juyoo
[container dev learn]: https://learn.microsoft.com/ko-kr/shows/beginners-series-to-dev-containers/?WT.mc_id=dotnet-78728-juyoo
[container dev codespaces]: https://docs.github.com/codespaces/setting-up-your-project-for-codespaces/setting-up-your-project-for-codespaces?langId=dotnet
[container dev dotfiles]: https://docs.github.com/codespaces/customizing-your-codespace/personalizing-github-codespaces-for-your-account

[vscode marketplace]: https://marketplace.visualstudio.com/VSCode?WT.mc_id=dotnet-78728-juyoo
[vscode settings]: https://code.visualstudio.com/docs/getstarted/settings?WT.mc_id=dotnet-78728-juyoo
[vscode extensions csharp]: https://marketplace.visualstudio.com/items?itemName=ms-dotnettools.csharp&WT.mc_id=dotnet-78728-juyoo
