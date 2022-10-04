---
title: "파워 플랫폼으로 라즈베리 파이 리모트 콘트롤러를 외부에서 실행시키기"
slug: remote-controlling-home-appliances-using-raspberry-pi-and-power-platform
description: "이 포스트에서는 라즈베리 파이로 만든 리모트 콘트롤러를 파워 플랫폼을 이용해 원격으로 제어하는 방법에 대해 알아봅니다."
date: "2020-08-19"
author: Justin-Yoo
tags:
- raspberry-pi
- remote-controller
- power-platform
- azure-functions
cover: /2020/08/remote-controlling-home-appliances-using-raspberry-pi-and-power-platform-00.png
fullscreen: true
---

[지난 포스트][post prev]에서는 [라즈베리 파이][rpi]를 [LIRC][lirc] 모듈을 이용해서 리모트 콘트롤러로 활용하는 방법에 대해 알아 보았다. 그렇다면, 이제는 이 LIRC 모듈을 원격지에서 조정해 볼 차례이다. 마침 이미 누군가가 이 LIRC 모듈을 [node.js][nodejs]를 통해 다룰 수 있는 패키지를 만들어 놓은 것이 있어서, 이를 이용하면 원격지에서 손쉽게 리모트 콘트롤러를 실행시킬 수 있다. 이 포스트에서는 이를 구현하는 방법에 대해 알아보기로 한다.

* [라즈베리 파이를 리모트 콘트롤러로 활용하기][post prev]
* ***파워 플랫폼으로 라즈베리 파이 리모트 콘트롤러를 외부에서 실행시키기***

> 이 포스트에 사용된 샘플 코드는 이곳 [깃헙 리포지토리][gh sample]에서 확인할 수 있다.


## node.js API 앱 서버 만들기 ##

npm 패키지에 보면 [node-lirc][npm node-lirc]라는 것이 있다. 이를 이용하면 자바스크립트 애플리케이션에서 손쉽게 LIRC 명령어를 실행시킬 수 있다.


### 리모트 콘트롤러 펑션 ###

아래와 같이 `remote.js` 라는 node 모듈을 하나 만들어 보자. 사용법은 아주 간단하다. `remote()` 라는 펑션을 하나 만들어서 리모트 콘트롤러 이름과 명령어를 보내기만 하면 된다 (line #5-9). 실제로 이 펑션은 `irsend SEND_ONCE <device> <command>` 명령어를 실행시키는 것에 불과하다 (line #6). 그리고, 명령어 개체를 `remoteControl`로 만들고 그 안에 `onSwitchOn`, `onSwitchOff` 펑션을 작성했다.

https://gist.github.com/justinyoo/352f186bf49fece4cb3dfb4cfae45cab?file=01-remote.js&highlights=5-9


### API 서버 앱 ###

이번에는 이 리모트 콘트롤러 모듈을 이용하는 API 앱을 만들어 보자. 이것 역시도 굉장히 간단하게 만들 수 있다. [express][npm express] 패키지를 이용하면 된다. 아래와 같이 전자제품을 켜는 엔드포인트와 끄는 엔드포인트를 각각 `/remote/switchon`, `/remote/switchoff`로 만든다 (line #6, 17). 여기서는 편의상 `GET` 방식을 썼지만, 일반적으로는 `POST` 방식이 더 정확할 것이다. 마지막으로 이 API 앱을 실행시킬 경우 4000번 포트로 실행하게끔 한다 (line #28).

https://gist.github.com/justinyoo/352f186bf49fece4cb3dfb4cfae45cab?file=02-app.js&highlights=6,17,28

여기까지 하면 이 리모트 콘트롤러를 API를 통해 실행시킬 수 있는 준비가 됐다. 마지막으로 `package.json` 파일에 이 API 앱을 실행시킬 수 있는 명령어가 아래와 같이 되어 있는지 확인한다 (line #4).

https://gist.github.com/justinyoo/352f186bf49fece4cb3dfb4cfae45cab?file=03-package.json&highlights=4

위와 같이 되어 있다면, 이제 `npm run start` 명령어를 통해 API 앱을 실행시킨다.

![][image-01]

API 앱이 실행된 것이 보일 것이다. 이제 웹 브라우저를 통해 API 앱을 실행시켜 보자.

https://youtu.be/ULZ2KYZejWw

<br/>
이렇게 리모트 콘트롤러를 node.js API 앱을 통해 실행시킬 수 있게 되었다.


## 홈 네트워크 터널링 &ndash; ngrok ##

위 API 앱은 라즈베리 파이에서 돌아가는데, 이 라즈베리 파이는 홈 네트워크 안에서만 접속 가능하다. 위 비디오 클립에서도 볼 수 있다시피 `172.30.xxx.xxx`는 사설 IP 대역이어서 외부에서는 특정 IP를 열어두지 않는 이상 접근이 불가능하다. 이는 보안상 당연한 것이기도 한데, 이를 해결하기 위해서는 여러 방법들이 있다. 그 중에서 사설 네트워크의 특정 애플리케이션만 외부에서 접근 가능하게 해 주는 [ngrok][ngrok] 이라는 서비스가 있다. 이 서비스를 이용하면 사설 네트워크의 특정 서비스에 대해서만 제한적으로 터널링이 가능하다. 자세한 방법은 [문서][ngrok docs]를 참조하도록 하고, 여기서는 이 ngrok 애플리케이션을 [라즈베리 파이에 설치][ngrok download]한 후, 아래 명령어를 실행시켜 HTTP 프로토콜과 4000번 포트에 대해 터널링을 구성한다.

https://gist.github.com/justinyoo/352f186bf49fece4cb3dfb4cfae45cab?file=04-run-ngrok.sh

ngrok을 실행시킨 화면은 대략 아래와 같다. 무료 버전으로 사용하고 있기 때문에 커스텀 도메인은 사용할 수 없고, 아래와 같이 실행시킬 때 마다 무작위로 URL이 만들어져서 붙는다. 이번에는 `https://48b6ff595189.ngrok.io`와 같은 형태로 생성됐지만, 다음번에 생성될 땐 또 다른 형태의 URL이 될 것이다.

![][image-02]

이렇게 만들어진 URL을 통해 홈 네트워크 대신 외부에서 리모트 콘트롤러에 접근이 가능하다. 아래 동영상을 통해 확인해 보자.

https://youtu.be/nT-CXCueGEc


## 애저 펑션으로 리모트 콘트롤러 제어 ##

ngrok을 통해 라즈베리 파이 리모트 콘트롤러를 홈 네트워크 외부로 노출시켰다면 이번에는 이 리모트 콘트롤러를 대신 호출해 주는 서버리스 애플리케이션을 [애저 펑션][az func]으로 만들어 보자. 애저 펑션 역시도 node.js 코드로 아래와 같이 손쉽게 작성이 가능하다. 아래 펑션은 쿼리스트링 또는 요청 개체를 이용해서 `device` 값과 `power` 값을 받아오게 되어 있다 (line #6-7). 또한 환경 변수 값을 이용해서 ngrok이 제공하는 URL 값을 받아온다 (line #9-11). 이들을 조합해서 (line #14) 라즈베리 파이 리모트 콘트롤러를 실행시킨다 (line #16).

https://gist.github.com/justinyoo/352f186bf49fece4cb3dfb4cfae45cab?file=05-function.js&highlights=6-7,9-11,14,16

이렇게 한 후, 실제로 로컬에서 애저 펑션을 실행시켜 라즈베리 파이 리모트 콘트롤러를 ngrok을 통해 실행시켜 보자.

https://youtu.be/o9Rhwktzvgs

<br/>
이후 애저로 배포한 후 이를 다시 실행시켜 보면 아래와 같다.<br/><br/>

https://youtu.be/r2lX6_qxXAE


## 파워 플랫폼으로 리모트 콘트롤러 제어 ##

위와 같이 애저 펑션으로 리모트 콘트롤러를 제어할 수 있게 됐다면, 이번에는 이를 좀 더 편하게 모바일 앱에서 제어를 해 보도록 하자. 먼저 [파워 오토메이트][pw automate]를 이용해 이 애저 펑션을 호출하는 것이고, [파워 앱][pw apps]를 통해 파워 오토메이트를 실행시킨다. 파워 앱에서 직접 애저 펑션을 호출할 수도 있지만, 그렇게 하기 보다는 파워 오토메이트를 통해 워크플로우를 제어하고, 파워 앱에서는 앱의 행위와 상태를 조절하는 것이 낫다. 먼저 파워 오토메이트로 워크플로우를 만들어 보자.


## 파워 오토메이트 워크플로우 ##

먼저 워크플로우 인스턴스를 하나 생성한다. 이 워크플로우는 파워 앱에서 호출하는 워크플로우이므로 아래와 같이 파워 앱을 트리거로 설정한다.

![][image-03]

인스턴스를 생성하면 아래와 같이 비주얼 디자이너 화면이 보인다. 트리거로 파워 앱이 설정되었다.

![][image-04]

이 워크플로우는 단순히 애저 펑션을 호출하는 것이므로 아래와 같이 HTTP 액션을 검색한다.

![][image-05]

앞서 애저 펑션은 `GET`, `POST` 둘 다 허용하는 것으로 만들어 놨는데, 여기 파워 오토메이트에서는 `POST` 방식으로 데이터를 넘겨주는 것으로 하자. 아래와 같이 애저 펑션 URL과 헤더를 설정한다. 이 때 요청 본문에 들어가는 값이 `HTTP_Body`와 `HTTP_Body_1`로 되어 있는 것이 보일텐데, 이것은 아래 그림에서 `Ask in PowerApps`를 클릭하면 자동으로 생성이 되는 것으로, `HTTP_Body`는 device 값을, `HTTP_Body_1`은 power 값을 나타낸다. 이는 아래 파워 앱 부분에서 다시 한 번 설명을 하기로 한다.

![][image-07]

위와 같이 애저 펑션을 통해 리모트 콘트롤러를 실행시킨 결과를 다시 파워 앱으로 보내주기 위해 아래와 같이 HTTP 응답 개체 액션을 검색한다.

![][image-08]

그리고, 이전 HTTP 액션의 결과값을 아래와 같이 그대로 반환하는 것으로 한다.

![][image-09]

위 그림에서 `Show advanced options` 버튼을 클릭하면 아래와 같은 추가 화면이 나오는데, 이는 파워 앱에 이 응답 개체의 형식을 알려줘야 하기 때문이다.

![][image-10]

여기서 `Generate from sample` 버튼을 클릭한다. 그러면 아래와 같이 JSON 페이로드를 입력할 수 있는 화면이 나타나는데, 여기에 실제 애저 펑션에서 받은 응답 개체 JSON 페이로드를 입력한다.

![][image-11]

그러면 아래와 같이 JSON 스키마가 자동으로 생성된 것을 볼 수 있다.

![][image-12]

여기까지 해서 [파워 오토메이트][pw automate]를 이용해서 애저 펑션을 호출하는 워크플로우를 만들었다.


### 파워 앱 으로 리모콘 앱 만들기 ###

이제 진짜 리모콘 앱을 만들어 보자. 아래와 같이 버튼 두 개와 라벨 하나를 추가한다. 첫번째 버튼은 ON, 두번째 버튼은 OFF 라고 이름을 준다.

![][image-13]

아래와 같이 `ON` 버튼을 클릭하면 앞서 파워 오토메이트로 만들어 둔 `Home Appliances Remote Controller` 워크플로우를 실행시키게끔 연결한다. 이 때 화면 상단에 함수가 어떤 식으로 호출이 되는지 살펴보자. `HomeAppliancesRemoteController.Run()` 펑션이 바로 이 워크플로우를 나타내는 것인데, 이 때 `tv`, `on` 값을 파라미터로 보낸다. 앞서 파워 오토메이트 워크플로우에서 `HTTP_Body`, `HTTP_Body_1` 값이 각각 device, power를 가리킨다고 했던 것을 기억하는가? 바로 이 `tv`, `on` 값이 그것이다.

![][image-14]

마찬가지로 `OFF` 버튼을 클릭하면 같은 워크플로우를 호출하는데, 이번에는 `tv`, `off` 값을 파라미터로 보내면 된다.

![][image-15]

그리고, 아래 화면은 이렇게 버튼을 클릭했을 때 그 결과값을 보여주는 라벨 값이다.

![][image-16]

지금 위에서는 단지 버튼 두 개를 가지고 TV 리모콘만 제어했지만, 만약 이 파워 앱에 버튼 두 개를 추가해서 같은 파워 오토메이트 워크플로우를 연결한 후 `winia`, `on` 값을 파라미터로 준다거나 `winia`, `off` 값을 파라미터로 보낸다면 이 버튼 두 개를 이용해서는 에어콘을 제어할 수 있게 된다.

아래와 같이 이 파워 앱을 저장하고 모바일 앱으로 배포한다.

![][image-17]

이제 실제로 이 파워 앱이 모바일 폰에서 제대로 작동하는지 아래 동영상을 통해 확인해 보도록 하자.

https://youtu.be/iI6nfwwzFQ4

<br/>
실제로 TV를 켜고 끄는 모습은 아래와 같다.<br/><br/>

https://youtu.be/JXPLTuloSWg

---

지금까지 가전 제품의 리모콘을 라즈베리 파이로 구현해서, 이를 퍼블릭 클라우드를 통해 파워 앱으로 제어할 수 있는 시나리오를 살펴보았다. 이렇게 만들어 놨다면, 집 밖에서도 충분히 집 안의 가전제품을 제어할 수 있게 되는 것이다. 이밖에도 파워 플랫폼의 용도는 무궁무진한데, 이 포스트를 읽는 여러분들도 재미있는 아이디어가 있다면 충분히 활용해 볼 만 하다.


[image-01]: /2020/08/remote-controlling-home-appliances-using-raspberry-pi-and-power-platform-01.png
[image-02]: /2020/08/remote-controlling-home-appliances-using-raspberry-pi-and-power-platform-02.png
[image-03]: /2020/08/remote-controlling-home-appliances-using-raspberry-pi-and-power-platform-03.png
[image-04]: /2020/08/remote-controlling-home-appliances-using-raspberry-pi-and-power-platform-04.png
[image-05]: /2020/08/remote-controlling-home-appliances-using-raspberry-pi-and-power-platform-05.png
[image-06]: /2020/08/remote-controlling-home-appliances-using-raspberry-pi-and-power-platform-06.png
[image-07]: /2020/08/remote-controlling-home-appliances-using-raspberry-pi-and-power-platform-07.png
[image-08]: /2020/08/remote-controlling-home-appliances-using-raspberry-pi-and-power-platform-08.png
[image-09]: /2020/08/remote-controlling-home-appliances-using-raspberry-pi-and-power-platform-09.png
[image-10]: /2020/08/remote-controlling-home-appliances-using-raspberry-pi-and-power-platform-10.png
[image-11]: /2020/08/remote-controlling-home-appliances-using-raspberry-pi-and-power-platform-11.png
[image-12]: /2020/08/remote-controlling-home-appliances-using-raspberry-pi-and-power-platform-12.png
[image-13]: /2020/08/remote-controlling-home-appliances-using-raspberry-pi-and-power-platform-13.png
[image-14]: /2020/08/remote-controlling-home-appliances-using-raspberry-pi-and-power-platform-14.png
[image-15]: /2020/08/remote-controlling-home-appliances-using-raspberry-pi-and-power-platform-15.png
[image-16]: /2020/08/remote-controlling-home-appliances-using-raspberry-pi-and-power-platform-16.png
[image-17]: /2020/08/remote-controlling-home-appliances-using-raspberry-pi-and-power-platform-17.png

[post prev]: /ko/2020/08/12/turning-raspberry-pi-into-remote-controller/

[gh sample]: https://github.com/devkimchi/Raspberry-PI-Remote-Controller-Sample

[rpi]: https://www.raspberrypi.org/
[lirc]: https://www.lirc.org/
[nodejs]: https://nodejs.org/
[ngrok]: https://ngrok.com/
[ngrok docs]: https://ngrok.com/docs
[ngrok download]: https://ngrok.com/download

[npm node-lirc]: https://www.npmjs.com/package/node-lirc
[npm express]: https://www.npmjs.com/package/express

[az func]: https://docs.microsoft.com/ko-kr/azure/azure-functions/functions-overview?WT.mc_id=aliencubeorg-blog-juyoo

[pw automate]: https://flow.microsoft.com/ko-kr/?WT.mc_id=aliencubeorg-blog-juyoo
[pw apps]: https://powerapps.microsoft.com/ko-kr/?WT.mc_id=aliencubeorg-blog-juyoo
