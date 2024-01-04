---
layout: post
title: '이번 프로젝트에 자동화는 처음이라'
author: 최윤석
date: 2022-10-17
tags: [RPA, QA, SDET, Python, Selenium, Appium]
---

안녕하세요. 11번가의 SQE팀에서 테스트 자동화를 개발하고 있는 최윤석입니다.

저희가 UI **테스트 자동화**를 개발하면서 선택이 필요한 시점마다 가장 중요하게 생각한 것은 유지 보수 시간을 최대한 줄이는 것이었으며, 이러한 선택을 적용한 사례를 중점으로 이야기해 보려고 합니다. 

<br/>
<figure>
  <img src="/files/post/2022-10-17-ui-test-automation/Step_01.png">
  <figcaption>Test Pyramid</figcaption>
</figure>
<br/>




# 목차
* [QA와 테스트 자동화 도구](#qa-test-automation_tool)
* [자동화 테스트 케이스의 입력 방식](#input-method-of-automation-test-case)
* [자동화 동작 및 리포트 생성 작업](#automated-actions-and-report)
* [모든 환경을 위한 하나의 자동화 TC](#one-tc-for-all)
* [마무리](#outro)
* [참고자료](#references)


# QA와 테스트 자동화 도구 <a id='qa-test-automation_tool'></a>


모든 프로젝트는 **자원 (시간, 인력, 돈)**이 부족합니다. 하지만 부족한 자원을 다양한 기법과 도구를 활용하여 제품의 목적을 이루고 안정성을 유지해야 합니다.

> 이러한 **자원** 부족 상황에서도 안정성 유지를 돕기 위해 다양한 QA 기법이 적용되고 있습니다.

- [QA 기법 소개](https://www.sten.or.kr/syllabus/index.php)
  1. [Pairwise Testing](https://www.pairwise.org/)
  2. [Risk-Based Testing Strategy](http://jidum.com/jidums/view.do?jidumId=579)
  3. [Exploratory Testing](http://jidum.com/jidums/view.do?jidumId=586)
  4. [Equivalence partitioning](https://en.wikipedia.org/wiki/Equivalence_partitioning)
  5. [Boundary Value Analysis](http://jidum.com/jidums/view.do?jidumId=571)

 

하지만 다양한 테스트 기법을 적용해 봐도 예기치 못한 상황이 추가로 발생하고 그로 인해 **자원** 부족이 심각해지는 것이 현실입니다.

이러한 **자원** 부족 현상을 조금이나마 개선하기 위해 **테스트 자동화** 도입은 충분히 좋은 고려 항목입니다.

예전 의류 공장에서 손바느질 작업으로 옷을 만들다가 재봉틀 도입으로 생산성이 올라간 것처럼 성공적으로 도입된 **테스트 자동화**도 효율적으로 제품 안정화에 기여할 수 있습니다.

하지만 재봉틀 도입만으로 완제품의 옷이 나오지는 못하는 것처럼 **테스트 자동화**도 만능은 아니란 점은 잊지 말아야 합니다.

<figure>
  <img src="/files/post/2022-10-17-ui-test-automation/Step_02.jpg">
  <figcaption>QA and Test Automation</figcaption>
</figure>
<br/>

> 테스트 자동화 도구에는 수많은 유 / 무료 도구가 있습니다. 그중 몇 가지를 소개하자면 다음과 같습니다.

- [유료 도구 소개] 
  1. [TestingWhiz](https://www.testing-whiz.com/)
  2. [lambdatest](https://www.lambdatest.com/selenium-automation)
  3. [testcomplete](https://smartbear.com/product/testcomplete/features/automated-ui-testing/)
  4. [Katalon Studio](https://katalon.com/)
  5. [basis](https://www.basistechnologies.com/products/testimony/)
  6. [UiPath](https://www.uipath.com/)
  7. [Automation Anywhere](https://www.automationanywhere.com/) 


- [무료 도구 소개] 
  1. [Selenium](http://www.seleniumhq.org/download/)
  2. [Appium](https://appium.io/)
  3. [Sikulix](http://www.sikulix.com/)
  4. [Autoit](https://www.autoitscript.com/site/)
  5. [uiautomator2](https://github.com/openatx/uiautomator2)
 
> 각각의 도구는 장단점이 있으므로 충분한 고려 후 도입이 필요합니다.


**테스트 자동화**를 도입할 때 동작 기댓값 확인 구조에 대한 충분한 고려 없이 개발하는 경우가 있습니다. 이런 방식은 개발 시간을 단축해 줄 수 있지만 장기적으로 보면 자동화 실패 발생 시 원인 파악에 추가적인 **유지 보수 시간**의 낭비를 발생시키게 됩니다. 이런 경우 자동화의 규모가 커질수록 점차 **유지 보수**가 어렵게 되며, **테스트 자동화**의 확장에 큰 장애물이 되기도 합니다.

특별히 예외적인 경우를 제외하면 **테스트 자동화**에 적용된 **테스트 케이스**는 각 단계 별로 성공과 실패에 대한 기준을 명확하게 확인하고 다음 단계로 넘어갈 수 있도록 설계하시는 것을 추천드립니다. 

이러한 방식을 도입하기 위해서는 **테스트 자동화** 도구는 테스트 케이스와 테스트 대상을 유기적으로 연결해 주는 것이 중요하며, 이것은 장기적인 관점에서 프로젝트에 **테스트 자동화**의 도입 성공 확률을 올려 주게 됩니다.


<figure>
  <img src="/files/post/2022-10-17-ui-test-automation/Step_03.png">
  <figcaption>자동화 도구 연결 구조</figcaption>
</figure>
<br/>

> 저희가 원하는 테스트 자동화 요구사항은 다음과 같습니다.

- **Android, iOS, Windows, Mac 등 OS 관계없이 동작 필요**
- **다양한 브라우저 종류 관계없이 동작 필요**
- **Mobile / Tablet 등 장비 종류 관계없이 동작 필요**
- **여러 개의 도구를 사용해야 한다면 사용된 도구들의 호환성**

무료 도구 중 하나의 도구로는 저희 요구사항을 모두 처리하기는 어려운 부분이 있었으며, 유료 도구 쪽에서 모두 가능하다고 명시되어 있더라도  해당 도구의 설명을 살펴보면  Selenium 과 Appium을 결합하여 자체 시스템을 구축한 경우가 많이 있었습니다.

유료 도구 중  Selenium 과 Appium을 사용한 경우가 많다면 이 두 무료 도구의 안정성과 호환성은 보장된다는 의미가 되었으므로, Selenium과 Appium을 기본 자동화 도구로 선택하게 되었습니다.


# 자동화 테스트 케이스의 입력방식 <a id='input-method-of-automation-test-case'></a>

저희가 선택한 자동화 도구(Appium, Selenium)는 기본적으로 코딩을 통해 **테스트 케이스(이하 TC 로 표기)**를 작성해야 합니다. 

사람마다 QA 실력에 따른 다양한 **TC**가 나올 가능성이 높으며, 각각의 코딩 실력에 따라 예외처리등 이슈로 품질도 달라질 수 있습니다. 이것은 흡사 자유도 높은 점토로 우주선을 만든 것처럼 다양한 방식의 **TC** 탄생 이유가 됩니다.

<figure>
  <img src="/files/post/2022-10-17-ui-test-automation/Step_04.png">
  <figcaption>TC의 자유도</figcaption>
</figure>
<br/>

이렇게 생성된 다양한 방식의 **TC**는 이슈 발생 시 코드 해석을 위한 **유지 보수 시간**을 추가로 소모시키는 원인이 되며, 제품에 대한 방대한 지식을 가진 QA가 코딩 언어의 벽을 넘지 못해서 테스트 자동화 **TC**를 추가하기 어려워질 수도 있습니다.

또한 이슈 발생 시 해당 **TC**를 직접 작업한 사람이 아니면 해당 **TC**를 분석하고 빠르게 수정하는 것이 어려워 질 수 있습니다. 

만약 어렵게 개발 적용한 테스트 자동화가 이러한 이유로 정체되기 시작한다면 몇 가지 케이스 만 동작하는 수준에서 개발이 멈추거나 특정 변화의 시점을 맞이하여 더 이상 쓰이지 못해 폐기될 수도 있습니다. 이러한 요인은 테스트 자동화가 효율적으로 사용되지 못하고 낭비되는 이유가 되기도 합니다.

> 이러한 문제를 줄이고자 자동화 **TC** 입력 방식 설정을 위해 고려 한 주요 항목은 다음과 같습니다.

- **모든 QA가 자동화를 위한 코딩을 배워야 하는지 여부**
- **만약 QA가 자동화 관련 교육(코딩)을 배운다면 안정적인 TC가 나올 때까지의 시간**
- **TC가 늘어나게 돼도 유지 보수 시간이 비약적으로 늘어나지 않는 방식**


저희 11번가는 코딩의 벽을 허물기 위해  **TC** 작성 시 세부적인 설정이나 옵션 같은 (String, int, sleep, Try ~) 고민은 자동화 파트에서 처리하고 기능 QA는 보다 쉽게 케이스를 작성할 수 있도록 각각의 **TC**를 레고 조각처럼 모듈화하였습니다. 

<figure>
  <img src="/files/post/2022-10-17-ui-test-automation/Step_05.png">
  <figcaption>레고 방식 코딩</figcaption>
</figure>
<br/>

또한 이러한 **TC** 작성 방법에 대한 정보를 조립 설명서와 같이 Excel로 가이드를 제공하여 누구나 쉽게 유지 보수 및 **TC** 추가가 가능하도록 하고 있습니다.

이러한 방식을 통해 작업한 **TC**는 Jenkins 서버나 개인 PC에서 쉽게 연동하여 동작할 수 있도록 지원하고 있습니다. 

<figure>
  <img src="/files/post/2022-10-17-ui-test-automation/Step_06.png">
  <figcaption>TC 가이드</figcaption>
</figure>
<br/>

또한 해당 방식을 활용한다면 별도의 사이드 프로젝트용 도구를 빠르게 제작할 수도 있습니다.

저희는 테스트 자동화용 ID를 수 백 개 운영하고 있습니다. 하지만 장기적으로 로그인을 하지 않는 ID는 휴면으로 분류되어 실제 테스트 과정에서 잠겨있는 계정으로 인해 테스트 진행이 불가능한 상황이 발생할 수도 있습니다. 만약 새벽 자동화 점검 시 이슈가 발생했는데 잠긴 계정을 풀 수 있는 사람이 연락이 안 된다면? 오히려 자동화가 프로젝트 진행에 발목을 잡는 경우를 발생시킬 수도 있습니다.

이러한 이슈를 막기 위해 주기적으로 로그인을 해줘야 하지만 사람이 하기엔 인력과 시간 낭비가 큰 동작이므로 몇 가지 모듈을 조립하여 로그인 전용 도구를 제작하였습니다.

이 로그인 용 도구는 로그인 동작과 함께 비정상 상태의 계정이 존재 파악하는 용도로 사용되고 있으며, Jenkins를 통해 주기적으로 동작하고 있습니다.

# 자동화 동작 및 리포트 생성 작업 <a id='automated-actions-and-report'></a>


저희 11 번가에서는 현재 1300 개가 넘는 자동화용 **TC**가 Jenkins에서 60개 Thread로 분리되어 동시 동작 가능하도록 설정되어 있습니다.

<figure>
  <img src="/files/post/2022-10-17-ui-test-automation/Step_07.png">
  <figcaption>Jenkins</figcaption>
</figure>
<br/>

각 Thread의 동작 시간은 최대 2시간이 넘지 않도록 설정되어 있으며, 이것은 Thread 동작 시간이 3시간이 넘어갈 경우 **유지 보수**를 위해 하루에 동작 가능한 횟수의 제약이 커지기 때문에 내린 결정입니다.

각각의 Thread는 매시간, 하루 1번 동작 등 상황에 따라 분리하여 스케줄을 설정 후 운영 중에 있으며, 주요한 케이스들은 출근 시작 전 동작하여 출근 후 결과를 확인할 수 있도록 설정되어 있습니다.

이렇게 Jenkins를 통해 동작 한 자동화 점검이 늘 정상적으로 진행된다면 좋겠지만 이슈 발생 시 최대한 원인 분석을 빠르게 진행할 수 있도록 자체 제작한 리포트를 제공하고 있습니다. (Slack 연결)

<figure>
  <img src="/files/post/2022-10-17-ui-test-automation/Step_08.png">
  <figcaption>Jenkins</figcaption>
</figure>
<br/>

이슈 발생 시 제공하는 리포트의 경우 정상적인 화면과 이슈 화면을 비교하여 볼 수 있도록 하여 시작적인 판단 자료를 제공하며, 추가적으로 문제가 발생한 개체의 데이터(세부 이미지, xpath 위치/값)도 함께 제공하고 있습니다.

만약 새로운 **TC**가 추가된다면 동작 성공 시 데이터를 자동으로 누적하도록 설계되어 있으며, 캡처 화면 등 기존 데이터를 교체하는 작업이 필요하다면 기존 데이터를 백업해 두고 해당 자동화 Thread를 동작하면 다시 데이터를 자동으로 누적해 줍니다.

<figure>
  <img src="/files/post/2022-10-17-ui-test-automation/Step_09.jpg">
  <figcaption>Jenkins</figcaption>
</figure>
<br/>

 
# 모든 환경을 위한 하나의 자동화 TC <a id='one-tc-for-all'></a>


테스트가 필요한 환경은 무엇이 있을까요?

OS로 구분한다면 Windows, Mac, Android, iOS 가 있으며, 이중 Windows는 10과 11이 있습니다. Windows 환경에서 구동하는 주요 브라우저는 Chrome, Edge, Whale, Firefox, (IE) 등이 있고, 여기서 다시 MW와 PC로 나뉩니다.

<figure>
  <img src="/files/post/2022-10-17-ui-test-automation/Step_10.jpg">
  <figcaption>Windows 환경</figcaption>
</figure>
<br/>

이러한 기준으로 Windows에서만 점검해야 하는 주요 환경은 (2 * 4 * 2 =) 16 개 입니다. 

만약 3,000 개의 점검용 **TC**를 가지고 있다면 Windows에서만 (3000 * 16 =) 48,000 개의 체크를 진행 해야 합니다. 

Mac 에서도 OS와 브라우저가 있고, Mobile / Tablet 환경인 Andoird와 iOS 에서는 다양한 OS 버전과 셀 수 없이 많은 단말기가 교차 되어 있으며, 추가적으로 **TC** 또한 고정이 아닌 지속적으로 업데이트가 이루어 지고 있습니다.

만약 MW, PC, Android, iOS 각각 다른 자동화 TC로 작업이 진행되다면, 이것은 TC 추가 작성 시간으로 끝나는 게 아니라 이슈 발생 시 유지 보수 시간 또한 크게 늘어나게 됩니다.

이러한 문제를 방지하기 위해 현재 저희가 개발한 테스트 자동화 시스템은 One TC for All 을 적용한 구조로 개발되어 있으며, 이것은 하나의 **TC**가 모든 환경에서 동작 가능한 구조를 가지고 있습니다.


<figure>
  <img src="/files/post/2022-10-17-ui-test-automation/Step_11.png">
  <figcaption>One TC for All</figcaption>
</figure>
<br/>

예를 들어 Windows > MW 점검을 목적으로 만든 **TC**가 OS, 브라우저 종류나 Android / iOS 기기와 상관없이 돌아가는 것을 목표로 지속적인 업데이트를 진행하고 있습니다. 이 방식을 통한 **테스트 자동화** 구현이 완료된다면 한번 작업 된 TC는 모든 환경에 대한 점검도 (이론상) 가능해집니다.


# 마무리 <a id='outro'></a>
위와 같이 **유지 보수 시간**을 중점으로 개발하게 되면 개발 시간은 좀 더 오래 걸리게 됩니다.

하지만 많은 수의 **테스트 자동화**가 **유지 보수**가 어려워서 폐기되는 경우가 많으므로 **테스트 자동화**의 성공적인 도입을 원한다면 하나의 좋은 선택지가 될 수 있습니다.

UI **테스트 자동화** 관련 내용인데 간단한 동작 영상도 없는 건 예의가 아닌 듯하여 4배속의 동작 화면을 준비했습니다.

> 동작 환경 :
- **Windows 11**
  - **Selenium**
    - **Chrome**
      - **MW**
      - **PC**


동작을 진행하는 아래 **TC** 는 [ 로그인 > 상품 이동 및 구매 후 > 금액 확인과 구매취소를 진행 ]하는 가장 기본적인 [ 케이스 ] 중 하나 입니다. 

UI 구조가 전혀 다른 2개의 항목이 하나의 **TC**로 동작 하는 예제 입니다.

<figure>
  <img src="/files/post/2022-10-17-ui-test-automation/Step_12.png">
  <figcaption>Test Case</figcaption>
</figure>
<br/>

> 자동화 동작 화면 
<figure>
  <img src="/files/post/2022-10-17-ui-test-automation/Step_13.gif">
  <figcaption>MW / PC 자동화 동작</figcaption>
</figure>
<br/>


지금도 SQE팀의 **테스트 자동화**는 지속적으로 업데이트 중에 있습니다. 앞으로도 가야 할 길이 많이 남아 있지만 점차 개선해 나갈 예정입니다.

긴 글 읽어주셔서 감사합니다.




 
# 참고자료 <a id='references'></a>

- **[https://www.sten.or.kr/syllabus](https://www.sten.or.kr/syllabus/index.php)**
- **[https://www.softwaretestinghelp.com/](https://www.softwaretestinghelp.com/)**
- **[https://www.toolsqa.com/software-testing](https://www.toolsqa.com/software-testing/ISTQB/experience-based-testing-technique)**
- **[https://en.wikipedia.org](https://en.wikipedia.org/wiki/Equivalence_partitioning)**
- **[http://jidum.com](http://jidum.com/jidums/view.do?jidumId=552)**




> **Contributors**: 박용호, 전민석, 최윤석
