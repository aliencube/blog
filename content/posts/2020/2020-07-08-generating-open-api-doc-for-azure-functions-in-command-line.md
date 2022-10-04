---
title: "애저 펑션을 위한 Open API 문서를 커맨드라인에서 생성하기"
slug: generating-open-api-doc-for-azure-functions-in-command-line
description: "이 포스트에서는 애저 펑션 API를 위한 Open API 문서를 커맨드라인에서 생성하는 방법에 대해 알아봅니다."
date: "2020-07-08"
author: Justin-Yoo
tags:
- swagger
- azure-functions
- open-api
- cli
cover: /2020/07/generating-open-api-doc-for-azure-functions-in-command-line-00.png
fullscreen: true
---

한참 전에 [애저 펑션을 위한 Swagger UI 익스텐션][post prev]을 소개한 후 많은 피드백을 받았는데, 그 중 하나가 바로 애저 펑션 런타임 상에서만 Open API 문서를 생성하지 말고, 커맨드라인에서도 직접 생성할 수 있었으면 좋겠다는 것이었다. 마침 이를 위한 작은 도구를 하나 만들어서 배포한 관계로, 이를 소개하는 포스트를 작성하기로 한다.


## CLI 다운로드 받기 ##

CLI는 [깃헙 리포지토리][gh release]에서 다운로드 받을 수 있다. 항상 `cli-<version>`의 형태로 태그를 해 놓기 때문에 최신 버전을 금방 찾을 수 있을 것이다. 익스텐션은 [애저 펑션][az func] v1 부터 최신 버전까지 다 지원하므로, 자신이 운영하는 애저 펑션의 런타임 버전과 운영체제에 맞춰 CLI를 다운로드 받으면 된다.

* 애저 펑션 v1
  * 윈도우 전용: `azfuncopenapi-v<version>-net461-win-x64.zip`
* 애저 펑션 v2 또는 그 이상
  * 리눅스용: `azfuncopenapi-v<version>-netcoreapp3.1-linux-x64.zip`
  * 맥용: `azfuncopenapi-v<version>-netcoreapp3.1-osx-x64.zip`
  * 윈도우용: `azfuncopenapi-v<version>-netcoreapp3.1-win-x64.zip`


## Open API 문서 생성하기 ##

앞서 다운로드 받은 CLI와 더불어 자신의 애저 펑션 앱에 [애저 펑션 Open API 익스텐션][gh doc openapi]을 설정했다면 모든 준비는 된 셈이다. 아래와 같이 명령어를 실행시킨다.

* 윈도우용 CLI:

https://gist.github.com/justinyoo/6da783b29c1f71fbee6c1f3b9bd59f6b?file=01-azfuncopenapi.ps1

* 리눅스/맥용 CLI:

https://gist.github.com/justinyoo/6da783b29c1f71fbee6c1f3b9bd59f6b?file=02-azfuncopenapi.sh

위에 보면 여러 옵션들이 있는데, 옵션별 자세한 내용은 아래와 같다.

| 옵션 | 설명 | 기본값 |
| --- | --- | --- |
| `--project|-p` | 프로젝트 경로. 디렉토리 형태의 전체 경로가 될 수도 있고, `.csproj` 파일을 포함한 전체 경로가 될 수도 있다. | 현재 디렉토리 |
| `--configuration|-c` | 설정값. 일반적으로는 `Debug` 또는 `Release`이지만, 사용자 설정에 따라 다른 무언가가 될 수도 있다. | `Debug` |
| `--target|-t` | 타겟 프레임워크. 애저 펑션 v1에는 반드시 `net4x` 형태가 되어야 한다 (예: `net461`). 애저 펑션 v2에는 `netcoreapp2.x` 형태 (예: `netcoreapp2.1`), 애저 펑션 v3를 `netcoreapp3.x` 형태 (예: `netcoreapp3.1`)를 따른다. | `netcoreapp2.1` |
| `--version|-v` | Open API 스펙 버전. 반드시 `v2` 또는 `v3`가 되어야 한다. | `v2` |
| `--format|-f` | Open API 문서 포맷. 반드시 `json` 또는 `yaml`가 되어야 한다. | `json` |
| `--output|-o` | 생성된 Open API를 저장할 경로. 전체 경로를 지정할 수도 있고, 상대 경로를 지정할 수도 있다. 상대 경로일 경우 `<PROJECT_ROOT>/bin/<CONFIGURATION>/<TARGET_FRAMEWORK>`을 기준으로 한다. | `output` |
| `--console` | 생성된 Open API 문서를 화면에 보여줄 지 아닐지를 결정한다. | `false` |

이렇게 해서 실제로 이 CLI를 돌려보면 지정한 경로에 `swagger.json` 또는 `swagger.yaml` 문서가 생성된 것을 볼 수 있다.

---

지금까지 애저 펑션 Open API 익스텐션을 위한 CLI 사용법에 대해 알아 보았다. 향후 [API Management][az apim] 또는 [파워 플랫폼][power platform]에서 사용하는 [커스텀 커넥터][az cuscon]에서 API를 사용할 경우 굉장히 유용하게 쓰일 수 있는 도구가 될 것이다.


[post prev]: /ko/2019/02/02/introducing-swagger-ui-on-azure-functions/

[gh release]: https://github.com/aliencube/AzureFunctions.Extensions/releases
[gh doc openapi]: https://github.com/aliencube/AzureFunctions.Extensions/blob/dev/docs/openapi.md

[az func]: https://docs.microsoft.com/ko-kr/azure/azure-functions/functions-overview?WT.mc_id=aliencubeorg-blog-juyoo
[az apim]: https://docs.microsoft.com/ko-kr/azure/api-management/api-management-key-concepts?WT.mc_id=aliencubeorg-blog-juyoo
[az cuscon]: https://docs.microsoft.com/ko-kr/connectors/custom-connectors/?WT.mc_id=aliencubeorg-blog-juyoo

[power platform]: https://powerplatform.microsoft.com/ko-kr/?WT.mc_id=aliencubeorg-blog-juyoo
