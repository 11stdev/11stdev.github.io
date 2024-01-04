---
layout: post
title: 'DB Function To Java 전환으로 기술부채 갚기'
author: 문상록
date: 2022-12-02
tags: [Oracle, DBFunction, java, msa, legacy]
---
안녕하세요 11번가 쿠폰/정산개발팀에서 프로모션 파트 개발을 하고 있는 문상록입니다.

올해에 십수년간 운영되던 프로모션 조회를 위한 오라클 DB Function을 Java API로 전환했습니다.
그 과정과 여러 이슈 해결과정을 공유드리고자 포스트를 작성하게 되었습니다.  
참고로 본 포스팅은 소스코드나 설계와 같은 세부사항은 담겨있진 않아서 전체적인 프로세스를 가볍게 읽어본다고 생각하시고 봐주시면 감사하겠습니다 ^^

일정 규모가 있는 회사라면 개발자가 함부로 건드리기 어려운 영역이 있습니다.
밑에 라이온킹 심바가 가지말라고 하는 '레거시 코드'라는 영역입니다.

<div style="text-align: center;">
    <img src="/files/post/2022-12-05-promotion-dbfunction-to-java/legacy_meme.jpg" width="330">
    <figcaption style="color: gray">어디든 발견할 수 있는 레거시들</figcaption>
</div>  

<br>
제가 담당하고 있는 프로모션쪽에도 여러 레거시 코드가 있습니다.
잘 돌아가고는 있지만 고치기가 꺼려지고, 점점 수정 비용이 올라가는 기술부채 같은 존재들이지요.
이 중 프로모션 조회용 오라클 DB Function 5개(각각 1천라인 이상)를 Java로 전환하되 성능도 가능하면 기존보다 빠르도록 개선시켜야 했습니다.

<hr/>

## 개선해야 할 이유와 개선 방향

구체적인 개선 이유는 아래와 같습니다.
1. 장애 시 대응 한계
- 로그를 통한 디버깅 불가로 원인 해석이 어려움
- 상용 환경 배포 후 롤백 어려움
2. 테스트 한계
- 개발 시 단위테스트 불가
- 상용 환경 배포 후 검증테스트 어려움
3. 유지보수 한계
- DB Function이 거대하고 내부 변수 간 의존이 심해, 시간이 갈수록 변경에 따른 부작용 발생 가능성 증가
- 비슷한 듯 비슷하지 않은 API 제작 요청이 들어올 경우, DB Function 1세트 추가제작 필요 (개발시간도 오래 걸림)
4. DB 트래픽 분산처리 한계
- 클라이언트에서 DB를 지정하고 해당 DB의 Function만 호출하므로, 트래픽 분산의 유연도가 떨어짐

이러한 개선 이유가 있으니, 앞으로는 Java API로 전환해서 장애 대응력, 테스트 유연성, 유지보수성을 키우고, DB의존도는 약화시켜서 향후 DB 분산처리도 유연하게 하자는 방향성을 가지게 되었습니다.

<hr/>

## 기존 DB Function 사용 방식
<div style="text-align: center;">
    <img src="/files/post/2022-12-05-promotion-dbfunction-to-java/asis_architecture.jpg" width="1000">
    <figcaption style="color: gray">대략적인 기존 업무흐름</figcaption>
</div>
<br>
그림 중앙에 빨간 네모 박스로 표현한 관심영역이 곧 개선대상 영역입니다.
상품,주문,검색,전시쪽에서 직간접적으로 DB Function을 직접 사용하는 구조였습니다.
클라이언트와 서버 모두 DB Function에 의존했던 것이죠.

<hr/>

## 개선안 스케치
위의 구조를 개선하기 위해 육각형 아키텍처를 기반으로 개선안을 스케치 해보았습니다.
이렇게 DB에 직접 의존하던 구조를 벗어나서 MSA환경의 서버로 API를 요청할 수 있게끔 구조를 변경시킬 계획입니다.
<div style="text-align: center;">
    <img src="/files/post/2022-12-05-promotion-dbfunction-to-java/solution_architecture.jpg" width="1000">
    <figcaption style="color: gray">개선안 스케치</figcaption>
</div>
<br>

<hr/>

## 비즈니스 로직 흐름 컨셉 및 기술스택 결정
다음으로는 자바 내의 로직 흐름 컨셉을 잡고, 사용할 기술스택을 결정했습니다.
팀 내 회의를 통해 로직의 흐름을 병렬과 순차처리의 조합으로 처리해서 성능을 끌어올리게 컨셉을 잡았습니다.
  
사용할 기술 스택은 여러 기술 후보 중에 WebFlux+R2DBC, WebMVC+CompletableFuture, 일반WebMVC를
검토 및 테스트를 해보았고, 최종적으로 WebMVC+CompletableFuture+ParallelStream 조합으로 사용하기로 했습니다.
  
WebFlux를 제외시킨 이유는  
1) 당시 오라클 R2DBC 드라이버를 지원하는 스프링부트 버전이 DB 커넥션풀을 지원하지 않았습니다.  
2) 저와 저희팀에 친숙한 기술이 아니라서, 러닝커브 비용을 고려하면 프로젝트가 오래 걸릴 것 같았습니다.
<div style="text-align: center;">
    <img src="/files/post/2022-12-05-promotion-dbfunction-to-java/logic_flow.png" width="1000">
    <figcaption style="color: gray">로직 흐름 컨셉 일부</figcaption>
</div>
<div style="text-align: center;">
    <img src="/files/post/2022-12-05-promotion-dbfunction-to-java/concept_performance_test.png" width="1000">
    <figcaption style="color: gray">기술 스택 결정을 위한 자체성능테스트 과정</figcaption>
</div>

<hr/>

## Java 버전 8 -> 17로 업그레이드
이후에 저희팀은 로직을 병렬처리로 처리하더라도 "과연 Java로 바꾼게 DB에서 처리하는것보다 빠를 수 있을까?"하는 걱정이 들었습니다.  
그래서 현재 MSA 서버가 Java8이니, 이것을 Java17로 업그레이드를 하면 어떨까? 하는 생각과 함께 Java버전 업그레이드 검토를 하게 되었지요.  
이후 Java 업그레이드 절차는 아래와 같이 진행했습니다.  

**1. 벤치마크 자료 조사**
- 벤치마크 자료를 통해 JDK8 -> JDK17로 상향 시, 연산속도가 약 25%가 상승할 수 있겠다고 추정하게 되었습니다.  
<p/>

**2. 상용 배포 및 롤백 방안 마련**
- 기존 서버 12대 이외에 JDK17용 신규 서버 12대를 생성했습니다.
- 카나리 배포로 천천히 JDK17 서버 12대를 배포할 계획을 세웠습니다.
- 기존서버와 잠시 병행운영 후 안정되면 기존서버 DOWN 시키기로 했습니다.
- 만약 롤백이 필요할 경우 JDK17 서버만 DOWN 시키기로 했습니다.
<p/>

**3. 모니터링 방안 마련**
- 기존 서버는 주로 제니퍼 APM을 통해 모니터링을 많이 했으므로, 서비스엔지니어링팀에 요청하여 JDK17 지원하는 제니퍼 에이전트 설치를 진행했습니다.
<p/>

**4. 소스코드 변경**
- JDK17(Temurin) 업그레이드를 위해 기존 소스코드를 고쳐야 했습니다. 예를 들어 스프링부트 버전을 2.5.x로 올렸고, 11번가 코어 라이브러리도 업그레이드 했습니다.
- 뿐만 아니라 JUnit, Gradle, 각종 Spring Cloud 의존성 변경이 필요했습니다.
<p/>

**5. 신규 CI/CD 환경 셋팅 후 배포**
- JDK17을 위한 CI/CD 파이프라인을 새로 만들고, 계획한대로 배포 및 모니터링을 실시 했습니다.    
<div style="text-align: center;">
    <img src="/files/post/2022-12-05-promotion-dbfunction-to-java/benchmark.png" width="1000">
    <figcaption style="color: gray">JDK8 vs 11, JDK11 vs 17 벤치마크 캡쳐</figcaption>
</div>

<hr/>

## 소스코드 이관 시작
자, 이제 기술스택도 정했고 서버의 Java 버전도 올렸으니, 본격적으로 소스코드 이관을 시작해야겠죠?  
소스 이관은 아래의 절차를 거쳐 진행했습니다.  

**1. DB Function to Mybatis 이관**
- 우선, 자바 어플리케이션에 최소한의 분기처리 로직만 두고, DB Function을 통째로 Mybatis로 쿼리를 옮겼습니다.
<div style="text-align: center;">
    <img src="/files/post/2022-12-05-promotion-dbfunction-to-java/mybatis_migration.jpg" width="1000">
    <figcaption style="color: gray">Mybatis로 마이그레이션 예시</figcaption>
</div>
- 이후, 쿼리에서 자바 로직으로 이관할 부분을 식별하기 위해 Mybatis 쿼리를 잘게 분리하여 약 30개의 쿼리로 만들었습니다.
<div style="text-align: center;">
    <img src="/files/post/2022-12-05-promotion-dbfunction-to-java/mybatis_split.jpg" width="1000">
    <figcaption style="color: gray">Mybatis 쿼리 분할 예시</figcaption>
</div>
<p/>

**2. 쿼리튜닝**
- Java로직 개발하기 전에, 분리한 쿼리를 DB개발팀에 요청하여 쿼리 튜닝 및 쿼리 통합을 진행했습니다.   
이 과정에서 쿼리가 30여개 -> 20여개로 줄었습니다  
<p/>

**3. 점진적 Java 전환**
- 우선 절차식으로 Java를 전환했습니다. 전반적으로 단위/통합테스트를 꼼꼼히 해나가며 작은 작업 단위로 이관해야 했습니다.
- 이후 비동기 병렬처리 가능한 부분을 식별하고
- 병렬처리와 순차처리의 조합으로 성능효율적으로 어플리케이션을 구성했습니다.
- 리팩토링을 하면서 육각형 아키텍처를 통해 핵심로직이 견고하고 확장에 열린 구조로 구축할 수 있게 되었습니다.
<div style="text-align: center;">
    <img src="/files/post/2022-12-05-promotion-dbfunction-to-java/java_migration.png" width="1000">
    <figcaption style="color: gray">자바 로직 흐름 및 아키텍처</figcaption>
</div>
<p/>

**4. Java 성능 추가 튜닝**  
Java로 전환하면서 성능측정을 해보았지만, 아직은 DB Function보다 느렸습니다. 이를 위해 좀 더 성능개선을 시도했고 드디어 DB Function보다 빨라질 수 있게 되었습니다.  
그 과정은 아래와 같습니다.
- IntelliJ의 AsyncProfiler를 통해 비동기 스레드의 병목지점을 탐색했습니다.
- 캐싱이 필요한 곳에 로컬캐시(Caffeine)를 설정했습니다.
- 병렬처리로 인한 부하 발생 시 처리를 위해 비동기 스레드풀을 여러개로 만들고, 스레드풀의 Queue가 넘쳤을 경우 Rejection Policy를 재정의해서 Retry를 할 수 있도록 했습니다.
<div style="text-align: center;">
    <img src="/files/post/2022-12-05-promotion-dbfunction-to-java/performance_tuning.png" width="1000">
    <figcaption style="color: gray">자바 추가 튜닝 중 일부</figcaption>
</div>

<hr/>

## 안정적인 배포를 위한 AS-IS/TO-BE 결과 비교로직 개발
기본적인 개발은 지금까지의 절차를 통해 완료했습니다.     
배포를 진행하기 전에 안정적인 배포를 위해 Java로직과 기존의 DB Function로직의 결과를 비교하는 로직을 만들어야 했습니다.  
그래야 상용 환경에 배포하더라도 로직의 이상유무를 신속하게 파악할 수 있었기 때문이지요.

결과비교로직 개발 과정은 아래와 같습니다.  
- 우선 API Controller단과 Service단 사이에 손상방지계층(Anti-Corruption Layer)를 설치하여, 핵심 도메인 로직은 지키면서 결과비교로직은 독립적으로 만들 수 있게끔 설계했습니다.
- AS-IS인 DB Function과 Java 결과 비교로직은 스위치로 on/off를 조절함으로써, 유연하게 상용환경에 반영할 수 있도록 만들었습니다.
- 결과비교는 AS-IS/TO-BE Response 객체를 String으로 비교하고, 틀리면 Slack Alarm으로 실시간으로 알 수 있게끔 구성했습니다.
- 통합테스트 클래스에도 수백개의 샘플을 순회하며 결과비교로직을 테스트하게 만들어 개발 신뢰성을 향상시켰습니다.
- 결과비교로직까지 개발을 다 한 뒤엔 시스템엔지니어링팀에 의뢰해서 성능테스트도 몇 차례 걸쳐서 통과를 했습니다.
<div style="text-align: center;">
    <img src="/files/post/2022-12-05-promotion-dbfunction-to-java/result_compare.png" width="1000">
    <figcaption style="color: gray">결과 비교로직 제작 과정 일부</figcaption>
</div>

<hr/>

## 배포 과정
이후 아래 배포 시나리오에 따라 상용환경에 배포를 진행했습니다.
- 안정적인 상용 배포를 위해 카나리 배포 적극 사용
- DB Function 1개 전환 개발 완료 시, 12대의 WAS 중에 1대의 WAS에만 우선 반영
- 1대 서버 배포 직전 신규 API 스위치 off, 결과 비교로직 스위치 off
- 배포 후 신규 API 스위치 on, 결과 비교로직 스위치 on
- 1대의 서버에서 안정된 것을 확인하면, 서서히 늘려가며 배포하여 부작용이 없는지 관찰
- 이슈 대응 후 안정화 되면, DB 부하 감소를 위해 결과 비교로직 스위치 off 

<hr/>

## 배포 이후 이슈 대응 과정
<div style="text-align: center;">
    <img src="/files/post/2022-12-05-promotion-dbfunction-to-java/internel_error.jpeg" width="600">
    <figcaption style="color: gray">서버 에러나면 요청자의 눈썹이 위로 향하는 것을 볼 수 있다</figcaption>
</div>
<p/>
물론 배포하고 이상이 없길 바랬지만, 모든 프로젝트가 그렇듯 이번 프로젝트도 안정화 과정이 필요했습니다.  
주요 이슈와 대응 과정을 나열해보겠습니다.

<hr/>

#### 기능이슈
**1. 현상 :**
   - 두번째 DB Function 상용 카나리 배포 후 모니터링 중 주문화면에 이상한점을 발견했습니다.  
   주문화면 진입 시 상품의 할인금액과 주문완료 후 상품의 할인금액이 서로 달랐던 것이었습니다.

**2. 해결 과정 :**
   - 주문개발팀과 쿠폰/정산개발팀 협업하여 정확한 현상 파악을 했습니다.
   - 주문 시 L.point 사용 항목 사라지는 것을 발견했고, 검증 환경에서 재현에 성공했습니다.
   - 프로모션 Java API가 주문에 상품번호를 return하지 않는 것을 발견했습니다.
   - 알고보니, 결과 비교 로직에서도 상품번호를 비교하고 있지 않아서 원인파악에 오래 걸렸습니다.
   - 향후 API 이관 시 기존의 Request, Response 항목을 꼼꼼히 따져야 한다는 교훈을 얻었습니다. 
   - 결과 비교로직 만들 경우, AS-IS의 Response 객체 기준으로 비교해야 한다는 교훈을 얻었습니다.  
   (이슈 당시에는 AS-IS와 TO-BE의 중간형태인 객체를 만들어서 변환했었습니다.)


<hr/>

#### 성능이슈 - NPE와 캐시 이슈 콜라보
**1. 현상 :**
   - 다중상품처리 API를 처리하던 도중 WAS 1대에서 Error로그를 다량 발견했습니다. (다행히 실제 서비스는 안하던 상태였습니다.)
   - 다중상품 병렬처리 시 한번씩 DB커넥션풀 과부하가 걸렸다가 풀리던 것이었습니다. 그로인해 부작용이 2가지가 발생했습니다.  
   - 부작용 1 : DB커넥션풀이 밀리니, 비즈니스 로직에서 NullPointerException(5xx 에러)이 발생했습니다.
   - 부작용 2 : NPE로 인해 Null로 리턴된 값이 캐싱되었습니다.(캐싱 시간 10분, 12시간도 포함되어 있었습니다.)

**2. 해결 과정 :**
   - 해결을 위해 우선 WAS의 스레드풀 수치를 튜닝했습니다.
   - Null 처리를 보완하고, Null로 리턴된 값은 캐싱하지 않게 수정했습니다.
   - 캐시 시간을 조절해서 추후 장애 리스크 감소시켰습니다.
   - DB개발팀과 협의하여 DB 커넥션풀 max size 증가시키고, DB 목적지를 변경해서 DB의 부하를 분산시켰습니다.
<div style="text-align: center;">
    <img src="/files/post/2022-12-05-promotion-dbfunction-to-java/npe_issue.png" width="700">
    <figcaption style="color: gray">NPE 당시 로그 화면</figcaption>
</div>
<hr/>

#### 성능이슈 - 메모리 누수
**1. 현상 :**
   - 상용 환경에 배포한지 약 1달 뒤 쯤, 모니터링툴에서 free메모리 영역 1% 미만 남았다고 알람이 왔습니다.  
   현상을 보니 메모리가 장기적으로 차오르고 있었습니다.

**2. 해결 과정 :**
   - 리눅스 커널은 권한부족으로 인해 건드릴 수 없어서 어플리케이션의 캐시 max사이즈를 낮추고, heap size도 낮췄으나, 개선되지 않았습니다.
   - 이후 제니퍼(JDK17지원) agent가 깔린 홀수 WAS만 메모리 사용량이 증가하는 것을 발견했습니다.
   - 시스템엔지니어링팀과 확인을 해보니 ZGC 옵션 중 –Xmx, -Xms 값들이 같을 경우 ZUncommit 옵션이 disable되어 메모리 환원이
   안됨을 확인했습니다.
   - 이 부분을 조치했으나, 일부 WAS는 여전히 메모리 증가하는 것을 발견했습니다.  
   여러 시도 끝에 제니퍼 수집서버 목적지 변경 후 상용서버가 드디어 안정화 되었습니다.
   (원인은 기존 제니퍼 수집서버에 부하가 많아서 저희 서버의 버퍼가 증가했던 것으로 보였습니다.)
   - 그러나 검증계에는 다른 현상의 메모리 누수가 있었습니다.  
   시스템엔지니어링팀과 분석을 하다 결국 JDK 버전을 17.0.1 -> 17.0.4.1 업그레이드하여 안정화 시켰습니다.  
   (17.0.4.1 릴리즈 노트를 보니, memory leak이라는 키워드로 많은 것이 개선된 것이 보였습니다.)
<div style="text-align: center;">
    <img src="/files/post/2022-12-05-promotion-dbfunction-to-java/memory_leak.png" width="700">
    <figcaption style="color: gray">메모리 누수 현상 캡쳐</figcaption>
</div>
<div style="text-align: center;">
    <img src="/files/post/2022-12-05-promotion-dbfunction-to-java/release_note.png" width="700">
    <figcaption style="color: gray">JDK 17.0.4.1 릴리즈 노트</figcaption>
</div>  


이렇게 상용 환경에서 발생했던 몇 가지 이슈를 공유했는데, 읽는 분들께 도움이 되었으면 좋겠습니다.

<hr/>


## 개발기간
개발은 첫 삽 뜨는 것 부터 안정화까지 총 1년이 걸렸다고 볼 수 있는데요. 조금 더 세부적으로 보자면 아래와 같습니다.
- 첫번째 DB Function 이관 : '21.07 ~ '22.01
- 두번째 DB Function 이관 : '22.01 ~ '22.03
- 세번째, 네번째 DB Function 이관 : '22.03 ~ '22.04
- 다섯번째 DB Function 이관 : '22.05 ~ '22.06

돌이켜보자면, 각 Function 개발 시, 기능개발보다는 결과 검증 로직 개발, 성능 최적화, 이슈 안정화 기간이 더 오래 걸렸습니다.
그리고 첫번째 DB Function 개발 시 핵심 도메인 로직을 완성했으므로, 이후 추가 DB Function 이관 시 개발시간을 단축 시킬 수 있었습니다.

<hr/>

## 성과 - 성능 측면
Java 이관 소스가 있는 서버 전체적으로 전환 이전에 비해 평균 Response Time이 8ms -> 6ms로 약 25% 향상되었습니다.
프로모션 Java 전환 API 호출량이 전체 대비 72%를 차지하고 있으므로, 의미 있는 성과로 보입니다.

검증계 성능테스트 당시 상품 1개에 대한 API는 Java가 DB Function에 비해 50% 느렸으나, 요청 상품 개수가 많아질 수록 병렬처리에 의한 성능 개선 효과를 볼 수 있었습니다.

<hr/>

## 성과 - 유지보수 측면
우선 배포 프로세스가 개선되었습니다. 기존의 번거롭고 테스트하기 어려웠던 DB 배포 방식에서 MSA 환경의 배포방식으로 간소화 되었습니다.
그리고 핵심 도메인 로직은 변경 가능성이 적어서 견고해졌고, 테스트 로직이 작성되어 있으므로 추후 신규/수정 개발건 진행 시 수정이 용이해졌습니다.

<hr/>

## 추후 개선 사항
1. 현재는 조회하는 DB Function만 이관했으나, 향후에는 각 클라이언트에서 직접 수행하는 모든 프로모션 CRUD 로직을 자바 API로 요청받아 처리하도록 변경 예정입니다.
2. 현재 프로모션 로직 내에 있는 타 도메인 영역 DB 조회 쿼리를 모두 각 도메인별 API 호출로 변경할 예정입니다.
3. 현재 사용중인 오라클 DB를 벗어나 다른 DB로 이관 및 교체할 예정입니다.
<div style="text-align: center;">
    <img src="/files/post/2022-12-05-promotion-dbfunction-to-java/future_architecture.jpg" width="1000">
    <figcaption style="color: gray">향후 개선안 스케치</figcaption>
</div>

<hr/>

## 마무리
지금까지 약 1년 간 레거시를 뜯어서 좀 더 효율적으로 개선하는 프로젝트의 과정을 같이 살펴보았습니다.  
한 편의 글로 담으려다 보니, 보다 상세히 적을 수 없어서 아쉽네요.
이번 포스팅이 비슷한 도전과제를 안고 있는 여러 개발자분께 조금이라도 도움이 되었으면 좋겠습니다.
비록 세부적인 트러블 슈팅 과정이나 고민거리들은 제공되진 않았지만, 참고하셔서 향후 방향성 설정에 도움이 되시길 바랍니다. 