---
title: "맥OS 에서 불필요한 .NET SDK를 수동으로 제거하기"
slug: removing-dotnet-sdks-from-macos-manually
description: "이 포스트에서는 맥OS에서 새 버전의 .NET SDK를 새로 설치한다거나 할 때 필요없어진 예전 버전을 수동으로 제거하는 방법에 대해 다뤄봅니다."
date: "2021-11-24"
author: Justin-Yoo
tags:
- dotnet
- dotnet-sdk
- macos
- installation
cover: /2021/11/removing-dotnet-sdks-from-macos-manually-00.png
fullscreen: true
---

다양한 운영체제를 지원하는 .NET Core 및 .NET 5, .NET 6 SDK를 설치하다보면 어느 순간 예전 버전을 삭제해야 한다거나 혹은 가장 최근에 설치한 프리뷰 SDK 버전을 삭제해야 한다거나 할 때가 있다. 이 때 만약 맥OS를 사용하는 경우에는 여러 가지 방법을 이용해 삭제할 수 있지만, 여기서는 수동으로 삭제할 때 알아둬야 할 내용을 중심으로 다뤄보도록 한다.


## .NET SDK 및 런타임 설치 경로 ##

맥OS에서 `dotnet --list-sdks` 라는 명령어를 실행시켜 보면 아래와 같이 설치된 모든 SDK의 버전과 경로를 알 수 있다.

![설치된 .NET SDK 버전들][image-01]

이번엔 `dotnet --list-runtimes` 라는 명령어를 실행시켜 닷넷 런타임의 버전과 설치 경로를 알아보자.

![설치된 .NET 런타임 버전들][image-02]

뭔가 차이점을 느낄 수 있는가? SDK와 런타임의 버전이 미묘하게 살짝 다르다. 따라서, SDK와 런타임을 삭제하기 위해서는 조금 조심스럽게 접근할 필요가 있다.

> .NET SDK와 런타임 설치 버전과 관련해서 좀 더 자세한 내용을 보고 싶다면, [.NET이 설치되어 있는지 확인하는 방법][dotnet check] 페이지를 참고한다.


## .NET SDK 및 런타임 삭제 ##

우선, [.NET 제거 도구][dotnet uninstall tool]를 사용해서 자동으로 삭제할 수 있는 방법이 있다. 하지만 이 도구로 완벽하게 지울 수 없을 수도 있기 때문에 항상 수동으로 각각의 디렉토리를 찾아서 하나하나 삭제하는 것이 좋다.

다만, 앞서 언급한 바와 같이 SDK 버전과 런타임의 버전이 살짝 다르기 때문에 이 부분만 확실하게 알면 좋다. 예를 들어 이 글을 쓰는 현재 가장 최근에 릴리즈한 .NET 6의 경우 SDK 버전은 `6.0.100` 이지만 런타임 버전은 `6.0.0` 이다. 같은 기간에 가장 최신의 .NET 5 SDK 버전은 `5.0.403` 이지만 런타임 버전은 `5.0.12`이다.

이런 식으로 SDK 버전과 런타임의 버전이 다르기 때문에 이를 맞춰서 수동으로 삭제해야 한다. 우선은 가장 먼저 [.NET SDK 및 런타임을 제거하는 방법][dotnet uninstall manual] 페이지를 참고하면 가장 좋다. 다만, 이 포스트에서는 좀 더 깊이 들어가 보도록 하자.


### .NET SDK 수동 삭제 ###

여기서는 .NET 6 SDK를 수동 삭제한다고 가정하자. 이 글을 쓰는 현재 가장 최신 버전의 SDK는 `6.0.100`이다. SDK와 관련된 디렉토리는 아래의 둘 뿐이므로 아래와 같이 bash 명령어를 실행시킨다.

https://gist.github.com/justinyoo/68c11fa00480fa15d38f7288815c6ba5?file=01-remove-sdk.sh

이렇게 하면 SDK가 삭제된다.


### .NET 런타임 수동 삭제 ###

이번에는 런타임도 수동으로 삭제를 해 보도록 하자. 아래와 같이 bash 명령어를 실행시킨다. 런타임이 정의되어 있는 디렉토리는 버전별로 상당히 다양하므로 꼭 확인해 보도록 하자. 아래의 디렉토리가 모두 쓰일 수도 있고 아닐 수도 있다.

https://gist.github.com/justinyoo/68c11fa00480fa15d38f7288815c6ba5?file=02-remove-runtime.sh

이렇게 하면 원하는 버전의 .NET 런타임이 모두 삭제된다.


## 파워셸을 이용한 수동 삭제 ##

위와 같이 수동으로 개별 디렉토리마다 삭제할 수도 있고, 스크립트를 이용하면 좀 더 간결하게 작업을 수행할 수도 있다. 이번에는 [파워셸][pwsh]을 이용해서 삭제해 보도록 하자. 파워셸 역시도 크로스 플랫폼을 지원하기 때문에 맥OS에서도 사용할 수 있다. 아래는 샘플 스크립트이다.

https://gist.github.com/justinyoo/68c11fa00480fa15d38f7288815c6ba5?file=03-remove-both.ps1

위와 같이 하면 깔끔하게 모든 .NET SDK와 런타임 버전을 한번에 삭제 가능하다.


## .NET SDK 및 런타임 수동 삭제후 재설치 ##

위와 같이 수동으로 삭제를 한 경우에는 종종 기존의 버전을 인식하지 못하는 경우가 있다. 이럴 땐 기존에 설치했던 SDK를 다시 한 번 더 설치해 주면 좋다. 예를 들어, .NET 6 SDK와 런타임을 삭제한 후 기존의 .NET Core 2.1, 3.1과 .NET 5가 작동하지 않는다면, 해당 버전의 가장 최신 SDK를 재설치하는 식이다. 필요한 SDK를 다시 설치하려면 [다운로드][dotnet download] 페이지에서 SDK를 다운로드 받아 설치하면 된다.

![.NET SDK 다운로드 페이지][image-03]

---

지금까지 맥OS에서 .NET SDK 및 런타임을 수동으로 삭제하는 방법에 대해 알아보았다. 별 문제가 없다면 굳이 삭제하지 않아도 상관없지만, 만일의 경우 수동으로 삭제해야 하는 상황이 생긴다면 위에 언급한 방법을 이용해 보면 좋을 것이다.


[image-01]: /2021/11/removing-dotnet-sdks-from-macos-manually-01.png
[image-02]: /2021/11/removing-dotnet-sdks-from-macos-manually-02.png
[image-03]: /2021/11/removing-dotnet-sdks-from-macos-manually-03.png


[dotnet check]: https://docs.microsoft.com/ko-kr/dotnet/core/install/how-to-detect-installed-versions?pivots=os-macos&WT.mc_id=dotnet-50035-juyoo&ocid=AID3035186
[dotnet download]: https://dotnet.microsoft.com/download/dotnet?WT.mc_id=dotnet-50035-juyoo&ocid=AID3035186
[dotnet uninstall tool]: https://docs.microsoft.com/ko-kr/dotnet/core/additional-tools/uninstall-tool?tabs=macos&WT.mc_id=dotnet-50035-juyoo&ocid=AID3035186
[dotnet uninstall manual]: https://docs.microsoft.com/ko-kr/dotnet/core/install/remove-runtime-sdk-versions?pivots=os-macos&WT.mc_id=dotnet-50035-juyoo&ocid=AID3035186

[pwsh]: https://docs.microsoft.com/ko-kr/powershell/scripting/overview?view=powershell-7.2&WT.mc_id=dotnet-50035-juyoo&ocid=AID3035186
