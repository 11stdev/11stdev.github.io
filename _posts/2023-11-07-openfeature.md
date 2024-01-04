---
layout: post
title: Feature Flag - 안전하고 신뢰할 수 있는 배포로 나아가는 열쇠 🔑
author: 전지원
date: 2023-11-07
tags: [kubernetes, openfeature, cncf, feature-flag]
---

안녕하세요. 11번가 Core플랫폼개발팀 전지원입니다. 저희 팀에서는 Spring Cloud 기반의 전사 MSA 플랫폼인 `Vine`의 공통 컴포넌트 개발과 운영을 담당하고 있습니다. 또한 각 소프트웨어 엔지니어링 조직이 Self-Service 기능을 사용할 수 있도록 툴체인과 워크플로우를 설계하고 있으며, 애플리케이션 전체 수명 주기에 필요한 운영 요구사항을 포괄하는 IDP (Internal Developer Platform)인 *Wheelhouse*를 개발하여 제공하고 있습니다.

이번 아티클에서는 Feature Flag 개념에 대해서 설명드리고, 기능을 Production 환경에서 안전하게 배포하고 실험하기 위해 Feature Flag를 도입한 사례를 소개해드리고자 합니다.

## Contents

- [Contents](#contents)
- [사례로 살펴보는 문제점](#사례로-살펴보는-문제점)
- [Feature Flag?](#feature-flag)
  - [Feature Flag 의 형태](#feature-flag-의-형태)
- [OpenFeature / Flagd](#openfeature--flagd)
  - [Flagd를 사용해 Flag Management System (Flag Backend, Flag Evaluation Engine) 구축하기](#flagd를-사용해-flag-management-system-flag-backend-flag-evaluation-engine-구축하기)
  - [Flagd에서 사용할 Flag 정의하기](#flagd에서-사용할-flag-정의하기)
  - [OpenFeature SDK (Java) 살펴보기](#openfeature-sdk-java-살펴보기)
    - [Evaluation API](#evaluation-api)
    - [Evaluation Context](#evaluation-context)
    - [Hooks](#hooks)
- [실제 애플리케이션 코드를 작성해볼까요?](#실제-애플리케이션-코드를-작성해볼까요)
- [마치며](#마치며)
- [References](#references)

## 사례로 살펴보는 문제점

Feature Flag를 소개드리기 전에, 문제 사례를 하나 소개드리겠습니다. 최근 저희 팀에서는 각 서비스 개발팀이 REST(HTTP) 호출 기반의 서버 개발을 보다 쉽게 gRPC 호출 기반으로 전환할 수 있게끔 전사 공통 라이브러리를 개발하여 제공하고 있었습니다.

![Switching between HTTP and gRPC](/files/post/2023-11-07-openfeature/image-20231030200656460.png)

개발한 서버 라이브러리는 개발 및 검증 환경에서는 충분히 검증이 되었지만, 실제 엄청난 트래픽이 들어오는 Production 환경에 적용되었을 때도 이슈 없이 완벽하게 동작할 수 있다는 확신이 없었습니다. 만약 **Production 환경에 배포한 후 예상치 못한 이슈가 발생한다면 신속하게 롤백을 진행해야 하지만, 롤백 재배포에도 적지 않은 시간이 소요될 수 있습니다.** 😱

따라서 코드를 배포할 때 안전장치를 함께 포함시켜 런타임에서 코드를 변경하지 않고, 신속하게 기능을 전환할 수 있는 매커니즘이 필요했습니다. HTTP 호출이 이루어지도록 우선 배포가 된 후, gRPC 호출이 되도록 외부에서 제어시킨 후 문제가 생기면 다시 원래대로 돌아가는 방식, **이런 경우에 Feature Flag를 활용해볼 수 있습니다.**

## Feature Flag?

Feature Flag란 **특정 기능을 동적으로 활성화 혹은 비활성화하기 위해 사용되는 조건부 코드 실행 매커니즘**입니다. 런타임 환경에서 특정 조건에 따라 코드 특정 부분을 스위치하여 실제 사용자에게 제공되는 서비스 기능을 다르게 제어할 수 있습니다. 특히 이러한 제어를 위해서 매번 코드를 수정해서 배포할 필요가 없다는 특징이 있습니다.

![Feature Flag 미리보기](/files/post/2023-11-07-openfeature/image-20231103155220565.png)

특히 Production 환경에 코드를 배포하게 되면 새롭게 반영되는 기능은 곧바로 사용자에게 전달되기 때문에 예상치 못한 이슈가 발생할 수 있습니다. (테스트를 충분히 거치더라도 예상치 못한 이슈는 언제든지 발생할 수 있기 때문입니다.)

Feature Flag를 사용하면 어떤 점이 좋을까요? Feature Flag를 사용하면 **코드 배포에서 기능 Rollout을 분리**할 수 있습니다. 이러한 분리는 **서비스의 신규 버전 릴리즈와 무관하게 배포하고자 하는 기능에 대해서 누가, 언제, 무엇을 볼 것인지에 대한 제어가 가능**해집니다.

이러한 Feature Flag는 다음과 같은 목적으로 사용할 수 있으며 그에 따른 이점은 아래와 같습니다.

1. 시스템 안정성
   - 서비스가 일시적으로 높은 부하를 받는 경우나 버그와 같은 긴급한 장애 상황에서 특정 기능을 즉시 비활성화하거나 대체할 수 있습니다.
   - 서버나 데이터베이스 등의 시스템 이전을 진행하는 과정에서 기존 시스템에서 신규 시스템으로의 트래픽을 제어할 수 있습니다.
2. A/B 테스트
   - 여러 버전의 기능을 사용자에게 제공하여 피드백을 받고 필요한 경우에는 변경할 수 있습니다.
3. 개인화
   - 사용자별로 특정 기능을 활성화하거나 비활성화하여 개인화된 경험을 제공할 수 있습니다.
4. Rollout 관리
   - 새로운 기능을 곧바로 모든 사용자에게 노출하지 않고 일부 사용자 혹은 그룹에 먼저 테스트하도록 구성할 수 있습니다. 따라서 낮은 위험도 내에서 안정성과 성능 문제를 식별해서 수정할 수 있습니다. 이러한 과정이 반복되면서 기능이 안전하게 보강되어 점진적인 릴리즈가 가능합니다.

> Feature Flag라는 용어는 Feature Toggle, Feature Bits, Feature Flippers 등 다양한 용어로 혼용되어 사용됩니다.

### Feature Flag 의 형태

Feature Flag는 일반적으로 아래의 의사코드(pseudo-code)와 같이 `조건문`과 `플래그` 변수를 사용하여 다루게 됩니다.

![Feature Flag Pseudo-code](/files/post/2023-11-07-openfeature/image-20231103165418500.png)

위 pseudo-code에서는 크게 아래와 같이 구분할 수 있습니다.

- Flag 정보가 정의된 `FlagConfiguration` (Flag Configuration)
- "FLAG_KEY"라는 이름의 Flag의 상태를 파악하여 결정 `flagConfiguration.getFlag("FLAG_KEY").getBooleanValue()` (Flag Router)
- 조건문을 사용하여 Flag의 상태에 따라 기능을 공개할지 (`RevealFeature()`) 숨길지 (`HideFeature()`) 결정 (Flag Point)

이 때 Flag의 상태를 런타임에 외부에서 제어할 수 있어야하므로 Flag Router가 호출될 때마다 Flag Configuration에서 Flag의 상태를 가져와 판단하도록 구현됩니다.

> #### Flag 상태를 배포된 애플리케이션 외부에서 불러오지 않는다면?
>
> 기능 동작을 변경하기 위해 매번 새롭게 배포가 이루어져야합니다.
>
> 다음과 같이 [Spring Boot 의 profiles](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.profiles) 를 활용해서 기능을 스위치하도록 구현하는 경우가 해당됩니다.
>
> ```java
> @Configuration
> public class FeatureConfiguration {
>   
>   @Profile("production")
>   @Bean
>   public OldFeatureBean() {
>     // New Feature is not yet ready in production
>     ...
>   }
>   
>   @Profile("verify")
>   @Bean
>   public NewFeatureBean() {
>     ...
>   }
> }
> ```
>
> 이렇게 Configuration을 구성하게 되면 기능 동작을 변경하기 위해 매번 Profile을 변경하여 재배포가 이루어져야 하기 때문에 빠른 반영, 불필요한 재배포와 같은 Feature Flag의 장점을 활용하지 못하게 됩니다.

실제 Feature Flag를 도입한 애플리케이션 코드를 작성하기 위해 아래에서 설명드릴 `OpenFeature`와 `Flagd`를 활용하는 방법을 소개해드리겠습니다.

## OpenFeature / Flagd

![OpenFeature](/files/post/2023-11-07-openfeature/openfeature.png)

[OpenFeature](https://openfeature.dev/)는 Cloud Native 환경에서 Feature Flag를 관리하고 사용하기 위한 표준 인터페이스와 프레임워크입니다. 2022년 6월에 CNCF의 Sandbox 프로젝트로 지정되었으며, 강력한 Feature Flag 생태계를 지원하고자 벤더 중립의 개방형 Feature Flag 표준을 정의하고 있습니다.

> CNCF(Cloud Native Computing Foundation)는 리눅스 재단 소속의 비영리 단체로, 클라우드 네이티브 컴퓨팅 환경에서 필요한 다양한 오픈소스 프로젝트를 추진하고 관리하고 있으며 500개 이상의 글로벌 기업들이 멤버로 참여하고 있습니다.

OpenFeature는 Feature Flag를 관리하는 표준화된 방법을 제공하기 때문에 개발자가 일관된 방식으로 Feature Flag를 구현하여 사용할 수 있습니다. 또한 런타임에 동적으로 제어하여 빠른 실험과 롤백 및 개인화가 가능하며, 모니터링 기능을 제공해 Feature Flag의 상태와 영향을 실시간으로 확인할 수 있습니다.

![OpenFeature Architecture](/files/post/2023-11-07-openfeature/of-architecture-a49b167df4037d936bd6623907d84de1.png)

Flag의 정의와 상태, 규칙 등을 Configuration으로 정의하면 [Flag Management System](https://openfeature.dev/specification/glossary#flag-management-system) (위 구조도에서 `Flags-R-us service`를 가리킵니다)에서 이를 Watch 하고, SDK와 Flag Management System의 중간 계층인 Provider는 (클라이언트 애플리케이션에서 Flag에 대해 평가를 요청하면) Flag Management System으로부터 Flag 정보를 조회하고, 이를 SDK의 Flag 평가 API에 매핑합니다.) 결과적으로 클라이언트는 SDK의 Flag 평가 API를 호출하여 Flag 상태를 판단하고, 이를 통해 기능에 대해서 제어할 수 있게 됩니다.

OpenFeature는 .NET, Go, Java, Node.js, PHP 등 다양한 언어로 작성된 SDK를 제공하고, 이와 함께 클라이언트 애플리케이션은 써드파티 Provider를 연결하여 Flag Management System과 연동시켜 Flag 상태를 추적하고 평가할 수 있습니다.

현재 CloudBee, Flagsmith, LaunchDarkly, Split 등 특정 Vendor의 서비스와 연결할 수 있는 Provider가 제공되고 있고, 이외에 Community에서 관리되는 오픈소스 Provider로 [Flagd](https://flagd.dev) Provider를 사용할 수 있습니다.

### Flagd를 사용해 Flag Management System (Flag Backend, Flag Evaluation Engine) 구축하기

![Flagd](/files/post/2023-11-07-openfeature/flagd.png)

Flagd는 OpenFeature와 호환되는 Feature Flag Backend System로, 오픈소스이며 사용하기 편한 구성으로 이루어져 있습니다.

Flagd는 실시간으로 변경되는 Flag 정보를 곧바로 반영할 수 있을 뿐만 아니라 아래와 같은 다양한 장점을 활용할 수 있게 됩니다.

- Boolean, String, Number, JSON과 같이 다양한 Flag 타입을 지원
- 상황 (Context)에 따른 규칙을 정의해 특정 사용자를 Targeting하도록 지원
- 실험적 용도로 사용하기 위해 Pseudo-random 할당 지원
- 새로운 기능을 점진적으로 릴리즈하도록 지원
- 분리된 Flag 정의를 Aggregation 할 수 있도록 지원

특히 Flagd는 다양한 인프라에 잘 맞도록 설계되어 있어서 바이너리를 사용하여 VM 환경에서도, Docker 이미지를 사용해 Kubernetes 환경에서도 실행시킬 수 있는 장점이 있습니다.

> #### 환경 구축 예시
>
> - **Kubernetes 환경에서 [`open-feature-operator`](https://github.com/open-feature/open-feature-operator) 활용**
>
>   OpenFeature에서는 Flagd를 Operator로 구성한 Helm Chart를 제공하고 있습니다. Operator는 클러스터 Pod 중 `openfeature.dev/enabled: true` annotation을 가지는 Pod에 `flagd` 를 사이드카로 주입시켜 Pod별로 별도의 분리된 Engine을 구축할 수 있습니다.
>
> ```yaml
> # Application Pod metadata
> metadata:
>  annotations:
>    openfeature.dev/enabled: "true"
>    openfeature.dev/flagsourceconfiguration: "flag-source-configuration"
> ```
>
> ![Kubernetes Operator 방식 배포](/files/post/2023-11-07-openfeature/20231103175022.png)
>
> 이 때 애플리케이션에서 참조하게 될 Flag 정의는 `FeatureFlagConfiguration`이라는 Kubernetes Custom Resource로 정의하여 클러스터에 배포하고, 이러한 `FeatureFlagConfiguration` 목록을 `FlagSourceConfiguration` Custom Resource에 정의하여 마찬가지로 Pod의 Metadata에 연결시킬 수 있습니다.
>
> ```yaml
> # FlagSourceConfiguration (K8s CR)
> apiVersion: core.openfeature.dev/v1alpha3
> kind: FlagSourceConfiguration
> metadata:
>   name: flag-source-configuration
> spec:
>   sources:                        # flag sources for the injected flagd
>     - source: featureflags/appName-flags  # FlagSourceConfiguration - namespace/name
>       provider: kubernetes        # kubernetes flag source backed by FlagSourceConfiguration custom resource
>   port: 8080                      # port of the flagd sidecar
> ```
>
> - **VM 환경과 Kubernetes 환경 모두에서 동일한 Engine을 바라보도록 구성**
>
>   11번가는 IDC와 AWS EKS를 사용한 Hybrid 인프라를 구축하여 서비스를 제공하고 있습니다. 애플리케이션이 어느 위치에 배포가 되었던 동일한 Engine을 바라볼 수 있도록 EKS 클러스터에 flagd Docker 이미지를 사용하여 Engine Pod를 배포하고, Engine에 대한 모든 요청은 Ingress를 통해 처리할 수 있도록 구성하였습니다.
>
>   ![Kubernetes Deployment + LB 방식 배포](/files/post/2023-11-07-openfeature/image-20231106145704944.png)
>
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   labels:
>     app: flagd
> spec:
>   replicas: 1
>   selector:
>     matchLabels:
>       app: flagd
>   template:
>     metadata:
>       labels:
>         app: flagd
>   spec:
>     containers:
>       - name: flagd
>         image: ghcr.io/openfeature/flagd:v0.6.6
>         imagePullPolicy: IfNotPresent
>         readinessProbe:
>           httpGet:
>             path: /readyz
>             port: 8014
>           initialDelaySeconds: 5
>           periodSeconds: 5
>         livenessProbe:
>           httpGet:
>             path: /healthz
>             port: 8014
>           initialDelaySeconds: 5
>           periodSeconds: 60
>         ports:
>           - containerPort: 8013
>         args:
>           - start
>           - --uri
>           - core.openfeature.dev/featureflags/appName-flags
> ```

### Flagd에서 사용할 Flag 정의하기

Flag 정의 Configuration는 JSON이나 YAML 형태로 작성할 수 있으며, Flagd 바이너리는 실행되는 동안 계속해서 Argument로 정의한 Flag Configuration을 Watch하여 변경될 경우 즉시 반영합니다. 이러한 Flag Configuration은 파일, 혹은 Kubernetes Custom Resource, 혹은 HTTP/gRPC 엔드포인트를 Watch하도록 구성할 수 있습니다.

```shell
$ ./flagd start \
    --uri file:/etc/flagd/my-flags.json
    --uri core.openfeature.dev/namespace/featureflagconfigurationCRName
    --uri https://featureflag-fake-url.11stcorp.com/featureflagName
    --uri grpc(s)://featureflag-fake-url.11stcorp.com/
```

```yaml
apiVersion: core.openfeature.dev/v1alpha1
kind: FeatureFlagConfiguration
metadata:
  name: featureflagconfigurationCRName
  namespace: name
spec:
  featureFlagSpec: |
    {
      "flags": {
        "betaFeature": { // Flag Key
          "state": "ENABLED",
          "variants": {
            "on": true,
            "off": false
          },
          "defaultVariant": "on",
          "targeting": {
            "if": [
              {"in": [{"var": ["host"]}, ["X.X.X.X"]]}, "on", "off"
            ]
          }
        }
      }
    }
```

![Flagd Logs](/files/post/2023-11-07-openfeature/image-20231106170629121.png)

Flag 정의의 각 항목이 의미하는 바는 아래와 같습니다.

- `state` (Required)

  - Flag가 사용 활성화되어 있는지 여부를 의미하는 것으로, `DISABLED`일 경우 Flag가 존재하지 않는 것과 동일하게 처리됩니다. (Flag 평가를 요청할 경우 Exception이 발생합니다.) 따라서 사용되는 Flag는 `ENABLED`로 지정하여야 합니다.

- `variants` (Required)

  - `variants`는 Flag가 지원할 수 있는 모든 값들을 사람이 의미적으로 이해할 수 있는 Unique한 식별자로 매핑한 구조체입니다. 식별자 (`variant`)를 key로, 값을 value로 하나의 pair를 이루며 모든 value는 동일한 타입을 가져야 합니다.
  - 클라이언트 애플리케이션에서는 `variants`의 각 value만을 참조하여 기능을 제어하게 되고, 실질적으로 Flag를 제어할 때에는 `variant`를 통해 제어하게 됩니다.

- `defaultVariant` (Required)

  - 아래에서 설명할 `targeting` 항목에 명시적으로 재정의하지 않는 이상 Flag 평가 API를 호출하면 `defaultVariant`에 지정된 `variant`와 일치하는 값이 반환되도록 동작합니다. 즉, `defaultVariant`는 Flag 의 기본값을 의미합니다.

- `targeting` (Targeting Rules, Optional)

  - Targeting Rules는 [변형된 JSONLogic](https://jsonlogic.com/) 기반의 조건문을 활용하여 상황에 따른 적절한 Flag 값이 반환되도록 구현하는 사전 정의된 규칙입니다.

  - 주로 애플리케이션에서 평가 API를 호출할 때 함께 넘기는 동적 컨텍스트 정보 (Evalutation Context, 아래에서 후술) 를 활용하게 됩니다.

  - 예를 들어 아래와 같이 평가 API가 호출되는 경우, Evaluation Context로 넘어온 사용자의 이메일이 `@sk.com` (SK 임직원) 을 포함하는 경우에만 Flag 값이 `true`로 반환되게끔 조건을 부여할 수 있습니다.

    - ```json
      {
        "flagKey": "betaFeature",
        "context": {
          "email": "jiwon_jeon@sk.com"
        }
      }
      ```

    - ```json
      {
        // ...
        "targeting": {
          "if": [
            {
              "ends_with": [
                { "var": ["email"] },
                "@sk.com",
              ],
            },
            "on",
            "off",
          ],
        },
      }
      ```

  - 조건문을 통해 도출되는 `variant`는 반드시 `variants` 항목에 정의된 것이어야 하고, 그렇지 않을 경우 `defaultVariant`가 설정됩니다.

  - 자세한 Syntax는 [https://flagd.dev/reference/flag-definitions/](https://flagd.dev/reference/flag-definitions/) 에서 확인 가능합니다.

> #### Targeting Rules로 Flag 세부 제어하기
>
> Targeting Rules를 사용하면 Static하게 Flag Definition의 `defaultVariant`를 변경해서 Flag 를 스위칭 (토글) 하는 것 외에도, Evaluation Context에서 넘어오는 데이터에 따라 Flag를 제어할 수도 있을 뿐만 아니라 요청 단위로 Weight를 부여하여 Flag에 Canary를 적용할 수도 있습니다.
>
> 아래와 같은 Flag 정의가 있다면,
>
> ```json
> {
>   "flags": {
>    "headerColor": {
>      "variants": {
>        "red": "#FF0000",
>        "blue": "#0000FF",
>        "green": "#00FF00"
>      },
>      "defaultVariant": "red",
>      "state": "ENABLED",
>      "targeting": {
>        "fractional": [
>          { "var": "email" },
>          [
>            "red",
>            50
>          ],
>          [
>            "blue",
>            20
>          ],
>          [
>            "green",
>            30
>          ]
>        ]
>      }
>    }
>   }
> }
> ```
>
> Evaluation Context로 `email` 정보가 넘어오는 요청에 대해서는 각각 50%, 20%, 30% 비율로 해당하는 `variant` 와 매핑되는 값을 적용받게 됩니다.

### OpenFeature SDK (Java) 살펴보기

OpenFeature를 사용해서 Flag를 평가하려면 애플리케이션 전역에서 사용할 수 있는 `Client` 인스턴스를 생성하여 Provider를 플러그인하여야 합니다.

```java
OpenFeatureAPI api = OpenFeatureAPI.getInstance();
api.setProvider(new FlagdProvider()); // Plug-in Flagd Provider

// 사이즈가 큰 애플리케이션의 경우 이름을 지정한 Multiple Client를 생성하여 Needs에 맞게 서로 다른 Configuration을 구성하여 사용할 수도 있고, 그렇지 않을 경우 이름 없이 단일 Client로도 사용 가능
Client client = api.getClient("clientName");
```

#### [Evaluation API](https://github.com/open-feature/java-sdk/blob/main/src/main/java/dev/openfeature/sdk/Features.java)

OpenFeature에서는 Feature Flag의 값 타입으로 Boolean, String, Integer, Double, Object 총 다섯가지가 제공됩니다. 아래 예시와 같이 `get[Type]Value()` 메서드를 호출해 단순 평가 (값만 반환) 하도록 하거나, 혹은 `get[Type]Details()` 메서드 호출로 Flag Key (Unique Identifier), 값, 반환 이유, 에러 코드 및 메시지 등 메타데이터를 포함하는 상세 평가를 진행할 수 있습니다.

```java
/* Basic Evaluation */
// get a boolean value
Boolean boolValue = client.getBooleanValue("boolFlag", false);

// get a string value
String stringValue = client.getStringValue("stringFlag", "default");

// get an integer value
Integer intValue = client.getIntegerValue("intFlag", 1);

// get a double value
Double doubleValue = client.getDoubleValue("doubleFlag", 0.9);

// get an object value
Value objectValue = client.getObjectValue("objectFlag", MyObjectInstance);

/* Detailed Evaluation */
// get details of boolean evaluation
FlagEvaluationDetails<Boolean> boolDetails = client.getBooleanDetails("boolFlag", false);

// get details of string evaluation
FlagEvaluationDetails<String> stringDetails = client.getStringDetails("stringFlag", "default");

// get details of integer evaluation
FlagEvaluationDetails<Integer> intDetails = client.getIntegerDetails("intFlag", 1);

// get details of double evaluation
FlagEvaluationDetails<Double> doubleDetails = client.getDoubleDetails("doubleFlag", .9);

// get details of object evaluation
FlagEvaluationDetails<Value> objectDetails = client.getObjectDetails<MyObject>("objectFlag", myObjectDefaultInstance);
```

#### Evaluation Context

Evaluation Context 는 Flag 평가에 사용될 수 있는 데이터 집합입니다. Flag 평가 API가 호출될 때 Flag Evaluation Engine에 함께 전송되어 조건에 따라 값이 동적으로 결정될 수 있으며, 이러한 동작은 백분율 기반 롤아웃이나 컨텍스트 데이터에 따라 기능의 동작을 제어하는 데에 사용될 수 있습니다.

OpenFeature SDK에서는 아래와 같이 생성 후 수정 가능한  `MutableContext`와 생성 후 수정이 불가능한 `ImmutableContext`가 제공되어 필요에 따라 적절한 Context를 생성할 수 있습니다.

```java
// MutableCotext
EvaluationContext context = new MutableContext();
context.addStringAttribute("host", new Value(getHostIp()));

// ImmutableContext
Map<String, Value> evaluationContextMap = Map.of("host", new Value(getHostIp()));
EvaluationContext context = new ImmutableContext();
```

#### Hooks

OpenFeature에서는 Flag 평가의 Lifecycle로 `Before`와 `After`, `Error` 그리고 `finally` (finally의 경우 언어마다 예약어로 지정된 사례가 있어 `finallyAfter` 등의 이름을 사용하기도 함) 네 단계로 구성하고 있습니다. Hook을 사용하면 Flag 평가의 각 Lifecycle 단계마다 수행할 로직을 작성할 수 있습니다.

```java
@Slf4j
class LoggerHook implements BooleanHook {
  @Override
  public Optional<EvaluationContext> before(HookContext<Boolean> ctx, Map<String, Object> hints) {
    log.debug("code to run before flag evaluation");
  }

  @Override
  public void after(HookContext<Boolean> ctx, FlagEvaluationDetails<Boolean> details, Map<String, Object> hints) {
    log.debug("code to run after successful flag evaluation");
  }

  @Override
  public void error(HookContext<Boolean> ctx, Exception error, Map<String, Object> hints) {
    log.error("code to run if there's an error during before hooks or during flag evaluation");
  }

  @Override
  public void finallyAfter(HookContext<Boolean> ctx, Map<String, Object> hints) {
    log.debug("code to run after all other stages, regardless of success/failure");
  }
}
```

```java
class ServiceImpl implements Service {
  
  private final Client client;
  
  @Override
  public Feature feature() {
    final Context context = getContext();
    final FlagEvaluationOptions options = FlagEvaluationOptions.builder()
                                                            .hook(new LoggerHook())
                                                            .build();
    if (client.getBooleanValue("betaFeature", false, context, options)) {
      return new BetaFeature();
    }
    
    return new Feature();
  }
  
}
```

## 실제 애플리케이션 코드를 작성해볼까요?

아티클의 맨 처음에 소개해드렸던 사례를 OpenFeature와 Flagd를 기반으로 하는, 실제 Spring Boot 애플리케이션 코드를 작성해보도록 하겠습니다.

우선 아래와 같이 OpenFeature SDK와 Flagd Provider를 의존성으로 포함하고 있어야 합니다.

 ```groovy
// build.gradle
dependencies {
  // OpenFeature SDK (작성일 기준 최신 버전 1.7.0)
  implementation 'dev.openfeature:sdk:1.7.0'

  // Flagd Provider (작성일 기준 최신 버전 0.6.6)
  implementation 'dev.openfeature.contrib.providers:flagd:0.6.6'
}
 ```

 그리고 애플리케이션 전역에서 Feature 평가를 위한 Client 인스턴스를 사용할 수 있도록 Application Context에 Bean으로 등록합니다.

 > Flagd Provider는 별도 설정을 하지 않을 경우 localhost:8013에 gRPC 요청을 하게 됩니다. (현재 애플리케이션과 Flagd 간에 gRPC 통신으로만 Evaluation이 가능합니다.)

 ```java
@Configuration
@RequiredArgsConstructor
@ConditionalOnProperty(value = "feature-flag.enabled", havingValue = "true")
@EnableConfigurationProperties(RemoteFeatureFlagClientProperties.class)
public class FeatureFlagConfig {
  
  private final RemoteFeatureFlagClientProperties flagdProperties;

  @Bean
  public Client featureFlagClient() {
      final OpenFeatureAPI instance = OpenFeatureAPI.getInstance();
      instance.setProvider(getFeatureFlagProvider());

      return instance.getClient();
  }

  private FeatureProvider getFeatureFlagProvider() {
      return new FlagdProvider(FlagdOptions.builder()
                                      .host(flagdProperties.host())
                                      .port(flagdProperties.port())
                                      .tls(flagdProperties.tls())
                                      .build());
  }

}

@ConfigurationProperties(prefix = "feature-flag")
public record RemoteFeatureFlagClientProperties(boolean enabled,
                                        String host,
                                        int port,
                                        boolean tls) {
}
 ```

> 비즈니스 로직 구현에서는 Bean으로 등록했던 `Client` 인스턴스의 메서드를 호출합니다. Flag Key (Flag 명) 와 Context를 파라미터로 넘겨주면 플러그인되어 있는 Flagd Provider가 Engine으로부터 Flag 정보를 가져와 평가하고, 값을 반환합니다. 이를 토대로 기능을 적절하게 분기시켜줄 수 있습니다.

 ```java
@Service
@RequiredArgsConstructor
class AmazonProductServiceImpl implements ProductService {
  private static final FLAG_IS_GRPC_ENABLED = "isGrpcEnabled";

  private final Client featureFlagClient;

  private final HTTPReviewService httpReviewService;
  private final GRPCReviewService grpcReviewService;

  /**
    featureFlagClient.getBooleanValue()
      - 1st Parameter : FLAG KEY (FLAG명)
      - 2nd Parameter : Default Value (FLAG가 존재하지 않을 경우 반환 기본값)
  */
  public List<Review> getProductReviews(final long productId) {
    final boolean isGrpcEnabled = featureFlagClient.getBooleanValue(FLAG_IS_GRPC_ENABLED, false);
  
    if (isGrpcEnabled) {
        return grpcReviewService.getProductReviews(productId);
    }
  
    return httpReviewService.getProductReviews(productId);
  }
}
 ```

 ```yaml
apiVersion: core.openfeature.dev/v1alpha1
kind: FeatureFlagConfiguration
metadata:
  name: review-flags
  namespace: feature-flags
spec:
  featureFlagSpec: |
    {
      "flags": {
        "isGrpcEnabled": {
          "state": "ENABLED",
          "variants": {
            "on": true,
            "off": false
          },
          "defaultVariant": "off" // 배포 이후 컨트롤
        }
      }
    } 
 ```

위와 같은 애플리케이션 및 Flag 구성이 배포되면 Flag의 `defaultVariant`가 변경되었을 때 Engine의 즉각적인 Sync Event에 의해 애플리케이션 측의 평가 API 또한 변경사항을 반영하게 됩니다.

아래 예시와 같이 HTTP 호출이 Default이던 메서드는 Flag의 `defaultVariant`가 gRPC로 전환되자마자 gRPC 호출을 진행하게 되는 것을 확인할 수 있습니다.

![Switching Between HTTP and gRPC](/files/post/2023-11-07-openfeature/image-20231106202556558.png)

## 마치며

지금까지 설명드렸던 Feature Flag를 적용하면 Production 환경에 안전하게 배포하고 실험할 수 있습니다. 이 때 Feature Flag 표준 인터페이스 프레임워크인 OpenFeature와 Flag Evaluation Engine인 Flagd를 사용하면 Feature Flag를 적용한 애플리케이션 코드를 쉽게 작성할 수 있습니다.  OpenFeature / Flagd 가 Feature Flag 를 고려하고 있는 여러분들에게 자그마한 인사이트가 되셨기를 바랍니다.

OpenFeature와 Flagd는 Early Project이기 때문에 지금 이 시간에도 기능의 변화가 계속 이루어지는만큼, 위 아티클에서 설명드린 구성 등의 기타 사항은 과 관련한 보다 상세한 내용은 Github Repository를 참고해주시기 바랍니다.

관련해서 추가 질의사항이나 개선사항이 있다면 언제든 아래 코멘트를 남겨주세요! 아티클을 읽어주신 모든 분들께 감사드립니다.

## References

- [https://martinfowler.com/articles/feature-toggles.html](https://martinfowler.com/articles/feature-toggles.html)
- [https://openfeature.dev](https://openfeature.dev)
- [https://flagd.dev](https://flagd.dev)
