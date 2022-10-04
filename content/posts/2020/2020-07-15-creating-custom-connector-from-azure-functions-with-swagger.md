---
title: "애저 펑션에서 Swagger를 이용해 파워 플랫폼을 위한 커스텀 커넥터 곧바로 생성하기"
slug: creating-custom-connector-from-azure-functions-with-swagger
description: "이 포스트에서는 애저 펑션 API를 이용해서 직접 커스텀 커넥터를 생성하는 방법에 대해 알아봅니다."
date: "2020-07-15"
author: Justin-Yoo
tags:
- azure-functions
- swagger
- custom-connector
- power-platform
cover: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-00.png
fullscreen: true
---

[애저 펑션을 위한 Open API 익스텐션][post prev]을 사용하면 좋은 점 중에 하나가 바로 애저 펑션으로 API를 개발할 때 이 API의 발견가능성(discoverability)을 높여준다는 것이다. 따라서, 이를 이용하면 굉장히 손쉽게 [애저 API 관리도구][az apim]에 연동시킬 수 있다. 또한 [애저 로직 앱][az logapp]이나 [파워 플랫폼][power platform] 에서 사용하는 [커스텀 커넥터][az cuscon] 역시도 쉽게 생성할 수 있다. 이 포스트에서는 이 [Open API 익스텐션][gh openapi]을 이용해 애저 펑션에 Swagger 문서를 통합한 후, 이를 이용해 커스텀 커넥터를 만들어 보는 방법에 대해 논의하기로 한다.


## 샘플 애저 펑션 코드 ##

우선 기본적인 뼈대만 갖춘 애저 펑션 샘플 코드는 아래와 같다. `/feeds/items`와 `/feeds/item` 이라는 두 개의 엔드포인트를 나타낸다 (line #7, 15).

https://gist.github.com/justinyoo/2b4bc731ff8f2cdb5e80e28bd7dff9e7?file=01-feed-reader.cs&highlights=7,15

이를 실행시키면 당연하겠지만, 아래와 같이 두 개의 엔드포인트를 확인할 수 있다.

![][image-01]


## NuGet 패키지 설치 ##

애저 펑션에 Open API 문서를 손쉽게 생성해 주는 NuGet 패키지를 설치한다.

https://gist.github.com/justinyoo/2b4bc731ff8f2cdb5e80e28bd7dff9e7?file=02-install-nuget-package.sh


## 보일러플레이트 코드 설치 ##

사실, 위 NuGet 패키지를 설치하면 이 보일러플레이트 코드가 자동으로 설치가 된다. 따라서, 이 부분은 딱히 고민할 부분이 없다. 앱을 빌드하고 실행시켜보자. 아래 그림과 같이 추가 엔드포인트가 보일 것이다. 이 세 엔드포인트가 바로 Open API 관련한 것들이다.

![][image-02]

이제 이 중에서 `http://localhost:7071/api/swagger/ui`를 브라우저에서 실행시켜 보면 아래와 같다.

![][image-03]

일단 Swagger UI 페이지는 나왔지만, 아직 엔드포인트를 설정하지 않았기 때문에 자세한 내용은 보이질 않는다.


## Open API 확장을 위한 데코레이터 지정 ##

이제 아래와 같이 각각의 엔드포인트에 데코레이터를 이용해 Open API 설정을 해 보자. `OpenApiOperation`, `OpenApiRequestBody`, `OpenApiResponseBody` 등의 데코레이터를 사용했다 (line #2-4, 13-15).

https://gist.github.com/justinyoo/2b4bc731ff8f2cdb5e80e28bd7dff9e7?file=08-add-decorators.cs&highlights=2-4,13-15

이렇게 컴파일 한 후, 다시 펑션 앱을 실행시켜 보면 아래와 같이 Swagger UI 페이지가 제대로 보이는 것을 볼 수 있다.

![][image-04]

이렇게 애저 펑션에 Open API 익스텐션을 추가해서 Swagger UI 페이지를 붙이는 것 까지 살펴봤다. 이를 애저로 배포한 후 다시 Swagger UI 페이지를 보면 아래와 같다.

![][image-05]

이제 배포가 끝났으니 실제 커스텀 커넥터를 만들기 위한 다음 단계로 넘어가도록 하자.


## 커스텀 커넥터 생성 ##

커스텀 커넥터는 한 번 만들어 놓으면 [파워 오토메이트][power automate]와 [파워 앱스][power apps] 어디서든 사용할 수 있다. 따라서 여기서는 파워 오토메이트에서 커스텀 커넥터를 만들어 보기로 한다. 먼저 아래와 같이 애저 펑션에서 제공하는 Swagger 문서의 URL을 지정한다.

![][image-06]

그런데, 가끔 아래와 같이 잘 안될 때가 있다.

![][image-07]

그럴 땐 당황하지 말고, Swagger 문서를 저장한 후 직접 업로드한다.

![][image-08]

이렇게 하면 그 다음부터는 그냥 자동으로 진행된다. 애초에 이 Open API 익스텐션이 바로 이 커스텀 커넥터를 염두에 두고 만든 것이어서 문제없이 진행된다. 아래와 같이 `✅ Create Connector` 버튼을 눌러 마무리한다.

![][image-09]

이제 커스텀 커넥터가 제대로 작동하는지 테스트를 해 볼 차례이다. 아래 그림과 같이 4. Test 탭에서 커스텀 커넥터를 연결한다.

![][image-10]

그러면 아래 그림과 같이 애저 펑션 API 키 값을 입력하라는 표시가 나타난다. 여기서 API 키 값을 입력한 후 연결한다.

![][image-11]

커스텀 커넥터에 성공적으로 커넥션이 만들어지면, 이제 아래와 같이 실제로 테스트를 진행한다. 아래 그림의 입력창은 바로 Swagger 문서에 정의된 요청 객체의 형식을 그대로 따라간다. 필요한 데이터를 입력하고 아래 `Test Operation` 버튼을 눌러보자.

![][image-12]

그러면 아래와 같이 테스트가 성공적으로 수행된 것을 확인할 수 있다.

![][image-13]

이제 커스텀 커넥터를 생성했으니, 파워 오토메이트를 하나 만들어 볼 차례이다.


## 커스텀 커넥터를 이용한 파워 오토메이트 플로우 만들기 ##

이번에 만드는 파워 오토메이트 플로우는 파워 앱에서 이용할 것이기 때문에 아래와 같은 순서로 생성한다. 먼저 `Instant Flow`를 선택한다.

![][image-14]

그 다음에는 아래 그림과 같이 파워 앱을 트리거로 지정한다.

![][image-15]

그러면 이제 본격적인 플로우 작성을 위한 디자이너 창이 만들어진다. 여기서 `➕ New Step` 버튼을 클릭한다.

![][image-16]

검색 창에 `ATOM`을 입력하면 방금 우리가 생성한 커스텀 커넥터가 보인다. 그리고, 애저 펑션에서 만든 두 개의 엔드포인트가 나타나는 것을 알 수 있다. 여기서 피드 아이템 하나만 가져오는 액션을 아래와 같이 선택한다.

![][image-17]

액션을 선택하면 앞서 커스텀 커넥터를 테스트 할 때와 비슷한 필드 입력 화면이 나타난다. 동일한 내용을 입력한다.

![][image-18]

이 플로우의 목적은 방금 커스텀 커넥터로 받아온 유튜브 피드를 소셜미디어에 포스팅하는 것이므로, 다음 액션으로 아래와 같이 트위터에 포스팅하는 액션을 선택한다.

![][image-19]

그리고 난 후, 앞서 커스텀 커넥터로부터 받아온 데이터를 아래와 같이 트위터 포스트 데이터로 입력한다.

![][image-20]

이번에는 파워 앱으로 이 플로우의 결과를 넘겨줘야 하니 아래와 같이 응답 객체 액션을 선택한다.

![][image-21]

그리고, 응답 객체의 본문에는 커스텀 커넥터에서 받아온 응답 객체 전부를 넣는다.

![][image-22]

여기까지 하면 파워 오토메이트 플로우 작성은 거의 다 끝났다. 한 번 테스트를 해 보도록 하자. 우측 상단의 `Test` 버튼을 클릭해서 아래와 같이 선택한 후 `Save & Test` 버튼을 클릭한다.

![][image-23]

그러면 다음 화면에서는 이 플로우에서 사용하는 커넥터들이 다 제대로 연결되어 있는지 확인하게 되는데, 다 연결 되었다면 아래 `Continue` 버튼을 눌러 계속 진행한다.

![][image-24]

모든 것이 잘 진행된다면 아래와 같이 테스트에 성공했다는 메시지를 볼 수 있다.

![][image-25]

이제 실제로 워크플로우가 어떻게 진행됐는지 살펴보자. 모든 액션은 아래와 같이 성공적으로 진행되었다. 여기서 응답 객체 데이터를 복사해 놓는다.

![][image-26]

그리고 실제로 트위터에도 성공적으로 포스팅이 된 것을 확인할 수 있다.

![][image-27]

이제 응답 객체를 파워앱에서 좀 더 확실하게 인식할 수 있게끔 마지막 설정을 해 줄 차례이다. 앞서 복사한 응답 객체 데이터를 가지고 JSON 스키마를 설정한다. 아래 그림의 `Generate from Sample` 버튼을 클릭한다.

![][image-28]

방금 복사해 놨던 JSON 응답 객체를 붙여넣고 `Done` 버튼을 클릭한다.

![][image-29]

그러면 JSON 응답객체 스키마가 생성된 것을 확인할 수 있다.

![][image-30]

여기까지 한 후 저장하면 파워 오토메이트 플로우 작성은 모두 끝났다.


## 파워 앱에 파워 오토메이트 연동하기 ##

이제 파워 앱을 만들어 볼 차례이다. 이번에 만드는 파워 앱에 앞서 만든 파워 오토메이트를 연결해 보도록 하자. 우선 새 앱 캔버스를 하나 생성한다.

![][image-31]

그 다음에 버튼 콘트롤 하나, 라벨 콘트롤 두 개, 이미지 콘트롤 하나를 캔버스에 추가한다.

![][image-32]

버튼을 눌렀을 때 필요한 액션이 바로 파워 오토메이트를 실행시키는 것이다. 이 액션을 아래와 같이 연결한다. 버튼을 클릭한 후 상단의 메뉴 바에서 `Action`을 클릭한다. 그리고 바로 아래에 있는 `Power Automate`를 선택한다. 그 다음에 나오는 화면에서 앞서 만들어 둔 파워 오토메이트를 선택하면 된다.

![][image-33]

연결이 끝나면 곧바로 함수 창에 어떤 작업을 할 것인지를 물어보는데, 여기에 `ClearCollect(result, AmplifyingaRandomYouTubeContent.Run())` 라고 입력한다. 여기서 `AmplifyingaRandomYouTubeContent()` 함수는 파워 오토메이트 이름을 의미한다.

![][image-34]

이제 다른 레이블 콘트롤 두 개와 이미지 콘트롤 한 개에는 이 파워 오토메이트 실행 결과를 표시한다. 각각의 콘트롤에 아래와 같이 입력한다.

* 상단 레이블 콘트롤: `First(result).title`
* 하단 레이블 콘트롤: `First(result).link`
* 이미지 콘트롤: `First(result).thumbnailLink`

이렇게 입력한 후 파워 앱을 실행시켜 버튼을 클릭해 보자. 그러면 아래와 같은 결과를 받을 수 있다.

![][image-35]

그리고 트위터에도 제대로 포스트가 잘 이루어진 것을 확인할 수 있다.

![][image-36]

---

지금까지, [애저 펑션][az func] 앱에 [Open API 익스텐션][gh openapi]을 설치해서 자동으로 Swagger 문서를 만들어주게끔 하는 것과 더불어, 이를 이용해 [파워 오토메이트][power automate]에 쓰이는 [커스텀 커넥터][az cuscon]를 손쉽게 만드는 방법, 그리고 파워 오토메이트에 이 커스텀 커넥터를 쉽게 붙이는 방법, 마지막으로 [파워 앱][power apps]에 파워 오토메이트를 쉽게 연동하는 방법에 대해 알아 보았다. 이렇게 애저 펑션에 간단한 익스텐션 하나만 설치하는 것으로 애저 펑션 API의 확장성이 엄청나게 높아지게 되는데, 이를 이용하면 [파워 플랫폼][power platform]에서 필요한 API를 정말 손쉽게 만들 수 있으리라 확신한다.


[image-01]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-01.png
[image-02]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-02.png
[image-03]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-03.png
[image-04]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-04.png
[image-05]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-05.png
[image-06]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-06.png
[image-07]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-07.png
[image-08]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-08.png
[image-09]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-09.png
[image-10]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-10.png
[image-11]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-11.png
[image-12]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-12.png
[image-13]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-13.png
[image-14]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-14.png
[image-15]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-15.png
[image-16]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-16.png
[image-17]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-17.png
[image-18]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-18.png
[image-19]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-19.png
[image-20]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-20.png
[image-21]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-21.png
[image-22]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-22.png
[image-23]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-23.png
[image-24]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-24.png
[image-25]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-25.png
[image-26]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-26.png
[image-27]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-27.png
[image-28]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-28.png
[image-29]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-29.png
[image-30]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-30.png
[image-31]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-31.png
[image-32]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-32.png
[image-33]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-33.png
[image-34]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-34.png
[image-35]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-35.png
[image-36]: /2020/07/creating-custom-connector-from-azure-functions-with-swagger-36.png


[post prev]: /ko/2019/02/02/introducing-swagger-ui-on-azure-functions/

[gh openapi]: https://github.com/aliencube/AzureFunctions.Extensions
[gh openapi docs openapi]: https://github.com/aliencube/AzureFunctions.Extensions/blob/dev/docs/openapi.md
[gh openapi release]: https://github.com/aliencube/AzureFunctions.Extensions/releases

[az func]: https://docs.microsoft.com/ko-kr/azure/azure-functions/functions-overview?WT.mc_id=aliencubeorg-blog-juyoo
[az logapp]: https://docs.microsoft.com/ko-kr/azure/logic-apps/logic-apps-overview?WT.mc_id=aliencubeorg-blog-juyoo
[az apim]: https://docs.microsoft.com/ko-kr/azure/api-management/api-management-key-concepts?WT.mc_id=aliencubeorg-blog-juyoo
[az cuscon]: https://docs.microsoft.com/ko-kr/connectors/custom-connectors/?WT.mc_id=aliencubeorg-blog-juyoo

[power platform]: https://powerplatform.microsoft.com/ko-kr/?WT.mc_id=aliencubeorg-blog-juyoo
[power automate]: https://flow.microsoft.com/ko-kr/?WT.mc_id=aliencubeorg-blog-juyoo
[power apps]: https://powerapps.microsoft.com/ko-kr/?WT.mc_id=aliencubeorg-blog-juyoo
