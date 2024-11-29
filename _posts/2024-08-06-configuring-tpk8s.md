---
layout: post
title: Tanzu Platform for Kubernetes 구성하기 (with EKS)
categories: [tp, tpk8s]
---

지난 5월 21일 VMware Tanzu Platform이 릴리즈 되었고, 2024년 8월 현재 SaaS 기반의 Tanzu Platform을 제공하고 있다. Tanzu Platform에 
대한 특징은 [VMware Tanzu 블로그의 링크](https://tanzu.vmware.com/content/blog/introducing-vmware-tanzu-platform)를 참고하기 바란다. 

여기서는 AWS 환경에서 EKS로 Tanzu Platform을 구성하고 앱을 배포하는 과정에 대해서 설명한다.

## 0. 사전 구성
### 1) 프로젝트 생성
Tanzu Platform에 접속해서 좌측 하단의 "Setup & Configuration" > "Projects"를 클릭한다. "NEW PROJECT"를 클릭하여, 원하는 프로젝트를 생성한다. 만약 버튼을 클릭할 수 없다면 권한이 없는 것이므로 [VCS(VMWare Cloud Services) 콘솔](https://console.cloud.vmware.com/)에서 부여받은 권한을 확인한다. VCS 메뉴 우측 상단의 "My Account"를 클릭하고, "Identity & Access Management" > "Active Users"에서 다음과 같이 나의 권한을 확인할 수 있다. 단, 부여받은 권한은 사용자 별로 상이할 수 있다.

![configuring-tpk8s 1](https://raw.githubusercontent.com/haewons-tanzu/haewons-contents/master/static/img/_posts/2024-08-06-configuring-tpk8s/1.png)

### 2) Tanzu CLI 및 플러그인 설치
Tanzu Platform 접속을 위해서는 v1.3.x 이상의 Tanzu CLI 설치가 필요하다. 2024년 8월 현재 v1.4.x가 최신 버전이며, 맥북 기준으로는 brew를 사용하여 다음과 같이 설치할 수 있다.
```bash
$ brew update
$ brew install vmware-tanzu/tanzu/tanzu-cli
```
만약 윈도우즈나 Linux 등 다른 OS를 사용한다면 [Installing and Using VMware Tanzu CLI v1.4.x](https://docs.vmware.com/en/VMware-Tanzu-CLI/1.4/tanzu-cli/index.html)
에서 설치 방법을 참조하여 Tanzu CLI를 설치한다.

설치가 완료되면 다음과 같이 Tanzu CLI가 제대로 설치되었는지 확인한다.
```
$ tanzu version
version: v1.4.0
buildDate: 2024-07-18
sha: af660c8b
arch: arm64
```

또한, Tanzu Platform 사용을 위해 다음과 같은 플러그인들이 필요하다. 먼저 다음 명령어를 실행해서 플러그인들의 목록을 확인한다. 2024년 8월 현재 플러그인 버전 정보는 다음과 같다.
```
$ tanzu plugin group search 
  GROUP                           DESCRIPTION                                           LATEST
  vmware-tanzu/app-developer      Plugins for Application Developer for Tanzu Platform  v0.1.5
  vmware-tanzu/platform-engineer  Plugins for Platform Engineer for Tanzu Platform      v0.1.6
  vmware-tanzucli/essentials      Essential plugins for the Tanzu CLI                   v1.0.0
  vmware-tap/default              Plugins for TAP                                       v1.11.1
  vmware-tkg/default              Plugins for TKG                                       v2.5.1
  vmware-tmc/default              Plugins for TMC                                       v1.0.0
  vmware-vsphere/default          Plugins for vSphere                                   v8.0.3

Note: To view all plugin group versions available, use 'tanzu plugin group search --show-details'.
```

이 중 Tanzu Platform에서 필요로 하는 플러그인을 다음 명령어를 이용해서 설치한다. 버전을 명시하지 않으면 최신 버전의 플러그인이 설치된다.

- vmware-tanzucli/essentials : Tanzu 필수 플러그인으로, 텔레메트리 관련 명령어를 수행할 수 있다.
  ```bash
  $ tanzu plugin install --group vmware-tanzucli/essentials 
  ```
- vmware-tanzu/app-developer: Tanzu Platform을 이용하는 앱 개발자가 사용하는 명령어들이 포함된 플러그인이다.
  ```bash
  $ tanzu plugin install --group vmware-tanzu/app-developer 
  ```
- vmware-tanzu/platform-engineer: Tanzu Platform을 이용하는 플랫폼 엔지니어가 사용하는 명령어들이 포함된 플러그인이다.
  ```bash
  $ tanzu plugin install --group vmware-tanzu/platform-engineer 
  ```

각각의 플러그인 설치가 완료되면, 설치된 플러그인 목록을 다음 명령어를 사용해서 조회한다.
```bash
$ tanzu plugin list 
```

## 1. 계정 및 크리덴셜 구성
### 1) 계정 생성
좌측 하단에 위치한 Tanzu Platform 메뉴에서 "Setup & Configuration" > "Cloud Accounts"를 클릭한다. "NEW ACCOUNT"를 클릭하고, "AWS Web Services"를 선택한다. 이후 나타나는 옵션은 다음과 같이 선택한다.
- Onboarding Method: Single account 
- Account Type: Commercial 

이후 나타나는 Account Information에서 연결할 AWS 계정 정보를 입력한다.
- Name: 계정 명
- Account ID: AWS 계정 ID로, AWS 콘솔 우측 상단에서 확인 (xxxx-xxxx-xxxx 형식)

"Next" 버튼을 클릭해서 AWS 계정과 연결한다. Tanzu Platform의 가이드에 따라 IAM Role을 AWS 콘솔에서 생성한다. Role이 생성되면 ARN 값을 AWS 콘솔에서 복사하여 Tanzu Platform UI에 입력한다. 

계정 생성이 완료되면 "OK" 상태임을 확인하고 다음 단계로 이동한다.

### 2) EKS Lifecycle Management용 크리덴셜 생성
좌측 하단에 위치한 Tanzu Platform 메뉴에서 "Setup & Configuration" > "Credentials"를 클릭한다. "ADD CREDENTIAL"을 클릭하고 "Tanzu Platform for Kubernetes"를 선택한다. 이후 나타나는 화면에서 다음과 같이 선택한다.
- Credential Type: Lifecycle Management
- Cluster Type: AWS EKSㅣ

"Next" 버튼을 클릭해서 Credential 명을 지정한다. 다시 Next 버튼을 클릭하고, Credential 생성을 위한 Cloudformation Stack을 생성한다. Cloudformation Stack은 AWS 리소스를 자동으로 구성해 주는 스크립트이다. AWS CLI를 이용하는 방법과 AWS 콘솔 UI를 이용하는 방법이 있지만, 여기서는 AWS CLI를 별도로 설치하지 않고 UI를 이용해서 구성하는 방법에 대해 설명한다. 

"AWS Console UI"를 선택하면 Cloudformation Stack을 생성하는 링크로 연결된다. 

> **NOTE** 
> 연결된 AWS 콘솔 UI 링크에서 내가 EKS 관리를 원하는 리전인지 확인한다. 크리덴셜은 리전 단위로 생성되기 때문에, 이 과정에서 잘못된 위치에 크리덴셜이 생성되면 이 과정부터 다시 구성해 주어야 하므로 꼼꼼하게 확인한다.


페이지 하단의 "Create Stack" 버튼을 클릭하고 스택을 생성한다. 몇 분 후에 스택 생성이 완료되었음을 확인하고 Output 탭에서 ARN 값을 복사한다. Tanzu Platform 페이지로 돌아와서 "Next" 버튼을 클릭하여 조금 전에 복사한 ARN 값을 입력한다. 이후, "CREATE" 버튼을 클릭하여 크리덴셜을 생성한다.

### 3) Route 53 GSLB용 크리덴셜 생성
Route 53 GSLB용 크리덴셜 생성 방법도 유사하다. 
좌측 하단에 위치한 Tanzu Platform 메뉴에서 "Setup & Configuration" > "Credentials"를 클릭한다. "ADD CREDENTIAL"을 클릭하고 "Tanzu Platform for Kubernetes"를 선택한다. 이후 나타나는 화면에서 다음과 같이 선택한다.
- Credential Type: Route 53 GSLB

"Next" 버튼을 클릭한 이후는 앞에서 설명한 크리덴셜 생성 방법과 동일하므로 설명을 생략한다. 스택이 생성된 후 설정이 완료되면 다음과 같이 완료 메세지를 확인할 수 있다.

![configuring-tpk8s 2](https://raw.githubusercontent.com/haewons-tanzu/haewons-contents/master/static/img/_posts/2024-08-06-configuring-tpk8s/2.png)

## 2. Cluster Group 생성 (Optional)
클러스터 그룹은 클러스터들의 논리적 집합으로, 이 구성을 통해 그룹 내의 모든 클러스터의 구성을 한번에 적용하여 관리할 수 있다. 
Tanzu Platform에서 기본적으로 제공되는 클러스터 그룹은 다음과 같으며, "Infrastructure" > "Kubernetes Clusters" 메뉴를 클릭한 후 "Clusters Group" 탭에서 확인할 수 있다.
- default: Application Engine disabled
- run: Application Engine enabled

Application Engine이란 앱이 구동되는데 필요한 여러 패키지들(Capabilities)의 모음이다. 구성된 패키지들을 확인하기 위해 "Application Spaces" > "Capabilities" 메뉴를 클릭한다. "Installed" 탭을 클릭하면 어느 클러스터 그룹에 설치되었는지 확인할 수 있는데, 기본적으로는 "run" 클러스터 그룹에 Capabilities가 설치되어 있는 것을 확인할 수 있다.

클러스터 그룹은 기본으로 제공되는 클러스터 그룹을 사용할 수도 있고, 신규로 생성할 수도 있다. 클러스터 그룹을 생성하려면 "Infrastructure" > "Kubernetes Clusters" 메뉴를 클릭한 후 "Clusters Group" 탭에서 "CREATE CLUSTER GROUP" 버튼을 클릭한다. 다음과 같이 값을 입력하여 클러스터 그룹을 생성한다.
- Name: 그룹 명 (에. kr-run)
- Tanzu Application Engine: 활성화

클러스터 그룹을 생성한 후, 이 그룹에서 사용할 Capabilities를 설정한다. 설치가 가능한 Capabilities는 "Application Spaces" > "Capabilities"에서 확인 가능하며, Capability 명 좌측에 보이는 아이콘(점 세개)을 클릭하여 설치를 진행한다. 이 단계에서는 패키지 명을 확인하고 설치를 진행할 클러스터 그룹을 지정해 주면 된다. (기본적으로는 "run" 클러스터 그룹이 선택되어 있으며, 여기서는 kr-run으로 선택)

> **NOTE**
> 클러스터 그룹을 생성한 후 이 단계를 건너뛸 경우, 클러스터 설치 이후 정상적으로 기능이 동작하지 않을 수 있으므로 설치가 정상적으로 완료되었는지를 반드시 확인한다. 

이 과정을 반복하여 필요한 Cabability를 설치한다. 필요 시 "ADVANCED CONFIGURATION"에서 추가 설정을 한다.

### "Registry Pull Only Credentials Installer" Capability 구성
이미지 레지스트리에 대한 정보를 설정하는 Cabability로 이 값을 지정하지 않으면 설치 시 일부 다른 Cabability들이 에러가 발생할 수 있으므로 반드시 설정한다. 설정 값들은 다음과 같다.
- username: 이미지 레지스트리 사용자명
- password: 이미지 레지스트리 바말번호
- _package_install_name: 임의의 값을 입력한다. (예. package_install_name)

  아래 메세지가 기본적으로 보이며 값을 비워두라고 하지만, 2024년 8월 현재 값을 비워두지 않을 경우 에러가 발생한다.
  ```
  Do not set manually, set automatically by DownwardAPI. Required.
  ```
  발생되는 에러는 다음과 같다.

  ![configuring-tpk8s 3](https://raw.githubusercontent.com/haewons-tanzu/haewons-contents/master/static/img/_posts/2024-08-06-configuring-tpk8s/3.png)

- registry: 이미지 레지스트리 주소

설치가 정상적으로 완료되면 신규로 생성된 클러스터 그룹에 대하여(여기서는 kr-run) 다음과 같이 Capabilities 목록을 확인할 수 있다.
![configuring-tpk8s 4](https://raw.githubusercontent.com/haewons-tanzu/haewons-contents/master/static/img/_posts/2024-08-06-configuring-tpk8s/4.png)

## 3. EKS 클러스터 설치
Tanzu Platform에서 EKS 클러스터를 설치하면 클러스터 생성, 삭제, 업그레이드, 스케일 등의 클러스터 라이프사이클을 관리할 수 있다. "Infrastructure" > "Kubernetes Clusters" 메뉴를 클릭한 후 "Clusters" 탭에서 "ADD CLUSTER" > "Create AWS EKS cluster" 메뉴를 선택한다. 다음 값들을 입력한 후 "Next" 버튼을 클릭한다.
- Cluster name: 생성할 클러스터 명 (예 my-cluster-1)
- Cluster group: Application Engine groups 선택 후 이전에 생성한 Cluster Goup 선택 (예. kr-run)

다음 값들을 입력한 후 "Next" 버튼을 클릭한다.
- Account credential: 1-2)에서 생성한 크리덴셜 선택
- Kubernetes version: 쿠버네티스 버전 선택 (2024년 8월 현재 사용가능한 쿠버네티스 최신 버전은 1.30임)
- Network
  - Region: 1-2)에서 생성한 크리덴셜과 같은 리전을 선택. 서울 리전의 경우 ap-northeast-2
  - VPC: 클러스터가 사용할 VPC를 선택. 없을 경우 VPC 셀렉트박스 하위에 위치한 링크를 참조하여 생성(클러스터 스택을 이용하여 VPC 자동 구성됨)
  - Subnets: 클러스터가 사용할 서브넷들을 선택
  - Cluster endpoint access: Public and private

다음 값들을 입력한 후 "Next" 버튼을 클릭하여 노드풀을 구성한다.
- Name: 노드풀 명 (기본값: default-node-pool)
- Compute
  - AMI type: 이미지 유형 (기본값: AL2_x86_64)
- Scaling: Fixed # nodes (AWS 오토스케일링을 사용하지 않을 경우 선택)
  - Desired # nodes: 3

"Create" 버튼을 클릭하여 EKS 클러스터를 생성한다. (약 30분 정도 소요)
> **NOTE**
> 클러스터 설치 이후에 관련 패키지가 설치된다. Tanzu Platform의 클러스터 정보 조회 화면에서는 패키지의 정상 설치 여부까지는 나오지 않으므로 다음과 같이 클러스터와 컴포넌트가 "Ready" 상태이더라도 추가 확인이 필요하다.

![configuring-tpk8s 5](https://raw.githubusercontent.com/haewons-tanzu/haewons-contents/master/static/img/_posts/2024-08-06-configuring-tpk8s/5.png)

클러스터에 접근하기 위해 위 화면 우측 상단의 "ACTION" > "Access this cluster"를 클릭한다. 

![configuring-tpk8s 6](https://raw.githubusercontent.com/haewons-tanzu/haewons-contents/master/static/img/_posts/2024-08-06-configuring-tpk8s/6.png)

위와 같은 화면이 나오면 kubeconfig 설정을 다운로드하고, kubectl CLI를 이용하여 클러스터 접속 가능 여부를 확인한다.

```bash
$ kubectl get node -o wide
```

연결이 정상적으로 되었다면, kubectl CLI를 이용한 다음 명령어를 실행하여 패키지들이 정상적으로 설치되었는지 확인한다. 
```bash
$ kubectl get pkgr -A
```

![configuring-tpk8s 7](https://raw.githubusercontent.com/haewons-tanzu/haewons-contents/master/static/img/_posts/2024-08-06-configuring-tpk8s/7.png)

> **NOTE**
> 만약 vss-k8s-collector-repo 패키지가 설치되지 않았다면, 다음과 같은 과정으로 클러스터를 재생성한다. 이 과정은 동일 이름일 경우 이전 설정 정보를 Tanzu Platform에서 인식하고 있어서 발생한다. 대부분의 경우 클러스터 그룹 생성 후 Capabilities를 설치하지 않아서 발생하므로, 위의 과정을 정상적으로 수행하였다면 패키지가 설치되지 않을 일은 없다.
> 1. 클러스터 삭제
> 2. Tanzu Platform 메뉴의 "Setup & Configuration" > "Kubernetes Management"에서 조금 전에 생성한 클러스터의 상태가 "Attached"인지 확인 후, 어태치 되어 있으면 클러스터명 좌측에 위치한 아이콘을 클릭하여 "Detach" 
> 3. 클러스터 생성

## 4. 스페이스 구성
### 1) 프로파일 구성
프로파일은 사전 정의된 것을 사용할 수도 있고, 직접 만들어서 사용할 수도 있다. 이 단계에서는 대부분의 경우 여러 사람이 공유해서 사용하는 환경이므로 직접 만드는 방법에 대해 설명한다. Spring Boot를 사용해서 앱을 구동하는 환경이라고 가정하면, 다음과 같은 2개의 프로파일이 필요하다. 하나의 프로파일에 지정할 수도 있지만, 프로파일의 재활용성을 생각한다면 기능별로 분리하는 것을 추천한다.

#### A) 네트워킹 관련 프로파일
이 프로파일은 네트워킹에 관련된 기능들의 묶음으로 정의한다. "Application Spaces" > "Profiles" 매뉴에서 "CREATE PROFILE" 버튼을 클릭하고 다음과 같이 입력하여 프로파일을 생성한다.
- Profile Name: 프로파일 명 지정 (예. haewons-custom-networking)
- Traits: 필요한 Traits를 선택. 네트워킹 관련 프로파일을 지정할 것이므로 다음 Traits를 선택
  - egress.tanzu.vmware.com 
  - multicloud-cert-manager.tanzu.vmware.com 
  - multicloud-ingress.tanzu.vmware.com: 라우팅 할 정보를 입력
    - Gslb Dns ZoneId: AWS Route 53에서 호스팅하고 있는 Zone의 ID로 21자리임.
    - Gslb Authentication CredentialRef:
    - Domain: AWS Route 53에 구성한 Hosted Zone 명이 포함된 서브도메인 (예. tanzu-korea.org이 Hosted Zone명인 경우 tanzu-hub.tanzukorea.org)
    - Name: 게이트웨이 명 (기본값. default-gatyeway)
- Additional required Capabilities: 기본적으로 선택되어 있는 Capabilities 외에 추가로 선택하지 않음

#### B) Spring Boot 관련 프로파일
이 프로파일은 Spring Boot 관련 기능들의 묶음으로 정의한다. Spring Boot 관련 프로파일은 다음과 같이 사전에 2개의 프로파일이 등록되어 있지만, 모든 Traits이 설정되어 있지도 않고 수정 불가능하므로 필요에 따라 프로파일을 생성한다.
- spring-dev.tanzu.vmware.com
- spring-prod.tanzu.vmware.com

위와 같이 사전에 정의된 프로파일들은 프로파일명 하단에 "Tanzu Provided"라고 명시되어 있어, 커스텀 프로파일과 구분된다. 2024년 8월 현재 사전에 정의된 프로파일 목록은 다음과 같다.
- networking.tanzu.vmware.com
- spring-dev.tanzu.vmware.com
- spring-prod.tanzu.vmware.com
- fluxcd-helm.tanzu.vmware.com

### 2) Availability Target 구성
Availability Target은 리전, Fault Domain 등 서비스 고가용성과 관련된 구성이다.
"Application Spaces" > "Availability Targets" 메뉴에서 "CREATE AVAILABILITY TARGETS" 버튼을 클릭하여 "Availability Target"을 생성한다. 생성하는 방법은 다음과 같이 2가지가 있다. 
- Upload Manifest: Availability Target 관련 YAML 파일을 생성하고 이를 업로드하여 구성하는 방식
- Step by Step: UI에서 설정값을 직접 입력하여 구성하는 방식

여기서는 "Step by Step"를 이용해서 설정하는 방식에 대해 설명한다. 특정 리전에 위치한 클러스터들을 "Availability Target"으로 지정하려고 하는 경우 다음과 같이 구성할 수 있다.
- Name: Availability Target 명 (예. kr-availability-target)
- Cluster Match Expressions
  - Cluster Affinity
    - Selector: Region
    - Value: 클러스터가 존재하는 리전 값. 서울 리전의 경우 "ap-northeast-2"를 입력

Availability Target 생성이 완료되면, 이전에 생성했던 클러스터(여기서는 my-cluster-1)가 다음과 같이 보이는 것을 알 수 있다. 만약 클러스터가 보이지 않는다면, 생성된 클러스터의 리전과 Availability Target에 지정한 리전이 일치하지 않는다는 것이므로, 조건을 다시 한 번 확인해 보도록 한다.

![configuring-tpk8s 8](https://raw.githubusercontent.com/haewons-tanzu/haewons-contents/master/static/img/_posts/2024-08-06-configuring-tpk8s/8.png)

### 3) 스페이스 생성
스페이스는 앱이 구성되는 환경이다. 이 역시 "Availability Target" 구성과 마찬가지로 두 가지 방식을 지원한다. 여기서는 "Step by Step"를 이용해서 설정하는 방식에 대해 설명한다.
"Application Spaces" > "Spaces" 메뉴에서 "CREATE SPACE" 버튼을 클릭하고 "Step by Step"를 선택한다.
- Space Name: 스페이스 명 (예. my-first-space)
- Profile: 스페이스에서 사용할 프로파일 들을 복수 선택
- Availability Targets
  - Name:
  - Replica count: 1 (스페이스가 구성될 기본 클러스터 개수. 앞 단계에서 클러스터를 1개만 생성했으므로 1로 설정함. 설정된 개수에 비해 "Availability Targets"에서 검색된 클러스터 개수가 부족할 경우 스페이스가 Ready 상태가 되지 않음)
  - Routing policy: Active

설정값을 지정한 후 스페이스를 생상한다. 스페이스가 생성되고 프로파일에서 지정한 Capabilities들이 클러스터에 직접 구성되므로 약간의 시간이 필요하며, 생성 시간은 약 5분 이내이다. 구성이 완료되면 쿠버네티스 클러스터에서 스페이스가 제대로 생성되었는지 확인한다. 정상적으로 생성되었다면 다음과 같이 쿠버네티스 클러스터에 스페이스 명을 접두어로 가지는 네임스페이스 2개가 생성된다.
 
![configuring-tpk8s 9](https://raw.githubusercontent.com/haewons-tanzu/haewons-contents/master/static/img/_posts/2024-08-06-configuring-tpk8s/9.png)


## 5. 스페이스 접속 확인
스페이스 구성이 완료되면 정상적으로 설치가 되었는지 확인이 필요하다. Tanzu Platform 환경에 대한 모든 구성은 Tanzu CLI를 통해 이루어진다.

로그인을 수행하기 위해 다음의 환경 변수 설정이 필요하다. 아래 정보들을 [VCS(VMWare Cloud Services) 콘솔](https://console.cloud.vmware.com/)에서 확인할 수 있다.

- TANZU_API_TOKEN: 우측 상단 메뉴를 클릭하여 "My Account"를 선택하고, "API Tokens" 탭에서 "Generate Token" 버튼을 클릭하여 토큰값을 획득함. 이 값은 이후에는 조회할 수 없으므로 별도로 저장이 필요.
- TANZU_CLI_CLOUD_SERVICES_ORGANIZATION_ID: 우측 상단 메뉴를 클릭 후 나타나는 "Organization ID" 값


이 값들을 다음과 같이 환경변수에 추가한다.
```
export TANZU_API_TOKEN=xxxxxx
export TANZU_CLI_CLOUD_SERVICES_ORGANIZATION_ID=xxxxxx
```

환경에 접속하기 위해 다음 명령어들을 수행한다.
### 1) 로그인
```bash
$ tanzu login
```
### 2) 프로젝트 목록 조회 및 사용할 프로젝트 선택
```bash
$ tanzu project list
$ tanzu project use APJ
```
### 3) 스페이스 목록 조회 및 사용할 스페이스 선택
```bash
$ tanzu space list
$ tanzu space use my-first-space
```

다양한 환경으로 구성할 필요가 있다면 컨텍스트를 이용할 수 있고, 컨텍스트 목록 조회를 통해 컨텍스트를 확인하여 그 정보를 사용할 수 있다. 이전에 로그인 한 이력이 있다면 다음과 같이 컨텍스트가 저장되어 있을 것이다.
```
$ tanzu context list
  NAME                                      ISACTIVE  TYPE   PROJECT      SPACE
  sa-tanzu-platform                         true      tanzu  APJ          my-first-space
```
```bash
$ tanzu context use sa-tanzu-platform
```

## 6. 앱 배포 및 확인
앱을 배포하는 방법은 다음 2가지가 있다. 각각의 용도 및 사용 방법은 다음과 같으며, 앱 소스 프로젝트 홈 디렉토리에서 실행한다. 여기서는 tanzu-java-web-app을 이용하여 앱을 배포한다.

### 1) 앱 배포 설정 구성
다음과 같이 GitHub에서 소스를 복제한다. 
```bash
$ git clone https://github.com/haewons-tanzu/tanzu-java-web-app
```

앱 소스 구성을 초기화하기 위해 다음 명령어를 수행한다. 
```bash
$ tanzu app init
```
위 명령어는 앱을 빌드하기 위해 필요한 기본 YAML들을 생성해 주는 명령어로, 다음과 같은 구조의 폴더 및 파일을 생성한다.

    tanz-java-web-app
    ├── ...
    ├── tanzu.yml                          # 앱 소스 디렉토리 내에서 설정 관련 디렉토리 및 파일 구성에 대한 정보
    └── .tanzu
        └── config          
            └── tanzu-java-web-app.yml     # ContainerApp 관련 YAML로, 컨테이너 설정 정보


각 파일 내용은 다음과 같다.
```YAML
$ cat tanzu.yml
apiVersion: config.tanzu.vmware.com/v1
configuration:
dev:
paths:
- .tanzu/config/
kind: TanzuConfig
```

```YAML
$ cat .tanzu/config/tanzu-java-web-app.yml
apiVersion: apps.tanzu.vmware.com/v1
kind: ContainerApp
metadata:
  creationTimestamp: null
  name: tanzu-java-web-app
spec:
  build:
    buildpacks: {}
    path: ../..
  ports:
  - name: main
    port: 8080
```
라우팅을 위해 Tanzu Platform에서 기본적으로 제공하는 게이트웨이 설정이 필요하다. 다음과 같이 라우팅 관련 YAML 파일을 생성한다.
```YAML
$ cat .tanzu/config/k8sGatewayRoutes.yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: tanzu-java-web-app-route
  annotations:
    apps.tanzu.vmware.com/promotable: ""
    apps.tanzu.vmware.com/promote-group: ContainerApp/tanzu-java-web-app
    healthcheck.gslb.tanzu.vmware.com/service: tanzu-java-web-app
    healthcheck.gslb.tanzu.vmware.com/path: /
    healthcheck.gslb.tanzu.vmware.com/port: "8080"
spec:
  parentRefs:
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: default-gateway
    sectionName: http-tanzu-java-web-app
  rules:
  - backendRefs:
    - group: ""
      kind: Service
      name: tanzu-java-web-app
      port: 8080
      weight: 1
    matches:
    - path:
        type: PathPrefix
        value: /
```

위의 설정 파일을 포함하여 다음과 같이 앱 배포 설정 파일이 구성되었다.

    tanz-java-web-app
    ├── ...
    ├── tanzu.yml                          # 앱 소스 디렉토리 내에서 설정 관련 디렉토리 및 파일 구성에 대한 정보
    └── .tanzu
        └── config          
            ├── tanzu-java-web-app.yml     # ContainerApp 관련 YAML로, 컨테이너 설정 정보
            └── k8sGatewayRoutes.yaml      # 라우팅 게이트웨이


### 2) 앱 빌드 및 배포
앱 빌드 및 배포는 아래 두 가지 방법 중 한 가지 방법을 선택하여 수행한다.
#### A. 단일 명령어로 앱 빌드 및 배포
앱 개발 중인 상태로, 지속적으로 앱에 대한 빌드 및 배포가 일어나는 경우 사용할 수 있다. 이 명령어를 사용하면 소스에서 이미지를 빌드하고, 쿠버네티스 환경으로 배포하는 일을 아래의 단일 커맨드로 수행한다.
```bash
$ tanzu deploy -y
```

#### B. 앱 빌드 및 배포 분리
앱에 대한 개발이 어느 정도 완료된 상태에서, 앱을 빌드하고 배포하는 작업을 분리하고 싶은 경우 사용한다.
아래 명령어는 앱 소스로부터 빌드 관련 bom 파일 및 설정 내용들을 특정 디렉토리(여기서는 build-output)에 저장하고, 이미지를 생성하여 레지스트리에 저장하는 작업이 진행된다.
```bash
$ tanzu build --output-dir build-output
```
특정 디렉토리(여기서는 build-output)에 저장된 빌드 관련 bom 파일 및 설정 내용들로부터 앱을 배포하기 위해서는 아래 명령어를 수행한다.
```bash
$ tanzu deploy --from-build build-output -y
```

### 3) 앱 접속 확인
"Application Spaces" > "Spaces" 메뉴에서 나의 스페이스로 접속한다. 앱이 정상적으로 구동되었다면 다음과 같이 "Network Topology"에 각 파드들 간의 연결 정보를 확인할 수 있을 것이다.

![configuring-tpk8s 10](https://raw.githubusercontent.com/haewons-tanzu/haewons-contents/master/static/img/_posts/2024-08-06-configuring-tpk8s/10.png)

토폴로지가 정상적으로 표시되는 것을 확인한 후, "Ingress & Egress" 탭을 클릭하여 앱 접속을 위한 URL을 획득한다.
![configuring-tpk8s 11](https://raw.githubusercontent.com/haewons-tanzu/haewons-contents/master/static/img/_posts/2024-08-06-configuring-tpk8s/11.png)

획득한 URL을 웹브라우저에 접속해서 앱이 정상적으로 구동되는지를 확인한다.

[http://tanzu-java-web-app.tanzu-hub.tanzukorea.org](http://tanzu-java-web-app.tanzu-hub.tanzukorea.org)

![configuring-tpk8s 12](https://raw.githubusercontent.com/haewons-tanzu/haewons-contents/master/static/img/_posts/2024-08-06-configuring-tpk8s/12.png)

이로써 Tanzu Platform 환경을 구성하고 앱을 배포해 보는 과정이 완료되었다. 좀 더 자세한 내용은 아래 링크를 접속하여 Tanzu Platform 매뉴얼에서 확인할 수 있다.

참고: [https://docs.vmware.com/en/VMware-Tanzu-Platform/services/create-manage-apps-tanzu-platform-k8s/index.html](https://docs.vmware.com/en/VMware-Tanzu-Platform/services/create-manage-apps-tanzu-platform-k8s/index.html)

