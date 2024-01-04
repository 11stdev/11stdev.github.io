---
layout: post
title: 'Java로 만들어진 Kubernetes Operator - 어떻게 테스트할까?'
author: 김보배
date: 2022-08-01
tags: [kubernetes, operator, test, java]
---

안녕하세요. 11번가 Core Platform 개발팀의 김보배입니다.

앞선 [블로그 글(Service Discovery 통합을 위한 Kubernetes Operator 구현 - Eurekube Operator)](https://11st-tech.github.io/2022/07/20/eurekube-operator/)에서 Eurekube Operator의 아이디어, 설계에 대해 이야기해보았습니다. 

이번 글에서는 Eurekube Operator의 개발을 마치고 제대로 동작하는지 확인하기 위해 어떻게 테스트했는지에 대해 이야기해보고자 합니다. 

이전 블로그 글을 읽었다는 전제하에 작성한 글로, 만약 아직 읽지 않으셨다면 읽고 오시길 추천드립니다.

<br/>

# **Integration test with Testcontainers**

11번가에서는 안정적인 애플리케이션을 개발하기 위해 테스트 코드를 최대한 작성하고 있습니다. 테스트는 유닛 테스트와 통합 테스트로 나누어 구성하고 있으며, 외부의 애플리케이션들에 종속적인 코드들이 있다면 [mockito](https://site.mockito.org/) 같은 Mocking 툴을 이용하여 테스트하고 있습니다.

Eurekube Operator의 경우도 마찬가지로 유닛 테스트/통합 테스트로 테스트를 구성하고자 하였습니다. 

하지만 타 애플리케이션들과 차이점이 있다면, Eureka server / Kubernetes와 커뮤니케이션하며 동작하는 기능들이 많고 Operator 특성상 Custom resource의 변화에 따라 Reconcile 작업이 정상적으로 동작하는지 테스트할 필요가 있었습니다. 

이를 위해 직접 Eureka, Kubernetes를 직접 동작시켜 테스트할 수 있도록 Testcontainers를 도입하였습니다.

<br/>

## **[Testcontainers](https://www.testcontainers.org/)**

![Figure 1. Testcontainers](/files/post/2022-08-01-eurekube-operator-test/testcontainers.png)

Testcontainers에 대한 간단한 설명을 하자면 Docker container를 이용해 Junit 테스트를 지원하는 library입니다.  

많이 사용하는 애플리케이션(데이터베이스, MQ, ES 등)에 대해서 Module을 지원하여 테스트 지원을 하고 있으며, [GenericContainer](https://www.testcontainers.org/features/creating_container/)를 기반으로 원하는 docker container를 직접 구성하여 테스트를 진행할 수도 있습니다.

하지만 장점만 있을 수는 없는 법! 직접 container를 띄우는 방법인 만큼 오래 걸려 테스트 시간이 증가하는 단점이 있습니다. 

이 문제를 해결하기 위해 [Junit의 Tag annotation](https://junit.org/junit5/docs/current/user-guide/#writing-tests-tagging-and-filtering)을 이용했습니다. Integration test는 따로 마킹하여 배포 전 CI 단계 같은 필요한 경우에만 진행하도록 설정해두어 운영 중입니다.

Eurekube Operator에서는 2개의 애플리케이션(Eureka, Kubernetes)에 대한 test container가 필요했습니다.

<br/>

### **Eureka**

Eureka의 경우, GenericContainer와 Eureka container image를 사용했습니다.

![Code 1. Eureka testcontainer code](/files/post/2022-08-01-eurekube-operator-test/eureka-testcontainer-code.png)

위 예시 코드처럼 `EurekaServerTestContainer` 추상 클래스를 상속받으면 Eureka를 docker container로 구성해 테스트를 진행할 수 있습니다.

그럼 Eurekube Operator에서 container로 띄운 Eureka 주소정보가 필요할 텐데, 어떻게 이용할 수 있을까요? 

`GenericContainer` 에서는 `GenericState` interface를 구현체인데요. 해당 interface에서 host, port 정보를 가져올 수 있도록 default method를 제공하고 있습니다. 

즉, `eurekaServerTestContainer.getHost()`, `eurekaServerTestContainer.getMappedPort(EUREKA_SERVICE_PORT)`를 통해 Testcontainers로 구성한 Eureka 주소를 가져올 수 있고, 이를 application property에 주입해 Eurekube Operator에서 사용할 수 있도록 구성했습니다.

<br/>

### **Kubernetes**

Kubernetes의 경우, Testcontainer의 [K3s module](https://www.testcontainers.org/modules/k3s/)을 사용했습니다. [K3s](https://rancher.com/products/k3s)는 lightweight kubernetes로서 testcontainers에서는 이를 이용해 Kubernetes APIs test를 지원하고 있습니다.

- 참고: Testcontainers (v1.17.2)에서 k3s module은 아직 incubating 상태입니다. 추후 Module에 큰 변경사항이 존재할 수 있습니다.

![Code 2. K3s testcontainer code](/files/post/2022-08-01-eurekube-operator-test/K3s-test-container-code.png)

`K3sContainer`는 `getKubeConfigYaml` method를 통해 kube config 파일을 읽어올 수 있도록 지원합니다. 

Eurekube Operator는 [Fabric8io kubernetes client](https://github.com/fabric8io/kubernetes-client)를 이용하므로 kubeConfigYaml을 이용하여 `DefaultKubernetesClient`를 설정하였습니다.

 `K3sTestContainer` 추상 클래스를 상속받은 클래스들에서는 client를 통해 k3s로 API request를 보낼 수 있게 되었고, 이를 이용해 reconcile이 제대로 동작하는지 확인하는지 테스트를 진행하였습니다. 

![Code 3. Namespace delete test code](/files/post/2022-08-01-eurekube-operator-test/namespace-delete-test-code.png)

참고로, Reconcile 과정에서 리소스가 추가/삭제 요청을 하면 kubernetes에서 직접 리소스를 생성/제거 하는데 **약간의 시간(delay)**이 걸리게 됩니다. 이러한 시간을 고려하지 않고 reconcile 직후에 리소스 유무를 검증하게 되면 생성/제거 시간 차이로 인해 오류가 날 수 있습니다. 

그렇기에  kubernetes resource 검증 테스트에서는 **awaitility를 활용하여 시간 차이를 고려하는 테스트를 작성**하였습니다.

![Code 4. Namespace delete test code with awaitility](/files/post/2022-08-01-eurekube-operator-test/namespace-delete-test-code-await.png)

Eureka, Kubernetes testcontainers를 통해서 Eurekube Operator 기능에 변경사항이 있더라도 두려운 마음없이 배포할 수 있게 되었습니다.

하지만 이러한 테스트들에도 불구하고, 실제 네트워크 환경의 장애가 발생한다면 어떠한 문제가 발생할지 확인할 수 없었습니다. 이에 저희 팀은 Chaos engineering 방법에 대해 고민해보기 시작했습니다.

<br/>

---

# **Chaos engineering with Chaos mesh**

Eurekube Operator의 핵심 목표 및 기능은 **On-premise Ereka와 Cloud kubernetes의 Service discovery 통합 지원**입니다. Eureka에는 Kubernetes Service의 목록들을 등록해주고, Kubernetes에는 Eureka 목록들을 Service 리소스로 등록해줍니다.

이렇듯, 핵심 기능이 **Network에 의존적**입니다. 

Eurekube Operator 코드 레벨에 기능적 이슈가 하나도 없다해도 On-premise <-> Cloud 사이에 Network 장애 발생 시 제대로 동작하지 않을 수 있고, 이에 따라 Network 장애 복구가 되었을 때 제대로 동작하는 지에 대한 테스트가 필요했습니다. 

<br/>

## **Chaos engineering**

Chaos engineering은 시스템 능력의 한계를 확인하고, 능력에 대한 확신을 가지기 위해 다양한 격동적인 조건(turbluent conditions)으로 시스템을 테스트하는 분야입니다. 

말이 좀 거창하지만, Chaos(혼돈, 혼란)라는 이름에서 알 수 있듯 시스템에 여러 혼란을 주며 발생할 수 있는 문제를 파악하는데 사용하는 방법입니다.

<br/>

## **[Chaos Mesh](https://chaos-mesh.org/)**

Eurekube Operator는 Kubernetes 환경에서 운영되고 있습니다. 그래서 Kubernetes 환경에서 사용할 수 있는 Chaos engineering 툴을 알아보았고, 저희는 Chaos Mesh를 도입하게 되었습니다.

![Figure 2. Chaos Mesh](/files/post/2022-08-01-eurekube-operator-test/chaos-mesh-logo.png)

Chaos mesh는 Cloud native Chaos engineering platform이라 소개하고 있으며, 오픈 소스([chaos-mesh github](https://github.com/chaos-mesh/chaos-mesh))로 CNCF의 incubating project이기도 합니다.

저희가 Chaos mesh를 선택한 이유는 여러가지가 있지만 가장 큰 이유는 Kubernetes를 위해 디자인 된 애플리케이션이라는 것입니다. 다양한 종류의 CRD(Custom Resource Definition)를 제공하여 CR(Custom Resource)로 쉽게 장애를 발생시킬 수 있습니다.

예를 들어, 아래의 리소스를 배포해 `temp` namespace의 모든 Pod에 Network disconnection을 발생시킬 수 있습니다.

```yaml
kind: NetworkChaos
apiVersion: chaos-mesh.org/v1alpha1
metadata:
  name: temp-network-disconnection
spec:
  selector:
    namespaces:
      - temp
  mode: all
  action: partition
  direction: both
```

이 밖에도 Poor network condition, Limit bandwidth 등 Network 장애 뿐 아니라 CPU/Memory stress 장애 등 [다양한 Fault](https://chaos-mesh.org/docs/simulate-pod-chaos-on-kubernetes/)들을 제공하고 있습니다. 

또한, 장애를 주기적으로 주입되도록 Scheduling 할 수 있고, 여러 장애들을 조합하는 Ochestration 기능도 제공하고 있습니다.

<br/>

11번가에서는 Chaos mesh를 적극적으로 활용하여 Kubernetes에 배포되는 애플리케이션들을 Chaos engineering 하고 있습니다. Eurekube Operator도 마찬가지입니다. Chaos mesh에서 제공되는 [다양한 Network Fault](https://chaos-mesh.org/docs/simulate-network-chaos-on-kubernetes/)를 통해 Chaos engineering을 진행하였습니다.

![Figure 3. Eurekube Operator overview](/files/post/2022-08-01-eurekube-operator-test/eurekube-operator-overview.png)

정상적인 상황에서는 Eurekube Operator는 On-premise Eureka와 Communication을 통해 

- Kubernetes Service를 생성/제거
- Kubernetes 애플리케이션(IP)들을 Eureka에 등록/제거
- Eureka에 직접 등록한 애플리케이션들에 대한 heartbeat 전송

등 여러 작업을 수행할 것입니다.

<br/>

이런 상황에서 Network 장애가 일어난다면 어떻게 될까요?

![Figure 4. Network failure](/files/post/2022-08-01-eurekube-operator-test/network-failure.png)

맞습니다. 위에서 이야기한 작업들이 정상적으로 수행되지 않을 것입니다. 

이 문제는 Infra 단에서 발생하는 문제로 직접 해결할 수 있기보다 Service discovery를 사용하는 애플리케이션들에서 Self healing, Circuit breaker, Fallback 등을 통해 최대한 오류를 제어해야 할 것입니다.

저희가 주목하고자 한 것은 **'장애가 복구된 후, 아무런 조치가 없어도 정상적으로 동작하는가? (Self healing)'** 였습니다. 

Chaos mesh를 통해 이 부분에서 Eurekube Operator의 문제점을 파악했고 해결한 경험을 이야기해보고자 합니다.

<br/>

### **Eurekube Operator 문제점 발견 그리고 해결**

장애 상황에 대해 이야기 하기 전에 On-premise Eureka의 설정에 대해 설명드리겠습니다.

### **Eureka와 Heartbeat**

On-premise Eureka는 등록된 주소(인스턴스)에 대한 Heartbeat가 특정 초 이상 오지 않는 경우, 주소 등록을 해제하도록 설정해 운영 중입니다.

보통은 서비스 서버가 다운되기 전 Eureka로 주소 등록 해제를 직접 하지만, 불가피하게 하지 못했을 경우 서비스 안정을 위해 알아서 주소 등록을 해제 해주기 위함입니다.

Eureka Client를 사용하면 일반적으로 heartbeat를 자동으로 수행해 주소 등록이 해제되는 것을 막아주지만, Eurekube Operator는 Eureka Client를 사용하는 것이 아니기에 직접 Scheduling을 통해 등록한 주소에 대한 heartbeat를 전송하고 있습니다.

![Figure 5. Eurekube Operator scheduling](/files/post/2022-08-01-eurekube-operator-test/eurekube-operator-scheduling.png)

![Code 5. Heartbeat scheduling code](/files/post/2022-08-01-eurekube-operator-test/heart-beat-code.png)

<br/>

### **문제 발견**

정상적으로 Eurekube Operator가 동작하고 있는 상황에서 Chaos-mesh를 통해 Network partition 장애를 발생시켰습니다.

```yaml
kind: NetworkChaos
apiVersion: chaos-mesh.org/v1alpha1
metadata:
  namespace: eurekube-operator
  name: eurekube-network-partition
spec:
  selector:
    namespaces:
      - eurekube-operator
  mode: all
  action: partition
  direction: to
```

장애를 몇 분여간 지속하고, 복구하였습니다. 그 후 **Eurekube Operator의 상황을 살펴보니, 정상작동하지 않고 있었습니다.** 

기존에 Eureka에 등록 했었던 주소들을 새로 등록하지 않았고, heartbeat도 실패하고 있었습니다. 저희 팀은 이러한 이유에 대해 살펴보기 시작했고, 원인을 찾아냈습니다.

제일 큰 원인은 **기존에 Eureka에 등록했던 Kubernetes Endpoints 주소들이 정상적으로 등록되어있지 않은 것**이었습니다. 그럼 왜 기존에 등록되어 있던 Endpoints 주소들이 등록되어 있지 않던 것 일까요?

답은 위에서 설명한 Eureka의 설정에 있었습니다. Network 오류로 인해 그 시간동안 **Eurekube Operator에서는 heartbeat를 전송하지 못했고, heartbeat가 오지 않는 시간이 길어지면서 Eureka에서 결국 주소 등록을 해제했던 것** 입니다.

<br/>

### **해결**

위 문제를 해결하기 위해서는 장애가 복구 되었을 때, **기존에 연결되어 있던 Endpoints 주소들을 Eureka에 다시 등록**해주어야만 했습니다.

하지만 기존 Eurekube Operator는 구조 상 특정 Kubernetes Resource 변화에 따라 Trigger 되어야지만 동작하도록 구현되어있었습니다. 그렇기에 장애가 복구되었더라도 알아서 자동으로 다시 주소를 등록하는 과정을 할 수 없었습니다.

그래서 이를 해결하기 위해 적용한 것이 reschedule 입니다.

![Figure 6. Eurekube Operator reschedule](/files/post/2022-08-01-eurekube-operator-test/reschedule.png)

java-opeartor-sdk 에서는 Reconcile 작업 후 **[rescheduleAfter 메서드](https://github.com/java-operator-sdk/java-operator-sdk/blob/main/operator-framework-core/src/main/java/io/javaoperatorsdk/operator/api/reconciler/BaseControl.java#L11-L23)를 통해 정의한 period마다 같은 EurekaSyncer resource로 reconcile을 trigger 해줍니다.** 이를 통해 period마다 계속해서 reconcile 작업이 수행되도록 설정할 수 있습니다.

[kubernetes client library](https://kubernetes.io/docs/reference/using-api/client-libraries/)들을 많이 사용해신 분이라면 이 작업이 informer의 resync의 동작과 유사하다고 생각하실 것 같네요. 거의 유사하나, java-operator-sdk framework 단에서 scheduling을 사용해 직접 구현해놓은 것이라고 보면 됩니다. 

java-operator-sdk에서는 [fabric8io의 kubernetes client](https://github.com/fabric8io/kubernetes-client)를 사용하고 이 부분에서 지원되는 resync 기능이 있었으나, 이를 framework 단에서 비활성해 사용을 막고 있었습니다. 그래서 저희는 java-operator-sdk에서 제공되는 reschedule을 사용하였습니다.

reschedule을 적용하기는 매우 간단합니다. 아래처럼요.

![Code 6. Reschedule code](/files/post/2022-08-01-eurekube-operator-test/reschedule-code.png)

Reschedule을 적용하고 똑같은 Network 장애를 주입하여 복구하였을 때, 설정한 **reschedule period 내 최신 버전의 resource들로 reconcile 작업되며 Eureka에서 제거되었던 Kubernetes Endpoints 주소들이 등록되는 것을 확인할 수 있었습니다.** 

**즉, 아무런 액션을 취하지 않아도 자동으로 복구가 진행되도록 구현되었습니다.**

<br/>

# Conclusion

이렇게 Testcontainer와 Chaos mesh를 이용해서 보다 더 안정적인 Eurekube Operator가 될 수 있었습니다. 이 경험을 기반으로 새로운 Kubernetes 서비스 오픈이나 feature 업데이트 등의 변화가 있을 때에도 앞선 Test 툴들을 이용하여 안정적인 서비스를 만들어가고 있습니다.

Java 언어로 Kubernetes를 다루고 계실 때 통합 테스트에 고민이 되신다면 Testcontainer를, Kubernetes 환경에서 Chaos engineering 툴을 고민하고 계시다면 Chaos mesh를 생각해보시는 것도 좋은 선택이라고 생각합니다 :)

긴 글 읽어주셔서 감사드리며, 언제나 피드백은 환영입니다.

> **Contributors**: 김광용, 김보배, 안희석, 장준영, 전지원, 최유진, 허서윤

