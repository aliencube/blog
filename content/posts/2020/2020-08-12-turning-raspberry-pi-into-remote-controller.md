---
title: "라즈베리 파이를 리모트 콘트롤러로 활용하기"
slug: turning-raspberry-pi-into-remote-controller
description: "이 포스트에서는 라즈베리 파이를 리모트 콘트롤러로 활용하는 방법에 대해 알아봅니다."
date: "2020-08-12"
author: Justin-Yoo
tags:
- raspberry-pi
- remote-controller
- lirc
- air-conditioner
cover: /2020/08/turning-raspberry-pi-into-remote-controller-00.png
fullscreen: true
---

얼마 전에 [치즈님(@seojeeee)][twt seojeee]님과 함께 이 주제를 갖고 라이브 코딩 방송을 진행한 적이 있었다 &ndash; [1부][yt lc part1] [2부][yt lc part2]. 이 때 왠지 나도 집에 있는 에어콘을 활용해 보면 좋겠다는 생각이 들어서 같은 장비를 활용해서 실제로 리모트 콘트롤러를 만들어 보았다. 이 포스트는 이 리모트 콘트롤러를 만들면서 알아두면 좋을 것들에 대한 메모의 성격이다.

* ***라즈베리 파이를 리모트 콘트롤러로 활용하기***
* [파워 플랫폼으로 라즈베리 파이 리모트 콘트롤러를 외부에서 실행시키기][post next]

> 이 포스트에 사용된 샘플 코드는 이곳 [깃헙 리포지토리][gh sample]에서 확인할 수 있다.


## 하드웨어 및 기본 소프트웨어 사양 확인 ##

[라즈베리 파이][rpi] 하드웨어와 거기에 붙이는 리모트 콘트롤러용 적외선 센서의 특성상 사양에 굉장히 민감하다. 인터넷에 올라온 정보들 중에 많은 수가 이미 버전 차이로 인해 이제는 더이상 유효하지 않은 정보들이 많다. 이 포스트 역시 이 글을 쓰는 당시에는 정확한 정보이지만, 곧 유효하지 않은 정보가 될 가능성이 높다. 따라서, 내가 사용한 하드웨어 및 소프트웨어 사양을 명시하는 것이 여러모로 정확할 것이다.

* 하드웨어
  * [라즈베리 파이 2 모델 B v1.1][rpi 2b]
  * [IR Remote Shield v1.0][rpi ir remote shield]
* 소프트웨어
  * [라즈베리 파이 OS LITE][rpi os lite]: Debian 10 (Buster) 기반, 2020-05-27 버전
  * [LIRC (Linux Infrared Remote Control)][lirc]: v0.10.1, 2017-09-10 버전


## LIRC 모듈 설치 ##

라즈베리 파이 OS를 다운로드 받아 실행시킨 후 가장 먼저 할 일은 LIRC 모듈을 설치하는 것이다. 아래 명령어를 통해 LIRC 모듈을 설치한다.

https://gist.github.com/justinyoo/34e3470ced2641221c69ebeab6e034f7?file=01-install-lirc.sh

위 명령어를 실행하면 라즈베리 파이 OS에 설치된 모든 패키지들이 최신으로 업데이트되고 LIRC 모듈까지 설치가 끝난다.

![][image-01]


## LIRC 모듈 설정 ##

이제 LIRC 모듈을 이용해서 리모트 콘트롤러 입력을 보내고 받는 설정을 해 보도록 하자.


### 부트 로더 설정 ###

가장 먼저 할 일은 부트 로더를 설정해서 라즈베리 파이 시작시 LIRC 모듈도 동시에 구동시키는 것이다. 아래 명령어를 통해 부트 로더를 설정한다.

https://gist.github.com/justinyoo/34e3470ced2641221c69ebeab6e034f7?file=02-modify-boot-config.sh

아래 부분을 찾아 주석을 해제하고 pin 번호를 수정한다. pin 번호 기본 값은 `gpio-ir` 값이 17번, `gpio-ir-tx` 값이 18번이지만, 이를 반대로 바꿔줘야 한다 (line #5-6).

> 물론 안 바꿔도 작동할 수 있다. 하지만, 내 경우에는 꼭 바꿔 줘야만 작동을 했다.

https://gist.github.com/justinyoo/34e3470ced2641221c69ebeab6e034f7?file=03-boot-config.sh&highlights=5-6


### LIRC 모듈 하드웨어 설정 ###

이번에는 LIRC 모듈 하드웨어를 설정해 보자. 아래 명령어를 통해 파일을 열거나 생성한다.

https://gist.github.com/justinyoo/34e3470ced2641221c69ebeab6e034f7?file=04-modify-lirc-hardware.sh

그리고 난 후 아래 내용을 입력한다.

https://gist.github.com/justinyoo/34e3470ced2641221c69ebeab6e034f7?file=05-lirc-hardware.sh


### LIRC 모듈 옵션 설정 ###

이번에는 LIRC 모듈의 옵션 값을 변경한다. 아래 명령어를 통해 해당 파일을 연다.

https://gist.github.com/justinyoo/34e3470ced2641221c69ebeab6e034f7?file=06-modify-lirc-lircoptions.sh

아래와 같이 `driver`와 `device` 값을 찾아 바꿔준다 (line #3-4).

https://gist.github.com/justinyoo/34e3470ced2641221c69ebeab6e034f7?file=07-lirc-lircoptions.sh&highlights=3-4

여기까지 다 됐으면 앞서 설정했던 부트 로더 값을 인식하기 위해 아래 명령어를 통해 라즈베리 파이를 재부팅한다.

https://gist.github.com/justinyoo/34e3470ced2641221c69ebeab6e034f7?file=08-reboot.sh

재부팅이 끝난 후 아래 명령어를 통해 LIRC 모듈이 제대로 작동하고 있는지 확인한다.

https://gist.github.com/justinyoo/34e3470ced2641221c69ebeab6e034f7?file=09-lircd-status.sh

![][image-02]

제대로 작동하는 것을 볼 수 있다.


## 리모트 콘트롤러 등록 ##

여기가 가장 중요한 부분이다. 내가 사용할 리모트 콘트롤러를 등록해야 한다.


### 리모트 콘트롤러 데이터베이스 활용 설정 파일 등록 ###

가장 쉬운 방법으로는 LIRC 웹사이트에 가서 [리모트 콘트롤러 데이터베이스][lirc remote]를 검색해 가져오는 것이다. 예를 들어 L 모사의 에어컨에 쓰이는 리모콘 정보는 모델명을 검색해서 아래와 같은 정보를 찾을 수 있다.

![][image-03]

만약 내가 찾는 리모트 콘트롤 정보가 있다면 다운로드 받은 후 아래 명령어를 통해 지정된 디렉토리로 복사한다.

https://gist.github.com/justinyoo/34e3470ced2641221c69ebeab6e034f7?file=10-copy-lircd-conf.sh


### 리모트 콘트롤러 설정 파일 직접 등록 ###

만약 데이터베이스에 내가 찾는 리모트 콘트롤러 데이터가 없을 경우에는 직접 만들어야 한다. 내 경우에는 김치냉장고를 잘 만드는 W 모사의 에어콘인데 리모트 콘트롤러가 데이터베이스에 없어서 직접 만들었다. 또한 TV 리모트 콘트롤러 역시도 데이터베이스에 없어서 직접 만들어야 했다. 직접 만들기 위해서는 라즈베리 파이에 붙여놓은 IR 센서가 신호를 잡을 수 있는지 여부를 먼저 확인해야 한다. 먼저 아래 명령어를 통해 LIRC 서비스를 중지시킨다.

https://gist.github.com/justinyoo/34e3470ced2641221c69ebeab6e034f7?file=11-lircd-stop.sh

![][image-04]

그 다음에는 아래 명령어를 실행시켜 리모트 콘트롤러 신호를 받을 수 있게끔 한다.

https://gist.github.com/justinyoo/34e3470ced2641221c69ebeab6e034f7?file=12-mode2.sh

이 때 아래와 같은 에러 메시지가 보일 수도 있다.

![][image-05]

이 경우는 LIRC 모듈이 수신과 송신 모두 활성화 되어 있기 때문인데, 지금 당장 필요한 것은 리모트 콘트롤러 신호를 받아서 데이터를 만드는 부분이므로 송신 부분을 비활성화 시킨다. 이는 또다시 부트 로더를 수정하고 재부팅하면 된다. 아래 명령어로 부트 로더 파일을 연다.

https://gist.github.com/justinyoo/34e3470ced2641221c69ebeab6e034f7?file=13-modify-boot-config.sh

위에서 우리는 `gpio-ir`, `gpio-ir-tx` 속성 모두를 활성화 시켰다. 신호를 받는 리시버 부분만 활성화 시켜야 하므로 아래와 같이 수정한다 (line #5-6).

https://gist.github.com/justinyoo/34e3470ced2641221c69ebeab6e034f7?file=14-boot-config.sh&highlights=5-6

이렇게 한 후 `sudo reboot` 명령어를 이용해 재부팅한다. 재부팅이 끝난 후 다시 아래 명령어를 실행시켜 보자. 이제는 제대로 작동할 것이다.

https://gist.github.com/justinyoo/34e3470ced2641221c69ebeab6e034f7?file=15-mode2.sh

![][image-06]

여기서 라즈베리 파이에 대고 리모트 콘트롤러의 버튼을 눌러보자. 그러면 대략 아래와 같이 리모트 콘트롤러의 데이터가 기록이 되는 것을 볼 수 있다.

https://youtu.be/WrTlpCHl1ZA

<br/>이제 제대로 기록이 되는 것을 확인했으니, 위와 같은 형식으로 데이터 파일을 만들어 보자. 아래 명령어를 통해 만들 수 있다.<br/>

https://gist.github.com/justinyoo/34e3470ced2641221c69ebeab6e034f7?file=16-irrecord.sh

명령어를 실행시킨 후 지시를 따라 리모트 콘트롤러 데이터를 기록하면 된다. 그런데, 어떤 상황에서는 리모트 콘트롤러 데이터가 제대로 기록이 되지 않는 경우도 있다. 내가 바로 그런 경우였는데, 그 땐 위 명령어 대신 아래와 같이 해서 직접 파일을 만들고 그 파일을 수정해야 한다.

https://gist.github.com/justinyoo/34e3470ced2641221c69ebeab6e034f7?file=17-mode2.sh

이렇게 한 후 앞서와 같이 리모트 콘트롤러를 라즈베리 파이에 대고 원하는 버튼을 눌러 기록한다. 그런 후에 저 `<remote_controller_name>.lircd.conf` 파일을 열면 대략 아래와 같이 보일 것이다.

![][image-07]

여기서 붉은색 사각형 안에 들어있는 값이 바로 리모트 콘트롤러 버튼의 수신값이다. 여기서 맨 마지막의 큰 숫자는 의미없는 값이므로 지운다. 이렇게 붉은색 사각형 블록만 남겨두고 나머지는 필요없는 값들이니 지운다. 그렇개 하면 내가 기록한 버튼 숫자만큼의 사각형 블록만 남아 있을 것이다. 이제 여기에 추가적으로 아래와 같이 수신값의 위, 아래, 사이에 코딩을 좀 해야 한다.

* 먼저 숫자 블록 앞에 `begin remote ... begin raw_codes` 부분을 넣어준다. 이 값들은 다른 데이터베이스 파일에서 가져온 것인데, 이게 무엇을 의미하는지는 나도 모르겠다. 그냥 복사해서 넣었다 (line #1-13).
* 그리고 각각의 숫자 블록에 `name SWITCH_ON` 등과 같이 이름을 준다. 각각의 버튼마다 다른 값을 갖고 있으니 이 부분이 명확해지면 좋다 (line #15, 22).
* 마지막으로 맨 아랫 부분에 `end raw_codes ... end remote`를 추가한다 (line #29-30).

https://gist.github.com/justinyoo/34e3470ced2641221c69ebeab6e034f7?file=18-lircd-conf.txt&highlights=1-13,15,22,29-30

위와 같이 파일을 수정한 후 이 파일을 아래 명령어를 통해 LIRC 디렉토리로 복사한다.

https://gist.github.com/justinyoo/34e3470ced2641221c69ebeab6e034f7?file=19-copy-lircd-conf.sh

그리고 난 후 해당 디렉토리로 가면 아래와 같은 파일들을 볼 수 있다. 나는 에어콘과 TV 리모트 콘트롤러를 등록했다.

![][image-08]

우리는 이제 더이상 `devinput.lircd.conf` 파일이 필요없으므로 이름을 바꾼다.

https://gist.github.com/justinyoo/34e3470ced2641221c69ebeab6e034f7?file=20-rename-devinput-lircd-conf.sh

리모트 콘트롤러 설정값 등록이 모두 끝났다. 이제 다시 부트 로더 파일을 연다.

https://gist.github.com/justinyoo/34e3470ced2641221c69ebeab6e034f7?file=21-modify-boot-config.sh

그리고, 리모트 콘트롤러 송신기 모드를 활성화 시킨다 (line #5-6).

https://gist.github.com/justinyoo/34e3470ced2641221c69ebeab6e034f7?file=22-boot-config.sh&highlights=5-6

여기까지 설정이 끝났다면 이제 `sudo reboot` 명령어를 통해 라즈베리 파이를 재부팅 시킨다. 재부팅이 끝난 후 다시 아래 명령어를 실행시켜 LIRC 모듈이 잘 작동하는지 확인한다.

https://gist.github.com/justinyoo/34e3470ced2641221c69ebeab6e034f7?file=23-lircd-status.sh

LIRC 모듈이 에어콘과 TV 리모트 콘트롤 설정 정보를 잘 읽어들여서 준비중이다!

![][image-09]

이론적으로는 라즈베리 파이에 내가 원하는 수 만큼의 리모트 콘트롤러를 등록시킬 수 있는 셈이다.


## 라즈베리 파이에서 직접 에어콘과 TV를 조종하기 ##

이제 리모트 콘트롤러가 잘 되는지 아닌지를 확인해 볼 차례이다. 아래 명령어를 통해 내가 어떤 명령어를 실행시킬 수 있는지 확인해 보자.

https://gist.github.com/justinyoo/34e3470ced2641221c69ebeab6e034f7?file=24-irsend-list.sh

나는 에어콘과 TV를 등록했으므로 `<remote_controller_name>` 대신 `winia` 또는 `tv`를 사용했다. 그러면 대략 아래와 같은 결과를 볼 수 있다. 각각의 리모트 콘트롤러는 더 많은 버튼을 갖고 있지만, 당장 나는 켜고 끄는 기능만 있으면 되므로 저 정도면 충분했다.

![][image-10]

이제 이를 실제로 실행시켜 보자.

https://gist.github.com/justinyoo/34e3470ced2641221c69ebeab6e034f7?file=25-irsend-sendonce.sh

터미널 상에서는 아무런 반응이 없지만 실제로 TV가 켜지고 꺼진다.

![][image-11]

https://youtu.be/QoUmSVAxBCs

<br/>안타깝게도 에어콘은 뭔가 설정이 잘못 되었는지 실행을 시킬 수 없었지만, TV는 실행을 시킬 수 있었다. 에어콘이 더 중요했는데, 이 부분은 좀 더 찾아봐야 할 것 같다. 리모트 콘트롤러를 등록하기 위해 다른 전문적인 장비가 있다면 또 괜찮을 것 같기도 하다.

---

지금까지 [라즈베리 파이][rpi]를 리모트 콘트롤러로 활용해서 집의 전자제품들을 켰다 껐다 하는 방법을 알아 보았다. [다음 포스트][post next]에서는 그렇다면 실제로 이 기능을 이용해서 외부에서도 [애저 펑션][az func]과 [파워 오토메이트][pw automate], [파워 앱][pw apps]을 통해 집 안의 전자제품들을 켰다 껐다 할 수 있는 방법에 대해 알아보기로 한다.


[image-01]: /2020/08/turning-raspberry-pi-into-remote-controller-01.png
[image-02]: /2020/08/turning-raspberry-pi-into-remote-controller-02.png
[image-03]: /2020/08/turning-raspberry-pi-into-remote-controller-03.png
[image-04]: /2020/08/turning-raspberry-pi-into-remote-controller-04.png
[image-05]: /2020/08/turning-raspberry-pi-into-remote-controller-05.png
[image-06]: /2020/08/turning-raspberry-pi-into-remote-controller-06.png
[image-07]: /2020/08/turning-raspberry-pi-into-remote-controller-07.png
[image-08]: /2020/08/turning-raspberry-pi-into-remote-controller-08.png
[image-09]: /2020/08/turning-raspberry-pi-into-remote-controller-09.png
[image-10]: /2020/08/turning-raspberry-pi-into-remote-controller-10.png
[image-11]: /2020/08/turning-raspberry-pi-into-remote-controller-11.png

[post next]: /ko/2020/08/19/remote-controlling-home-appliances-using-raspberry-pi-and-power-platform/

[twt seojeee]: https://twitter.com/seojeee

[yt lc part1]: https://youtu.be/q8TuRTw0OsQ
[yt lc part2]: https://youtu.be/2iaSu9kOQGE

[gh sample]: https://github.com/devkimchi/Raspberry-PI-Remote-Controller-Sample

[rpi]: https://www.raspberrypi.org/
[rpi 2b]: https://www.raspberrypi.org/products/raspberry-pi-2-model-b/
[rpi ir remote shield]: https://www.amazon.com/IR-Remote-Control-Transceiver-Raspberry/dp/B0713SK7RJ
[rpi os lite]: https://www.raspberrypi.org/downloads/raspberry-pi-os/

[lirc]: https://www.lirc.org/
[lirc remote]: http://lirc-remotes.sourceforge.net/remotes-table.html

[az func]: https://docs.microsoft.com/ko-kr/azure/azure-functions/functions-overview?WT.mc_id=aliencubeorg-blog-juyoo

[pw automate]: https://flow.microsoft.com/ko-kr/?WT.mc_id=aliencubeorg-blog-juyoo
[pw apps]: https://powerapps.microsoft.com/ko-kr/?WT.mc_id=aliencubeorg-blog-juyoo
