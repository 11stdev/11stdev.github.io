---
layout: post
title: '11번가의 오픈소스 활동 (1)'
author: 최유진
date: 2022-03-02 13:00
tags: [OpenSource]
---
안녕하세요.
11번가 Core Platform 개발팀 최유진이라고 합니다.
 
개발자라면 누구나 오픈소스 활동을 꿈꾸는데요. 11번가에서는 이 오픈소스 활동을 적극적으로 장려하고 있습니다.<br/>
이 포스트에서는 11번가의 오픈 소스 활동을 소개해드리겠습니다.<br/><br/>
 
### 목차
- [트러블 슈팅을 위한 컨트리뷰션](#트러블-슈팅을-위한-컨트리뷰션)
  - [MSA 전환 과정에서 발견한 이슈에 대한 컨트리뷰션](#msa-전환-과정에서-발견한-이슈에-대한-컨트리뷰션)
  - [Spring Boot 업그레이드 중 발견한 이슈에 대한 컨트리뷰션](#spring-boot-업그레이드-중-발견한-이슈에-대한-컨트리뷰션)
    1. [Model Mapper](#1-model-mapper)
    2. [QueryDSL](#2-querydsl)
    3. [Spring Cloud Openfeign](#3-spring-cloud-openfeign)
    4. [Resilience4J](#4-resilience4j)

# 트러블 슈팅을 위한 컨트리뷰션

업무 진행 중 발견한 문제를 해결하기 위해 오픈소스에 직접 컨트리뷰션 한 경우입니다.<br/><br/>
오픈 소스를 사용하다보면 버그를 발견하여 수정해야 할 때가 있습니다. 이 때 버그를 내부적으로만 해결하거나 혹은 귀찮아서 버그를 알리지 않는다면 오픈소스를 사용하는 여러 사람이 해당 문제로 인해 어려움을 겪을 수 있습니다. <br/><br/>
저희 팀에서는 사용 중인 오픈소스에서 버그가 발견됐을 때, 누군가 대신 버그 수정해주는 것을 기다리기보다는 적극적으로 문제를 리포팅하거나 직접 문제를 해결하고 있습니다.

## MSA 전환 과정에서 발견한 이슈에 대한 컨트리뷰션
11번가는 2017년 초 Spring Cloud를 이용하여 MSA로 전환했습니다. <br/><br/>
API 게이트웨이로 Spring Cloud Zuul을 사용했는데, Spring Cloud Zuul에서는 Fault Tolerance Library로 Hystrix를 사용합니다.<br/>
Hystrix에서는 Circuit Breaker 별로 isolation을 지정할 수 있습니다.<br/><br/>
11번가에서는 Circuit Breaker간 isolation을 Thread로 설정하여 동일한 commandKey를 갖는 CircuitBreaker 끼리는 Thread Pool을 공유하도록 설정했습니다.<br/>
따라서 아래와 같은 모습의 Thread Pool이 생성되기를 기대했습니다.
<figure>
  <img src="/files/post/2022-03-02-11st-and-open-source/msa1.jpg">
  <figcaption>예상 설정</figcaption>
</figure>
<br/>
하지만, 실제 확인해보니 예상과는 달리 모든 CircuitBreaker에 대해 하나의 Thread Pool이 생성된 것을 확인했습니다.<br/>
<figure>
  <img src="/files/post/2022-03-02-11st-and-open-source/msa2.jpg">
  <figcaption>실제 설정</figcaption>
</figure>
<br/>

11번가에서 사용한 Spring Cloud Zuul은 Spring Cloud Netflix의 일부분으로 Netflix OSS에서 공개된 코드를 스프링으로 통합시킨 것인데, 코드를 옮겨오는 과정에서 실수가 있었던 것으로 추정되어 [Pull Requests](https://github.com/spring-cloud/spring-cloud-netflix/pull/2074)를 통해 직접 코드를 수정 했습니다.
<figure>
  <img src="/files/post/2022-03-02-11st-and-open-source/msapr.png">
  <figcaption>Pull Request 내용</figcaption>
</figure>
 
이 외에도 MSA로 전환하는 과정에서 다양한 이슈를 제보하고 직접 수정했습니다.
<figure>
  <img src="/files/post/2022-03-02-11st-and-open-source/msaprs.png">
  <figcaption>머지된 PR들</figcaption>
</figure>

## Spring Boot 업그레이드 중 발견한 이슈에 대한 컨트리뷰션
저희 팀은 스프링 최신 버전을 빠르게 적용하고 있습니다.<br/>
하지만 버전을 올리는 과정이 항상 평탄치 만은 않습니다. 업그레이드 과정에서 다양한 이슈가 발견되기도 합니다. <br/>
스프링 버전 업그레이드 과정에서 발견한 여러 이슈를 해결하기 위해 다양한 컨트리뷰션을 했습니다.
 
### 1. Model Mapper 
RPS가 일정 수준 이상으로 높아지면 CPU full이 발생하고, 이로 인해 애플리케이션이 멈추는 현상을 발견했습니다.<br/>

확인 결과 modelmapper 라이브러리가 원인이었고, 버전을 바꿔가며 문제가 생기는 버전을 찾았습니다. <br/>
그리고 문제가 생기는 버전에서부터 디버깅을 하여 [스레드 병목 구간을 찾아 제보](https://github.com/modelmapper/modelmapper/issues/520)했습니다.

<figure>
  <img src="/files/post/2022-03-02-11st-and-open-source/modelmapper1.png">
  <figcaption>문제 발생 원인 분석 및 제보</figcaption>
</figure>
 
덕분에 메인테이너가 이를 수정하여 해당 버그가 수정된 새로운 버전이 출시되었습니다.<br/>
### 2. QueryDSL
컴파일 단계에서 SQL Syntax 에러를 발견했습니다. Spring Boot2에서는 hibernate5가 기본적으로 사용되는데, QueryDSL에서 hibernate를 지원하지 못해서 발생한 이슈였습니다.<br/>

알고 보니 당시 QueryDSL 팀은 프로젝트 관리를 하지 않고 있어 새로운 메인테이너를 구하고 있던 상황이었습니다.<br/>

그래서 hibernate를 지원하도록 [핸들러 코드를 공유](https://github.com/querydsl/querydsl/issues/2264#issuecomment-564635512)했고, 새로운 메인테이너 팀에서 이를 반영해주었습니다.

<figure>
  <img src="/files/post/2022-03-02-11st-and-open-source/querydsl1.png">
  <figcaption>공유한 핸들러 코드 내용</figcaption>
</figure>
<br/>
 
### 3. Spring Cloud Openfeign
11번가에서는 Client-Side Load-Balancing을 위해 Spring Cloud Openfeign을 사용하고 있습니다.<br/>
그런데, Spring Cloud Openfeign 2020.0.2에서 하위 호환이 불가능하게 Circuit Breaker 이름을 변경하여 문제가 생겼습니다.

Circuit Breaker 이름이 변경됨에 따라 이 이름을 가지고 생성하는 모니터링 메트릭 명이 모두 변경되어 모니터링이 불가능해졌습니다.<br/>
또한, 저희가 버전을 올린 서비스의 경우 모든 Circuit Breaker가 동일한 기본 설정을 사용하고 있었지만, Circuit Breaker별로 별도의 설정을 사용하고 있었다면 설정이 동작하지 않는 문제가 발견됐습니다.<br/>

더군다나, Circuit Breaker 이름을 예전과 동일한 방식으로 사용할 수 있는 방법 또한 제공하고 있지 않았습니다.<br/>
이에 Circuit Breaker 이름을 사용자가 커스터마이징을 할 수 있도록 Resolver를 제공하는 [컨트리뷰션](https://github.com/spring-cloud/spring-cloud-openfeign/pull/575)을 했습니다.<br/>
<br/>
<figure>
  <img src="/files/post/2022-03-02-11st-and-open-source/feign1.png">
  <figcaption>새로 추가된 Resolver</figcaption>
</figure>

### 4. Resilience4J
가장 첫번째 사례에서 알 수 있듯이, 11번가에서는 Spring Cloud 를 기반으로 MSA로 전환을 했습니다. 
그런데, 그 중 Spring Cloud Netflix의 대부분의 모듈이 Maintenance 모드에 들어갔고, Spring Boot 버전 업그레이드를 위해서는 Spring Cloud Netflix 모듈들을 교체할 필요가 생겼습니다.

그 중 Hystrix의 대안으로 Resilience4j를 선정했습니다. 그런데 불필요하게 circuit이 열리는 상황을 발견했습니다.<br/>
확인 결과 Resilience4J와 Hystrix의 ThreadPool 구현이 달라서 발생한 이슈였습니다.

Resilience4j의 경우 항상 `ArrayBlockingQueue`를 사용하여 task 가 queue 사이즈 만큼 찼을 때 thread를 늘리게 됩니다.<br/>
하지만, 생각만큼 빨리 thread가 늘어나지 않았고 이로 인해 queue에 등록된 task들이 빠르게 처리되지 않아 circuit이 열린 것으로 확인했습니다.<br/>
반면 Hystrix는 `SynchronousQueue`를 기본으로 사용하여 task가 지연없이 바로 worker thread로 전달되는 구조입니다.

따라서 Resilience4j에서도 Synchronous Queue를 사용할 수 있도록 변경하는 [Pull Request](https://github.com/resilience4j/resilience4j/pull/1504)를 작성했습니다.

<figure>
  <img src="/files/post/2022-03-02-11st-and-open-source/resilience4j1.png">
  <figcaption>새로 추가된 Resolver</figcaption>
</figure>

---

지금까지는 11번가에서 업무 중 발견한 이슈들을 해결하기 위한 오픈 소스 활동 내용을 공유드렸습니다.<br/>
다음 편에서는 다른 목적의 오픈소스 활동들과 저의 첫 컨트리뷰션 경험에 대해 공유드리겠습니다.<br/>
감사합니다 :)
