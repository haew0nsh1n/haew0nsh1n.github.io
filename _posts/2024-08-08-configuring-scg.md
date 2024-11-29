---
layout: post
title: Tanzu Platform for Kubernetes에서 Spring Cloud Gateway 구성하기
categories: [tp, tpk8s, scg]
---

Tanzu Platform에서 기본적으로 제공하는 쿠버네티스 게이트웨이 라우팅 외에도, Spring Cloud Gateway를 이용한 라우팅을 구성할 수 있다. 여기서는 스페이스, 프로파일 등의 구성은 완료되었다고 가정하고 Spring Cloud Gateway 구성 방법과 이를 이용한 앱 배포 과정을 알아볼 것이다. 만약 구성에 대한 이해가 필요하다면 [여기](2024-08-06-configuring-tpk8s.md)를 클릭하여 Tanzu Platform 구성 방법에 대해 확인하기 바란다.

## 1. Spring Cloud Gateway 구성
먼저 Spring Cloud Gateway를 사용할 수 있는지 확인한다. Space에서 사용하는 Profile을 확인한 뒤, 해당 프로파일을 클릭하여 설치된 Traits 정보에 Spring Cloud Gateway가 있는지 확인한다. 만약 없다면 추가로 선택하여 설치한다.
![configuring-scg 1](https://raw.githubusercontent.com/haewons-tanzu/haewons-contents/master/static/img/_posts/2024-08-08-configuring-scg/1.png)

기본 설정으로 두어도 동작하는데는 지장이 없지만, 고가용성을 고려하여 replica 개수를 조정하거나, 리소스에 대한 제한을 거는 설정을 조정할 수 있다. 이와 관련된 설정 값은 다음과 같다.
- Count: 1 (기본값)
- resources 하위 항목: CPU, 메모리 리소스 설정 (필요 시)

![configuring-scg 2](https://raw.githubusercontent.com/haewons-tanzu/haewons-contents/master/static/img/_posts/2024-08-08-configuring-scg/2.png)

기본 값으로 설정하고 프로파일을 업데이트 한 뒤, 파드들의 변동을 확인해 보면 다음과 같다.
```bash
$ kubectl get po -A
```

![configuring-scg 3](https://raw.githubusercontent.com/haewons-tanzu/haewons-contents/master/static/img/_posts/2024-08-08-configuring-scg/3.png)

위의 스크린샷에서 구동된 파드들에 대한 설명은 다음과 같다. 
1. 기존에 설정되었던 게이트웨이에 대한 정보가 변경되면서 파드가 재구동 되고 있음
2. Spring Cloud Gateway Operator에 의해서 설치된 스페이스에서 사용될 게이트웨이
   - 앞에서 개수를 1개로 지정했기 때문에 spring-cloud-gateay-0으로 표시됨
   - SCG 개수가 N개로 지정될 경우, 다음과 같이 N개의 게이트웨이 파드가 구성됨
     - spring-cloud-gateay-0
     - spring-cloud-gateay-1
     - ...
     - spring-cloud-gateay-{N-1}
3. Spring Cloud Gateway에 대한 Operator가 설치됨

## 2. 앱 배포 및 확인
[Tanzu Platform 구성 방법](2024-08-06-configuring-tpk8s.md)에서 설명했던 동일한 앱 소스코드(tanzu-java-web-app)를 가지고 Spring Cloud Gateway를 구성해서 설정을 비교해 보려고 한다. 다음과 같이 GitHub에서 소스를 복제한다.
```bash
$ git clone https://github.com/haewons-tanzu/tanzu-java-web-app
```

앱 빌드를 위한 설정 파일들의 구조는 다음과 같다.

    tanz-java-web-app
    ├── ...
    ├── tanzu.yml                          # 앱 소스 디렉토리 내에서 설정 관련 디렉토리 및 파일 구성에 대한 정보
    └── .tanzu
        └── config          
            ├── tanzu-java-web-app.yml     # ContainerApp 관련 YAML로, 컨테이너 설정 정보
            ├── k8sGatewayRoutes.yml       # 라우팅 게이트웨이
            └── scgRoutes.yaml             # Spring Cloud Gateway 설정, 라우팅 및 매핑 정보를 포함함


Spring Cloud Gateway를 이용한 라우팅 사용을 위해 라우팅 설정 및 매핑 정보를 다음과 같이 추가한다.
```YAML
cat .tanzu/config/scgRoutes.yaml
apiVersion: tanzu.vmware.com/v1
kind: SpringCloudGatewayRouteConfig
metadata:
  name: tanzu-java-web-app-route-config
spec:
  routes:
  - predicates:
    - Path=/**
    filters:
    - StripPrefix=0
  service:
    name: tanzu-java-web-app
    port: 8080

---
apiVersion: "tanzu.vmware.com/v1"
kind: SpringCloudGatewayMapping
metadata:
  name: tanzu-java-web-app-route-mapping
spec:
  gatewayRef:
    name: spring-cloud-gateway
  routeConfigRef:
    name: tanzu-java-web-app-route-config

```
기존의 쿠버네티스 게이트웨이 라우팅은 앱을 직접 바라보고 있었지만, Spring Cloud Gateway를 통과하도록 설정이 수정될 것이다. 다음과 같이 변경해 준다.
```YAML
$ cat .tanzu/config/k8sGatewayRoutes.yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: tanzu-java-web-app-scg-route
  annotations:
    apps.tanzu.vmware.com/promotable: ""
    apps.tanzu.vmware.com/promote-group: ContainerApp/tanzu-java-web-app
    healthcheck.gslb.tanzu.vmware.com/service: spring-cloud-gateway
    healthcheck.gslb.tanzu.vmware.com/path: /
    healthcheck.gslb.tanzu.vmware.com/port: "80"
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
      name: spring-cloud-gateway
      port: 80
      weight: 1
    matches:
    - path:
        type: PathPrefix
        value: /
```

설정이 완료되었으면 다음 명령어를 사용해서 앱을 빌드 및 배포한다.
```bash
$ tanzu deploy -y
```

이전에 수행했던 것과 마찬가지로, "Application Spaces" > "Spaces" 메뉴에서 나의 스페이스로 접속한다. "Network Topology"에 각 파드들 간의 연결 정보를 확인한다.
![configuring-scg 4](https://raw.githubusercontent.com/haewons-tanzu/haewons-contents/master/static/img/_posts/2024-08-08-configuring-scg/4.png)

토폴로지가 정상적으로 표시되는 것을 확인한 후, "Ingress & Egress" 탭을 클릭하여 앱 접속을 위한 URL을 획득한다.
![configuring-scg 5](https://raw.githubusercontent.com/haewons-tanzu/haewons-contents/master/static/img/_posts/2024-08-08-configuring-scg/5.png)

주어진 웹 URL을 통해 앱이 정상적으로 접속되는 것을 확인한다.

좀 더 자세한 내용은 아래 링크를 접속하여 Tanzu Platform 매뉴얼에서 확인할 수 있다.

참고: [https://docs.vmware.com/en/VMware-Tanzu-Platform/services/create-manage-apps-tanzu-platform-k8s/index.html](https://docs.vmware.com/en/VMware-Tanzu-Platform/services/create-manage-apps-tanzu-platform-k8s/index.html)

