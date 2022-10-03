---
title: "Windows 11에서 node-gyp 패키지 트러블슈팅하기"
slug: troubleshooting-node-gyp-package-on-windows11
description: "이 포스트에서는 윈도우 11 환경에서 node.js 앱 개발시 흔히 볼 수 있는 node-gyp 패키지 에러를 해결하는 방법에 대해 알아봅니다."
date: "2021-11-26"
author: Justin-Yoo
tags:
- windows11
- nodejs
- node-gyp
- troubleshooting
cover: /2021/11/troubleshooting-node-gyp-package-on-windows11-00.png
fullscreen: true
---


윈도우 환경에서 프론트엔드 애플리케이션을 개발하다 보면 요즘은 거의 당연하게도 [node.js][node js]를 기반으로 한다. [Windows Subsystem for Linux (WSL)][wsl]을 사용하면 리눅스 환경에서 node.js를 활용해서 앱을 개발할 수도 있겠지만, 만약 윈도우 네이티브 환경에서 node.js 앱을 개발하기 위해서는 몇가지 고려할 사항이 있다. 그 중 하나가 바로 [`node-gyp`][node gyp] 패키지 오류 해결과 관련된 것이다. 이 포스트에서는 윈도우 11 환경에서 흔히 발생하는 이 `node-gyp` 패키지 오류 해결과 관련해서 논의해 보기로 한다.


## `node-gyp` 패키지 의존성 문제 해결 ##

[`node-gyp`][node gyp] 패키지는 [npm][node npm]의 한 부분으로서 node.js를 설치하면 자동으로 설치된다. 따라서, 현재 설치되어 있는 npm 버전에 따라 어떤 식으로 문제를 해결해야 할 지를 결정할 수 있다. 이 포스트를 쓰는 현재 가장 최신의 node.js LTS 버전을 설치하면 아래와 같이 node.js `16.13.0` 버전과 npm `8.1.0` 버전이 보인다.

![node.js 및 npm 버전][image-01]

일반적인 경우, 깃헙에 어떤 프로젝트가 하나 있고 거기서 협업을 한다고 가정할 때, 해당 프로젝트를 로컬에서 빌드하기 위해 어떤 절차를 거치게 될까?

1. 리포지토리를 로컬에 클론한다
2. `npm install` 명령어를 실행시켜 필요한 node 모듈들을 다운로드 받는다.
3. `npm run <command>` 명령어를 실행시켜 해당 프로젝트에 준비된 애플리케이션을 실행시킨다.

완전히 똑같지는 않겠지만, 대략 위의 세 단계를 거치게 되는데, 이 때 경우에 따라 이 `node-gyp` 패키지를 사용할 수 있고, 그러다 보면 아래 정리할 에러들을 만나게 될 가능성이 높다.


### Python 설치 ###

npm 패키지를 보면 이 node-gyp 패키지에 직접적으로 의존성을 갖는 것들이 있다. 따라서, 운영체제에 따라 기본적으로 미리 설치되어 있어야만 하는 것들이 있다. [Python][python]이 그러한 의존성 중 하나에 해당한다.

`npm install` 명령어를 사용해서 필요한 node 모듈들을 다운로드 받는 도중에 아래와 같은 에러 메시지를 만나게 되는 경우가 있다.

> 참고로, node.js `14.13.0` 버전과 함께 기본으로 설치되는 `node-gyp` 버전은 `8.2.0`이다.

![Python 미설치 에러][image-02]

바로 Python이 설치되어 있지 않기 때문에 생기는 에러인데, 이는 간단히 Python을 설치하면 된다. 아래 [Python 웹사이트][python]를 방문해서 최신 버전을 다운로드 받아 설치한다. 이 글을 쓰는 현재 가장 최신의 Python 버전은 3.10.0이다.

![Python 웹사이트][image-03]

위의 웹사이트를 통해 Python을 설치한 후 제대로 설치가 되었는지 아래와 같이 확인한다.

![Python 버전 확인][image-04]


### Visual Studio 2019 Build Tools 설치 ###

이제 다시 `npm install` 명령어를 실행시켜 보자. 아래와 같은 에러 메시지가 나타난다.

![Visual Studio 미설치 에러][image-05]

이는 `node-gyp` 패키지가 참조하는 C++ 컴파일러를 설치하지 않았다는 의미이다. 따라서, 위 스크린샷의 하단에 나온 메시지와 같이 [비주얼 스튜디오][vs]를 설치해도 되고, 아니면 [단독으로 빌드 도구][vs 2019 build tools]를 설치해도 된다. 여기서는 비주얼 스튜디오 대신 [단독 빌드 도구][vs 2019 build tools]를 설치하도록 하자. 설치하면서 아래와 같이 "Desktop development with C++" 워크로드를 선택하면 된다.

![Visual Studio 2019 설치 화면][image-06]

이 글을 쓰는 현재 빌드 도구 다운로드 링크는 비주얼 스튜디오 2019 버전에 해당하므로 설치가 끝나고 나면 아래와 같은 화면을 볼 수 있다.

![Visual Studio 2019 Build Tools 설치 완료][image-07]

그리고 실제 설치 경로는 아래와 같다.

```text
C:\Program Files (x86)\Microsoft Visual Studio\2019\BuildTools
```

![Visual Studio 2019 Build Tools 설치 경로][image-08]

이렇게 한 후 다시 `npm install` 명령어를 실행시켜 보자. 그러면 에러 없이 모든 node 의존성 패키지 설치를 할 수 있다. 만약, 위의 에러가 다시 나타난다면, 아래 명령어를 실행시켜 현재 설치한 빌드 도구의 버전을 강제로 설정해 준다.

```powershell
npm config set msvs_version 2019
```

그러면 `npm install` 명령어를 에러 없이 실행시킬 수 있다.

> 참고로, 위의 [빌드 도구 다운로드 링크][vs 2019 build tools]는 시점에 따라 가장 최신의 빌드 도구를 다운로드 받을 수 있게 해 준다. 따라서, 이 글을 보는 시점에서 가장 최신 버전을 다운로드 받을 수 있으니, 꼭 비주얼 스튜디오 2019가 아닐 수도 있음을 기억해 두면 좋다.



### Visual Studio 2022 Build Tools 설치 ###

최근 비주얼 스튜디오 2022 버전이 출시됐으므로, 빌드 도구 역시도 2022 버전에서 사용해야 하는 상황이 생길텐데, [이 페이지][vs 2022 build tools]를 통해 가장 최신의 빌드 도구를 다운로드 받아서 설치한다. 앞서와 마찬가지로 설치하면서 아래와 같이 "Desktop development with C++" 워크로드를 선택하면 된다.

![Visual Studio 2022 설치 화면][image-09]

설치가 끝난 후에는 아래와 같은 경로에 빌드 도구가 설치된 것을 확인할 수 있다.

```text
C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools
```

![Visual Studio 2022 Build Tools 설치 경로][image-10]

그런데, 문제는 현재 설치된 node.js에 함께 설치된 `node-gyp` 버전이 비주얼 스튜디오 2022를 지원하지 않기 때문에, 아래 명령어를 통해 npm 버전을 업데이트해야 한다.

```powershell
npm install -g npm
```

업데이트가 끝나고 나면 npm 버전이 `8.1.0` 에서 `8.1.4`로 올라갔다.

![npm 버전 업데이트][image-11]

이와 더불어 `node-gyp` 패키지도 함께 `8.2.0`에서 `8.4.0`으로 버전업이 됐다.

![node-gyp 버전 업데이트][image-12]

따라서, 다시 `npm install` 명령어를 실행시키면 문제 없이 node 의존성 패키지들을 설치할 수 있게 된다. 만약 필요하다면 아래 명령어를 통해 강제로 버전을 설정해도 된다.

```powershell
npm config set msvs_version 2022
```


## Visual Studio 2022 설치 ##

앞서는 비주얼 스튜디오 전체 설치 없이 빌드 도구만 설치했을 경우를 다뤄봤다면, 이번에는 [비주얼 스튜디오 2022 버전][vs 2022 community]을 설치했을 경우도 다뤄보도록 하자. 비주얼 스튜디오 2022 부터는 이제 x64 모드로 작동하기 때문에 빌드 도구를 함께 설치할 경우 설치 경로가 달라진다.

```text
C:\Program Files\Microsoft Visual Studio\2022\Community
```

![Visual Studio 2022 설치 경로][image-13]

앞서 이미 `node-gyp` 버전이 `8.4.0`으로 올라갔기 때문에, `npm install` 명령어를 실행시켰을 경우 아무 문제 없이 모든 node 의존성 패키지들이 설치된다. 마찬가지로 필요하다면 아래 명령어를 통해 강제로 버전을 설정할 수도 있다.

```powershell
npm config set msvs_version 2022
```

> 참고로, 이 포스트에서는 커뮤니티 버전을 다운로드 받아 설치하지만 프로 버전, 엔터프라이즈 버전을 다운로드 받아도 괜찮다.


## Long Path 문제 해결 ##

이 이슈는 `node-gyp`와는 다르지만, node.js를 이용해 앱 개발을 하다보면 흔히 겪는 이슈이다. [윈도우의 고질적인 최대 260자 경로 이슈][win10 long path]를 해결할 수 있게 됐는데, 윈도우 11에서는 아래와 같이 로컬 그룹 정책 편집기를 이용한다.

![Local Group Policy Editor][image-14]

로컬 그룹 정책 편집기를 실행시켜 나오는 화면에서 "Local Computer Policy" ➡️ "Computer Configuration" ➡️ "Administrative Templates" ➡️ "System" ➡️ "File System" 메뉴로 이동한 후 "Enable Win32 long paths" 항목을 변경한다.
![Local Group Policy Editor][image-15]

이 값은 기본적으로 "Not Configured"로 설정되어 있기 때문에 아래와 같이 "Enabled"로 바꿔주면 된다.

![Local Group Policy Editor][image-16]

이렇게 하면 이제 더이상 경로 길이 문제로 고통 받을 일이 없어진다.

---

지금까지 윈도우 11 환경에서 node.js 앱을 개발할 때 흔히 겪는 `node-gyp` 이슈를 해결하는 방법에 대해 알아보았다. 앱 개발에 도움이 될 수 있기를 희망한다.


[image-01]: /2021/11/troubleshooting-node-gyp-package-on-windows11-01.png
[image-02]: /2021/11/troubleshooting-node-gyp-package-on-windows11-02.png
[image-03]: /2021/11/troubleshooting-node-gyp-package-on-windows11-03.png
[image-04]: /2021/11/troubleshooting-node-gyp-package-on-windows11-04.png
[image-05]: /2021/11/troubleshooting-node-gyp-package-on-windows11-05.png
[image-06]: /2021/11/troubleshooting-node-gyp-package-on-windows11-06.png
[image-07]: /2021/11/troubleshooting-node-gyp-package-on-windows11-07.png
[image-08]: /2021/11/troubleshooting-node-gyp-package-on-windows11-08.png
[image-09]: /2021/11/troubleshooting-node-gyp-package-on-windows11-09.png
[image-10]: /2021/11/troubleshooting-node-gyp-package-on-windows11-10.png
[image-11]: /2021/11/troubleshooting-node-gyp-package-on-windows11-11.png
[image-12]: /2021/11/troubleshooting-node-gyp-package-on-windows11-12.png
[image-13]: /2021/11/troubleshooting-node-gyp-package-on-windows11-13.png
[image-14]: /2021/11/troubleshooting-node-gyp-package-on-windows11-14.png
[image-15]: /2021/11/troubleshooting-node-gyp-package-on-windows11-15.png
[image-16]: /2021/11/troubleshooting-node-gyp-package-on-windows11-16.png


[node js]: https://nodejs.org/ko/
[node gyp]: https://github.com/nodejs/node-gyp
[node npm]: https://www.npmjs.com/

[wsl]: https://docs.microsoft.com/ko-kr/windows/wsl/about?WT.mc_id=dotnet-50103-juyoo&ocid=AID3035186

[python]: https://python.org/

[vs]: https://visualstudio.microsoft.com/?WT.mc_id=dotnet-50103-juyoo&ocid=AID3035186
[vs 2019 build tools]: https://visualstudio.microsoft.com/thank-you-downloading-visual-studio/?sku=BuildTools&WT.mc_id=dotnet-50103-juyoo&ocid=AID3035186
[vs 2022 build tools]: https://docs.microsoft.com/ko-kr/visualstudio/install/create-an-offline-installation-of-visual-studio?view=vs-2022&WT.mc_id=dotnet-50103-juyoo&ocid=AID3035186#step-1---download-the-visual-studio-bootstrapper
[vs 2022 community]: https://visualstudio.microsoft.com/thank-you-downloading-visual-studio/?sku=Community&rel=17&WT.mc_id=dotnet-50103-juyoo&ocid=AID3035186

[win10 long path]: https://docs.microsoft.com/ko-kr/windows/win32/fileio/maximum-file-path-limitation?tabs=powershell&WT.mc_id=dotnet-50103-juyoo&ocid=AID3035186
