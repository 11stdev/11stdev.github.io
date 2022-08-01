---
layout: post
title: 'Service Discovery 통합을 위한 Kubernetes Operator 구현 - Eurekube Operator'
author: 안희석
date: 2022-07-20
tags: [kubernetes, operator, msa]
---

안녕하세요. 저는 11번가 Core Platform 개발팀에서 `MSA 플랫폼 - Vine` 을 개발하고 있는 안희석입니다.

이번 글에서는 11번가에서 쿠버네티스 플랫폼을 도입하면서 기존에 `Spring Cloud` 로 이루어진 `Vine Platform` 과 서비스 디스커버리를 통합하기 위해 `Kubernetes Operator` 패턴을 이용해
`Eurekube Operator` 를 개발 한 것을 공유드리고자 합니다.


## 1. Context & Mission
---

Eureka 와 Kubernetes 서비스 디스커버리를 통합하기 위해서 11번가에서는 Kubernetes Operator 패턴으로 `Eurekube Operator` 를 개발하였습니다.
`Eurekube Operator` 에 대해서 설명하기 전에 먼저 개발하게된 배경들에 대해서 설명드리겠습니다.

### AS-IS. **Spring Cloud and MSA**

 2016년 11번가에서는 monolithic 아키텍처 구조에서 MSA 를 도입하기 위하여 Spring Cloud 기반의 Vine 플랫폼 프로젝트를 시작하였습니다.

 점진적으로 monolithic 아키텍처에서 서비스들을 Spring Cloud 기반으로 분리하면서 현재는 약 600여개의 인스턴스와 60여개의 어플리케이션 서비스가 Vine 플랫폼 위에서 성공적으로 운영되고 있습니다.

![Figure 1-1. Vine Platform Micro Service Dependency Graph](/files/post/2022-07-20-eurekube-operator/vine-distributed-tracing.png)


위의 이미지를 보면 알 수 있듯이 다양한 Micro Service들이 서로 호출하고 있는 것을 알 수 있습니다.

작게 쪼개어진 서비스들이 생성과 소멸을 반복하며 서로 호출을 하고 있다보니 Micro Service 들이 어디에 있는지 알려줄 수 있는 컴포넌트가 필요합니다. 이 컴포넌트의 이름을 Service Discovery 라고 합니다.

Spring Cloud 에서는 Netflix OSS 중 하나인 [Eureka](https://github.com/Netflix/eureka) 를 통해서 이러한 Service Discovery 구현하고 있습니다. 

Vine Platform 에서 Eureka 를 통한 서비스 디스커버리는 잘 동작했고, 현재에도 Vine Platform 내에 많은 서비스들이 Eureka 컴포넌트를 중심으로 동작하고 있습니다.

### TO-BE. Kubernetes 의 도입과 Service Discovery 통합

2021년 11번가에서는 AWS Cloud 를 사용하게 되면서 다양한 AWS 서비스를 사용하게 되었습니다.

그러면서 몇몇 서비스들이 IDC 내에 구축된 `Spring Cloud 기반의 MSA 플랫폼 - Vine` 을 AWS Cloud 환경에서도 사용하려는 Needs 들이 생겨나기 시작했습니다.

이러한 Needs 를 충족시키고 Cloud Native 개발 생태계를 이용할 수 있도록 Core Platform 개발팀에서는 AWS EKS (Elastic Kubernetes Service) 를 채택하여 도입하게 되었습니다.

그러나 여기에서 직면하게 된 문제점 중 하나는 위에서 설명한 서비스 디스커버리 부분이였습니다.

![Figure 1-2. Different Service Discovery](/files/post/2022-07-20-eurekube-operator/idc-vs-eks-discovery.png)

IDC 내에 Vine 플랫폼에서는 `Eureka` + `Client Side Loabalancer` 를 기반으로 `Client Side Service Discovery` 로 동작하지만 쿠버네티스에서는 쿠버네티스에서 제공하는 `Server-Side Service Discovery` 로 동작합니다.

서로 다른 서비스 디스커버리로 인해서 IDC Cluster 와 EKS Cluster 간 배포된 서비스 간 통신을 위해서는 Client Side Loadbalancing 을 이용하지 못하고 직접 FQDN 을 통해 호출을 해야 했습니다.

![Figure 1-3. eureka-based micro services - review and rating](/files/post/2022-07-20-eurekube-operator/eureka-based.png)

한 예를 살펴보도록 하겠습니다. Eureka 를 기반으로 Review 서비스와 Rating 서비스가 IDC VM 환경에 있습니다. 이 경우 Review 가 Rating 을 호출할 때는 아래와 같은 Http Client 정의로 호출을 할 수 있습니다.

![Figure 1-4. Calling fegin via service name](/files/post/2022-07-20-eurekube-operator/calling-service-name.png)

Eureka에 의해서 Service Registry 에 등록이 되어있기 때문에 위와 같이 Feign Client 를 통해서 Service Name `rating` 으로 호출이 가능합니다.

`RatingClient` 에서 `getRatingsByType` 을 호출하게 되면 `Eureka Server` 로 부터 service name `rating`을 기준으로 매칭되는 service instance의 endpoint list를 받게 됩니다.

그리고 Ribbon 이나 Spring Cloud Loadbalancer에 의해서 로드밸런싱 되어 서비스를 호출하게 됩니다.

그런데 만약 `Rating` 서비스를 Kubernetes 플랫폼으로 이전하면 어떻게 될까요?

![Figure 1-5. Migrate rating service to kubernetes](/files/post/2022-07-20-eurekube-operator/migrate-rating-service.png)


이제 Rating 서비스는 Eureka Service Discovery에 등록되어있지 않으므로 Service Name 을 통해서 호출을 할 수 없습니다. 

Feign 에서는 이런 경우 url 을 직접 넣어서 호출할 수 있도록 지원하고 있습니다.

![Figure 1-6. Calling fegin via url](/files/post/2022-07-20-eurekube-operator/calling-by-url.png)

**Figure 1-6. Calling fegin via url**

하지만 이 부분에서 알 수 있듯이 Eureka 기반으로 운영하던 서비스 `Rating` 을 EKS 로 이전하면서 Http Client 에서의 ***코드 변화***가 발생하였습니다. 

만약 점진적으로 여러 서비스들을 이전한다면 해당 서비스를 호출하던 모든 Client에서 코드의 변화가 생겨야 합니다. MSA 에서는 많은 Micro Service가 inter communication 하기 때문에 모든 서비스에서 코드 변화 수정을 하는 것은 부담이 되는 선택이였습니다.

  또한, 기존에 `Rating` 서비스를 IDC → Kubernetes 로 한번에 이전하는 것은 리스크가 존재했습니다.

대부분 쿠버네티스에서 상용 서비스를 운영하는 경험이 처음이였기 때문에 빅뱅 방식처럼 한번에 서비스를 이전하여 운영하는 것은 운영 부담이 되는 선택이였습니다.

위에서 살펴본 문제점들을 해결하고자 다음과 같은 Mission 을 수립하였습니다.

> **Mission. 서로 다른 서비스 디스커버리를 통합하여 서비스 네임으로 호출 할 수 있도록 투명성 레벨을 유지한다.**

![Figure 1-7. Service Discovery Integration and Transparency](/files/post/2022-07-20-eurekube-operator/service-discovery-transparency.png)

서비스 디스커버리가 통합 된다면 아래와 같은 장점을 얻을 수 있습니다.

- **Feign 클라이언트는 서비스 네임으로 서비스를 호출할 수 있습니다.**
- **서비스가 IDC에 배포되던 Kubernetes 에 배포되던 장소 투명성이 지켜지기 때문에 점진적으로 서버를 이전시킬 수 있습니다.**

서로 다른 두 서비스 디스커버리를 통합하는 방법은 여러가지가 있겠지만 저희가 선택한 방식은 Kubernetes Operator 패턴을 사용하여 `EureKube Operator` 를 개발하는 것이였습니다.

## 2. Kubernetes Operator 패턴과 Java Operator SDK
---

Kubernetes 의 디자인 패턴 중 `Control loop` 라는 것이 존재합니다. `Control loop` 는 쉽게 생각하면   
사용자가 원하는 `Desired State` 를 지속적으로 `Current State` 에 반영하며 `Desired State` 와 `Current State` 를 맞춰주는 작업을 합니다.

한 예로 보일러 컨트롤러를 생각해볼 수 있습니다.

![Figure 2-1. Control Loop Example - Boiler Controller](/files/post/2022-07-20-eurekube-operator/boiler-controller.png)

보일러 컨트롤러는 설정한 목표 온도와 현재 온도가 일치하지 않으면 지속적으로 온도가 맞을 때 까지 온도를 올리게 됩니다.  
이것을 쿠버네티스 리소스에 대입해서 생각해보면 Deployment 에서 replica count 를 1 에서 5로 올린다면 쿠버네티스는 replica count 를 5로 맞추기 위해 Pod 갯수가 5가 될 때까지 지속적으로 동작하며 결국 Desired State와  Current State 를 만족시키게 됩니다.

![Figure 2-2. Reconcile Loop](/files/post/2022-07-20-eurekube-operator/control-loop-example.png)

쿠버네티스에서는 이런 패턴을 사용자가 사용할 수 있도록 `Custom Resource` 를 사용자가 직접 정의할 수 있도록 `Extension API` 를 제공하고 있으며 이 API를 사용해 리소스를 정의하고 해당 리소스에 반응하는 쿠버네티스 컨트롤러를 직접 개발할 수 있습니다. 그리고 이런 패턴을 오퍼레이터 패턴이라고 합니다.

오퍼레이터를 구현하기 위해서는 `Figure 2-2` 처럼 Kube Api server 로 원하는 리소스를 Watch 하고 이벤트가 발생한다면 `Reconcile loop` 를 통해서 지속적으로 `Desired State`와 `Current State` 를 맞춰주어야 합니다.

`kubernetes operator` 구현을 위해서 다양한 오퍼레이터 프레임워크가 있지만 저희 팀에서는 주로 Java 언어를 사용하여 개발하기 때문에 `Kubernetes Operator` 를 Java 로 구현할 수 있는 [java operator sdk](https://github.com/java-operator-sdk/java-operator-sdk) 를 선택하게 되었습니다.

## 3. Eurekube Operator

---

 `Eurekube Operator` 는 `Eureka` 와 `Kubernetes`의 서비스 디스커버리 통합을 위해 오퍼레이터 패턴으로 개발한 프로젝트입니다.

두 서비스 디스커버리를 통합하기 위해서 다양한 방식들이 있겠지만 쿠버네티스 오퍼레이터 패턴을 이용하여 얻을 수 있던 장점은 아래와 같습니다.

- **GitOps 방식과 통합하여 리소스를 관리할 수 있다.**

    Kubernetes Resource 들은 yaml 확장자로 매니페스트 형태로 관리되기 때문에 GitOps 를 통해 관리되기 쉽습니다. GitOps 에는 다양한 장점들이 있는데 좀 더 자세한 내용은 [이 문서](https://www.weave.works/technologies/gitops/)를 참고하시면 좋습니다.

- **kubectl 커맨드로 쉽게 리소스의 현재 상태를 파악할 수 있다.**

    kubernetes 에서는 command-line tool 로 kubectl 을 제공하고 있습니다. kubectl 은 쿠버네티스의 리소스들을 쿼리하고 터미널에 출력해주고 리소스의 Status 도 간단하게 확인할 수 있습니다.
    operator 패턴을 사용하여 Custom Resource 를 생성하게 된다면 이런 kubectl 도 통합하여 사용할 수 있다는 장점이 있습니다.

- **언제든지 쉽게 Attach & Detach 할 수 있다.**

    Operator 에 정의된 `Custom Resource` 를 yaml 로 선언하여 쿠버네티스 클러스터에 배포하면 `오퍼레이터`에 의해서 리소스의 `생성 / 삭제 / 업데이트` 에 이벤트에 맞춰 그에 맞는
    비즈니스 로직이 수행되게 됩니다.

    그렇기 때문에 `Custom Resource` 를 쿠버네티스 클러스터에 생성하였다가 추후에 해당 리소스를 삭제하게되면 `오퍼레이터`에 의해서 해당 리소스에 대한 clean-up 이 수행될 수 있습니다.
    즉, 특정 리소스를 사용자가 원할 때 생성하고 사용되지 않을 때 쉽게 삭제할 수 있습니다.

    `Eurekube Operator` 의 역할은 Eureka 와 Kubernetes 서비스 디스커버리를 통합하는 것입니다. 만약 Eureka와 Kubernetes에 배포된 서비스가 더이상 `Sync`가 필요하지 않다면
    언제든지 생성했던 `Custom Resoruce`를 삭제하여서 쉽게 `Sync` 작업을 중단할 수 있습니다.

- **Reconciliation Loop 를 통해 지속적으로 Desired State와 Current State 를 일치시킬 수 있다.**
    
    Operator 는 Reconciliation Loop 를 통해서 사용자가 정의한 Desired State 를 지속적으로 Current State 와 일치시켜주는 작업을 진행합니다.
    
    그렇기 때문에 만약에 실제 쿠버네티스의 리소스 상태에 변화가 생기더라도 Reconciliation Loop에 의해서 Desired State 로 복구되게 됩니다.
    
    만약 어떠한 큰 장애로 쿠버네티스 클러스터가 모두 다운되더라도 정의한 Kubernetes Resource 만 존재한다면 쉽게 기존 Desired State 로 복구할 수 있습니다.
    

### Eurekube Operator 동작 방식 및 예제

![Figure 3-1. Eurekube operator Simple Architecture](/files/post/2022-07-20-eurekube-operator/eurekube-operator-simple-arch.png)

`Figure 3-1` 은 `Eurekube operator` 에 `simple architecture` 입니다. `Eurekube Operator` 는 위의 이미지에서 보면 알수 있듯이 `Kubernetes Cluster` 에 `Service Endpoints` 를 `Eureka Server` 에 등록해주는 것을 알 수 있습니다.

이렇게 쿠버네티스 클러스터에서 외부 서비스에 `Desired State` 를 반영하는 패턴을 `Provisioner Pattern` 이라고도 합니다. 저희 팀에서는 [kubecon 2019. Growth and Design Patterns in the KRM](https://static.sched.com/hosted_files/kccncna19/5e/eric-tune-kcon-slides-final.pdf) 에 정의된 패턴을 주로 사용하고 있습니다.

이제 위의 그림을 따라가면서 간단하게 `Eurekube Operator` 가 어떻게 동작하는지 살펴보도록 하겠습니다.

### (1) CRD - EurekaSyncer

먼저 `Eurekube Operator` 에서 사용할 `Custorm Resource` 를 정의하여야 합니다. `Figure 3-1` 의 (1) 을 보면 `EurekaSyncer Resource` 를 확인할 수 있습니다.

 `Custorm Resource` 를 정의하기 위해서는 `Kubernetes API Extension` 인 [Custom Resource Definition](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/) 을 사용하여야 합니다. 해당 API 로 CRD 를 작성하고 쿠버네티스에 적용하면 아래와 같이 정의한 `Custom Resource` 를 쿠버네티스에서 사용할 수 있습니다.

![Figure 3-2. Custom Resource Definition - EurekaSyncer For Eurekube Operator](/files/post/2022-07-20-eurekube-operator/custom-resource-definition.png)

위와 같이 `Custom Resource Definition` 을 정의하고 쿠버네티스 클러스터에 배포하면 쿠버네티스에서는 이제 `EurekaSyncer` 라는 리소스에 대해서 인지하고 사용할 수 있게 됩니다.

![Figure 3-3. Custom Resource Definition - additionalPrinterColumns](/files/post/2022-07-20-eurekube-operator/additional-param.png)

![Figure 3-4. Custom Resource Definition - additionalPrinterColumns Example](/files/post/2022-07-20-eurekube-operator/additional-param-ex.png)

특히 CRD에서 `additionalPrinterColumns` 를 사용하게되면 kubectl 로 리소스를 가져왔을 때 status 정보를 콘솔에 출력할 수 있어서 리소스의 현재 상태를 파악하는데에 도움이 될 수 있습니다.

![Figure 3-4. Custom Resource - EurekaSyncer for EurekubeOperator](/files/post/2022-07-20-eurekube-operator/custom-resource.png)

Figure 3-4는 Figure 3-2 에서 정의한 CRD 를 사용하여 생성된 `Custom Resource` 입니다. spec 에서 의미하는 것이 어떤 의미인지 살펴보도록 하겠습니다.

**(1) mode** 

kubernetes에서는 pod의 endpoint 도 있지만 service의 endpoint 도 존재합니다. 이런 경우 사용자가 원하는 다양한 리소스의  IP 를 지원하기 위해 `mode` spec을 정의하였습니다. (pod or service)

> **참고** 
>
> 일반적으로 cluster ip 는 가상의 IP로 외부에서 접근하는 IP로 사용할 수 없지만 aws eks에서는 CNI 플러그인으로 ENI 를 사용하고 있습니다. 
> 또한 11번가에서는 IDC 네트워크망과 AWS의 VPC를 Direct Connect를 통해서 연결하였습니다.
> 그렇기 때문에  pod의 ip 를 통해 IDC에서도 직접 pod 호출을 할 수 있습니다.


**(2) selector**

`selector`는 `Eurekube Operator` 가 어떤 리소스를 Eureka 에 Sync 하고 싶은지 선택하는 지시자입니다.

이 예제 `Figure 3-4` 에서는 namespace rating 에 rating app 을 pod ip 를 eureka 에 싱크 하도록 정의되어 있습니다.

### (2) Watch Eureka Syncer Resource & Reconciliation

 위에서 Custom Resource Definition 을 정의하여 Custom Resource 를 배포하였다면 이제 Eurekube Operator 에서 해당 Custom Resource에 반응하여 비즈니스 로직이 수행되어야 합니다. `[Figure 3-5]`

![Figure 3-5. watch EurekaSyncer Custom Resource](/files/post/2022-07-20-eurekube-operator/watch-eureka-syncer.png)

 그러기 위해서는 kube api server 에 kubernetes client의 informer를 이용해서 resource type에 대해서 watch를 해주어야 합니다.

![Figure 3-6. Reconciler Interface](/files/post/2022-07-20-eurekube-operator/reconciler-interface.png)

`java operator sdk` 에서는 개발자가 직접 이런 구현 작업을 하지 않고 간단히 `Figure 3-6` 같은 인터페이스를 구현하면 프레임워크단에서 `Target Custom Resource` 에 대해서 이벤트가 발생할 때마다 reconcile 메서드를 트리거 해주는 feature 를 지원해주고 있습니다.

![Figure 3-7. [code snippet] Reconciler for EurekaSyncer Resource](/files/post/2022-07-20-eurekube-operator/reconciler-for-custom-resource.png)

`Fiugre 3-6` 에서 제공되는 인터페이스를 이용해 `EurekaSyncerReconciler` 를 구현해보겠습니다. 

`Figure 3-7` 에서 보면 알 수 있듯이 `EurekaSyncerReconciler` 는 `Reconciler<EurekaSyncer>` 를 구현하여 `reconcile` 메서드에 비즈니스 로직이 등록되어있습니다.

이제 쿠버네티스에서 `EurekaSyncer` Custom Resource에 대한 생성 / 업데이트 등 이벤트가 발생하게 되면 java operator sdk 프레임워크 안에서 이 이벤트를 감지하여 `reconcile` 메서드를 호출해주게 됩니다.

간단하게 `reconcile` 메서드 내부의 로직을 살펴보면 `Figure 3-2` 에서 설정한 `spec.mode (pod or service)` 의 값을 받아서 mode에 따라 적합한 syncStratrgy 를 통해 eureka server에 sync 작업을 진행하게 됩니다.

![Figure 3-8. get endpoints & register eureka instance](/files/post/2022-07-20-eurekube-operator/get-endpoints-and-sync.png)

그러면 `Figure 3-8` 에서 볼 수 있듯이 (2) EurekaSyncer 리소스의 spec 정보를 활용하여 kubernetes에서 endpoints 정보들을 가져오고 (3) 가져온 endpoints 들을 eureka server에 Instance 로 등록하게 됩니다.

만약 kubernetes에 `rating` 이라는 서비스가 존재하고 2대의 POD가 있다고 가정해보겠습니다.

- **pod_rating_1 : 10.0.0.1:8080**
- **pod_rating_2 : 10.0.0.2:8080**

이 상황에서 위에서 살펴본 `Figure 3-4` 와 같은 EurekaSyncer 리소스가 생성된다면 Eurekube Operator는 위의 두 pod의 endpoint 정보를  쿠버네티스 서버로 부터 가져와서 eureka server의 instance 로 등록을 시도하게 됩니다.

성공적으로 등록이 된다면 `Figure 3-8` 에서 볼수 있듯이 Eureka 서버에는 `rating` 이라는 서비스가 생성되고 instance 로 두 rating pod의 endpoint 가 등록되게 됩니다.

그러면 Eureka Client 를 사용하는 IDC 내의 `Review` 서버에서는 rating에 대한 service registry 정보를 통해 직접 호출을 할 수 있게 됩니다.

### (3) ErrorStatusHandler

만약 Reconcile 중에 에러가 발생하면 이 에러에 대한 핸들링은 어떻게 할 수 있을까요?

java operator sdk 에서는 ErrorHandling 을 위해서 `ErrorStatusHandler` 라는 인터페이스를 제공하고 있습니다.

![Figure 3-9. ErrorStatusHandler Interface](/files/post/2022-07-20-eurekube-operator/error-status-interface.png)

ErrorStatusHandler 는 `Figure 3-4` 에서 볼 수 있듯이 updateErorrStatus 메서드를 지닌 간단한 인터페이스입니다. 이 인터페이스를 사용하면 Reconcile 도중에 에러가 발생하면 에러에 대한 처리를 할 수 있습니다.

![Figure 3-10. [code snippet] Get visibility when reconcile fails with kubernetes event and status](/files/post/2022-07-20-eurekube-operator/error-status-impl.png)

`Figure 3-10` 은 `Figure 3-9` 인터페이스를 `EurekaSyncerReconciler` 에서 구현하여 Reconcile 시 발생하는 에러에 대한 핸들링을 처리하고 있는 코드입니다.

에러가 발생하면 에러에 대한 가시성을 외부에서 얻기 위해서 첫째로 쿠버네티스 클러스터에 Event 리소스를 생성하여 이벤트를 알리고, 뿐만 아니라 kubectl을 통해 에러를 확인할 수 있도록 EurekaSyncer의 Status 에도 에러에 대한 정보를 작성하고 있습니다.

### (4) Cleaner

위에서 주로 EurekaSyncer Custom Resource에 대한 생성 / 업데이트 시의 처리를 살펴보았다면 이제 리소스를 삭제하는 경우에는 어떻게 동작하는지 살펴보겠습니다.

![Figure 4-1. cleaner interface](/files/post/2022-07-20-eurekube-operator/cleaner-interface.png)

java operator sdk 는 `Figure 4-1` 같이 `Cleaner` 라는 인터페이스를 구현하면 [kubernetes finalizer](https://kubernetes.io/docs/concepts/overview/working-with-objects/finalizers/) 를 리소스 메타데이터에 자동으로 선언해줍니다. 

그렇기 때문에 해당 리소스에 삭제 명령을 보내면 쿠버네티스 API 서버에서는 해당 리소스를 바로 삭제하지않고 컨트롤러에 의해서 clean up 로직이 수행됩니다.

![Figure 4-2. [code snippet] Implemented Cleaner in EurekaSyncerReconciler](/files/post/2022-07-20-eurekube-operator/cleaner-impl.png)

CleanUp에 대한 구현은 `Figure 4-2` 와 같이 cleanup 메서드를 구현하면 해당 리소스에 대한 삭제가 발생하였을 때 이에 대한 처리를 직접 구현할 수 있습니다.

위의 코드의 경우에는 `EurekaSyncer` 리소스가 삭제될 때 Eureka 에 등록된 Endpoint 리스트를 삭제하는 작업을 수행하게 됩니다.

참고로, 기존에 Eureka 에 등록되어있던 `Endpoint` 를 지울 때 주의해야할 점이 있습니다. Eureka Server에 등록된 Service Registry 정보는 Eureka Client, Client Side Loadbalancing client 등에서 cache 되어져 있을 수 있습니다.

그렇기 때문에 Eureka Server에서 특정 Endpoint 를 지우더라도 일정 시간동안 캐시가 지워지기 전까지 Endpoint가 가리키는 Pod는 살아있어야 합니다.

만약 Pod가 바로 Shutdown이 될 경우 해당 Pod의 Endpoint 를 호출하는 클라이언트들에서 장애가 발생할 수 있기 때문입니다.

그렇기 때문에 EurekaSyncer 가 바라보는 Pod의 경우 [lifecycle hook](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/) 을 이용하여  pod shutdown이 바로되지 않도록 sleep을 걸어두어야 합니다.

![Figure 4-3. lifecycle hook](/files/post/2022-07-20-eurekube-operator/lifecycle-hook.png)

11번가에서는 EurekaSyncer가 바라보는 Pod 에 `Figure 4-3` 같이 sleep 커맨드를 lifecycle.prestop 을 걸어서 안전하게 쿠버네티스와 Eureka 서비스 간에 `zero-downtime` 배포가 되도록 처리하였습니다. 

### (5) Register Event Source

만약에 `EurekaSyncer` 리소스가 아니라 다른 리소스의 변화로 `EurekaSyncer` 이벤트를 트리거 하려면 어떻게 해야할까요?

Eurekube Operator의 케이스에서는 만약 Pod 가 생성 및 제거가 될 경우 `Endpoints` 리소스에 변화가 생기게 됩니다.

이 경우에는 `Endpoints` 리소스 변화를 감지하고 새롭게 Eureka Server에 적용 (생성 또는 삭제) 을 해주어야 합니다.

`Java Operator SDK` 에서는 `EventSourceInitializer` 인터페이스를 제공하여 `prepareEventSources` 메서드를 구현하면 다른 리소스의 이벤트를 등록하여 reconcile이 동작할 수 있도록 제공합니다.

현재 Eurekube Operator 의 경우에도 `EurekaSyncer` 가 바라보는 대상 `Endpoints` 들에 변화가 생기면 이벤트를 받아서 Eureka Server에 새롭게 register or de-register 를 수행하고 있습니다.


## 4. 마치며

---

위와 같이 `Java Operator SDK` 를 통해서 구현된 `Eurekube Operator` 를 통해서 십일번가에서는 Spring Cloud 기반의 서비스들과 Kubernetes 기반의 서비스들을 쉽게 Service Discovery 통합을 할 수가 있었습니다.

현재 Eurekube Operator에 의해서 관리되는 `EurekaSyncer` 리소스는 Kubernetes에 배포되는 API 서버들의 Helm Chart에 포함되어져 있습니다.

그렇기 때문에 Kubernetes에 배포되는 시점에 모든 API 서버는 Eureka 에 자동으로 등록되고 있습니다.

2022년 7월 기준으로 약 4개의 서비스가 `production` 환경에 배포되어 Kubernetes & IDC Spring Cloud 가 성공적으로 서비스 간 통신을 수행하며 마이그레이션 되고 있습니다.

`Operator` 는 쿠버네티스 컨트롤러 패턴에 비즈니스 로직을 추가하여 커스텀 리소스를 사용할 수 있게 해줍니다. 

만약 Java 언어를 통해서 Kubernetes Operator 서비스를 개발한다면 Java Operator SDK 에 하나의 좋은 선택지가 될 수 있을 것 같습니다.

긴 글 읽어주셔서 감사합니다. 🙇

> Contributors : 김광용, 김보배, 안희석, 장준영, 전지원, 최유진, 허서윤