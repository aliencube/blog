---
title: "애저 Bicep 배포 스크립트 리소스"
slug: azure-bicep-deployment-script
description: "이 포스트에서는 애저 Bicep에서 제공하는 배포 스크립트 리소스 기능을 이용해서 한번에 애저에 리소스를 프로비저닝하고 동시에 앱도 배포하는 방법에 대해 알아봅니다."
date: "2022-04-06"
author: Justin-Yoo
tags:
- azure
- autopilot
- bicep
- developer-experience
cover: /2022/04/azure-bicep-deployment-script-00.png
fullscreen: true
---

[이전 포스트][post 1]에서는 [깃헙 액션][gh actions]의 다양한 이벤트 트리거와 [애저 Bicep][az bicep]을 활용해서 오토파일럿 기능을 구현했다면, 이번 포스트에서는 깃헙 액션 없이 애저 Bicep만 활용해서 오토파일럿 기능을 구현해 보기로 한다.

> 이 포스트에서 사용한 샘플 코드 리포지토리는 [이곳][gh sample]을 참조한다.

## 솔루션 아키텍처 ##

마이크로서비스 아키텍처를 구성하다보면 흔히 보이는 API 게이트웨이와 다양한 API 사이의 관계를 [API 관리자][az apim]와 [애저 펑션][az fncapp] API을 이용해 구현했다. 마이크로서비스 아키텍처는 요구사항에 따라 좀 더 복잡한 구조가 될 수도 있겠으나, 이 포스트의 주제와는 벗어나므로 여기서는 작동하는 최소한의 컴포넌트로만 구성하기로 한다. 대략의 그림은 아래와 같다.

![애저 애플리케이션 아키텍처][image-01]

## 애저 리소스 프로비저닝 ##

애저에 프로비저닝할 리소스는 크게 다섯 가지이다.

* [애저 저장소][az st] 👉 [소스보기][gh sample st]
* [애플리케이션 인사이트][az appins] 👉 [소스보기][gh sample appins]
* [앱 서비스 플랜][az csplan] 👉 [소스보기][gh sample csplan]
* [펑션 앱][az fncapp] 👉 [소스보기][gh sample fncapp]
* [애저 API 관리자][az apim] 👉 [소스보기][gh sample apim]

각각의 리소스당 bicep 파일 하나씩 대응하게끔 모듈화를 시켜놨고, 저장소, 애플리케이션 인사이트, 앱 서비스 플랜은 애저 펑션앱을 위해 필요한 만큼 이를 하나로 묶는 오케스트레이션 파일을 [`provision_functionapp.bicep`][gh sample fncapp provision] 이라는 이름으로 작성한다. 마찬가지로 API 관리자도 애플리케이션 인사이트를 필요로 하는 만큼 이를 하나로 묶는 오케스트레이션 파일을 [`provision_apimamagement.bicep`][gh sample apim provision] 이라는 이름으로 작성한다.

이후 펑션앱을 배포한 후 이를 API 관리자 인스턴스에 등록하는 파일을 [`provision_apimanagementapi.bicep`][gh sample apim api provision] 이라는 이름으로 작성한다.

> **NOTE**: 이 포스트에서는 리소스 별로 모듈화를 했지만, 상황에 따라서는 모듈화를 하기 보다는 하나의 파일에 한꺼번에 작성할 수도 있다. 상황에 따라 선택하는 것이 좋다.

여기까지 bicep 파일을 작성했다면 대략의 순서를 예상할 수 있을 것이다.

1. 펑션앱 관련 리소스를 프로비저닝한다 👉 [`provision_functionapp.bicep`][gh sample fncapp provision]
2. API 관리자 관련 리소스를 프로비저닝한다 👉 [`provision_apimamagement.bicep`][gh sample apim provision]
3. ➡️ 펑션앱을 배포한다 ⬅️
4. API 관리자에 펑션앱을 연결시킨다 👉 [`provision_apimanagementapi.bicep`][gh sample apim api provision]

실제로 프로비저닝이 끝나면 아래와 같이 다양한 리소스가 생긴다.

![애저 리소스 프로비저닝][image-02]

그런데, 여기서 3번의 경우에는 앱을 직접 배포하는 과정이므로 1번, 2번, 4번에서 진행한 리소스 프로비저닝과는 완전히 결이 다르다. 따라서, 이 3번 과정을 리소스 프로비저닝 과정의 일부분으로 편입시킬 수만 있다면 우리가 의도하는 오토파일럿 기능을 완성시킬 수 있게 된다.

## 애저 Bicep 배포 스크립트 리소스 ##

[애저 리소스 관리자][az rm]에서는 [배포 스크립트(Deployment Script)][az bicep ds]라는 개념을 도입했다. 이는 파워셸 스크립트 혹은 bash 스크립트를 리소스 프로비저닝 과정의 일부분으로 편입시켜 리소스 프로비저닝 과정을 좀 더 다양한 용도로 확장할 수 있다는 말이다. 즉, 배포 스크립트를 이용해 [애저 파워셸][az pwsh] 또는 [애저 CLI][az cli]를 실행시킬 수 있다는 말과 동일하다. 이게 어떻게 가능한 걸까? 아래 bicep 내용을 보자.

```javascript
resource ds 'Microsoft.Resources/deploymentScripts@2020-10-01' = {
    name: 'my-deployment-script'
    location: resourceGroup().location
    kind: 'AzureCLI'
    identity: {
        type: 'UserAssigned'
        userAssignedIdentities: {
            '<user-assigned-identity-id>': {}
        }
    }
    properties: {
        azCliVersion: '2.33.1'
        containerSettings: {
            containerGroupName: 'my-container-group'
        }
        environmentVariables: [
            {
                name: 'RESOURCE_NAME'
                value: '<resource-name>'
            }
        ]
        primaryScriptUri: '<bash-script-url>'
        retentionInterval: 'P1D'
    }
}
```

* `kind` 어트리뷰트를 통해 이 배포 스크립트는 애저 CLI를 사용한다고 명시한다.
* bash 스크립트를 통해 애저 CLI를 실행시키려면 로그인 정보가 필요한데, 이를 위해 사용자 지정 ID를 지정한다.
* bash 스크립트에서 사용할 애저 CLI 버전은 `2.33.1`로 고정시킨다. 여기서는 특정 버전을 명시하는 것이 좋은데, 이 부분은 잠시 후에 따로 설명한다.
* bash 스크립트에 적용시킬 환경 변수로 `RESOURCE_NAME`을 정의하고 값을 할당한다.
* `primaryScriptUri` 어트리뷰트를 통해 실행시킬 bash 스크립트의 주소를 입력 받는다.

> **NOTE**: 위 bicep 파일은 필요한 부분만 나타낸 것이고, 좀 더 자세한 내용은 [이 파일][gh sample ds]을 직접 확인하도록 하자.

이 배포 스크립트 리소스가 작동하는 원리를 살펴보자면, 배포 스크립트 리소스를 프로비저닝하는 도중 자동으로 애저 컨테이너 인스턴스를 생성하고 그 안에서 bash 스크립트를 실행시키는데, [이 문서][az bicep ds]를 보면 현재 시점으로 30일 이전에 배포한 애저 CLI 버전을 사용하도록 권장하고 있다. 따라서 이 글을 쓰는 시점을 기준으로 하면 [`2.33.1` 버전][az cli release notes]이 한달 전에 배포된 것이라서 이를 포함한 컨테이너 버전을 사용하는 것으로 지정했다.

## 배포 스크립트 &ndash; bash 스크립트 ##

이제 실제로 애저 CLI를 실행시킬 bash 스크립트를 살펴보자. 가장 먼저 깃헙 리포지토리에 저장되어 있는 애저 펑션앱 아티팩트의 URL을 가져온다. 아티팩트 파일 이름을 `api.zip` 이라고 가정한다면 대략 아래와 같이 해당 아티팩트의 URL을 가져올 수 있다.

```powershell
# Get artifacts from GitHub
urls=$(curl -H "Accept: application/vnd.github.v3+json" \
  https://api.github.com/repos/devkimchi/APIM-OpenAPI-Sample/releases/latest | \
  jq '.assets[] | { name: .name, url: .browser_download_url }')

apizip=$(echo $urls | jq 'select(.name == "api.zip") | .url' -r)
```

다음으로는 이 아티팩트 URL을 이용해서 애저 CLI를 통해 펑션앱을 배포한다. 아래에서 보이는 `$RESOURCE_NAME` 환경변수 값은 앞서 배포 스크립트 bicep 파일을 통해 받아온 값이다.

```powershell
# Deploy function apps
ipapp=$(az functionapp deploy \
  -g rg-$RESOURCE_NAME \
  -n fncapp-$RESOURCE_NAME-api \
  --src-url $apizip \
  --type zip)
```

배포가 끝났다면, 이 펑션앱에서 제공하는 OpenAPI 문서를 애저 API 관리자 인스턴스에 추가해야 한다. 이는 아래와 같이 또다른 bicep 파일을 실행시키면 된다. 이 때 중요한 것은, 애저 CLI로 외부 URL을 통해 리소스 배포를 할 경우, bicep 파일을 직접 사용할 수 없고 반드시 JSON 형태의 ARM 템플릿으로 변환해 놓아야 한다.

```powershell
# Provision APIs to APIM
az deployment group create \
  -n ApiManagement_Api \
  -g rg-$RESOURCE_NAME \
  -u https://raw.githubusercontent.com/devkimchi/APIM-OpenAPI-Sample/main/Resources/provision-apimanagementapi.json \
  -p name=$RESOURCE_NAME
```

이렇게 bash 스크립트 작성까지 끝마쳤다.

## 전체 리소스 오케스트레이션 ##

마지막으로 위에 작성한 모든 것을 하나로 묶어야 한다. 아래와 같이 [`main.bicep` 파일][gh sample main]을 통해 주요 리소스 및 배포 스크립트 리소스를 오케스트레이션한다. 아래에 보면 배포 스크립트용 리소스는 `dependsOn` 어트리뷰트를 이용해 가장 마지막에 실행되게끔 하는 것을 알 수 있다.

```javascript
// Provision API Management
module apim './provision-apimanagement.bicep' = {
    name: 'ApiManagement'
    params: {
        ...
    }
}

// Provision function app
module fncapp './provision-functionapp.bicep' = {
    name: 'FunctionApp'
    dependsOn: [
        apim
    ]
    params: {
        ...
    }
}]

// Provision deployment script
module uai './deploymentScript.bicep' = {
    name: 'UserAssignedIdentity'
    dependsOn: [
        apim
        fncapp
    ]
    params: {
        ...
    }
}
```

이 모든 과정이 끝난 후 최종 파일을 ARM 템플릿 형태로 변환한 후 아래와 같이 버튼에 연결시키면 모든 작업이 끝난다. 이제 이 버튼만 클릭한 후 필요한 정보만 입력하면, 별도의 작업 없이도 애저에 리소스를 프로비저닝하고 앱도 배포하면서 모든 것이 즉시 실행 가능한 상태로 준비될 것이다.

[![Deploy To Azure](https://raw.githubusercontent.com/Azure/azure-quickstart-templates/master/1-CONTRIBUTION-GUIDE/images/deploytoazure.svg?sanitize=true)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fdevkimchi%2FAPIM-OpenAPI-Sample%2Fmain%2FResources%2Fazuredeploy.json)

실제로 위 버튼을 눌러보면 아래와 같이 애저 포탈에서 실행시킬 수 있는 화면을 볼 수 있다.

![애저 포탈 커스텀 템플릿 프로비저닝][image-03]

이 과정이 모두 끝나면 실제로 [애저 API 관리자][az apim] 인스턴스에 연결된 애저 펑션 API를 아래와 같이 API 관리자 URL을 통해 접속할 수 있다.

![애저 API 관리자를 통한 API 접근][image-04]

---

지금까지 애저 리소스 프로비저닝 및 앱 배포와 관련한 오토파일럿 기능을 [애저 bicep][az bicep]의 [배포 스트립트][az bicep ds] 기능을 통해 구현해 보았다. 이를 활용한다면 다양한 상황에서 별도의 번거로운 작업 없이도 스스로 리소스를 만들고 앱을 배포할 수 있게 될 것이다.


[image-01]: /2022/04/azure-bicep-deployment-script-01-ko.png
[image-02]: /2022/04/azure-bicep-deployment-script-02.png
[image-03]: /2022/04/azure-bicep-deployment-script-03.png
[image-04]: /2022/04/azure-bicep-deployment-script-04.png


[post 1]: /ko/2022/03/11/azure-apps-autopilot/
[post 2]: /ko/2022/04/06/azure-bicep-deployment-script/

[az apim]: https://docs.microsoft.com/ko-kr/azure/api-management/api-management-key-concepts?WT.mc_id=62548-dotnet-juyoo
[az appins]: https://docs.microsoft.com/ko-kr/azure/azure-monitor/app/app-insights-overview?WT.mc_id=62548-dotnet-juyoo

[az bicep]: https://docs.microsoft.com/ko-kr/azure/azure-resource-manager/bicep/overview?tabs=bicep&WT.mc_id=62548-dotnet-juyoo
[az bicep ds]: https://docs.microsoft.com/ko-kr/azure/azure-resource-manager/bicep/deployment-script-bicep?WT.mc_id=62548-dotnet-juyoo

[az cli]: https://docs.microsoft.com/ko-kr/cli/azure/what-is-azure-cli?WT.mc_id=62548-dotnet-juyoo
[az cli release notes]: https://docs.microsoft.com/ko-kr/cli/azure/release-notes-azure-cli?WT.mc_id=62548-dotnet-juyoo

[az csplan]: https://docs.microsoft.com/ko-kr/azure/app-service/overview-hosting-plans?WT.mc_id=62548-dotnet-juyoo
[az fncapp]: https://docs.microsoft.com/ko-kr/azure/azure-functions/functions-overview?WT.mc_id=62548-dotnet-juyoo
[az rm]: https://docs.microsoft.com/ko-kr/azure/azure-resource-manager/management/overview?WT.mc_id=62548-dotnet-juyoo
[az st]: https://docs.microsoft.com/ko-kr/azure/storage/blobs/storage-blobs-overview?WT.mc_id=62548-dotnet-juyoo
[az pwsh]: https://docs.microsoft.com/ko-kr/powershell/azure/what-is-azure-powershell?WT.mc_id=62548-dotnet-juyoo

[gh sample]: https://github.com/devkimchi/APIM-OpenAPI-Sample
[gh sample st]: https://github.com/devkimchi/APIM-OpenAPI-Sample/blob/main/Resources/storageAccount.bicep
[gh sample appins]: https://github.com/devkimchi/APIM-OpenAPI-Sample/blob/main/Resources/appInsights.bicep
[gh sample csplan]: https://github.com/devkimchi/APIM-OpenAPI-Sample/blob/main/Resources/consumptionPlan.bicep
[gh sample fncapp]: https://github.com/devkimchi/APIM-OpenAPI-Sample/blob/main/Resources/functionApp.bicep
[gh sample apim]: https://github.com/devkimchi/APIM-OpenAPI-Sample/blob/main/Resources/apiManagement.bicep
[gh sample fncapp provision]: https://github.com/devkimchi/APIM-OpenAPI-Sample/blob/main/Resources/provision-functionapp.bicep
[gh sample apim provision]: https://github.com/devkimchi/APIM-OpenAPI-Sample/blob/main/Resources/provision-apimanagement.bicep
[gh sample apim api provision]: https://github.com/devkimchi/APIM-OpenAPI-Sample/blob/main/Resources/provision-apimanagementapi.bicep
[gh sample ds]: https://github.com/devkimchi/APIM-OpenAPI-Sample/blob/main/Resources/deploymentScript.bicep
[gh sample main]: https://github.com/devkimchi/APIM-OpenAPI-Sample/blob/main/Resources/main.bicep

[gh actions]: https://github.com/features/actions
