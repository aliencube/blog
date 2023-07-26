---
title: "GitHub 코파일럿으로 손쉽게 APIM 정책 문서 작성하기"
slug: gh-copilot-for-apim-policies
description: "이 포스트에서는 애저 API 관리자에서 사용하는 다양한 정책 문서들을 ChatGPT, 애저 OpenAI 그리고 GitHub 코파일럿을 이용해 손쉽게 작성하는 방법에 대해 알아봅니다."
date: "2023-07-31"
author: Justin-Yoo
tags:
- azure
- api-management
- github-copilot
- azure-openai
cover: /2023/07/gh-copilot-for-apim-policies-00.png
fullscreen: true
---

[애저 API 관리자(API Management; 이하 APIM)][apim]는 회사의 다양한 백엔드 API를 손쉽게 관리해 줄 수 있는 도구이다. APIM은 다양한 기능을 제공하는데, 그 중 하나가 [APIM 정책][apim policies]이다. APIM 정책은 APIM에서 제공하는 기능을 확장하거나, API를 보호하기 위한 보안 정책을 적용하는 등 다양한 용도로 사용할 수 있다. APIM 정책은 XML 기반의 문서로 작성하는데, 너무나 다양한 레퍼런스가 있기도 하고 복잡해서 이 정책 문서를 작성하기는 쉽지 않다. 이번 포스트에서는 APIM 정책을 손쉽게 작성하는 방법에 대해 알아보고, 이를 위해 [GitHub 코파일럿(GitHub Copilot)][gh copilot]과 [애저 OpenAI 서비스(AOAI)][aoai]를 사용하는 방법에 대해 알아보기로 한다.

## APIM 정책 문서 작성하기

가장 먼저 APIM 인스턴스를 만들고 나면 아래와 같은 글로벌 정책 문서가 기본값으로 만들어져 있다.

```xml
<policies>
    <inbound />
    <backend>
        <forward-request />
    </backend>
    <outbound />
    <on-error />
</policies>
```

![APIM 기본 정책 문서 &ndash; 글로벌][image-01]

그리고, 각 API별로 기본 정책 문서가 기본값으로 아래와 같이 만들어져 있다.

```xml
<policies>
    <inbound>
        <base />
    </inbound>
    <backend>
        <base />
    </backend>
    <outbound>
        <base />
    </outbound>
    <on-error>
        <base />
    </on-error>
</policies>
```

![APIM 기본 정책 문서 &ndash; API][image-02]

마찬가지로 각 오퍼레이션별로 기본 정책 문서가 기본값으로 아래와 같이 만들어져 있다.

```xml
<policies>
    <inbound>
        <base />
    </inbound>
    <backend>
        <base />
    </backend>
    <outbound>
        <base />
    </outbound>
    <on-error>
        <base />
    </on-error>
</policies>
```

![APIM 기본 정책 문서 &ndash; 오퍼레이션][image-03]

이 기본 문서를 바탕으로 비즈니스 로직에 맞게끔 API 정책을 각 수준별로 정의해야 한다. 이 때 아래 그림과 같이 스니펫을 이용하면 좀 더 손쉽게 필요한 정책을 추가할 수 있다.

![APIM 정책 스니펫][image-04]

하지만, 이렇게 스니펫을 하나하나 찾아서 넣는 것은 여전히 번거롭다. 러닝 커브도 꽤 높은 편이다. 이를 위해 [GitHub 코파일럿][gh copilot]을 사용해보자. 마침 [GitHub 코파일럿 Chat][gh copilot chat beta] 기능이 퍼블릭 베타 버전으로 출시됐다고 하니 더욱 좋은 타이밍이다.

## 준비물

- [Visual Studio Code][vs code]
- [GitHub 코파일럿 구독][gh copilot subscription] &ndash; 학생의 경우에는 [Student Developer Pack][gh student dev pack] 프로그램에 가입하면 무료로 구독을 할 수 있다.
- [GitHub 코파일럿][vs code extensions copilot]
- [GitHub 코파일럿 Chat][vs code extensions copilot chat]

## GitHub 코파일럿으로 APIM 정책 문서 작성하기

APIM 정책 문서는 글로벌 수준, API 수준, 오퍼레이션 수준에서 작성할 수 있고 사용할 수 있는 컨텍스트도 살짝 다르다.

### 글로벌 정책 문서 작성

가장 먼저 글로벌 APIM 정책 문서를 작성해 보자.

> 시나리오: 글로벌 수준에서는 보통 프론트엔드 애플리케이션과 백엔드 애플리케이션 사이의 CORS 정책을 적용하는 경우가 많다. 이 CORS 정책을 글로벌 정책 문서에 반영해 보기로 한다.

아래와 같이 Visual Studio Code를 열고 GitHub 코파일럿 Chat 윈도우를 연다.

![GitHub 코파일럿 Chat &ndash; 윈도우][image-05]

그리고 아래와 같이 [제로샷(zero-shot) 프롬프트][aoai prompt-engineering]를 입력한다.

```yaml
Azure API Management 정책 문서를 글로벌 수준에서 아래 내용을 포함해서 작성해 줘.

- CORS origins: https://make.powerapps.com, https://make.powerautomate.com
- CORS methods: GET, POST, PUT, PATCH, DELETE
```

![GitHub 코파일럿 Chat &ndash; 글로벌 제로샷 프롬프트][image-06]

그러면 대략 아래와 같이 정책 문서를 작성해 준다.

![GitHub 코파일럿 Chat &ndash; 글로벌 제로샷 프롬프트 결과][image-07]

그리고, 이 결과가 만족스럽다면 아래와 같이 "Insert at Cursor" 메뉴를 클릭해서 정책 문서를 오른쪽에 열려있는 `policy-global.xml` 파일에 삽입한다.

![GitHub 코파일럿 Chat &ndash; 글로벌 결과 삽입][image-08]

그러면 아래와 같이 정책 문서가 만들어진다.

![GitHub 코파일럿 Chat &ndash; 글로벌 정책 문서 생성 #1][image-09]

물론 새 파일을 열어 정책 문서를 작성할 수도 있다.

![GitHub 코파일럿 Chat &ndash; 글로벌 정책 문서 생성 #2][image-10]

이렇게 만들어진 문서가 처음부터 완벽하다면 아주 좋겠지만, 대부분의 경우에는 약간의 수정이 필요할 수 있다. 이제부터는 GitHub 코파일럿을 이용해 APIM 정책 문서를 수정해 보자. 먼저 모든 요청 헤더를 받아주는 정책을 추가한다. `</allowed-methods>` 태그 밑에 아래와 같이 주석을 추가한 후 엔터키를 쳐 보자.

```xml
</allowed-methods>
<!-- add the allowed-headers node and accept everything -->
```

그러면 아래와 같이 GitHub 코파일럿이 제안을 해 올 텐데, Tab 키를 눌러 받아들인다.

![GitHub 코파일럿 &ndash; 글로벌 정책 문서 수정 #1][image-11]

이후 엔터 키를 칠 때 마다 GitHub 코파일럿이 제안을 해 올 것이다. 이를 반복해서 아래와 같이 정책 문서를 수정해 준다.

![GitHub 코파일럿 &ndash; 글로벌 정책 문서 수정 #2][image-12]

이번에는 그 아래에 응답 헤더를 받아주는 정책을 추가해 보자. `</allowed-headers>` 태그 밑에 아래와 같이 주석을 추가한 후 엔터키를 쳐 보자.

```xml
</allowed-headers>
<!-- add the expose-headers node and accept everything -->
```

GitHub 코파일럿이 제안해 오는 결과를 Tab 키를 눌러 받아들인다.

![GitHub 코파일럿 &ndash; 글로벌 정책 문서 수정 #3][image-13]

이를 반복해서 아래와 같이 정책 문서를 수정해 준다.

![GitHub 코파일럿 &ndash; 글로벌 정책 문서 수정 #4][image-14]

지금까지 APIM의 글로벌 정책 문서를 GitHub 코파일럿을 이용해서 만들어 봤다.

### API 정책 문서 작성

이번에는 API 정책 문서를 작성해 보자.

> 시나리오: API의 모든 엔드포인트에 동일한 API 키를 적용하는 정책을 작성한다. API 키는 이미 APIM의 Named Values 기능에 저장되어 있다고 가정한다.

마찬가지로 GitHub 코파일럿 Chat을 열고 아래와 같이 제로샷 프롬프트를 입력한다.

```yaml
Azure API Management 정책 문서를 API 수준에서 아래 내용을 포함해서 작성해 줘.

- 요청 헤더 삽입
- 헤더 이름: x-functions-key
- 헤더 값: Named Values 기능에 "{{X_FUNCTIONS_KEY}}"로 저장된 API Key 값
```

그러면 대략 아래와 비슷한 형태로 답이 올 것이다.

![GitHub 코파일럿 &ndash; API 정책 문서 생성 #1][image-15]

이제 이 내용을 저장하기 위해서는 아래 그림과 같이 새 파일을 자동으로 만들고 저장하게 하면 된다.

![GitHub 코파일럿 &ndash; API 정책 문서 생성 #2][image-16]

아래와 같이 새 파일이 하나 만들어졌다.

![GitHub 코파일럿 &ndash; API 정책 문서 생성 #3][image-17]

이 파일을 `policy-api.xml`로 저장한다.

이제 API 수준에서 정책 문서를 GitHub 코파일럿 Chat 기능을 이용해 작성했다. 물론 좀 더 필요하다면 이 `polocy-api.xml` 파일을 열고 추가적인 정책을 GitHub 코파일럿을 이용해 작성해도 좋다.

### 오퍼레이션 정책 문서 작성

이번에는 오퍼레이션 수준에서 정책 문서를 작성해 보도록 하자. URL rewriting 이라든가 백엔드 서버 주소를 변경하는 것들이 오퍼레이션 수준에서 정책으로 적용되는 경우가 많다.

> 시나리오: 엔드포인트의 `/products/{id}` 오퍼레이션에 대해 `/products?id={id}` 형태로 URL rewriting을 해 주고, 백엔드 서버 주소를 `https://fabrikam.com/api`로 변경하는 정책을 작성한다.

마찬가지로 GitHub 코파일럿 Chat을 열고 아래와 같이 제로샷 프롬프트를 입력한다.

```yaml
Azure API Management 정책 문서를 오퍼레이션 수준에서 아래 내용을 포함해서 작성해 줘.

- URL rewriting: /products/{id}를 /products?id={id} 형태로 변환
- 백엔드 서버 URL 변경: https://fabrikam.com/api
```

위와 같은 프롬프트를 통해 만들어지는 정책 문서는 아래와 같다.

![GitHub 코파일럿 &ndash; 오퍼레이션 정책 문서 생성 #1][image-18]

이를 앞서와 마찬가지로 새 파일로 작성한 후 `policy-operation.xml` 파일로 저장한다.

이제 이를 복사해서 애저 포탈을 통해 정책 문서를 수정하면 된다.

---

이렇게 해서 APIM 정책 문서를 GitHub 코파일럿을 이용해 글로벌 수준, API 수준, 오퍼레이션 수준에서 작성해 보았다. APIM의 정책 문서를 작성하거나 수정하는 것은 꽤 복잡할 수 있다. 따라서 러닝 커브가 상대적으로 높은 편인데, 이 때 GitHub 코파일럿을 이용해 본다면 좀 더 손쉽게 APIM 정책 문서를 작성할 수 있지 않을까 한다.

## APIM에 대해 좀 더 알고 싶다면...

만약 APIM과 APIM 정책 문서에 대해 좀 더 공부해 보고 싶다면 아래 링크를 참고하면 좋다.

- [API 관리자 구현][apim learn]


[image-01]: /2023/07/gh-copilot-for-apim-policies-01.png
[image-02]: /2023/07/gh-copilot-for-apim-policies-02.png
[image-03]: /2023/07/gh-copilot-for-apim-policies-03.png
[image-04]: /2023/07/gh-copilot-for-apim-policies-04.png
[image-05]: /2023/07/gh-copilot-for-apim-policies-05.png
[image-06]: /2023/07/gh-copilot-for-apim-policies-06.png
[image-07]: /2023/07/gh-copilot-for-apim-policies-07.png
[image-08]: /2023/07/gh-copilot-for-apim-policies-08.png
[image-09]: /2023/07/gh-copilot-for-apim-policies-09.png
[image-10]: /2023/07/gh-copilot-for-apim-policies-10.png
[image-11]: /2023/07/gh-copilot-for-apim-policies-11.png
[image-12]: /2023/07/gh-copilot-for-apim-policies-12.png
[image-13]: /2023/07/gh-copilot-for-apim-policies-13.png
[image-14]: /2023/07/gh-copilot-for-apim-policies-14.png
[image-15]: /2023/07/gh-copilot-for-apim-policies-15.png
[image-16]: /2023/07/gh-copilot-for-apim-policies-16.png
[image-17]: /2023/07/gh-copilot-for-apim-policies-17.png
[image-18]: /2023/07/gh-copilot-for-apim-policies-18.png

[apim]: https://learn.microsoft.com/ko-kr/azure/api-management/api-management-key-concepts?WT.mc_id=dotnet-102583-juyoo
[apim policies]: https://learn.microsoft.com/ko-kr/azure/api-management/api-management-howto-policies?WT.mc_id=dotnet-102583-juyoo
[apim learn]: https://learn.microsoft.com/ko-kr/training/paths/az-204-implement-api-management/?WT.mc_id=dotnet-102583-juyoo

[aoai]: https://learn.microsoft.com/ko-kr/azure/ai-services/openai/overview?WT.mc_id=dotnet-102583-juyoo
[aoai prompt-engineering]: https://learn.microsoft.com/ko-kr/azure/ai-services/openai/concepts/prompt-engineering?WT.mc_id=dotnet-102583-juyoo#examples

[gh copilot]: https://docs.github.com/ko/copilot/quickstart
[gh copilot subscription]: https://docs.github.com/ko/billing/managing-billing-for-github-copilot/about-billing-for-github-copilot
[gh copilot chat beta]: https://github.blog/2023-07-20-github-copilot-chat-beta-now-available-for-every-organization

[gh student dev pack]: https://education.github.com/pack

[vs code]: https://code.visualstudio.com?WT.mc_id=dotnet-102583-juyoo
[vs code extensions copilot]: https://marketplace.visualstudio.com/items?itemName=GitHub.copilot&WT.mc_id=dotnet-102583-juyoo
[vs code extensions copilot chat]: https://marketplace.visualstudio.com/items?itemName=GitHub.copilot-chat&WT.mc_id=dotnet-102583-juyoo
