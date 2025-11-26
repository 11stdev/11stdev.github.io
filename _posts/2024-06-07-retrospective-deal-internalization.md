---
layout: post
title: "전시 딜 내재화 프로젝트 회고: MongoDB 기반 데이터 구축과 API 개선 과정"
author: 서장원
date: 2024-06-07
tags: [java, ApacheKafka, spring-batch, MongoDB]
---

안녕하세요, 11번가 전시서비스개발팀의 서장원입니다. 

전시 딜 내재화 업무를 맡아 진행했던 과정과 개선 작업이 갖는 의미에 대한 개인적인 회고를 공유해 보고자 합니다.

---

내용 이해를 돕기위해, 기초적인 질문을 던져봅니다.

<figure style="width: 50%; margin-left: auto; margin-right: auto;">
    <a href="/files/post/2024-06-07-retrospective-deal-internalization/Untitled.png" class="mg-link">
        <img src="/files/post/2024-06-07-retrospective-deal-internalization/Untitled.png">
    </a>
    <figcaption class="caption">출처: https://chimhaha.net/story/111311</figcaption>
</figure>

<br/>
- **‘딜’ 그리고 ‘내재화’ 라는게 무슨 뜻인가요?**<br/>
    - 딜은 상품 판매를 위한 판촉행사라고 생각하면 쉽습니다.<br/>
        예를 들어, 단 10일간! 사과 한 박스에 단돈 3만원!
    - 내재화는 외부 기술/데이터를 가져와서<br/>
        우리만의 특성에 맞게 조정하고 직접 관리한다는 뜻입니다.<br/>

  **즉, 상품의 판촉행사 정보를 보여주는 영역에 필요한 정보는 우리 전시서비스개발팀에서 직접 구축하고 입맛에 맞게 최적화 시킨다는 뜻입니다.**

<br/>
- **왜 내재화 하나요?**<br/>

    내재화하는 이유는 간단히 말해, 전시에 사용하는 딜 정보를 최적화 시켜서 더 쉽고 빠르게 유지 보수 하기 위함입니다.<br/>
    아래 이미지에서처럼 이미 **THOR 플랫폼** **에서 전시에 사용하는 상품정보는 내재화(with MongoDB)가 완료된 상태**였고<br/>
    **딜 정보(=행사정보)는 내재화하기 전 반정규화 테이블(with OracleDB)을 사용하고있는 상태**입니다.<br/>
    다시 말해, [행사 판매 중인 상품] 정보를 가져오기 위해 상품정보는 MongoDB에서,<br/>
    딜정보는 Oracle에서 조회하여 조합하고 있었습니다.

![반정규화 상품정보(MongoDB)와 반정규화 딜정보(OracleDB)를 각각 조회해서 병합](/files/post/2024-06-07-retrospective-deal-internalization/Untitled1.png)

<br/>
이러한 조회방식은 아래와 같은 **단점**을 지니고 있습니다.

- 매 호출마다 두 번, 서로 다른 DB에 커넥션을 받아야 하는 낭비
- 개발이나 이슈 발생 시 딜정보인지 상품정보인지에 구분하여 디버깅 필요(관리 포인트 중복)
- 전시서비스개발팀에서 즉각 대응/수정 가능한 상품정보와 달리, <br/>
  딜정보는  OracleDB 관리 팀에 요청하는 프로세스로 개발과정 단위 테스트와 즉각적인 대응에 어려움
    

이를 반정규화 MongoDB로 전환하면 위의 단점을 극복하고 MongoDB의 **장점**을 그대로 살릴 수 있습니다.

- 탈 오라클(비용 절감, DB 분리에 및 종속성 제거)
- 유연한 스키마로 구조 변경에 용이하여 개발 단위 테스트 및 개발속도 향상(OracleDB 변경 요청 프로세스가 사라짐)
- 수평 확장이 용이해져 대량트래픽 대응 비용 감소

이러한 이유로 OracleDB 반정규화 테이블은 MongoDB 반정규화 테이블로 전환하게 되었고<br/>
THOR 프로젝트의 일환으로 이번에 딜 내재화를 이어 맡아서 진행하게 되었습니다.

* THOR 플랫폼 이란?

> 다양한 요구사항들에 기민한 대응을 위해 **전시서비스 개발팀에 필요한 정보만을** <br/>
**직접 구축하고 운용하는 MongoDB 기반 전시서비스 데이터 플랫폼.** <br/>
기존에는 반정규화된 PL/SQL 기반 Oracle Serving 테이블을 참조하였지만<br/>
전시상품 수 증가, 업무로직 복잡성 증가로 인한 PL/SQL 의 한계, PL/SQL의 유지보수에 들어가는 리소스의 증가에 <br/>
기민한 대응을 위해 만들어진 전시데이터 플랫폼입니다
> 

이제 기본적인 궁금한 것도 알았고 왜 하는 건지 이해했으니 다음으로 넘어가보겠습니다.

# 프로젝트 진행 과정

---

## 기존 운영 방식의 문제점

![MongoDB 상품 정보 + OracleDB 딜 정보를 각각 조회하고 가공하여 전시 API로 응답하는 방식.](/files/post/2024-06-07-retrospective-deal-internalization/Untitled2.png)

MongoDB 상품 정보 + OracleDB 딜 정보를 각각 조회하고 가공하여 전시 API로 응답하는 방식.

- 기존 방식의 구체적인 문제점
    - DB 트래픽 낭비 문제
        - 한 API 응답에서 두 개의 DB Pool Connection 을 맺는 추가 리소스
        - 대량 트래픽 발생시, 오라클 커넥션 풀이 부족한 경우가 발생했고 이는 지연 발생의 원인이 될 수 있음
    - 유지 보수 한계 (이슈 발생시 딜 정보 추적이 어려움)
        - 직접 관리하는 테이블이 아니다 보니, Oracle Serving 테이블의 생성 로직과 엮여있는 배치의 프로세스와 주기, 관계 파악 쉽지 않음
        - 이슈의 원인을 파악하기 위해 Oracle SP의 생성 로직과 갱신 주기 등을 매번 다른 팀에 문의하거나, <br/>
          MongoDB(상품 정보)와 OracleDB(딜 정보)를 모두 확인해야 합니다.
            

**→ Oracle Serving 테이블 참조를 제거하고 자체 구축한 MongoDB 컬렉션을 참조하도록 변경필요**

<br/><br/>
지금부터는 좀 더 깊게 생각해봅니다

<figure style="width: 70%;">
    <a href="/files/post/2024-06-07-retrospective-deal-internalization/Untitled3.png" class="mg-link">
        <img src="/files/post/2024-06-07-retrospective-deal-internalization/Untitled3.png">
    </a>
    <figcaption class="caption">https://namu.wiki/</figcaption>
</figure>

## 고려할점들

### 1. 매우 넓은 영향범위

기존에 딜 정보를 사용하는 API를 전수조사를 해보니 약 60개, 각 API 마다 사용처는 그보다 훨씬 많았습니다.

![API 내부 딜 조회 메서드에 대한 클래스 다이어그램](/files/post/2024-06-07-retrospective-deal-internalization/Untitled4.png)

**다행히, 이미 잘 짜여진 구조와 적용된 패턴 덕분에, 일일이 60여개의 API의 코드를 모두 확인해봐야 할 필요는 적었습니다.**<br/>
**대부분의 API에서는 facade의 메서드(findProductsBy)만을 호출하여 필요한 정보를 조회하도록 캡슐화되어 있었고,**<br/>
**factory에서 반환하는 하위 객체 별 조회 동작 구현부만 수정하면 되었기 때문입니다.**<br/>
하지만 앞서 말했듯이, API자체가 많았고 이를 이용하는 사용처가 많다보니 로직상 숨겨진 의도와 데이터변환 조건등을 잘 챙겨야만 했습니다.<br/>
API별 실제 사용되는 필드가 다르고 재변환로직이 조금씩 달랐기 때문에 필드 하나하나 신중한 접근이 필요해 보였습니다.<br/>
그래서 이를 정확히 검증하기 위해, <br/>
최종 작업에서 **as-is VS to-be API 간 응답 값을 확인할 수 있는 validation 기능을 가진 프로세스를 추가**하여 검증하는 것이 필요했습니다.
<br/><br/>
### 2. 시시각각 변하는 유기적인 데이터

각 제품마다 딜은 **[신청-승인-확정-진행-종료] 형태의 라이프사이클이 존재**합니다.<br/>
그리고 라이프사이클이 진행되는 동안 **딜 데이터는 계속 변하며, 이를 추적하고 빠르게 갱신 해주어야 합니다.**

![상품의 딜 종류별 라이프사이클이 반드시 동일하지 않을 수 있는 조건을 반영하여 MongoDB 에 적재한다](/files/post/2024-06-07-retrospective-deal-internalization/Untitled5.png)


하지만 AS-IS 모델의 Serving 데이터와 TO-BE 모델의 새로 구축한 Serving 데이터가 갱신되는 시점이 서로 다르다보니,<br/>
**AS-IS와 TO-BE 간에 데이터의 정합성을 비교할 때, 비교하는 시점에 따라 일시적인 다름인지, 잘못된 데이터인지 판단할 수 있어야 합니다.**<br/>
특정 시점, 예를 들어 15:00:00에 판매 수량 데이터를 확인했을 때,<br/>
A 상품은 15건, B 상품은 10건으로 표시되는 상황이라면, 세 가지 가능성을 염두하고 판단할 수 있습니다.
- A 상품의 판매 수량 15건은 최신 갱신된 결과이지만, B 상품의 10건은 아직 갱신 이전의 값 일 수 있습니다.<br/>
    이 경우, B 상품의 판매 수량이 다음 갱신을 통해 15건으로 변경된다면 TO-BE로의 전환이 잘 되었고 일시적인 다름이었음을 판단 할 수 있습니다.
- 반대로, 만약 B 상품의 판매 수량 10건이 갱신 이후의 값이었다면, B상품의 판매 수량 결과값이 잘못된 것으로 판단할 수 있습니다.
- 추가로, B의 10건이 정확한 값이었고, 오히려 A값이 잘못 계산되어왔던 것인지 추가로 확인해 볼 수도 있습니다.<br/>
예시 이외에도, 각 필드마다 갱신 조건이 있고 어떤 시점에 어떻게 변경될 수 있는지 알아야 검증 과정에서 정상적인 데이터인지 아닌지 판단 할 수 있습니다.

**또한, 딜의 라이프사이클뿐 아니라 상품의 라이프사이클도 영향을 미치기 때문에**<br/>
딜 한정 수량이 품절되어도 상품 수량이 남아있는지 구분하여 전시 노출 해야하는 부분,<br/>
전시종료 여부는 딜의 기간종료인지 품절종료인지, 품절은 상품품절인지 딜품절인지를 구분하여 전시 노출 결정을 할지를 명확히 알아야합니다.<br/>
**이를 위해, 전시 딜의 동작방식을 위한 딜의 라이프사이클에 대한 이해도를 높일 필요가 있습니다**
<br/><br/>
### 3. 롤백(Rollback)

11번가를 이용하는 대부분의 사람들은 할인행사들을 잘 파악하고 행사기간을 이용하여 물품을 구매하고 있습니다.<br/>
행사상품정보는 11번가에서 물품을 구매할때 가장 먼저 보는 진입점인 만큼 중요성이 매우 크다고 생각합니다.<br/>
**그래서 전환과정에 잘못된 데이터가 노출 되었을때 AS-IS 모듈로 가장 빠르게 롤백 할 수 있도록 전환하는 장치를 마련해야 합니다.**<br/>
롤백을 위한 재 배포없이 이전 모듈로 동작 할 수 있는 간편한 방법으로는<br/>
DB 테이블에 AS-IS와 TO-BE 모듈의 실행을 분기할수 있는 플래그를 넣어두고 실시간으로 플래그값을 변경하는 방법이 있습니다. <br/>

## 작업 진행

---

### 📖작업목록 계획

- **✅ 전시딜 MongoDB 구축**
  - 수집 속도 개선(=갱신속도 개선)
  - 적재대상 딜 대상을 추출 및 Kafka로 메시지 수집 및 MongoDB적재

- **✅ 전시 API MongoDB 컬렉션 참조로 변경**
  - 롤백 장치 생성(기존 Oracle Serving 테이블 조회)
  - Oracle 서빙 테이블과 MongoDB 서빙 컬렉션의 구조 차이로 인한 가공 로직 변경

- **✅ 검증기 구축**
  - MongoDB 적재 데이터 검증을 위한 Validator 작성
  - 영향범위 API 응답값 검증을 위한 Validator 작성
      - Oracle 서빙 테이블 vs MongoDB 서빙 컬렉션 조회 정합성 비교 (총 개수, 각 필드 별 값)
      - Elasticsearch를 활용한 실시간 모니터링

<br/><br/>

### 이제 열심히 개발을 해봅니다

<figure style="width: 70%; margin-left: auto; margin-right: auto;">
    <a href="/files/post/2024-06-07-retrospective-deal-internalization/Untitled6.png" class="mg-link">
        <img src="/files/post/2024-06-07-retrospective-deal-internalization/Untitled6.png">
    </a>
    <figcaption class="caption">출처: https://blog.naver.com/youth_pear/222842134763</figcaption>
</figure>

### 완성: TO-BE 모델

<figure style="width: 140%; margin-left: auto; margin-right: auto;">
    <a href="/files/post/2024-06-07-retrospective-deal-internalization/Untitled7.png" class="mg-link">
        <img src="/files/post/2024-06-07-retrospective-deal-internalization/Untitled7.png">
    </a>
    <figcaption class="caption" style="text-align:left;">플로우(flow):  원천DB에서 대상 데이터 수집 → 메시지발행 → 메시지 수집 및 데이터 가공 → MongoDB 데이터 적재/구축 → API에서 적재된 데이터 사용 → API 응답값 검증</figcaption>
</figure>

### 데이터 구축 (SpringBatch, Kafka, MongoDB)

- 딜 정보 수집대상 추출 및 메시지 발행 배치 생성
    - Oracle Serving 테이블 생성 PL/SQL 중, 필요한 딜 정보 생성부분을 분리해내어 추출 배치 생성
        - 정각에 정확한 딜정보를 노출해야하는 타임딜은 더 빠른 상품정보 갱신을 위해 **타임딜상품 정보 갱신배치 추가 생성**하여 정확도를 높임
        - 성능 향상을 위해, **데이터가 많은 딜 정보는 별도의 배치를 추가 생성**하여 데이터를 추출
        - **멀티스레드(thread-pool=5)로 동작**하도록 적용하여 빠른 수집이 이루어지도록 작업
        - **추출된 딜 정보는 카프카로 메시지 발행**하고 ZeroPayload 방식으로 키값만 송신하여 효율을 높임

<br/>

- 딜정보 가공 및 적재
    - 중요한 딜이 덜 중요한 딜정보 메시지에 밀리는 일이 없도록, **배치마다 각각 토픽 나누어 운영하고 메시지를 읽도록 처리**
    - Lag의 빠른 소진을 위해 **데이터가 많은 딜은 멀티스레드로(concurrency=3) Conesume** 처리
    - API 조회를 더욱 빠르게 처리하기 위한 전략으로, **데이터 구축 과정에서 데이터 가공 로직을 집중적으로 처리**
        - 데이터가 이미 가공되어 있으므로 API 조회 시에는 복잡한 처리 없이 데이터를 즉시 사용할 수 있음
    - 조회에 쓰이는 필드에 맞는 단일, 복합 인덱스 생성

### API 적용

- OracleDB 조회는 새로 구축된 MongoDB 레포지토리에서 조회되도록 변경
- MongoDB 조회쿼리 인덱스 활용 조정
- 기존 로직을 담은 AS-IS 소스코드는 유지하고 변경된 TO-BE 조회 로직은 배포없이 빠르게 롤백 가능한 장치 구성

### 검증기 생성

- 5분, 10분 단위 등으로 설정한 스케줄링 단위시간마다 비교할 검증 데이터를 LogStash에 송신하도록 구성
- ELK 스택을 활용하여 데이터를 수집 및 파싱 후, 데이터 비교 대시보드 구성
- 1차: MongoDB 신규 컬렉션과 Oracle 기존 테이블 데이터 Validator 생성
    - 방식: 총 데이터 수, 데이터의 차집합을 양방향으로 비교, 랜덤하게 뽑은 딜데이터 비교
- 2차: AS-IS API와 TO-BE API 호출 응답값 데이터 Validator 생성
    - 실제 호출된 Access Log 패턴별 차이나는 응답필드가 있는 경우만 확인할 수 있도록 구성

<br/>


## 물론, 시행착오도 있었다.

---

<figure style="width: 35%; margin-left: auto; margin-right: auto;">
    <a href="/files/post/2024-06-07-retrospective-deal-internalization/Untitled8.png" class="mg-link">
        <img src="/files/post/2024-06-07-retrospective-deal-internalization/Untitled8.png">
    </a>
    <figcaption class="caption">출처: 데브 경수님의 인스타툰 @waterglasstoon</figcaption>
</figure>

### 1. Embedded Document 미갱신

<figure style="width: 60%; margin-left: auto; margin-right: auto;">
    <a href="/files/post/2024-06-07-retrospective-deal-internalization/Untitled9.png" class="mg-link">
        <img src="/files/post/2024-06-07-retrospective-deal-internalization/Untitled9.png">
    </a>
    <figcaption class="caption">출처: 데브 경수님의 인스타툰 @waterglasstoon</figcaption>
</figure>

딜 정보 하위에 임베디드 문서로 상품 정보를 사용했지만,<br/>
상품 정보와 딜 정보의 갱신이 개별적으로 이루어지고, 딜 정보가 갱신될 때만 하위의 상품 정보를 갱신하는 문제로,<br/>
**상품 정보의 갱신분이 반영되지 않은 채로 API에서 조회되는 경우가 생겼습니다.**<br/>
딜 정보는 분 단위로 갱신되기 때문에 대부분 빠르게 잘 갱신되었지만,<br/>
**가끔 하위의 상품 정보가 미갱신된 상태로 조회되는 경우가 있었는데 정확하게 캐치하지 못한 케이스가 종종 있었습니다.**

→ 딜 하위의 임베디드 상품 정보를 활용하지 않는 방향으로 변경하여 대응했고,
성능도 중요하지만 복잡한 로직에서는 로직을 단순하게 가져가는 것도 좋은 방향이라고 생각했습니다.

### 2. 예상치 못한 데이터 급증

1-2분 단위 갱신 딜 정보 → 10분 간의 갱신 지연된 상황이 발생했습니다.<br/>
최초 상용 배포 직후까지 상용 수집 딜 개수는 16만 건이었고 분 단위 수집 배치 속도에 문제가 없었습니다.<br/>
하지만, 이후 **37만 건으로 급증한 데이터로 인해 수집 배치 시간이 10분까지 늘어났고 그 동안 딜 정보는 갱신되지 않아 미노출** 되었던 상황이 발생했습니다.

![](/files/post/2024-06-07-retrospective-deal-internalization/Untitled10.png)

→ 해결을 위해, **수집과 적재 처리 시 멀티스레딩 방식으로 변경**하여 처리 속도를 높였고<br/>
하나의 배치로 수집했던 배치의 구분점을 찾아 두 개의 배치로 나누고 **각각의 토픽(Topic)으로 메시징 처리되도록 분리**하였습니다.<br/>
처리 성능을 올릴 수 있는 두 가지 방법을 동원해 해결했습니다.

### 3. 복잡한 롤백 플랜

최초 상용 테스트 기간에 여러 상황에 대비한 롤백 플랜을 만들고자 하는 생각에 **상황별 대응 가능한 롤백 플랜으로** 만들었습니다.
- 2단계까지 단계별로 되돌아갈 수 있도록 구성
- 롤백 장치 4개를 삽입하여 원하는 부분만 롤백할 수 있도록 구성
    - 롤백 장치를 조정하면 직접 데이터 Delete 작업이 필요
하지만, **유연한 롤백 플랜이라고 생각했던 방식이 실제 긴박한 이슈라이징 상황에 어떤 단계로 돌아가야 하는지 오히려 혼란이 되었습니다**

→ 이후, 롤백 장치 개수는 최소한으로, 롤백 단계는 1단계로 줄이고 롤백 장치 조정 이외 추가 작업은 모두 제거했습니다.<br/>
앞으로 **롤백플랜은 가능한 심플하게 만들고 꼭 필요한 상황이 아니라면 롤백 단계도 최소한으로 줄이는 것이 더 좋은 방향**이라는 생각이 들었습니다.

### 4. Validation 대상 선정의 빈틈

초기 MongoDB에 **적재된 데이터 대상으로만 Validator를 구축**하고 검증 진행을 했습니다.<br/>
이 방법은 구축된 데이터를 사용하는 로직 부분에서 발생할 수 있는 문제를 캐치하지 못하는 문제가 있었습니다.

<figure style="width: 70%; margin-left: auto; margin-right: auto;">
    <a href="/files/post/2024-06-07-retrospective-deal-internalization/Untitled11.png" class="mg-link">
        <img src="/files/post/2024-06-07-retrospective-deal-internalization/Untitled11.png">
    </a>
    <figcaption class="caption">출처: https://chimhaha.net/story/111311</figcaption>
</figure>

→  validation 대상을 5분, 10분, 20분 단위로 스케줄러를 생성하여 주기적으로 **최종 API의 응답 결과를 비교**하는 방향으로 수정되었습니다<br/>
새로 구축된 데이터 검증도 중요하지만, <br/>
결국 최종적으로 영향이 가는 API 응답 결과를 검증하는 것이 더 우선으로 보아야 모든 문제를 체크할 수 있었습니다.<br/>


## 프로젝트 결과 및 개선 사항

---

- 프로젝트 결과
    - MongoDB 적용으로 오라클 커넥션이 제거되어 대량 트래픽에서 API 응답 지연이 크게 개선되었습니다.
    
    ![](/files/post/2024-06-07-retrospective-deal-internalization/Untitled12.png)

    <figure style="width: 150%; margin-left: auto; margin-right: auto;">
        <a href="/files/post/2024-06-07-retrospective-deal-internalization/Untitled13.png" class="mg-link">
            <img src="/files/post/2024-06-07-retrospective-deal-internalization/Untitled13.png">
        </a>
    </figure>    
  
    <figure style="width: 150%; margin-left: auto; margin-right: auto;">
        <a href="/files/post/2024-06-07-retrospective-deal-internalization/Untitled14.png" class="mg-link">
            <img src="/files/post/2024-06-07-retrospective-deal-internalization/Untitled14.png">
        </a>
    </figure>
    
    - 딜 갱신 속도가 빨라져 행사 기간 중 급격한 데이터 증가에도 안정적으로 대응할 수 있게 되었습니다.
    - Oracle 의존성이 제거되었습니다.
    - OracleDB와 mongoDB 모두 확인해야 했던 디버깅 지점이 MongoDB로 단순화되었습니다.
        
        이로서, 이슈 대응이 편해졌고, 요구사항에 대한 기민한 대응도 가능해졌습니다.
        

![짜란다 짜란다 , 출처: 데브 경수님의 인스타툰 @waterglasstoon](/files/post/2024-06-07-retrospective-deal-internalization/Untitled15.png)

## 마치며…

---

- **추가 개선 포인트와 향후 계획**
    - 최초 데이터 적재 이후 갱신이 필요 없는 필드와 지속적인 갱신이 필요한 필드를 구분하면 갱신에 필요한 리소스를 줄여 더 가벼워질 수 있음
    - 이어서 향후 Event 기반의 CDC Platform인 카시타(Casita) 에서 딜 정보를 받아오는 방식으로 전환한다면 딜 수집 리소스도 줄일 수 있음
    - 상품 정보와 딜 정보를 로직상 완전히 분리되도록 리팩토링하면 유지 보수성을 더 높일 수 있음

현재는 안정적인 운영이 이루어지는지 모니터링하고 있는 중이고<br/>
조만간 롤백을 위해 만들어두었던 코드 및 기타 AS-IS 코드를 삭제할 예정입니다.<br/>
그리고 프로젝트 진행 간 생긴 지저분해진 코드 정리와 개선 포인트의 진행도 계획하고 있습니다.<br/>

- **느낀점**

이번 회고 글에 넣기에는 너무 소소했던 삽질부터<br/>
실제 상용 환경에서 겪은 큰 이슈들까지 많은 시행착오를 통해 이전보다 더 단단해진 느낌입니다.<br/>
돌이켜 생각해보면, 평소 스스로 꼼꼼한 편이라고 생각했던 저는, 가끔 놓치는 부분들이 분명히 존재했고<br/>
작업 흐름과 필요한 정보를 알고 있지만, 정확히 이해하고 활용할 줄 몰랐던 상황들이 있었습니다.<br/>
특히 복잡하고 깊은 고민이 필요한 부분을 쉽게 해결하지 못하고 혼자 고민만 하고 있었습니다.<br/>
이러한 부족한 점들을 개선하기 위해 앞으로 프로젝트 진행 상황과 내용을 단계별로 메모하여 주기적으로 정리하려고 합니다.<br/>
크고 복잡한 문제는 작게 쪼개어 분할하여 해결하고, 이렇게 간결한 내용들이 이어져 전체적인 플로우가 만들어지도록 해야겠다고 느꼈습니다.<br/>
다행히도 프로젝트 진행 간, 11번가의 일을 위해서는 자기의 시간을 아끼지 않고 너나 할 것 없이 적극적으로 도움을 주고 싶어 하시는 분들이 많았기에<br/>
어려운 상황들을 하나씩 헤쳐나갈 수 있어 뿌듯함과 동료애를 느낄 수 있었고,<br/>
앞으로 저도 어려운 상황에 다른 분들에게 적극적인 도움을 줄 수 있는 사람이 되어야겠다고 다짐했습니다.<br/>
느꼈던 바를 발판 삼아, 앞으로도 지속적인 개선을 통해 더 나은 서비스를 제공할 수 있도록 노력하겠습니다.<br/>

개선작업에 많은 도움을 주신 손지성, 반태형님 감사합니다<br/>
긴글 읽어주셔서 감사합니다.