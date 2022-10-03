---
title: "자바로 애저 펑션을 개발할 때 알아두면 좋은 것들"
slug: things-to-know-when-writing-azure-functions-in-java
description: "이 포스트에서는 애저 펑션 앱을 자바로 개발하고 배포할 때 알고 있으면 좋을만한 것들에 대해 다뤄봅니다."
date: "2022-02-04"
author: Justin-Yoo
tags:
- azure
- azure-functions
- java
- intellij-idea
cover: /2022/02/things-to-know-when-writing-azure-functions-in-java-00.png
fullscreen: true
---

[애저 펑션][az fncapp]은 C# 뿐만 아니라 자바, 자바스크립트, 파이썬, 파워셸 등 다양한 언어로 작성할 수 있다. 이 포스트에서는 자바로 애저 펑션 앱을 개발하고 애저로 배포할 때 알고 있으면 좋을만한 것들에 대해 간략하게 정리해 보기로 한다.


## 로컬 개발 환경 설정 ##

닷넷으로 애저 펑션 앱을 개발할 때에는 [비주얼 스튜디오][vs] 혹은 [비주얼 스튜디오 코드][vs code]를 사용하면 개발 환경 설정이 굉장히 쉽게 되기 때문에 크게 문제가 되지 않는다. 마찬가지로 자바스크립트로 애저 펑션을 개발할 때 역시도 [비주얼 스튜디오 코드][vs code]를 사용하면 개발 환경 설정이 그다지 어렵지 않다.

> 자바로 애저 펑션 앱을 개발하기 위해 필요한 기본적인 레퍼런스 문서는 [애저 펑션 자바 개발자 가이드][az fncapp java] 페이지를 참조하면 된다.

하지만, 자바로 애저 펑션 앱을 개발하려면 보통 [인텔리제이 아이디어 (인텔리제이)][intellij]를 많이 사용하는데, 인텔리제이를 사용해서 애저 펑션 앱을 자바로 개발할 때 생기는 몇 가지 특성에 대해 미리 알아둬야 당황하지 않을 수 있다.


### 인텔리제이 사용시 알아두면 좋을 것들 ###

1. **인텔리제이 에디션**:

    애저 펑션 앱은 스프링 프레임워크에 종속성이 없으므로 굳이 인텔리제이 울티밋 에디션이 필요 없다. 커뮤니티 에디션만으로 충분하다. 물론 스프링 프레임워크 뿐만 아니라 다양한 다른 도구들을 사용해야 한다면 울티밋 에디션이 필요하긴 하겠지만, 기본적으로는 커뮤니티 에디션만으로도 충분히 개발할 수 있다.

    ![인텔리제이 UE/CE 비교][image-01]

    또한 인텔리제이에서 애저 펑션 앱을 개발하기 위해서는 기본적으로 [애저 개발도구 플러그인][intellij az toolkits]을 설치하면 기본적인 부트스트래핑 및 템플리팅을 해주기 때문에 필수적으로 설치하는 것이 좋다.

    ![인텔리제이 애저 개발도구 플러그인][image-02]

    [인텔리제이를 사용해서 애저에서 첫번째 자바 펑션 앱 만들기][az fncapp java intellij] 페이지를 참조하면 가장 기초적인 자바 펑션 앱을 만들어서 애저에 배포할 수 있다.

1. **Maven 빌드**:

    인텔리제이로 애저 펑션 앱을 개발할 때에는 기본적으로 [Maven 빌드][maven]에 종속성을 갖는다. 만약 [Gradle 빌드][gradle]로 작업하고 싶다면 [Gradle을 사용해서 애저 펑션 앱 만들기][az fncapp java gradle] 페이지를 참조해서 펑션 앱 프로젝트를 생성한 후 인텔리제이로 임포트해야 한다.


### 맥 환경에서 인텔리제이 사용시 알아두면 좋을 것들 ###

"애저 펑션 개발"이라는 맥락으로 한정하자면, 인텔리제이는 윈도우 환경과 맥 환경에서 작동하는 방식이 사뭇 다르다. 윈도우 환경에서는 아무 문제 없는 것이 맥에서는 문제가 생기는 경우가 있는데, 대략 아래의 두 가지 정도를 들 수 있다.

1. **애저 펑션 코어 툴 인식 문제**:

    아래 그림을 보면 [윈도우 환경에서 애저 펑션 코어 툴][az fncapp core tools win]을 설치한 후 인텔리제이를 통해 로컬로 애저 펑션 앱을 실행시키면 아무 문제 없이 잘 작동한다.

    ![윈도우에서 애저 펑션 앱을 인텔리제이로 로컬 실행][image-03]

    하지만, 아래 그림을 보자. [맥 환경에서 애저 펑션 코어 툴][az fncapp core tools mac]을 설치한 후 GUI 환경에서 실행한 인텔리제이 인스턴스에서 로컬로 애저 펑션 앱을 실행시키면 이를 인식하지 못한다.

    ![맥에서 애저 펑션 앱을 인텔리제이로 로컬 실행][image-04]

    그런데, 터미널에서 `idea .` 명령어를 통해 인텔리제이 인스턴스를 실행시켜보자.

    ![맥 터미널에서 인텔리제이 실행][image-05]

    이 인스턴스에서 로컬로 애저 펑션 앱을 실행시키면 [애저 펑션 코어 툴][az fncapp core tools mac]을 잘 인식해서 실행시킨다.

    ![맥 인텔리제이에서 애저 펑션 앱 실행][image-06]

    이것은 인텔리제이 인스턴스가 파인더에서 아이콘을 클릭해서 실행시켰을 때(Finder launched application)와 셸에서 커맨드라인으로 실행시켰을 때(Shell launched application) 읽어들이는 환경 변수 값이 달라서 생기는 이슈이다. 이 이슈로 인해 JVM이 로드되는 위치가 달라지기 때문이다.

    [이슈 보기][intellij az toolkits issue 1]

    실제로 환경 변수를 파인더에서 인텔리제이를 실행시켰을 때 확인해 보면 아래와 같다.

    ![맥 파인더에서 인텔리제이 실행시킨 후 확인한 환경 변수][image-07]

    이번엔 터미널에서 인텔리제이를 실행시켰을 때 환경 변수를 확인해 보자. 값이 다른 것이 보이는가?

    ![맥 터미널에서 인텔리제이 실행시킨 후 확인한 환경 변수][image-08]

    따라서, 현재로서는 인텔리제이 인스턴스를 셸에서 커맨드라인으로 실행시키는 방법 말고는 해결책이 없다.

1. **애저 펑션 실행 후 포트 록 문제**:

    앞서와 같이 인텔리제이를 이용해 로컬에서 애저 펑션 앱을 실행시켰다 종료시킨 후 다시 실행을 시킬 경우 아래와 같은 에러가 나타난다. 앞서 실행시켰던 애저 펑션 앱이 종료된 후 포트를 반환하지 않았다는 에러메시지이다.

    ![맥 인텔리제이에서 애저 펑션을 다시 실행시킬 때 나타나는 에러][image-09]

    애저 펑션은 두 개의 프로세스를 사용한다. 애저 펑션 런타임은 메인 프로세스에서 돌아가고, 각 언어별 런타임은 서브 프로세스에서 돌아가는 구조인데, 셸 환경이라든가 [비주얼 스튜디오 코드][vs code]에서는 메인 프로세스가 종료될 경우 서브 프로세스도 함께 잘 종료되는데 비해, 인텔리제이에서는 메인 프로세스가 죽어도 서브 프로세스는 여전히 살아 있어서 계속 포트를 붙잡고 있는 현상이 발생한다.

    이 현상은 비단 애저 펑션 뿐만 아니라 비슷한 형태로 메인 프로세스와 서브 프로세스의 이원 체제로 애플리케이션이 작동할 경우 언제든 발생하는 것이어서, 이 이슈를 해결하기 위해서는 인텔리제이 안의 터미널에서 아래와 같이 프로세스를 찾아 강제로 종료하는 방법 말고는 없다.

    https://gist.github.com/justinyoo/a41dc3bb8c19141f3844f214593956c8?file=01-kill-process.sh

    이마저도 귀찮다면, 아래 스크립트를 저장해 두고 필요할 때 사용하면 된다.

    https://gist.github.com/justinyoo/045201b2c4280c818ca872af6284c720?file=kill-process.sh

    혹시나 해서 플러그인 리포지토리에 [이슈를 등록해 뒀으니][intellij az toolkits issue 2] 나중에 이 포스트를 통해 업데이트하기로 한다.


## 애저 배포 환경 ##

애저 펑션 앱을 자바로 작성해서 배포할 때, [Java 8과 11][az fncapp java version] 버전을 지원한다. 따라서, `pom.xml` 파일에서 아래와 같이 `java.version` 값을 1.8 혹은 11로, `javaVersion` 값을 8 혹은 11로 설정하면 된다 (line #5-6, 17-18, 31).

https://gist.github.com/justinyoo/a41dc3bb8c19141f3844f214593956c8?file=02-pom.xml&highlights=6-7,18-19,32

여기서 주의할 점이 있다. 애저 포탈을 통해 애저 펑션 앱 인스턴스를 프로비저닝할 경우에는 Java 8과 Java 11을 선택할 수 있다.

![애저 포탈에서 자바 펑션앱 런타임 선택][image-10]

하지만, bicep 또는 ARM 템플릿을 이용해서 펑션 앱 인스턴스를 프로비저닝할 경우, 별도의 설정을 하지 않는 이상 Java 8을 기본값으로 설정하게 된다. 따라서, 만약 Java 11 버전으로 앱을 배포하고 싶다면 반드시 bicep 또는 ARM 템플릿에 아래와 같이 명시적으로 Java 11 런타임을 사용한다고 선언을 해 주어야 한다. 아래는 bicep 파일의 예시이다. 리눅스 인스턴스를 사용할 때에는 line #8과 같이 `linuxFxVersion` 값을 `Java|11`으로 설정하고, 윈도우즈 인스턴스를 사용할 때에는 line #11과 같이 `javaVersion` 값을 `11`으로 설정해 주면 된다.

https://gist.github.com/justinyoo/a41dc3bb8c19141f3844f214593956c8?file=03-functionapp.bicep&highlights=7-8,10-11

위와 같이 해두면, Java 11 런타임 사용 선언 후 Java 8 앱을 배포하는 경우에는 문제가 되지 않지만, 반대로 Java 8 런타임을 사용하는 경우에는 Java 11 앱은 배포가 되더라도 실행되지 않는다.

---

지금까지 자바로 애저 펑션을 개발할 때 맥OS 환경에서 발생하는 이슈와 해결 방법, 그리고 애저로 배포할 때 생기는 이슈와 해결 방법에 대해 논의해 봤다. 본격적으로 애저 펑션을 자바로 개발하다 보면 더 많은 이슈를 찾을 수도 있겠지만, 기본적으로 이정도를 대비하고 있으면 맥 환경에서 애저 펑션 앱을 자바로 개발하는 데 있어서 큰 무리는 없을 것으로 판단한다.


[image-01]: /2022/02/things-to-know-when-writing-azure-functions-in-java-01.png
[image-02]: /2022/02/things-to-know-when-writing-azure-functions-in-java-02.png
[image-03]: /2022/02/things-to-know-when-writing-azure-functions-in-java-03.png
[image-04]: /2022/02/things-to-know-when-writing-azure-functions-in-java-04.png
[image-05]: /2022/02/things-to-know-when-writing-azure-functions-in-java-05.png
[image-06]: /2022/02/things-to-know-when-writing-azure-functions-in-java-06.png
[image-07]: /2022/02/things-to-know-when-writing-azure-functions-in-java-07.png
[image-08]: /2022/02/things-to-know-when-writing-azure-functions-in-java-08.png
[image-09]: /2022/02/things-to-know-when-writing-azure-functions-in-java-09.png
[image-10]: /2022/02/things-to-know-when-writing-azure-functions-in-java-10.png


[gh sample]: https://github.com/fusiondevkr/fusiondevkr

[vs]: https://visualstudio.microsoft.com/vs/?WT.mc_id=dotnet-56308-juyoo
[vs code]: https://code.visualstudio.com/?WT.mc_id=dotnet-56308-juyoo

[intellij]: https://www.jetbrains.com/idea/
[intellij az toolkits]: https://plugins.jetbrains.com/plugin/8053-azure-toolkit-for-intellij
[intellij az toolkits issue 1]: https://github.com/microsoft/azure-tools-for-java/issues/6243#issuecomment-1012500771
[intellij az toolkits issue 2]: https://github.com/microsoft/azure-tools-for-java/issues/6374

[maven]: https://maven.apache.org/
[gradle]: https://gradle.org/

[az fncapp]: https://docs.microsoft.com/ko-kr/azure/azure-functions/functions-overview?WT.mc_id=dotnet-56308-juyoo
[az fncapp java]: https://docs.microsoft.com/ko-kr/azure/azure-functions/functions-reference-java?tabs=bash%2Cconsumption&WT.mc_id=dotnet-56308-juyoo
[az fncapp java version]: https://docs.microsoft.com/ko-kr/azure/azure-functions/functions-reference-java?tabs=bash%2Cconsumption&WT.mc_id=dotnet-56308-juyoo#java-versions
[az fncapp java intellij]: https://docs.microsoft.com/ko-kr/azure/azure-functions/functions-create-maven-intellij?WT.mc_id=dotnet-56308-juyoo
[az fncapp java gradle]: https://docs.microsoft.com/ko-kr/azure/azure-functions/functions-create-first-java-gradle?WT.mc_id=dotnet-56308-juyoo
[az fncapp core tools win]: https://docs.microsoft.com/ko-kr/azure/azure-functions/functions-run-local?tabs=v4%2Cwindows%2Cjava%2Cportal%2Cbash%2Ckeda&WT.mc_id=dotnet-56308-juyoo#install-the-azure-functions-core-tools
[az fncapp core tools mac]: https://docs.microsoft.com/ko-kr/azure/azure-functions/functions-run-local?tabs=v4%2Cmacos%2Cjava%2Cportal%2Cbash&WT.mc_id=dotnet-56308-juyoo#install-the-azure-functions-core-tools
