---
title: "CI/CD 파이프라인 안에서 애저 펑션 OpenAPI 문서 자동 생성하기"
slug: generating-openapi-document-from-azure-functions-within-cicd-pipeline
description: "이 포스트에서는 CI/CD 파이프라인 안에서 애저 펑션 앱을 빌드한 후 자동으로 OpenAPI 문서를 생성하는 방법에 대해 알아봅니다."
date: "2022-02-23"
author: Justin-Yoo
tags:
- azure
- azure-functions
- openapi
- github-actions
cover: /2022/02/generating-openapi-document-from-azure-functions-within-cicd-pipeline-00.png
fullscreen: true
---

[애저 펑션][az fncapp] 앱에서 자동으로 OpenAPI 문서를 생성해 주는 [확장 기능][az fncapp ext openapi]을 이용하다 보면 흔히 볼 수 있는 상황이 있다. 그 중 하나가 바로 OpenAPI 문서를 펑션앱을 빌드한 직후 곧바로 생성해서 [파워 플랫폼][pp] 혹은 [애저 API 관리자][az apim] 같은 다른 서비스와 바로 연동하는 것이다. 이 과정을 CI/CD 파이프라인 안에서 모두 처리를 하고 싶다면 어떻게 해야 할까?

* ***CI/CD 파이프라인 안에서 애저 펑션 OpenAPI 문서 자동 생성하기*** 👈
* [CI/CD 파이프라인 안에서 애저 펑션 OpenAPI 문서를 애저 API 관리자로 자동 발행하기][post 2]

이 포스트에서는 깃헙 액션을 이용해서 애저 펑션앱을 빌드하고, 곧바로 OpenAPI 문서를 생성한 후, 아티팩트로 저장하는 방법에 대해 알아보기로 한다.

> **NOTE**: 여기서는 [애저 펑션 OpenAPI 확장 기능][az fncapp ext openapi] 리포지토리에서 제공하는 예제 앱을 이용하기로 한다.

우선 CI/CD 파이프라인 상에서 [애저 펑션 코어 툴][az fncapp core tools]을 설치한다. 사용하고자 하는 운영체제마다 설치 방법이 살짝 다르니 문서를 보고 따라서 설치하면 된다. 그 이후에 로컬에서 애저 펑션 앱을 실행시키려면 보통 아래와 같이 명령어를 실행시킨다.

```powershell
func start
```

터미널 같은 셸 환경이라면 세션 하나를 열어두고 거기서 실행시킨 후 터미널을 하나 더 열어놓고 거기서 작업을 하면 되기 때문에 큰 문제가 되지 않는다. 그런데, CI/CD 환경이라면 살짝 얘기가 다르다. 세션을 한 번에 여러 개 만들어서 사용할 수 없기 때문이다. 따라서 애저 펑션 앱에서 OpenAPI 문서를 생성하려면 애저 펑션 앱을 우선 백그라운드로 실행을 시켜야 한다.


## Bash 셸에서 실행시키기 ##

bash 셸에서는 `func start &`와 같이 `func start` 명령어 뒤에 `&`를 붙여주면 그만이다 (line #2). 따라서, 대략 아래와 같이 명령어를 실행시키면 된다.

```powershell
# Run the function app in background
func start &

# Send request to the function app and save it to swagger.json
curl http://localhost:7071/api/swagger.json > swagger.json

# Read swagger.json
cat swagger.json
```

아주 간단하다.


## 파워셸에서 실행시키기 ##

파워셸에서 명령어를 백그라운드로 실행시키기 위해서는 `Start-Process` 커맨들릿을 `-NoNewWindow` 스위치와 함께 사용해야 한다 (line #2). 아래는 윈도우 운영체제 이외의 파워셸 환경에서 애저 펑션 앱을 백그라운드로 실행시키기 위한 명령어이다.

```powershell
# Run the function app in background
Start-Process -NoNewWindow func @("start")

# Send request to the function app and save it to swagger.json
Invoke-RestMethod -Method Get -Uri http://localhost:7071/api/swagger.json | ConvertTo-Json -Depth 100 | Out-File -FilePath swagger.json -Force

# Read swagger.json
Get-Content -Path swagger.json
```

반면에 윈도우 운영체제에서는 `Start-Process` 커맨들릿이 직접 `func` 명령을 수행할 수 없기 때문에 아래와 같이 `Start-Process` 이전에 명령어 전처리가 필요하다 (line #2, 5).

```powershell
# Change the function app runner from .ps1 to .cmd
$func = $(Get-Command func).Source.Replace(".ps1", ".cmd")

# Run the function app in background
Start-Process -NoNewWindow "$func" @("start")

# Send request to the function app and save it to swagger.json
Invoke-RestMethod -Method Get -Uri http://localhost:7071/api/swagger.json | ConvertTo-Json -Depth 100 | Out-File -FilePath swagger.json -Force

# Read swagger.json
Get-Content -Path swagger.json
```

위와 같은 방식으로 파워셸에서 처리할 수 있다.


## 깃헙 액션 워크플로우에 적용시키기 ##

위의 내용을 이제는 CI/CD 파이프라인 상에서 애저 펑션 앱을 백그라운드로 실행시키고 OpenAPI 문서를 저장할 수 있다. 아래는 깃헙 액션 워크플로우에서 활용하는 예이다. 불필요한 부분은 삭제하고 펑션 앱을 빌드하고 OpenAPI 문서를 저장하는 부분만 남겨뒀다 (line #26-33).

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
        pushd MyFunctionApp

        Start-Process -NoNewWindow func @("start","--verbose","false")
        Start-Sleep -s 60

        Invoke-RestMethod -Method Get -Uri http://localhost:7071/api/swagger.json | ConvertTo-Json -Depth 100 | Out-File -FilePath outputs/swagger.json -Force

        Get-Content -Path outputs/swagger.json

        popd
```

---

지금까지 깃헙 액션 워크플로우 안에서 애저 펑션을 백그라운드 프로세스를 이용해 로컬로 실행시키고 OpenAPI 문서를 호출하여 저장하는 방법에 대해 알아보았다. 이렇게 저장한 OpenAPI 문서는 다른 액션을 통해 아티팩트로 저장한 후 [파워 플랫폼][pp] 또는 [애저 API 관리자][az apim]와 같은 다른 서비스에 통합시키면 된다. [다음 포스트][post 2]에서는 실제로 CI/CD 파이프라인에서 생성한 OpenAPI 문서를 [애저 API 관리자][az apim]에 직접 발행하는 방법에 대해 알아보기로 하자.


[post 1]: /ko/2022/02/23/generating-openapi-document-from-azure-functions-within-cicd-pipeline/
[post 2]: /ko/2022/03/02/publishing-openapi-doc-from-azfunc-to-apim-within-cicd-pipeline/

[pp]: https://powerplatform.microsoft.com/ko-kr/?WT.mc_id=dotnet-58527-juyoo

[az fncapp]: https://docs.microsoft.com/ko-kr/azure/azure-functions/functions-overview?WT.mc_id=dotnet-58527-juyoo
[az fncapp core tools]: https://docs.microsoft.com/ko-kr/azure/azure-functions/functions-run-local?WT.mc_id=dotnet-58527-juyoo
[az fncapp ext openapi]: https://aka.ms/azfunc-openapi

[az apim]: https://docs.microsoft.com/ko-kr/azure/api-management/api-management-key-concepts?WT.mc_id=dotnet-58527-juyoo
