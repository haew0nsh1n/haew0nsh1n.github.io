---
layout: post
title: Tanzu Platform for Kubernetes 위에 구동되는 앱에서 서비스 바인드하기
categories: [tp, tpk8s]
---

Tanzu Platform은 앱에 필요한 다양한 서비스들을 Bitnami 서비스(Tanzu Catalog Services)를 통해 제공하고 있다. 이 기능을 이용하면, 서비스를 바인드하여 앱에서 서비스에 쉽게 접근할 수 있다.

## 사전 확인
Tanzu Platform에서 생성된 서비스는 스토리지를 동적으로 프로비저닝 한다. EKS 클러스터를 최초 생성했을 때 "Amazon EBS CSI Driver" 플러그인이 설치되며 스토리지 클래스가 함께 생성되는데 이 설정을 확인한다.

다음 명령어를 수행하여 자동으로 생성된 스토리지클래스가 디폴트로 구성되어 있는지 확인한다. (기본적으로 Tanzu Platform에서 클러스터를 생성하면 스토리지클래스는 디폴트로 구성되어 있지 않다.)
```
$ k get sc -A
NAME   PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
gp2    kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer   false                  14h
```

이 스토리지클래스를 기본 스토리지클래스로 설정해 준다. 만일 이 설정을 하지 않으면, persistentvolumeclass가 어떤 스토리지클래스를 사용할지 알 수가 없기 때문에 서비스 파드들이 펜딩되며 서비스가 정상으로 구동되지 않는다.
```
$ kubectl patch sc gp2 -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
storageclass.storage.k8s.io/gp2 patched
```
스토리지클래스를 조회하는 명령어를 한번 더 수행하여 디폴트 스토리지클래스가 설정되었는지 확인한다.
```
$ kubectl get sc -A
NAME            PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
gp2 (default)   kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer   false                  14h
```

## Tanzu Platform 콘솔에서 서비스 생성 및 바인드하기
먼저 서비스를 생성하고 바인드하는 기능을 살펴보기 위해 Tanzu Platform 콘솔 화면에 접속한다.

### 서비스 생성
서비스를 생성하기 위해서는 "Application Spaces" > "Spaces" 메뉴에서 나의 스페이스(예. my-first-space)로 접속한다. "Services" 탭을 클릭하고, 우측 상단의 "CREATE SERVICE" 버튼을 클릭하여 사용할 수 있는 서비스 목록을 조회한다.

![binding-services-on-tpk8s 1](https://raw.githubusercontent.com/haew0nsh1n/haew0nsh1n.github.io/master/static/img/_posts/2024-08-07-binding-services-on-tpk8s/1.png)

서비스를 생성하기 위해 다음과 같이 입력한다. MySQL 서비스를 생성하기 위한 설정 값은 다음과 같다. 
- Provision Type: Dynamic Provision Flow
- Service Type: MySQLInstance
- Service Name: 생성할 서비스명 (예. my-mysql)
- Storage Size (GB): 1

설정 값들을 입력하고 "CREATE SERVICE" 버튼을 클릭하여 서비스를 생성한다.

![binding-services-on-tpk8s 2](https://raw.githubusercontent.com/haew0nsh1n/haew0nsh1n.github.io/master/static/img/_posts/2024-08-07-binding-services-on-tpk8s/2.png)

### 서비스 바인드
생성된 서비스를 앱과 바인딩 하기 위해 우측 상단의 "BIND TO APPLICATION" 버튼을 클릭한다. 테스트를 위해, 이전에 생성한 tanzu-java-web-app과 바인드 시키기 위해 다음과 같이 입력한다.
- Application Type: Container App
- Application: tanzu-java-web-app
- Service Binding Alias: 서비스 별칭 값(예. db)

값 입력 후, "CREATE SERVICE BINDINGS"를 클릭하여 서비스를 바인딩한다. 성공적으로 바인딩 되었으면 다음과 같은 화면이 나타난다.

![binding-services-on-tpk8s 3](https://raw.githubusercontent.com/haew0nsh1n/haew0nsh1n.github.io/master/static/img/_posts/2024-08-07-binding-services-on-tpk8s/3.png)

### 서비스 언바인드
서비스를 앱과 언바인드 시키려면 언바운드할 앱 옆에 위치한 "UNBIND" 버튼을 클릭한다.

### 서비스 삭제
바인드된 앱이 존재하지 않음을 확인하고, 서비스 조회 화면 우측 상단에서 "DELETE" 버튼을 클릭하여 서비스를 삭제한다. 

## CLI를 이용하여 서비스 생성 및 바인드하기

서비스들은 tanzu CLI를 이용해서도 조회할 수 있다.
```bash
$ tanzu service type list
```

![binding-services-on-tpk8s 4](https://raw.githubusercontent.com/haew0nsh1n/haew0nsh1n.github.io/master/static/img/_posts/2024-08-07-binding-services-on-tpk8s/4.png)

또한 서비스들을 tanzu CLI를 이용하여 생성, 삭제, 바인드 할 수 있으나, 여기서는 YAML 파일들을 활용해서 앱 배포 시 함께 서비스를 관리하는 방법에 대해 살펴보겠다.

### spring-music 앱 소개
앱 바인드에 사용할 앱은 Spring Music이다. Spring Music 앱은 스프링 스프링 부트의 기능을 테스트하기 위해 스프링 진영에서 기본적으로 제공하고 있는 앱이다. 이 앱은 "mysql" 이라고 하는 스프링 profile을 사용하는데 이 경우 MySQL을 사용하여 데이터를 처리하도록 코딩되어 있다. 

다음 명령어를 사용하여 GitHub에서 소스를 복제한다.

```bash
$ git clone https://github.com/haew0nsh1n/spring-music
```

### 서비스 관련 설정 구성
추가될 서비스 관련 설정 파일들에 대한 구조는 다음과 같다.

    spring-music
    ├── ...
    ├── tanzu.yml                          # 앱 소스 디렉토리 내에서 설정 관련 디렉토리 및 파일 구성에 대한 정보
    └── .tanzu
        └── config          
            ├── spring-music.yml           # ContainerApp 관련 YAML로, 컨테이너 설정 정보
            ├── k8sGatewayRoutes.yml       # 라우팅 게이트웨이
            └── bitnamiServices.yaml       # 비트나미 서비스 및 바인드 설정. 여기서는 MySQL 서비스


```YAML
$ cat .tanzu/config/bitnamiServices.yaml
---
apiVersion: bitnami.database.tanzu.vmware.com/v1alpha1
kind: MySQLInstance
metadata:
  name: my-mysql
spec:
  storageGB: 1
---
apiVersion: services.tanzu.vmware.com/v1
kind: ServiceBinding
metadata:
  name: spring-music-mysql
spec:
  targetRef:
    apiGroup: apps.tanzu.vmware.com
    kind: ContainerApp
    name: spring-music
  serviceRef:
    apiGroup: bitnami.database.tanzu.vmware.com
    kind: MySQLInstance
    name: my-mysql
  alias: db
```

다음은 spring-music 앱 소스의 application.yaml 파일의 일부이다. spring.datasource.url 부분이 localhost로 설정되어 있는 것을 알 수 있다. 서비스로 바인드 했기 때문에 localhost로 접속 할 수 있으며, MySQL 파드에 접속하기 위해서 해당 서비스명을 직접 호출해야 하는 번거로움을 피할 수 있다.

```YAML 
---
spring:
  config:
    activate:
      on-profile: mysql
  ...
  datasource:
    url: "jdbc:mysql://localhost/music"
    ...
```

### 앱 배포 및 확인
구성이 완료되었으면 다음 명령어를 사용해서 앱을 빌드 및 배포한다.
```bash
$ tanzu deploy -y
```

획득한 웹 URL을 접속해서 앱이 잘 구동되어 있는지 확인한다.

[http://spring-music.tanzu-hub.tanzukorea.org/](http://spring-music.tanzu-hub.tanzukorea.org/)

![binding-services-on-tpk8s 5](https://raw.githubusercontent.com/haew0nsh1n/haew0nsh1n.github.io/master/static/img/_posts/2024-08-07-binding-services-on-tpk8s/5.png)
