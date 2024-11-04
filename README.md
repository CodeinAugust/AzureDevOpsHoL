# Azure DevOps CI 구축 핸즈온랩

## 소개

이 핸즈온랩은 Azure DevOps와 Azure Portal을 사용하여 CI(Continuous Integration) 파이프라인을 구축하는 방법을 다룹니다.   
실습을 통해 가상 머신(VM)을 설정하고, Azure DevOps 환경에서 Self-hosted Agent를 구성한 후 Node.js 애플리케이션을 배포하는 CI 파이프라인을 구현합니다.   
또한, Microsoft Security DevOps 확장을 설치하여 보안 강화 기능을 추가합니다.

### 목적
이 핸즈온랩의 목적은 Azure DevOps 환경에서 자체 호스팅된 에이전트를 사용하는 CI 파이프라인을 구축하는 과정을 익히는 것입니다.  
실습을 통해 다음의 기술을 학습할 수 있습니다:
- Azure 가상 머신 생성 및 구성
- Azure Container Registry 설정
- Azure DevOps 프로젝트 생성 및 에이전트 풀 설정
- Node.js 애플리케이션에 대한 파이프라인 설정 및 실행
- Microsoft Security DevOps 확장 추가

---

## 실습 단계

### 1. Azure Portal에 접속하여 가상 머신 생성
- [Azure Portal](https://portal.azure.com)에 접속합니다.
- 새로운 Virtual Machine을 생성합니다.
  - VM 이름: `VM-DevOps-Private-Agent`
  - 이미지: `Windows Server 2019 Datacenter - x64 Gen2`
  - 크기: `Standard_D4s_v3 - 4 vcpus`
  - Administrator 계정의 Username과 Password를 설정합니다. 이 정보는 나중에 VM 접속 시 필요합니다.

### 2. VM에 접속
- VM 접속 방법 중 선호하는 방식을 선택합니다. 이번 실습에서는 **Bastion**을 사용합니다.

### 3. Azure Container Registry 생성
- Azure Portal에서 **Azure Container Registries**를 생성합니다.
  - Registry 이름: `ACRAzureDevOpsCI`

### 4. Azure DevOps에 접속하여 조직 및 프로젝트 생성
- [Azure DevOps](https://dev.azure.com)에 접속합니다.
- 새로운 조직(Organization)을 생성합니다.
- 새로운 프로젝트를 생성합니다. 
  - 프로젝트 Visibility: `Private`

### 5. Azure DevOps에서 에이전트 풀 설정
- **Project Settings**에서 **Pipeline > Agent pools**를 선택합니다.
- **Add pool**을 선택하고, **Pool type**을 **Self-hosted**로 설정합니다.
  - **Grant access permission to all pipelines** 옵션을 선택합니다.

### 6. Personal Access Token(PAT) 생성
- 상단 메뉴에서 **User settings > Security**로 이동하여 **Personal access tokens**을 선택합니다.
- **New Token**을 생성합니다.
  - **더보기**를 선택하고, **Agent Pools**에서 **Read & manage**를 선택한 뒤 **Create**합니다.
- 생성된 PAT 값을 복사해둡니다.

### 7. Azure DevOps에서 서비스 연결 설정
- **Project Settings > Pipeline > Service connections**를 선택합니다.
- **Docker Registry**를 선택하고, 다음과 같이 설정합니다:
  - Authentication Type: **Service Principal**
  - Subscription: *My Subscription*
  - Azure container registry: *My ACR*
  - Service connection name: `ACR`
  - **Grant access permission to all pipelines** 옵션을 선택합니다.

### 8. Self-hosted Agent 다운로드 및 설정
- **Pipeline > Agent pools**에서 **Agents** 탭 내 **New agent**를 클릭합니다.
- 에이전트 파일 다운로드 링크를 복사하여 VM에 접속한 뒤 다운로드합니다.
- VM의 PowerShell에서 다음 명령어를 실행하여 에이전트를 설치합니다:
```powershell
PS C:\> mkdir agent ; cd agent
PS C:\agent> Add-Type -AssemblyName System.IO.Compression.FileSystem ; [System.IO.Compression.ZipFile]::ExtractToDirectory("$HOME\Downloads\vsts-agent-win-x64-3.246.0.zip", "$PWD")
```
- 에이전트 설정을 위해 다음 명령어를 입력합니다:
```powershell
PS C:\agent> .\config.cmd
```
- 에이전트를 활성화하기 위해 다음 명령어를 실행합니다:
```powershell
PS C:\agent> .\run.cmd
```

### 9. 코드 가져오기 및 파이프라인 생성
- Azure DevOps의 **Repos > Files**에서 **Import**를 클릭하여 코드를 가져옵니다.
- **Repository type**으로 **Git**을 선택하고, Clone URL에 다음을 입력합니다:
  - `https://github.com/CodeinAugust/AzureDevOpsHoL`

### 10. 파이프라인 생성 및 구성
- **Pipelines** 탭에서 **Create Pipeline**을 선택합니다.
- 가져온 **Repository**를 선택한 뒤, **Node.js**를 선택합니다.
- YAML 파일을 작성합니다.
  - 참고 링크: [Azure DevOps CI 예제 YAML 파일](https://github.com/CodeinAugust/AzureDevOpsHoL/blob/main/pipelines/azure-pipelines.yml)
```yml
trigger:
- main

pool:
  name: 'PrivateAgent001'

steps:
- checkout: self

- task: NodeTool@0
  inputs:
    versionSpec: '20.x'
  displayName: 'Install Node.js'

- script: |
    cd source
    npm install
    npm test
  displayName: 'Install dependencies and run tests'

- task: PublishBuildArtifacts@1
  inputs:
    PathtoPublish: '$(Build.SourcesDirectory)/source' 
    ArtifactName: 'drop'
    publishLocation: 'Container'
  displayName: 'Publish build artifacts'
  condition: succeededOrFailed()

- task: Docker@2
  inputs:
    containerRegistry: 'ACR'
    repository: 'nodejs-aks-sample'
    command: 'buildAndPush'
    Dockerfile: '$(Build.SourcesDirectory)/source/Dockerfile'
    tags: |
      $(Build.BuildId)
  displayName: 'Build and push Docker image'
```

### 11. 파이프라인 실행 및 결과 확인
- Agent 상태와 Job 상태를 확인합니다.
- 모든 Job이 완료된 후 생성된 Artifact를 확인합니다.

### 12. Microsoft Security DevOps 확장 설치
- 상단 우측의 **Marketplace** 아이콘을 클릭합니다.
- **Microsoft Security DevOps**를 검색하고, **Get it free**를 선택하여 확장을 설치합니다.
- 사용 중인 Azure DevOps Organization을 선택하고 확장을 추가합니다.

> 💡 **Note:** 더욱 강력한 보안 기능을 사용하기 위해서는 [GitHub Advanced Security for Azure DevOps](https://azure.microsoft.com/ko-kr/products/devops/github-advanced-security) 를 추천합니다.
> Azure DevOps 플랫폼에 제공되는 보안 테스트 도구 모음, 비밀 검사, 종속성 검사, 그리고 코드 검사를 할 수 있습니다.

### 13. 파이프라인 업데이트
- Microsoft Security DevOps 확장 설치 이후 파이프라인을 업데이트 해봅니다.

```yml
trigger:
- main

pool:
  name: 'PrivateAgent001'

steps:
- checkout: self

- task: NodeTool@0
  inputs:
    versionSpec: '20.x'
  displayName: 'Install Node.js'

- script: |
    cd source
    npm install
    npm run build
  displayName: 'Install dependencies and build'

# ESLint - JavaScript/Node.js 코드 분석
- script: |
    cd source
    npx eslint . --ext .js,.jsx,.ts,.tsx 
  displayName: 'Run ESLint for static code analysis'

- task: Docker@2
  inputs:
    containerRegistry: 'ACR'
    repository: 'nodejs-aks-sample'
    command: 'buildAndPush'
    Dockerfile: '$(Build.SourcesDirectory)/source/Dockerfile'
    tags: |
      $(Build.BuildId)
  displayName: 'Build and push Docker image'

- task: PublishBuildArtifacts@1
  inputs:
    PathtoPublish: '$(Build.SourcesDirectory)/source/dist'  
    ArtifactName: 'drop'
    publishLocation: 'Container'
  displayName: 'Publish build artifacts'

```


### 13. 추가 기능 테스트
- 파이프라인에 사용 가능한 기능들을 추가한 뒤 테스트합니다.

---

## 마무리
이 핸즈온랩을 통해 Azure DevOps 환경에서 CI 파이프라인을 설정하고, 자체 호스팅된 에이전트를 활용한 Node.js 애플리케이션의 빌드 및 배포 과정을 학습하였습니다.  
또한, 핸즈온랩을 끝마치고 나서는 Azure 내 리소스를 정리하여 불필요한 비용이 발생하지 않도록 유의하시기 바랍니다.

# 작성자
[어거스트 리]( https://github.com/CodeinAugust)

