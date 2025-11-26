---
layout: post
title: Micrometer 객체 증가로 인한 메모리 이슈 회고
author: 이경진
date: 2025-05-29
tags: [java, spring-boot, gc, monitoring]
---

## 목차
- [들어가며](#들어가며)
- [시스템 구성 및 개선 배경](#시스템-구성-및-개선-배경)
- [주요 이슈 분석](#주요-이슈-분석)
  - [1. 이슈 1: Micrometer 객체 증가 현상](#1-이슈-1-micrometer-객체-증가-현상)
  - [2. 이슈 2: 메모리 점진적 증가 현상](#2-이슈-2-메모리-점진적-증가-현상)
- [최종 결론](#최종-결론)
- [마무리하며](#마무리하며)

---

## 들어가며
안녕하세요. 주문플랫폼개발팀 이경진입니다.\
최근 주문플랫폼개발팀과 쿠폰/PCS개발팀이 함께 진행한 최대할인가 구조 개선 과정에서 예상치 못한 몇 가지 성능 이슈를 마주하게 되었습니다.\
이 글에서는 함께 논의하며 문제를 해결해 나갔던 과정을 되짚어보고, 어떤 이슈가 있었고 어떻게 대응했는지 공유하고자 합니다.

---

## 시스템 구성 및 구조 변경
해당 서비스는 다음과 같은 환경에서 운영 중입니다:
- JDK 21
- Spring Boot 3.2.1
- CompletableFuture 기반 병렬 처리 구조

기존에는 할인 계산에 필요한 데이터를 외부 API(**Spring Boot 2.x**) 를 통해 조회하는 구조였으나,\
해당 API 로직을 내부 서비스(**Spring Boot 3.x**) 로 이관하며 구조를 변경하였습니다.

---

## 주요 이슈 분석

## 1. 이슈 1: Micrometer 객체 증가 현상

### 문제

서비스 개선 이후 메모리 사용률이 점점 상승하였으며, GC가 정상적으로 발생해도 `Meter$Id`, `Tag`, `ImmutableTag` 객체가 Old 영역에서 정리되지 않고 계속 쌓이는 현상이 발생하였습니다.

![jmap 객체 분석 결과:<small> Micrometer 관련 객체들이 메모리 상위 점유</small>](/files/post/2025-05-29-maxdsc-memory-issue/jmap_micrometer_obj.png)


### 원인 분석

#### 변경된 부분

- 기존: 쿠폰 API 1회 호출로 필요한 데이터를 한 번에 조회
- 변경: 쿠폰 API 로직이 서비스 내부로 이관되어, 상품 단위로 여러 DB 조회 + 다수의 API 호출 수행
  - 호출 API 예시:
    - 상품 정보: `/api/products/{productNo}/~`
    - 회원 정보: `/api/members/{memberNo}/~`
    - 기획전 정보: `/api/plan/{productNo}/~`

#### 변경된 부분이 미친 영향

- **Spring Boot 3.x에서는 Micrometer가 기본적으로 활성화**되어 있으며,
- Micrometer는 HTTP 요청의 URI 값을 Tag로 저장하여 `Meter$Id`를 관리합니다.
- PathVariable 값이 다르면 Micrometer의 Tag 값도 달라지며, 새로운 `Meter$Id` 객체가 생성됩니다.
- Spring Boot 3.x에서는 URI 내 PathVariable을 `{}`로 정규화하여 동일한 URI로 간주하는 기능이 기본 활성화되어 있으며, 이를 통해 객체 재사용이 가능해야 합니다.
- `/actuator/metrics/http.client.requests`로 확인하였을 때 URI는 정규화된 형태로 보였음에도, 실제 메모리에서는 관련 객체(`Meter$Id`, `Tag`, `ImmutableTag`)가 계속 증가하는 문제가 발생하였습니다.
- URI 외의 Tag 값(`status_code`, `host`, `error`)의 영향 가능성도 점검하였으나, `availableTags`가 매 요청마다 동일하게 유지되고 있었기에 이 또한 원인은 아니었습니다.

##### 정규화된 URI 예시

![정규화된 URI 예시](/files/post/2025-05-29-maxdsc-memory-issue/reg_uri.png)

> `/actuator/metrics/http.client.requests`에서 확인할 수 있으며, Micrometer가 URI 정규화를 수행하고 있음을 보여줍니다.  
> 그럼에도 불구하고 LoadBalancer 계층에서는 URI가 정규화되지 않은 채 메트릭이 수집되는 것으로 보이며, 이로 인해 객체 증가가 계속 발생하는 것으로 추정됩니다.

##### LoadBalancer 메트릭 설정 영향

- 문제의 원인은 `spring.cloud.loadbalancer.stats.micrometer.enabled: true` 설정이었습니다.
- 이 설정이 활성화되면 LoadBalancer가 별도로 URI를 태그로 수집하며, 이 과정에서는 정규화가 적용되지 않습니다.
- LoadBalancer 경유 시 PathVariable 값이 실제 URI로 반영되어 각기 다른 Meter 객체가 생성되는 구조였으며, 이로 인해 객체가 계속해서 Old 영역에 쌓이게 되었습니다.

**관련 문서**: [Spring Cloud LoadBalancer Metrics](https://docs.spring.io/spring-cloud-commons/reference/spring-cloud-commons/loadbalancer.html)\
**유사 사례**: [StackOverflow - Micrometer high memory usage](https://stackoverflow.com/questions/69217202/micro-meter-high-memory-usage)

#### 과거에는 왜 문제가 없었을까?

> 아래와 같은 경우에는 Micrometer 객체가 과도하게 증가하지 않았습니다:

- **쿠폰 API 서버에서는 문제가 없었던 이유**  
  Spring Boot 2.7.1 기반으로 Micrometer 관련 설정이 기본 비활성화되어 있었음

- **변경 전에도 쿠폰 API를 호출했는데 왜 문제가 없었을까?**  
  PathVariable 없이 호출되는 API(`/api/coupon/discount`)였기 때문에 Meter 객체가 계속 생성되지 않음

- **변경 전에도 PathVariable 포함 API(시스템 관리 API)가 있었는데 왜 문제가 없었을까?**  
  호출된 시스템 API (`/system/api/{appId}`)는 동일한 ID로 반복 호출되어 객체가 재사용됨

- **성능 테스트에서는 왜 문제가 발생하지 않았을까?**  
  테스트에 사용된 상품번호, 회원번호가 제한적이었기 때문에 다양한 PathVariable 값이 들어가지 않았음

### 대응 방안

#### 1) `@SpringBootApplication`에서 Micrometer 자동 설정 제외
```java
@SpringBootApplication(exclude = {
    MetricsAutoConfiguration.class,
    MicrometerTracingAutoConfiguration.class,
    ObservationAutoConfiguration.class,
    LogbackMetricsAutoConfiguration.class
})
```

#### 2) application.yml에서 메트릭 수집만 비활성화
```yaml
spring.cloud.loadbalancer.stats.micrometer.enabled: false
spring.cloud.openfeign.micrometer.enabled: false
resilience4j.circuitbreaker.metrics.enabled: false
management.metrics.enable.circuit-breaker: false
management.metrics.distribution.percentiles-histogram.resilience4j: false
```

✅ 최종적으로는 application.yml에서 LoadBalancer, Feign, Resilience4j 관련 메트릭 수집을 선택적으로 비활성화하여, 불필요한 Micrometer 객체 생성을 차단하였습니다.

### 결론

- Spring Boot 3.x의 자동 메트릭 수집 설정과 PathVariable 사용 증가가 맞물려, Micrometer 객체가 GC되지 않고 Old 영역에 누적되었습니다.
- LoadBalancer 설정을 비롯한 메트릭 수집 기능을 상황에 맞게 비활성화하거나 조정하는 것이 필요합니다.

---

## 2. 이슈 2: 메모리 점진적 증가 현상

### 배경 및 경과

- 3월 10일: Micrometer 객체 누적 대응으로 ZGC → G1GC 변경
- 3월 14일: 메트릭 수집 비활성화 설정 배포
- 이후 약 3일간, G1GC 환경에서 메모리 사용률이 점진적으로 증가\
![메모리 사용률 증가](/files/post/2025-05-29-maxdsc-memory-issue/inc_memory_used.png)


#### G1GC로의 임시 전환 배경
- Micrometer로 인해 생성되는 `Meter$Id`, `Tag` 등의 객체가 GC 대상에 계속 쌓이며  
- ZGC의 컨커런트 수거만으로는 이를 감당하지 못해 **Old 영역이 빠르게 증가**하고 **CPU 부하도 함께 상승**하였습니다.
- 이를 회피하기 위해 G1GC로 일시 전환하였습니다.


### 문제 분석
- 메모리 사용률이 점진적으로 증가하였으며, 로그 상에서는 Mixed GC가 정상적으로 발생하는 것으로 보였습니다.\
그러나 Mixed GC 이후에도 Old 영역의 사용률은 지속적으로 상승하는 패턴을 보였습니다.
- `jmap -histo:live` 명령어를 통한 강제 Full GC 수행 시 Old 영역의 객체 수가 유의미하게 감소하는 현상이 확인되었고,\
Full GC 없이 실행한 `jmap -histo` 결과에서도 시간이 지남에 따라 객체 수가 일부 줄어드는 것을 확인할 수 있었습니다.
- 이로 미루어 보아 메모리 누수라기보다는, G1GC의 기본 정책상 Mixed GC가 Old 영역의 객체를 충분히 수거하지 못해  
메모리가 점진적으로 쌓이는 현상이 발생한 것으로 판단됩니다.

![G1GC Mixed GC 로그: <small>G1GC Mixed GC 수행 후 Old 영역이 50 → 44로 소폭 감소</small>](/files/post/2025-05-29-maxdsc-memory-issue/g1gc_mixedgc.png)
![<small>
`jmap -histo` 명령을 통해 확인한 결과. 시간이 지남에 따라 일부 객체 수가 줄어드는 것으로 보아 누수보다는 GC 수거 강도 부족으로 판단됨.
</small>
](/files/post/2025-05-29-maxdsc-memory-issue/gc_jmap_histo2.png)



### ZGC로의 재전환 및 적용

✅ 기존에 사용하던 **ZGC로 복귀**하였습니다.

```bash
-XX:+UnlockExperimentalVMOptions
-XX:+UseZGC
-XX:ZUncommitDelay=60
-XX:ConcGCThreads=4
-Xms6g -Xmx10g
```
- 실제 모니터링 결과, Heap 사용률이 안정적이고 평탄하게 유지되었습니다.

![](/files/post/2025-05-29-maxdsc-memory-issue/metrics_chart.png)


### G1GC vs ZGC 비교 및 결론

| 항목               | G1GC                                       | ZGC                                               | 설명 |
|--------------------|---------------------------------------------|----------------------------------------------------|------|
| Old 영역 GC        | Mixed GC에 의존, 제한적 수거                | 지속적인 백그라운드 수거                         | ✅ G1GC는 Old 영역 수거를 위해 Mixed GC를 트리거하지만, 대상 Region을 일부만 수거하므로 제한적입니다. ZGC는 백그라운드에서 Old 영역을 포함한 전체 영역을 점진적으로 수거합니다. |
| Heap 사용률 안정성 | 우상향 패턴, 점진적 증가                   | 일정하게 유지                                     | ✅ G1GC는 Old 영역을 조금씩만 수거하므로 Heap 사용량이 점차 증가할 수 있습니다. ZGC는 실시간 수거로 Heap 사용량을 안정적으로 유지합니다. |
| TPS 대응           | 비교적 안정적이나, GC 지연 발생 가능        | Pause-free 특성으로 고 TPS 환경에 유리            | ✅ G1GC는 GC pause가 짧은 편이지만, TPS가 높고 객체 생성/소멸이 빈번한 경우 지연이 누적될 수 있습니다. ZGC는 stop-the-world 구간이 매우 짧아 높은 TPS 환경에서도 안정적으로 동작합니다. |


#### 결론

최대할인가 서비스처럼 TPS가 높고 객체 생성이 많은 환경에서는 **ZGC가 더 안정적이며 운영에 적합한 GC 전략**임을 확인하였습니다.


#### 참고 문서

- [G1GC 공식 가이드 (JDK 21)](https://docs.oracle.com/en/java/javase/21/gctuning/garbage-first-g1-garbage-collector1.html)
- [ZGC 공식 가이드 (JDK 21)](https://docs.oracle.com/en/java/javase/21/gctuning/z-garbage-collector.html)
- [Red Hat: Java GC 선택 가이드](https://developers.redhat.com/articles/2021/11/02/how-choose-best-java-garbage-collector#z_garbage_collector__zgc_)

---

## 최종 결론
- 최대할인가 개선 과정에서 PathVariable 기반 API 호출이 늘어나면서, Micrometer 객체가 GC되지 않고 Old 영역에 누적되는 문제가 발생하였습니다.
- Spring Boot 3.x 환경에서는 Micrometer가 기본 활성화되어 있어, 동일한 구조에서도 메모리 이슈가 발생할 수 있습니다.
- 이 경우, Micrometer 메트릭 수집 설정을 적절히 조정하거나 비활성화를 고려해야 합니다. 
- GC 전략(G1GC, ZGC) 튜닝을 통해 메모리 점진적 증가 문제를 완화할 수 있습니다.
- 유사한 구조의 서비스에서도 유사한 이슈가 재현될 수 있으므로, 사전에 설정과 구조를 점검하는 것이 필요합니다.


---

## 마무리하며
이번 개선 과정에서 Micrometer 관련 메모리 이슈와 GC 전략 문제를 겪으며, 운영 환경에서 발생할 수 있는 다양한 변수에 대해 다시 생각해보게 되었습니다. 
프레임워크의 기본 설정이 실제 서비스에 어떤 영향을 줄 수 있는지, 그리고 그 영향이 어떻게 나타나는지를 직접 경험할 수 있었던 시간이었습니다. 
이 글이 비슷한 문제를 마주한 분들께 조금이나마 참고가 되기를 바랍니다.