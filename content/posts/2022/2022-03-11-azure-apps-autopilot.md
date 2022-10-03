---
title: "애저 앱 오토파일럿"
slug: azure-apps-autopilot
description: "이 포스트에서는 깃헙 액션 워크플로우를 이용해서 한번에 애저에 리소스를 프로비저닝하고 동시에 앱도 배포하는 방법에 대해 알아봅니다."
date: "2022-03-11"
author: Justin-Yoo
tags:
- azure
- autopilot
- github-actions
- developer-experience
cover: /2022/03/azure-apps-autopilot-00.png
fullscreen: true
---

며칠 전 팀 동료인 [David][dave]으로부터 흥미로운 제안이 들어왔다.

> "[애저 정적 웹 앱][az swa]을 애저 리소스 프로비저닝부터 시작해서 배포까지 한 번에 할 수 있는 방법이 있을까? 그렇게만 된다면, PoC 앱이 들어 있는 깃헙 리포지토리 주소만 주면 곧바로 알아서 실행시켜 볼 수 있을 것 같아."

이를 위해서는 두 가지 도전 과제가 있는데, 이 도전 과제들을 하나로 자연스럽게 엮을 수만 있다면, 사용자는 버튼 한 번만 눌러 리소스 설치부터 앱 배포까지 오토파일럿이 될 것이고, 이를 통해 애저에 앱을 배포하는 경험이 비약적으로 향상될 것 같다는 생각이 들었다. 이 포스트에서는 실제로 어떤 형태로 오토파일럿 기능을 제공할 수 있는지 다뤄보기로 한다.

> 이 포스트에서 사용한 샘플 코드 리포지토리는 [이곳][gh sample]을 참조한다.


## 아키텍처 ##

* 프론트엔드: [블레이저 웹 어셈블리][blazor wasm]
* 백엔드: [애저 펑션][az fncapp]
* 데이터베이스: [코스모스 DB][az cosdba]

위 세가지 앱을 프로비저닝하게 되면 프론트엔드 앱과 백엔드 앱은 [애저 정적 웹 앱][az swa]으로 한번에 묶을 수 있고, 이 앱은 [깃헙 액션][gh actions]을 통해 배포하게 된다. 따라서 전체적인 그림은 대략 아래와 같다.

![애저 애플리케이션 아키텍처][image-01]


## 애저 리소스 프로비저닝 ##

위의 아키텍처를 바탕으로 필요한 리소스를 구성해 보자. [애저 bicep][az bicep]을 이용하면 손쉽게 애저 리소스를 정의하고 프로비저닝할 수 있다.


### 코스모스 DB ###

먼저 [코스모스 DB][az cosdba]를 정의한다. [서버리스 모델][az cosdba serverless]의 [SQL API][az cosdba sql]를 사용하고, 데이터베이스 이름을 `AdventureWorks`, 컨테이너 이름을 `products`로 지정한다. 불필요한 부분을 제외하고 표시한다면 대략 아래와 같다.

```javascript
// cosmosDb.bicep: Cosmos DB
param resourceName string
param resourceLocation string
param databaseName string
param containerName string

resource cosdba 'Microsoft.DocumentDB/databaseAccounts@2021-10-15' = {
    name: 'cosdba-${resourceName}'
    location: resourceLocation
    kind: 'GlobalDocumentDB'
    properties: {
        ...
        capabilities:  [
            {
                name: 'EnableServerless'
            }
        ]
        ...
    }
}

resource cosdbasql 'Microsoft.DocumentDB/databaseAccounts/sqlDatabases@2021-10-15' = {
    name: '${cosdba.name}/${databaseName}'
    properties: {
        resource: {
            id: databaseName
        }
    }
}

resource cosdbasqlcontainer 'Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers@2021-10-15' = {
    name: '${cosdbasql.name}/${containerName}'
    properties: {
        resource: {
            id: containerName
            partitionKey: {
                paths: [
                    '/category'
                ]
            }
        }
    }
}

// Outputs
output connectionString string = 'AccountEndpoint=https://${cosdba.name}.documents.azure.com:443/;AccountKey=${cosdba.listKeys().primaryMasterKey};'
```

맨 마지막 `output` 값을 보면 `listKeys()` 함수를 이용해 코스모스 DB의 커넥션 스트링을 출력하게끔 지정했다. 이 값은 애저 정적 웹 앱에 사용할 예정이다.


### 애저 정적 웹 앱 ###

다음으로 [애저 정적 웹 앱][az swa] 인스턴스를 정의한다. [기본으로 제공하는 무료][az swa plan] 인스턴스에 애저 펑션에서 코스모스 DB에 연결할 커넥션 스트링을 추가해야 한다. 불필요한 부분을 제외한 bicep 파일의 내용은 아래와 같다.

```javascript
// staticWebApp.bicep: Azure Static Web Appas
param resourceName string
param resourceLocation string
@secure()
param cosmosDbConnectionString string

resource sttapp 'Microsoft.Web/staticSites@2021-03-01' = {
    name: 'sttapp-${resourceName}'
    location: resourceLocation
    sku: {
        name: 'Free'
    }
    ...
}

resource sttappconfig 'Microsoft.Web/staticSites/config@2021-03-01' = {
    name: '${sttapp.name}/appsettings'
    properties: {
        ConnectionStrings_CosmosDB: cosmosDbConnectionString
    }
}

// Outputs
output deploymentKey string = sttapp.listSecrets().properties.apiKey
```

맨 마지막 `output` 값을 보면 `listSecreets()` 펑션을 이용해 애저 정적 웹 앱을 배포할 때 사용할 배포 키 값을 출력하게끔 했다. 이 값은 나중에 앱 자동 배포시에 활용할 예정이다.


### `main.bicep` &ndash; 오케스트레이션 ###

위와 같이 코스모스 DB와 애저 정적 웹 앱 인스턴스를 개별적으로 정의했다면 이를 묶어서 한번에 프로비저닝할 오케스트레이션 bicep 파일이 필요하다. 아래는 대략의 내용이다. [`module` 키워드][az bicep module]를 이용해서 각 리소스별 정의한 bicep 파일을 호출한다.

```javascript
// main.bicep: Orchestration
param resourceName string

param cosmosDbLocation string
param cosmosDbDatabaseName string
param cosmosDbContainerName string

param staticAppLocation string

// Cosmos DB
module cosdba './cosmosDb.bicep' = {
    name: 'CosmosDB'
    params: {
        resourceName: resourceName
        location: cosmosDbLocation
        databaseName: cosmosDbDatabaseName
        containerName: cosmosDbContainerName
    }
}

// Static Web App
module sttapp './staticWebApp.bicep' = {
    name: 'StaticWebApp'
    params: {
        resourceName: resourceName
        location: staticAppLocation
        cosmosDbConnectionString: cosdba.outputs.connectionString
    }
}

// Outputs
output connectionString string = cosdba.outputs.connectionString
output deploymentKey string = sttapp.outputs.deploymentKey
```

맨 마지막 `output` 내용을 보면 코스모스 DB의 커넥션 스트링과 애저 정적 웹 앱의 배포 키를 반환한다.


### `azuredeploy.bicep` &ndash; 구독수준 스코프 확장 ###

위와 같이 작성만 해도 일차로 인스턴스의 배포는 가능하다. 하지만, 오토파일럿을 위해서라면 리소스 그룹까지 자동으로 만들어야 한다. 따라서, 앞서 작성한 `main.bicep` 파일을 한 번 더 래핑해서 리소스 그룹까지 만들어 보자. 이 때 반드시 [`targetScope` 값을 `subscription`으로 지정][az bicep targetscope]해야 한다.

여기서 눈여겨 봐야 할 부분은 최대한 기본값을 정의해서 오토파일럿 과정에서 사용자의 개입을 최소화시키게끔 하는 것이다. 아래 제공하는 파라미터들은 `resourceName`을 제외하고는 기본값을 지정해 두었다.

```javascript
// azuredeploy.bicep
targetScope = 'subscription'

param name string
@allowed([
    ...
    'Central US'
    'East Asia'
    'East US 2'
    'Korea Central'
    'West Europe'
    'West US 2'
    ...
])
param cosmosDbLocation string = 'Korea Central'
param cosmosDbDatabaseName string = 'AdventureWorks'
param cosmosDbContainerName string = 'products'
@allowed([
    'Central US'
    'East Asia'
    'East US 2'
    'West Europe'
    'West US 2'
])
param staticAppLocation string = 'East Asia'

// Resource Group
resource rg 'Microsoft.Resources/resourceGroups@2021-04-01' = {
    name: 'rg-${name}'
    location: cosmosDbLocation
}

// Resource Orchestration
module resources './main.bicep' = {
    name: 'Resources'
    scope: rg
    params: {
        resourceName: name

        cosmosDbLocation: rg.location
        cosmosDbDatabaseName: cosmosDbDatabaseName
        cosmosDbContainerName: cosmosDbContainerName

        staticAppLocation: staticAppLocation
    }
}

// Outputs
output connectionString string = resources.outputs.connectionString
output deploymentKey string = resources.outputs.deploymentKey
```

마찬가지로 위의 `output` 값은 코스모스 DB의 커넥션 스트링과 정적 웹 앱의 배포 키 값이다.

여기까지 했다면 거의 다 됐다. 마지막으로 위에 작성한 `azuredeploy.bicep` 파일을 ARM 템플릿 형태로 빌드해서 `azuredeploy.json` 파일을 생성한다.

```powershell
az bicep build -f azuredeploy.bicep
```

> **NOTE**: 위의 bicep 파일은 개별 리소스, 오케스트레이션, 구독수준 스코핑 등으로 하나씩 나눠 작성되었다. 개인적으로는 단일책임원칙에 따라 한 번에 한 가지씩 하는 방향을 선호하기 때문에 이렇게 했지만, 그냥 `azuredeploy.bicep` 파일 하나에 모든 것을 작성해도 상관 없다.

이후 아래와 같이 "Deploy to Azure" 버튼에 이 `azuredeploy.json` 파일의 URL을 연결시킨다.

[![Deploy To Azure](https://raw.githubusercontent.com/Azure/azure-quickstart-templates/master/1-CONTRIBUTION-GUIDE/images/deploytoazure.svg?sanitize=true)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fdevkimchi%2FASWA-AutoPilot-Sample%2Fmain%2Finfra%2Fazuredeploy.json)

위 버튼을 눌러보면 아래와 같이 애저 포털을 통해 곧바로 방금 생성한 ARM 템플릿을 실행할 수 있는 화면이 나온다. 다른 값들은 이미 기본값으로 지정되어 있으므로 `Name` 필드만 입력하면 된다.

![애저 포털을 통한 ARM 템플릿 실행][image-02]

이렇게 ARM 템플릿을 실행시키고 나면 아래와 같이 리소스가 자동으로 프로비전된다.

![애저 리소스 프로비전 결과][image-03]

그리고 실제로 코스모스 DB의 커넥션 스트링도 애저 정적 웹 앱에 모두 저장되어 있다.

![코스모스 DB 커넥션 스트링][image-04]

여기까지 하면 리소스 프로비저닝이 끝난다. 하지만, 아직 앱 배포가 끝나지 않았다. 따라서, 다음으로 앱 배포를 해 보도록 하자.


## 애저 앱 배포 ##

현재 [애저 정적 웹 앱][az swa]을 배포하기 위해서는 CI/CD 파이프라인을 거치는 것이 유일한 방법이다. 문제는 애저 정적 웹 앱의 배포 방식이 깃헙 액션에 너무 의존적이어서 이 의존성을 우선 끊어내야 한다.

앞서 프로비저닝이 끝난 정적 웹 앱 인스턴스는 아직 깃헙 액션과 연결되어 있지 않다. 따라서, 앞서 받아온 배포 키를 깃헙 리포지토리에 저장한 후 깃헙 액션과 연결시켜 보기로 한다.


### 배포 키 저장 ###

깃헙 리포지토리의 시크릿에 배포 키를 저장한다. 저장을 위한 키는 `AZURE_STATIC_WEB_APPS_API_TOKEN`이다.

![깃헙 액션 시크릿 - 정적 웹 앱][image-05]


### 깃헙 액션 `workflow_call` 플로우 작성 ###

앱 배포를 시작하는 이벤트는 크게 두 가지가 있다.

1. 코드를 변경한 후: [`push`][gh actions events push] 또는 [`pull_request`][gh actions events pr]
2. 오토파일럿을 통한 리소스 프로비저닝 후: [`workflow_dispatch`][gh actions events wd]

이 두 가지 경우에 동일한 방식으로 앱을 배포해야 하기 때문에 두 이벤트에서 모두 호출할 수 있는 [`workflow_call`][gh actions events wc] 이벤트를 기반으로 워크플로우를 작성한다. 아래는 이를 반영해서 작성한 워크플로우이다. 콜러 워크플로우에서 변수값과 시크릿 값을 입력 받아 [Azure/static-web-apps-deploy][gh actions aswa] 액션을 실행시킨다. 이 때 파라미터 값은 이미 디폴트 값을 정해뒀으므로 콜러 워크플로우에서는 시크릿 값만 넘겨도 된다.

```yaml
# app-deploy.yaml
name: 'App Deploy'

on:
  workflow_call:
    inputs:
      job_name:
        type: string
        required: false
        default: 'Deploy Static Web App'
      app_location:
        type: string
        required: false
        default: app
      api_location:
        type: string
        required: false
        default: api
      output_location:
        type: string
        required: false
        default: wwwroot
    secrets:
      gh_token:
        required: true
      aswa_token:
        required: true

jobs:
  deploy:
    name: ${{ inputs.job_name }}

    runs-on: ubuntu-latest

    steps:
    - name: Checkout the repo
      uses: actions/checkout@v2
      with:
        submodules: true

    - name: Deploy Static Web App 
      uses: Azure/static-web-apps-deploy@v1
      with:
        azure_static_web_apps_api_token: ${{ secrets.aswa_token }}
        repo_token: ${{ secrets.gh_token }}
        action: upload
        app_location: ${{ inputs.app_location }}
        api_location: ${{ inputs.api_location }}
        output_location: ${{ inputs.output_location }}
```


### 깃헙 액션 `workflow_dispatch` 플로우 작성 ###

위에서 작성한 `workflow_call` 워크플로우를 실행시키기 위해서 호출하는 워크플로우이다. 외부에서 직접 실행시킬 수 있도록 하려면 아래와 같이 `workflow_dispatch` 워크플로우를 작성한다. 입력 받는 변수는 앞서 `workflow_call` 이벤트와 동일하다. 여기서는 일부러 구별을 위해 변수명만 살짝 바꿨다.

```yaml
# main-dispatch.yaml
name: 'App Deploy Dispatch'

on:
  workflow_dispatch:
    inputs:
      jobName:
        type: string
        description: Job name
        required: false
        default: 'Deploy Static Web App'
      appLocation:
        type: string
        description: Web app location
        required: false
        default: app
      apiLocation:
        type: string
        description: API app location
        required: false
        default: api
      outputLocation:
        type: string
        description: Web app artifact location
        required: false
        default: wwwroot

jobs:
  call_app_deploy:
    uses: devkimchi/ASWA-AutoPilot-Sample/.github/workflows/app-deploy.yaml@main
    with:
      job_name: ${{ github.event.inputs.jobName }}
      app_location: ${{ github.event.inputs.appLocation }}
      api_location: ${{ github.event.inputs.apiLocation }}
      output_location: ${{ github.event.inputs.outputLocation }}
    secrets:
      gh_token: ${{ secrets.GITHUB_TOKEN }}
      aswa_token: '${{ secrets.AZURE_STATIC_WEB_APPS_API_TOKEN }}'
```

위에서 볼 수 있다시피, `call_app_deploy` 라는 잡을 통해 앞서 작성했던 `app-deploy.yaml` 워크플로우를 호출한다. 그리고 각종 파라미터와 시크릿을 전달한다.

여기까지 한 후, 터미널에서 아래 명령어를 실행시켜 보자. 여기서는 [GitHub CLI][gh cli]를 사용해서 방금 작성했던 `workflow_dispatch` 이벤트를 호출한다.

```powershell
gh workflow run "App Deploy Dispatch" \
  --repo devkimchi/ASWA-AutoPilot-Sample
```

> **NOTE**: GitHub CLI를 사용해서 위 명령어를 사용하려면 이미 로그인이 되어 있어야 한다. `gh auth status` 명령어를 통해 로그인 여부를 확인해 볼 수 있고, 만약 로그인 되어 있지 않다면 `gh auth login` 명령어를 통해 로그인하면 된다.

이렇게 하면 리소스 프로비저닝 이후 앱을 배포할 수 있다.


### 깃헙 액션 `push`/`pull_request` 플로우 작성 ###

이제 코드 변경이 생길 때 마다 앱을 지속적으로 배포를 하고 싶다면, `push` 이벤트 및 `pull_request` 이벤트에 대응하는 깃헙 액션 워크플로우를 작성해야 한다. 이미 `workflow_call` 워크플로우를 작성해 뒀기 때문에 이를 활용하기만 하면 된다.

```yaml
# main-app.yaml
name: 'App Deploy'

on:
  push:
    branches:
    - main

  pull_request:
    types:
    - opened
    - synchronize
    - reopened
    - closed
    branches:
    - main

jobs:
  call_app_deploy:
    if: github.event_name == 'push' || (github.event_name == 'pull_request' && github.event.action != 'closed')
    uses: devkimchi/ASWA-AutoPilot-Sample/.github/workflows/app-deploy.yaml@main
    with:
      job_name: 'Deploy Static Web App'
      app_location: 'app'
      api_location: 'api'
      output_location: 'wwwroot'
    secrets:
      gh_token: ${{ secrets.GITHUB_TOKEN }}
      aswa_token: '${{ secrets.AZURE_STATIC_WEB_APPS_API_TOKEN }}'
```

이를 바탕으로 이제 새로운 코드 변경 상황을 깃헙 리포지토리에 반영한다면, 곧바로 이 `main-app.yaml` 워크플로우가 실행될 것이다.


## 한번에 엮기 ##

그런데, 여전히 "한번에 탁"하고 되지는 않는다. 우선 리소스 프로비저닝을 하고, 별도로 명령어를 실행시켜 앱을 배포하는 구조인데, 이 둘을 한꺼번에 엮을 수 있는 방법은 없을까? 현실적으로 가장 가능한 대안은 앞서 작성했던 `workflow_dispatch` 이벤트를 활용해 리소스 프로비저닝을 하고 앱도 배포하는 것이다. 그렇다면, 이 작업을 시작해 보자.


### 리소스 프로비저닝 워크플로우 작성 ##

이미 리소스 프로비저닝을 위한 bicep 파일을 작성해 놓았으므로 이를 실행만 시키면 된다. 아래 `workflow_call` 워크플로우를 살펴보자. `resource_name` 파라미터를 제외하고는 모두 기본값을 지정해 놓았다.

```yaml
# resource-provision.yaml
name: 'Resource Provision'

on:
  workflow_call:
    inputs:
      job_name:
        type: string
        required: false
        default: 'Provision Resources'
      resource_name:
        type: string
        required: true
      cosdba_location:
        type: string
        required: false
        default: 'Korea Central'
      sttapp_location:
        type: string
        required: false
        default: 'East Asia'
    secrets:
      pa_token:
        required: true
      az_credentials:
        required: true

jobs:
  deploy:
    name: ${{ inputs.job_name }}

    runs-on: ubuntu-latest

    steps:
    - name: Checkout the repo
      uses: actions/checkout@v2
      with:
        submodules: true

    - name: Login to Azure
      uses: Azure/login@v1
      with:
        creds: ${{ secrets.az_credentials }}

    - name: Provision resources
      id: provisioned
      shell: pwsh
      run: |
        cd ./infra

        $result = ./Provision-Resources.ps1 `
          -ResourceName "${{ inputs.resource_name}}" `
          -Location "${{ inputs.cosdba_location }}" `
          -CosmosDbPrimaryRegion "${{ inputs.cosdba_location }}" `
          -StaticWebAppLocation "${{ inputs.sttapp_location }}" `
          -TargetScope Subscription

        $provisioned = $result | ConvertFrom-Json
        $deploymentKey = $provisioned.properties.outputs.sttappDeploymentKey.value

        echo "::add-mask::$deploymentKey"
        echo "::set-output name=sttapp_deploymentkey::$deploymentKey"

    - name: Update GitHub repository secrets
      uses: hmanzur/actions-set-secret@v2.0.0
      with:
        token: ${{ secrets.pa_token }}
        repository: ${{ github.repository }}
        name: AZURE_STATIC_WEB_APPS_API_TOKEN
        value: ${{ steps.provisioned.outputs.sttapp_deploymentkey }}
```

애저에 리소스를 프로비저닝 하기 위해서는 [애저 CLI][az cli]를 사용하는데, 이를 위해 파워셸 스크립트를 [`Provision-Resources.ps1`][gh sample provision] 이라는 이름으로 하나 작성해 뒀다. 이 스크립트를 실행하면 정적 웹 앱의 배포 키를 받을 수 있으므로, 마지막 액션에서 이를 깃헙 시크릿으로 저장한다.

여기까지 해서 리소스 프로비저닝을 위한 워크플로우를 작성했다.


### 오토파일럿 워크플로우 작성 ###

이제 오토파일럿 워크플로우를 통해 리소스 프로비저닝과 앱 배포를 실행시켜야 한다. 먼저 오토파일롯 용도의 `workflow_dispatch` 워크플로우를 작성한다. 사용자 입력을 최소한으로 유지해야 하므로 가능한 한 입력 파라미터에 수동 입력 필드를 줄이고 선택지를 넣는 식으로 진행한다.

```yaml
# main-autopilot.yaml
name: 'Autopilot'

on:
  workflow_dispatch:
    inputs:
      resourceName:
        type: string
        description: 'Resource name'
        required: true
      location:
        type: choice
        description: 'Cosmos DB location. Default is "Korea Central"'
        required: false
        options:
        ...
        - 'Central US'
        - 'East Asia'
        - 'East US 2'
        - 'Korea Central'
        - 'West Europe'
        - 'West US 2'
        ...
        default: 'Korea Central'
      staticWebAppLocation:
        type: choice
        description: 'Static Web App location. Default is "East Asia".'
        required: false
        options:
        - 'Central US'
        - 'East Asia'
        - 'East US 2'
        - 'West Europe'
        - 'West US 2'
        default: 'East Asia'

jobs:
  call_resource_provisioning:
    uses: devkimchi/ASWA-AutoPilot-Sample/.github/workflows/resource-provision.yaml@main
    with:
      job_name: 'Provision Resources'
      resource_name: ${{ github.event.inputs.resourceName }}
      cosdba_location: ${{ github.event.inputs.location }}
      sttapp_location:  ${{ github.event.inputs.staticWebAppLocation }}
    secrets:
      pa_token: ${{ secrets.PA_TOKEN }}
      az_credentials: ${{ secrets.AZURE_CREDENTIALS }}

  call_workflow_dispatch:
    needs: call_resource_provisioning
    name: 'Deploy Static Web App'

    runs-on: ubuntu-latest

    steps:
    - name: Invoke workflow with inputs
      uses: benc-uk/workflow-dispatch@v1
      with:
        workflow: 'App Deploy Dispatch'
        token: ${{ secrets.PA_TOKEN }}
        inputs: '{ "jobName": "Deploy Static Web App", "appLocation": "app", "apiLocation": "api", "outputLocation": "wwwroot" }'
```

위의 내용을 보면 첫번째 잡을 통해 리소스를 프로비저닝하는데, 두번째 잡에서는 앞서 작성해 놓았던 또다른 `workflow_dispatch` 워크플로우인 `main-dispatch.yaml` 파일을 호출한다. 그런데, 굳이 `workflow_call` 이벤트 대신 `workflow_dispatch` 이벤트를 호출하는 이유가 있을까?

`workflow_call` 이벤트를 통해 `app-deploy.yaml` 워크플로우를 호출해도 워크플로우 자체는 정상적으로 작동한다. 하지만, 이렇게 하면 앞서 리소스 프로비저닝시 생성했던 깃헙 시크릿 값에 접근할 수가 없는데, 이는 동일한 파이프라인 안에서 생성된 것이기 때문이다. 동일한 파이프라인 안에서 리포지토리와 관련한 뭔가가 생성됐다거나 변경되었다면 이를 동일한 컨텍스트 안에서는 인식하지 못한다. 따라서, 다른 컨텍스트로 전환시켜야 하는데, 그 때 `workflow_dispatch` 이벤트를 사용하면 된다.

이렇게 별도의 `workflow_dispatch` 워크플로우를 호출하게 되면 이는 다른 파이프라인에서 실행이 되기 때문에, 즉 컨텍스트가 달라지기 때문에 이제는 다른 컨텍스트에서 설정한 배포 키 값을 인식할 수 있다.

여기까지 해서 오토파일럿 기능이 완성되었다. 실제로 이를 실행시키려면 새로운 깃헙 시크릿을 아래와 같이 추가해야 한다. `AZURE_CREDENTIALS` 값과 `PA_TOKEN` 값을 추가한다.

> 기존에 추가했던 `AZURE_STATIC_WEB_APPS_API_TOKEN` 시크릿은 삭제해도 좋다. 만약 삭제하지 않는다면 오토파일럿 과정에서 새로운 값으로 변경된다.

![깃헙 액션 시크릿 - 애저 & 깃헙][image-06]


### `AZURE_CREDENTIALS` 값 저장 ###

워크플로우 안에서 애저 CLI를 사용하기 때문에 이 로그인 정보를 필요로 한다. 아래 명령어를 통해 값을 생성한 후 깃헙 시크릿으로 저장한다.

```powershell
az ad sp create-for-rbac \
  --name "myApp" \
  --role contributor \
  --sdk-auth
```

> 이와 관련해서 좀 더 자세한 내용이 필요하다면 [이 문서][az cli login]를 참조한다.


### `PA_TOKEN` 값 저장 ###

한 `workflow_dispatch ` 워크플로우 안에서 애저 정적 웹 앱의 배포 키를 저장하기 위해서, 그리고 다른 `workflow_dispatch` 워크플로우를 실행시키기 위해서 깃헙의 개인 액세스 토큰(PAT) 값이 필요하다. [이 문서][gh pat]를 이용해 PAT 값을 생성하고 이를 `PA_TOKEN` 시크릿에 저장한다.


### 오토파일럿 실행 ###

위의 절차가 모두 끝났다면, 이제 아래와 같이 오토파일럿 화면으로 넘어가서 실행시켜 보자.

![깃헙 액션 오토파일럿][image-07]

1. "Actions" 탭 이동
2. "Autopilot" 탭 클릭
3. "Run workflow" 버튼 클릭
4. 리소스 이름 입력
5. Cosmos DB 지역 선택
6. 애저 정적 웹 앱 지역 선택
7. "Run workflow" 버튼 클릭

이 절차가 모두 끝나면 아래와 같이 리소스 프로비저닝 및 앱 배포가 한번에 끝난다. 코스모스 DB에 저장된 데이터가 없기 때문에 아래 그림과 같이 "No product found" 라는 메시지가 보이는 것이 정상이다.

![애저 정적 웹 앱][image-08]

이렇게 해서 애저 리소스 프로비저닝 및 앱 배포와 관련한 오토파일럿 기능을 구현해 보았다.


## 풀어야 할 숙제 ##

위와 같이 한 번에 해결할 수 있긴 하지만, 앞으로 몇 가지 좀 더 풀어야 할 것들이 있다.

1. [애저 CLI][az cli] 및 [깃헙 PAT][gh pat] 수동 저장

    이 부분은 현재 애저 CLI를 통해 직접 정적 웹 앱을 배포할 수 있는 기능이 없기 때문인데, 명세는 있지만 아직 구현이 덜 된 것으로 보인다. 따라서, 곧 가능하지 않을까 예상한다. 이게 가능해 진다면 깃헙 PAT를 이용해 별도의 `workflow_dispatch` 이벤트를 호출할 필요가 없어진다.

2. "Deploy to Azure" 버튼 활용

    현재 애저 bicep에서 [배포 스크립트][az bicep deploymentscript] 기능을 활용해 애저 CLI 명령어를 실행시킬 수 있다. 단, 앞서 언급한 바와 같이 애저 CLI에서 정적 웹 앱을 배포할 수 있다는 것을 전제로 한다.

---

지금까지 애저 리소스 프로비저닝 및 앱 배포와 관련한 오토파일럿 기능을 [깃헙 액션][gh actions]의 다양한 이벤트 트리거와 [애저 bicep][az bicep]을 활용해서 구현해 보았다. 이를 활용하면 PoC 애플리케이션을 만들어서 고객에게 데모를 보여준다거나 할 때, 누구든 깃헙 리포지토리만 접근할 수 있다면 별도의 지식이 없이도 스스로 리소스를 만들고 앱을 배포할 수 있게 될 것이다. [다음 포스트][post 2]에서는 실제로 애저 bicep의 [배포 스크립트][az bicep deploymentscript] 기능을 활용해서 오토파일럿을 구현해 보기로 한다.


[image-01]: /2022/03/azure-apps-autopilot-01.png
[image-02]: /2022/03/azure-apps-autopilot-02.png
[image-03]: /2022/03/azure-apps-autopilot-03.png
[image-04]: /2022/03/azure-apps-autopilot-04.png
[image-05]: /2022/03/azure-apps-autopilot-05.png
[image-06]: /2022/03/azure-apps-autopilot-06.png
[image-07]: /2022/03/azure-apps-autopilot-07.png
[image-08]: /2022/03/azure-apps-autopilot-08.png


[post 1]: /ko/2022/03/11/azure-apps-autopilot/
[post 2]: /ko/2022/04/06/azure-bicep-deployment-script/

[az bicep]: https://docs.microsoft.com/ko-kr/azure/azure-resource-manager/bicep/overview?tabs=bicep&WT.mc_id=dotnet-59944-juyoo
[az bicep deploymentscript]: https://docs.microsoft.com/ko-kr/azure/azure-resource-manager/bicep/deployment-script-bicep?WT.mc_id=dotnet-59944-juyoo
[az bicep module]: https://docs.microsoft.com/ko-kr/azure/azure-resource-manager/bicep/modules?WT.mc_id=dotnet-59944-juyoo
[az bicep targetscope]: https://docs.microsoft.com/ko-kr/azure/azure-resource-manager/bicep/deploy-to-subscription?tabs=azure-cli&WT.mc_id=dotnet-59944-juyoo#set-scope

[az cli]: https://docs.microsoft.com/ko-kr/cli/azure/what-is-azure-cli?WT.mc_id=dotnet-59944-juyoo
[az cli login]: https://github.com/Azure/login#configure-a-service-principal-with-a-secret

[az cosdba]: https://docs.microsoft.com/ko-kr/azure/cosmos-db/introduction?WT.mc_id=dotnet-59944-juyoo
[az cosdba sql]: https://docs.microsoft.com/ko-kr/azure/cosmos-db/choose-api?WT.mc_id=dotnet-59944-juyoo#coresql-api
[az cosdba serverless]: https://docs.microsoft.com/ko-kr/azure/cosmos-db/serverless?WT.mc_id=dotnet-59944-juyoo

[az fncapp]: https://docs.microsoft.com/ko-kr/azure/azure-functions/functions-overview?WT.mc_id=dotnet-59944-juyoo

[az swa]: https://docs.microsoft.com/ko-kr/azure/static-web-apps/overview?WT.mc_id=dotnet-59944-juyoo
[az swa plan]: https://docs.microsoft.com/ko-kr/azure/static-web-apps/plans?WT.mc_id=dotnet-59944-juyoo

[blazor wasm]: https://docs.microsoft.com/ko-kr/aspnet/core/blazor/host-and-deploy/webassembly?WT.mc_id=dotnet-59944-juyoo

[dave]: https://twitter.com/davrous

[gh cli]: https://cli.github.com/

[gh pat]: https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token

[gh sample]: https://github.com/devkimchi/ASWA-AutoPilot-Sample
[gh sample provision]: https://github.com/devkimchi/ASWA-AutoPilot-Sample/blob/main/infra/Provision-Resources.ps1

[gh actions]: https://github.com/features/actions
[gh actions aswa]: https://github.com/marketplace/actions/azure-static-web-apps-deploy
[gh actions events push]: https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#push
[gh actions events pr]: https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#pull_request
[gh actions events wc]: https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#workflow_call
[gh actions events wd]: https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#workflow_dispatch
