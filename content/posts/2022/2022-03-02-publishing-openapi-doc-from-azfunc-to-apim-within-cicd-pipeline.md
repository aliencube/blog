---
title: "CI/CD 파이프라인 안에서 애저 펑션 OpenAPI 문서를 애저 API 관리자로 자동 발행하기"
slug: publishing-openapi-doc-from-azfunc-to-apim-within-cicd-pipeline
description: "이 포스트에서는 CI/CD 파이프라인 안에서 애저 펑션 앱을 빌드한 후 자동으로 OpenAPI 문서를 생성하는 방법에 대해 알아봅니다."
date: "2022-03-02"
author: Justin-Yoo
tags:
- azure
- azure-functions
- openapi
- azure-api-management
cover: /2022/03/publishing-openapi-doc-from-azfunc-to-apim-within-cicd-pipeline-00.png
fullscreen: true
---

[지난 포스트][post 1]에서는 애저 펑션 앱을 CI/CD 파이프라인 안에서 백그라운드 프로세스로 실행하고 거기서 OpenAPI 문서를 자동으로 생성하는 방법에 대해 알아보았다. 이렇게 자동으로 생성된 문서는 아티팩트 형태로 어딘가에 저장한 후 나중에 [파워 플랫폼][pp] 혹은 [애저 API 관리자][az apim] 같은 다른 서비스와 연동시킬 수도 있다.

* [CI/CD 파이프라인 안에서 애저 펑션 OpenAPI 문서 자동 생성하기][post 1]
* ***CI/CD 파이프라인 안에서 애저 펑션 OpenAPI 문서를 애저 API 관리자로 자동 발행하기*** 👈

이 포스트에서는 깃헙 액션을 이용해서 CI/CD 파이프라인 상에서 자동으로 생성한 OpenAPI 문서를 아티팩트의 형태로 저장한 후 애저 펑션 앱 배포와 상관 없이 애저 API 관리자 서비스에 발행하는 방법에 대해 알아보기로 한다.

> **NOTE**: 여기서는 [애저 펑션 OpenAPI 확장 기능][az fncapp ext openapi] 리포지토리에서 제공하는 예제 앱을 이용하기로 한다.


## 깃헙 액션 워크플로우 수정 ##

[지난 포스트][post 1]에서 다뤘던 깃헙 액션 워크플로우는 아래와 같다. 우분투 러너를 사용한다고 가정한다 (line #21-35).

```yaml
name: Build

on:
  push:

jobs:
  build_and_test:
    name: Build
    runs-on: 'ubuntu-latest'

    steps:
    - name: Build solution
      shell: pwsh
      run: |
        pushd MyFunctionApp

        dotnet build . -c Release -v minimal

        popd

    - name: Generate OpenAPI document
      shell: pwsh
      run: |
        cd MyFunctionApp

        Start-Process -NoNewWindow func @("start","--verbose","false")
        Start-Sleep -s 60

        Invoke-RestMethod -Method Get -Uri http://localhost:7071/api/swagger.json | ConvertTo-Json -Depth 100 | Out-File -FilePath outputs/swagger.json -Force

        Get-Content -Path outputs/swagger.json

        cd ..
```


위와 같은 방식으로 생성한 OpenAPI 문서는 기본 주소 값이 `https://localhost:7071/api`이다. 하지만, 실제로 배포가 되는 애저 펑션의 주소는 `https://<azure-functions-app>.azurewebsites.net/api`와 같은 형태이다. 따라서, 애저 API 관리자에 통합시키기 위한 OpenAPI 문서에서는 실제로 배포하지는 않았지만 곧 배포할 펑션 앱의 주소를 적용해야 한다. 다행히도 [문서][az fncapp ext openapi doc]에는 이와 관련한 언급이 있다.

* `OpenApi__HostNames` 라는 환경 변수를 설정하면 `localhost` 대신 실제 내가 사용할 서버 주소를 문서에 적용시킬 수 있다 (line #4).
* 또다른 환경 변수인 `AZURE_FUNCTIONS_ENVIRONMENT` 값을 `Production`으로 수정하면 마치 로컬에서 실행시키는 애저 펑션 앱을 애저 배포 환경에서 실행시키는 효과를 주기 때문에 OpenAPI 문서가 실제로 배포된 펑션 앱에서 가져오는 것과 동일한 형태로 생성된다 (line #5).
* 일반적으로는 `local.settings.json` 파일이 리포지토리에 포함되어 있지 않다. 하지만, 로컬에서 애저 펑션을 백그라운드로 실행시키려면 반드시 있어야 하므로, 임시로 하나 생성하는 것이 좋다. 아래 코드를 보면 `local.settings.sample.json` 파일을 복사해서 사용하는 것으로 되어 있다 (line #12).

위 세 가지 사항을 적용시켜 아래와 같이 깃헙 액션을 수정한다 (line #4,5,12).

```yaml
    - name: Generate OpenAPI document
      shell: pwsh
      env:
        OpenApi__HostNames: 'https://<azure-functions-app>.azurewebsites.net/api'
        AZURE_FUNCTIONS_ENVIRONMENT: 'Production'
      run: |
        cd MyFunctionApp

        mkdir outputs

        # Create local.settings.json
        cp ./local.settings.sample.json ./local.settings.json

        Start-Process -NoNewWindow func @("start","--verbose","false")
        Start-Sleep -s 60

        Invoke-RestMethod -Method Get -Uri http://localhost:7071/api/swagger.json | ConvertTo-Json -Depth 100 | Out-File -FilePath outputs/swagger.json -Force

        Get-Content -Path outputs/swagger.json -Raw

        cd ..
```

위의 예에서는 샘플로 제공되는 `local.settings.sample.json` 파일을 복사해서 사용했지만, 일반적인 방법은 아니다. 따라서, 보통의 경우에는 직접 `local.settings.json` 파일을 직접 생성해야 하는데, 이 때 `local.setting.json` 파일에는 반드시 `FUNCTIONS_WORKER_RUNTIME` 값을 포함시켜야 한다 (line #3). 다시 말하자면 가장 최소한의 `local.settings.json` 파일의 내용은 대략 아래와 같다.

```json
{
  "Values": {
    "FUNCTIONS_WORKER_RUNTIME": "dotnet"
  }
}
```

이렇게 해서 생성한 OpenAPI 문서를 보면 아래와 같이 바뀐 서버 주소가 적용된 것을 확인할 수 있다 (line #4,16).

```json
// OpenAPI v2
{
  "swagger": "2.0",
  "host": "<azure-functions-app>.azurewebsites.net",
  "basePath": "/api",
  "schemes": [
    "https"
  ]
}

// OpenAPI v3
{
  "openapi": "3.0.1",
  "servers": [
    {
      "url": "https://<azure-functions-app>.azurewebsites.net/api"
    }
  ]
}
```

지금까지, 애저 펑션 앱을 배포하지 않고도 배포한 앱에서 생성한 것과 동일한 OpenAPI 문서를 CI/CD 파이프라인 상에서 생성했다.


## 애저 API 관리자로 발행 ##

OpenAPI 문서를 위와 같이 준비했다면, 이를 [애저 API 관리자][az apim]로 발행시킬 차례이다. 다양한 방법이 있을 수 있겠지만, 여기서는 [bicep 파일][az bicep]을 작성하고 이를 [애저 CLI][az cli]를 이용해 배포하기로 한다. 아래는 이와 관련한 bicep 파일의 내용이다. 먼저 [`existing` 키워드][az bicep existing]를 이용해 기존 애저 API 관리자 인스턴스의 정보를 가져온다.

```javascript
// azuredeploy.bicep
param servicename string

resource apim 'Microsoft.ApiManagement/service@2021-08-01' existing = {
    name: servicename
    scope: resourceGroup(apiManagement.groupName)
}
```

이제 아래와 같이 API 인스턴스를 프로비저닝한다.

```javascript
// azuredeploy.bicep
param openapidoc string

resource apimapi 'Microsoft.ApiManagement/service/apis@2021-08-01' = {
    name: '${apim.name}/my-api'
    properties: {
        type: 'http'
        displayName: 'My API'
        description: 'This is my API.'
        path: 'myapi'
        subscriptionRequired: true
        format: 'openapi+json-link'
        value: openapidoc
    }
}
```

위 bicep 파일을 보면 `format`과 `value`라는 속성이 있는데 (line #12-13), 이를 사용하기 위해서는 아래 내용을 알아두면 좋다. 좀 더 자세한 내용은 [애저 Bicep 템플릿 레퍼런스][az apim bicep template] 문서를 참조한다.

1. 앞서 저장한 파일을 직접 JSON 문자열로 변환한 후 사용하는 방법이 있다.
   * OpenAPI v2 문서일 경우에는 `format` 값을 `swagger-json`로 지정한다.
   * OpenAPI v3 문서일 경우에는 `format` 값을 `openapi+json`로 지정한다.
   * 그리고 `value` 값을 JSON 문자열로 설정한다.
2. 만약 어딘가에 저장한 링크를 이용할 경우에는 아래와 같이 사용한다.
   * OpenAPI v2 문서일 경우에는 `format` 값을 `swagger-link-json`로 지정한다.
   * OpenAPI v3 문서일 경우에는 `format` 값을 `openapi+json-link`로 지정한다.
   * 그리고 `value` 값을 OpenAPI 문서를 저장한 링크로 설정한다. 이 때 설정하는 링크는 반드시 공개적으로 접속 가능한 URL이어야 한다.

개인적으로는 첫번째 방법을 선호하지 않는데, JSON 문자열로 변환하는 과정이 너무 고통스럽기 때문이다. 따라서 CI/CD 파이프라인에서 생성한 OpenAPI 문서를 [애저 블롭 저장소][az st blob]에 업로드한 후 그 URL을 사용하는 방법을 추천한다.

위와 같이 bicep 파일을 완성했다면, 아래 애저 CLI 명령을 이용해 배포한다.

https://gist.github.com/justinyoo/4b0138f38c8b15edbaa6fafe2c2a9e1e?file=06-az-cli-deploy.ps1

이후 애저 API 관리자에서 API가 제대로 발행된 것을 확인할 수 있다.

---

지금까지 깃헙 액션 워크플로우 안에서 [애저 펑션][az fncapp]을 백그라운드 프로세스를 이용해 로컬로 실행시키고 OpenAPI 문서를 추출한 후, 이를 [애저 API 관리자][az apim]로 직접 발행하는 방법에 대해 알아보았다. 이렇게 함으로써 애저 펑션과 애저 API 관리자 사이에 필요한 통합 작업을 자동화 할 수 있어 좀 더 업무 효율이 높아질 수 있을 것이다.


[post 1]: /ko/2022/02/23/generating-openapi-document-from-azure-functions-within-cicd-pipeline/
[post 2]: /ko/2022/03/02/publishing-openapi-doc-from-azfunc-to-apim-within-cicd-pipeline/

[pp]: https://powerplatform.microsoft.com/ko-kr/?WT.mc_id=dotnet-59289-juyoo

[az fncapp]: https://docs.microsoft.com/ko-kr/azure/azure-functions/functions-overview?WT.mc_id=dotnet-59289-juyoo
[az fncapp ext openapi]: https://aka.ms/azfunc-openapi
[az fncapp ext openapi doc]: https://github.com/Azure/azure-functions-openapi-extension/blob/main/docs/openapi.md#configure-custom-base-urls

[az apim]: https://docs.microsoft.com/ko-kr/azure/api-management/api-management-key-concepts?WT.mc_id=dotnet-59289-juyoo
[az apim bicep template]: https://docs.microsoft.com/ko-kr/azure/templates/microsoft.apimanagement/service/apis?WT.mc_id=dotnet-59289-juyoo&tabs=bicep#apicreateorupdateproperties

[az bicep]: https://docs.microsoft.com/ko-kr/azure/azure-resource-manager/bicep/overview?tabs=bicep&WT.mc_id=dotnet-59289-juyoo
[az bicep existing]: https://docs.microsoft.com/ko-kr/azure/azure-resource-manager/bicep/existing-resource?WT.mc_id=dotnet-59289-juyoo

[az st blob]: https://docs.microsoft.com/ko-kr/azure/storage/blobs/storage-blobs-introduction?WT.mc_id=dotnet-59289-juyoo

[az cli]: https://docs.microsoft.com/ko-kr/cli/azure/what-is-azure-cli?WT.mc_id=dotnet-59289-juyoo
