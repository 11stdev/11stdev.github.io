---
layout: post
title: 'AWS 보안 모니터링 환경 구성 하기-1부'
author: 이상옥
date: 2021-10-01 16:00
tags: [aws,security,siem]
---

안녕하세요!!

11번가 정보보안팀에서 보안관제 및 CERT 업무를 맡고 있는 이상옥 입니다.<br/>
11번가 AWS 클라우드 서비스 환경으로 전환되고 있으며, 이에 맞춰 보안 모니터링 환경을 구성하고 개선하는 작업을 하고있습니다.<br/>
구성 작업을 하면서 부족하지만 AWS 보안 모니터링 환경을 구성했던 경험 내용을 공유드리고자 합니다.

---

### 목차
- [1. 보안 모니터링 구성도](#aws-security-monitoring)<br/>
- [2. AWS 보안 데이터](#aws-security-data)<br/>
- [3. 보안 데이터 수집](#aws-security-input)<br/>
  - [3.1 로그 수집 솔루션 검토](#aws-security-input-list)
  - [3.2 Splunk 구성하기](#aws-security-input-splunk)
  - [3.3 ElasticSearch 구성하기](#aws-security-input-elasticsearch)
- [4. Splunk Custom Search Command 만들기](#aws-security-customcommand-splunk)<br/>
- [5. 마치며...](#aws-security-end)

---

## 1. 보안 모니터링 구성도<a id='aws-security-monitoring'></a>

현재 구성하고 보안 모니터링 진행 중인 AWS 보안 모니터링 구성도 입니다.<br/>
클라우드 환경에서 보안 데이터 수집을 위한 SIEM 환경 구성 및 분석에 필요한 보안 서비스 데이터를 수집 가능하도록 하고, 보안 데이터 속에서 보안 위협을 인지하고 분석할 수 있는 환경이 필요하여 보안 모니터링 환경을 구성하게 되었습니다.
또한 지속적인 보안 데이터 수집, 상관 분석 처리 및 대응 프로세스 개선을 통해 한층더 고도화된 보안 모니터링 환경을 만들어 가려고 노력하고 있습니다.
구성을 진행하면서 아직도 개선하고 보완할 점이 많지만 현재까지 구성했던 보안 모니터링 환경은 어떻게 만들어갔는지 알아 보도록 하겠습니다.

- 보안 모니터링 구성도

![](/files/post/2021-10-01-aws-security1/security_monitoring.png){: width="1200"}  

  
## 2. AWS 보안 데이터<a id='aws-security-data'></a>

우선 보안 모니터링 환경 구성을 위해 AWS 클라우드 서비스로 제공하는 보안 서비스 또는 보안 분석에 필요한 데이터가 무엇이 있을지 확인하는 작업이 필요했습니다.
온프라미스 기준으로 FW, IPS, WAF 등의 기본 보안 장비와 보안 데이터가 있는 것과 같이 클라우드 환경에서도 이에 맞는 보안 서비스와 데이터를 확인 할 수 있었습니다.<br/>
아래 표와 같이 로그 수집 솔루션 기준으로 보안 모니터링을 위해서 수집이 필요한 데이터를 검토하게 되었습니다.

- 보안 데이터 검토

|        솔루션         |        수집 대상         |                         데이터 유형                          |                             내용                                                         |
| :-----------------------------: | :----------------------: | :-------------------------------------------------------: | :---------------------------------------------------------------------------- |
|    Splunk  |        GuardDuty         |                      GuardDuty findings                      | AWS 클라우드 환경 내에서 발생하는 보안 위협 모니터링 보안 서비스 |
|                                 |         AWS WAF          |            Web Access Logs<br/>(Detect/Block Filter)             | AWS 에서 제공하는 웹방화벽(Web Application Firewall, WAF) 보안 서비스<br/>- 탐지 및 차단 로그 필터링 수집 |
|                                 |       Security Hub       | Security Hub findings<br/>(CIS, Best Practice, PCI DSS, Prolwer) | AWS Security Hub는 여러 보안 플랫폼 연동하여 중앙에서 관리 할 수 있도록 하는 서비스<br/>- 클라우드 환경에서의 컴플라이언스(BestPractice, CIS, PCI-DSS)기준을 점검 기능 활용<br/>- OpenSource인 Prolwer를 추가 연동하여 컴플라이언스 점검 강화|
| ElasticSearch |      AWS CloudTrail      |                    Management<br/>S3 Bucket(Object)                    | AWS 클라우드 환경에서 발생하는 모든 행위에 대한 요청,응답 데이터 수집<br/>- Object 수준 설정을 통한 S3 데이터 모니터링 강화 |
|                                 |        Amazon VPC        |                        VPC Flow Logs<br/>(Version 5)                       | AWS 클라우드 환경에서 발생하는 네트워크 통신에 대한 연결 정보 제공<br/>- Version 5 수준 로깅을 통한 Flags 정보 및 방향성 등 추가 분석 데이터 확보 |
|                                 | Amazon Route 53 Resolver |                        DNS query log                         |     AWS 클라우드 환경에서 발생하는 DNS 요청 데이터 수집      |
|                                 |         AWS WAF          |                       Web Access Logs                        | AWS 에서 제공하는 웹방화벽(Web Application Firewall) 보안 서비스<br/>- 모든 로그 수집 |
|                                 |   Firebeat<br/>Winlogbeat    |       Linux: /var/log/message, audit, secure<br/>Windows: WinEventLog, Sysmon        |      엔드포인트(서버)에 대한 Security 및 Audit 로그 수집       |
|              기타               |        AWS Shield<br/>Standard       |                제공 로그 없음               | AWS 에서 기본으로 제공하는 DDOS 대응 서비스<br/>- AWS 환경 내 L4 수준의 대역폭 공격 대응<br/>- AWS Shield Advanced 서비스 적용 검토 진행 |
|                                 |        AWS Macie         |                    Amazon Macie findings                     |      개인 정보 모니터링 보안 서비스<br/>- 연동 검토 진행 중      |
|                                 |      AWS Inspector       |                  Amazon Inspector findings                   |       취약점 점검 보안 서비스<br/>- 별도 보안 솔루션 운영        |
|                                 |        자체 개발         |                    AWS 리소스 자산 정보                    |             수집 모듈 개발을 통한 AWS 자산 정보 수집              |


## 3. 보안 데이터 수집<a id='aws-security-input'></a>
### 3.1 로그 수집 솔루션 검토<a id='aws-security-input-list'></a>
이전에 검토 했던 보안 데이터를 수집하고 분석을 하기 위해서는 SIEM 기반 로그 수집 및 분석 가능한 환경이 필요했습니다.<br/>
구성과 운영이 용이한 SIEM을 선정하고,이를 기준으로 장단점 비교 및 활용 방안을 검토하게 되었습니다.

|      구분       |                                 장점                                    |                     단점                        |                                                  활용 방안                                                    |
|:-------------:    |:-------------------------------------------------------------------:  |:-------------------------------------------:  |:-----------------------------------------------------------------------------------------------------------:  |
|     Splunk        | SPL문을 통한 유연한 데이터 분석 기능<br>다양한 App을 통한 연동 지원   | 상용 라이센스<br>대용량 처리 시간 다소 느림    | 탐지 기반 이벤트 데이터 수집<br>상관 분석을 통한 보안 탐지 이벤트 생성 및 관리<br>실시간 모니터링 환경 구성     |
| ElasticSearch     |             오픈소스(무료)<br>대용량 처리 시간 다소 빠름               |               데이터 분석 기능 활용 제한적               |                         대용량의 RawData 수집<br>대용량 검색 및 분석 시스템으로 활용                           |

최소한 비용 및 장단점, 활용방안 등을 고려하여 Splunk는 보안 모니터링 알람 및 이벤트 처리를 위한 SIEM 으로 활용하고, ElasticSearch는 대용량 로그 수집 및 검색, 분석에 활용하는 방향으로 구성하기로 하였습니다.

### 3.2 Splunk 구성하기<a id='aws-security-input-splunk'></a>

보안 모니터링 알람 및 탐지 이벤트 처리를 위한 Splunk 환경을 구축하도록 하겠습니다.<br/>
배포 방식은 Standalone, Basic, Multi-Instance 등이 존재하며, 분산 환경에서 Clustering 구성을 통해 검색 과 저장 성능을 크게 향상 될 수 있도록 Multi-Instance 배포 방식으로 구성하였습니다.

- Splunk 구성도

![](/files/post/2021-10-01-aws-security1/splunk_infra.png){: width="800"} 

 클러스터 환경을 위해 Search Head, Indexer, Forwarder, SHC Deployer, Cluster Master(Deployment)로 구성하였으며, 각 역할에 대해서 알아보도록 하겠습니다.(라이센스는 On-Promise 연동)

|        구분        |                             역할                             |
| :----------------: | :----------------------------------------------------------: |
|    Search Head     | SPL 문을 통한 Indexer 데이터 조회 및 분석<br>Custom Search Command  활용한 ElasticSearch 데이터 조회 및 분석 |
|      Indexer       | Forwarder 에서 데이터를 전달 받아 저장하고, Search Head로 부터 SPL 데이터 조회 요청을 처리 |
|     Forwarder      | UDP/TCP input, S3, Kinesis, Script 등의 모듈을 통한 데이터를 수집된 데이터는 Indexer 로 전달 |
|    SHC Deployer    | Search Head 노드 간 클러스링 구성을 통해 App 관리 및 배포   |
| Cluster Master<br/>(Deployment) | Indexer Cluster 간 Indexer 추가 및 분배 관리<br/>Splunk 클러스터 상태 모니터링 |

클러스터 구성 요소들의 구분과 역할을 정의 하였으니, Splunk 클러스터 구성을 해보도록 하겠습니다.

- 방화벽 오픈<br/>
클러스터 구성으로 노드간 통신을 위해서 아래 구성도와 같이 방화벽 오픈이 필요합니다.<br/>
    - Splunk 노드 간 통신 포트 구성도

    ![](/files/post/2021-10-01-aws-security1/splunk_port.png){: width="800"} 

- 설치 작업<br/>
Splunk는 Linux, Windows, MacOS 등의 OS 설치가 가능하며, RPM 설치파일 기준으로 진행 하겠습니다.
```
Splunk RPM 설치 파일 다운로드
rpm -ivh splunk-8.2.2.1-ae6821b7c64b-linux-2.6-x86_64.rpm
$SPLUNK_HOME/bin/splunk start --accept-license   //splunk 실행
admin 계정 생성(admin 계정 및 패스워드 입력)
$SPLUNK_HOME/bin/splunk enable boot-start // 부팅 자동 재시작
```
- Splunk 로그인 접속 화면<br/>
![](/files/post/2021-10-01-aws-security1/splunk_login.png){: width="800"} 


- 마스터(Cluster Master) 노드 설정
    1. 설정 → 인덱스 클러스터링 → 인덱서 클러스터링 활성화 → 마스터 노드 → 마스터 노드 설정 → 재기동<br/>
    \- 보안 키는 노드 별 클러스터 통신 설정을 위해 사용<br/>
    ![](/files/post/2021-10-01-aws-security1/splunk_master1.png){: width="500"} 

    2. 설정 → 데이터 전달 및 수신 → 데이터 전달 → 새 전달 호스트 적용(모든 Indexer 적용)<br/>
    ![](/files/post/2021-10-01-aws-security1/splunk_master2.png){: width="500"} 


- 인덱서(Indexer) 노드 설정
    1. 설정 → 인덱스 클러스터링 → 인덱서 클러스터링 활성화 → 피어 노드 → 피어 노드 설정 → 재기동<br/>
    \- 마스터 URI, 피어 복제 포트(8080), 보안 키(마스터 노드 설정) 입력<br/>
    ![](/files/post/2021-10-01-aws-security1/splunk_indexer1.png){: width="500"}

    2. 설정 → 데이터 전달 및 수신 → 수신 설정 → 새 수신 포트 추가(기본 9997)<br/>
    \- Indexer 노드에 데이터를 모두 수집하기 위해 수신 포트를 구성<br/>
    ![](/files/post/2021-10-01-aws-security1/splunk_indexer2.png){: width="500"}

- 검색 헤드(Search Head) 배포 노드 설정
    1. 설정 → 인덱스 클러스터링 → 인덱서 클러스터링 활성화 → 검색 헤드 노드 → 검색 헤드 설정 → 재기동<br/>
    \- 마스터 URI, 보안 키(마스터 노드 설정) 입력<br/>
    ![](/files/post/2021-10-01-aws-security1/splunk_head1.png){: width="500"}

    2. 설정 → 데이터 전달 및 수신 → 데이터 전달 → 새 전달 호스트 적용(모든 Indexer 적용)<br/>
    ![](/files/post/2021-10-01-aws-security1/splunk_head2.png){: width="500"}

- 검색 헤드 클러스터(Search Head Cluster) 구성

    1. Search Cluster 배포 서버 설정<br/>
    \- $SPLUNK_HOME/bin 에서 splunk edit shcluster-config -adhoc_searchhead true<br/>
    \- $SPLUNK_HOME/etc/system/local 경로에 server.conf 생성하여 아래와 같은 내용 추가

        ```
        [shclustering]
        pass4Symmkey = 서치 헤드 클러스터 암호키 입력
        shcluster_label = 서치 헤드 클러스터 이름 입력
        ```

    2. Search Cluster 멤버 설정<br/>
    \- 각각 서치 헤드 노드에서 아래 명령어 실행<br/>
    \- $SPLUNK_HOME/bin 경로에서 아래 내용 입력<br/>
    \- 8089 : 관리포트, 8080: 복제포트

        ```
        ./splunk init shcluster-config -auth admin:PW -mgmt_uri https://멤버서버IP(자신):8089 -replication_port 8080 -replication_factor 3 -conf_deploy_fetch_url https://SHC서버IP:8089 -secret SHC패스워드 -shcluster_label sh_cluster
        ```
    
    3. Search Cluster 캡틴 설정<br/>
    \- 캡틴 서버로 설정할 서치 헤드 노드에서 명령어 실행<br/>
    \- SPLUNK_HOME/bin 경로에서 아래 내용 입력
        ```
        ./splunk bootstrap shcluster-captain -servers_list "https://맴버1IP:Port, https://멤버2IP:Port, https://....." -auth admin:PW
        ```


- 서치 헤드 클러스터 구성 화면

![](/files/post/2021-10-01-aws-security1/splunk_shc2.png){: width="500"} 



### 3.3 ElasticSearch 구성하기<a id='aws-security-input-elasticsearch'></a>

대용량 기반의 RawData 수집 및 검색, 분석을 위한 ElasticSearch 플랫폼 환경을 구축하도록 하겠습니다.<br/>
ElasticSearch는 AWS에서 Amazon OpenSearch Service(구 Amazon ElasticSearch Service)이름의 서비스 형태로 제공하고 있어, 별도 설정 없이 클러스터 구현이 가능하여 어렵지 않게 구성이 가능했습니다.

- ElasticSearch 구성도

![](/files/post/2021-10-01-aws-security1/elasticsearch_infra.png)

클러스터 환경을 위해 Master, Hot Node, Warm Node, Cold, Logstash, Kibana(자동 구성)을 활용하여 구성하였으며, 각 역할에 대해서 알아보도록 하겠습니다.

|              구분              |                              역할                             |
| :----------------------------: | :---------------------------------------------------------- |
|            Logstash            | UDP/TCP input, S3, Kinesis 등의 모듈을 통한 데이터 수집<br/>- 데이터 Parsing 작업 및 강화(IP 국가코드 추가 등) <br/>- Hot Node 로 전달 |
|             Master             | ElasticSearch Cluster 상태 정보를 관리<br/>- Split Brain 현상(마스터 간 네트워크 단절 발생 시 클러스터 분리 이슈)을 고려하여 구성<br/>- 홀수 3개 노드 및 최소한 마스터 노드 2개 Active 상태 체크 구성|
|            Hot Node            | 실시간 및 최근 데이터 저장 및 검색을 위한 노드<br/>- 데이터 쓰기, 읽기 등의 작업을 많이 하여 고사양의 스펙 요구(CPU, MEM, DISK 성능[IOPS] 확보 필요) |
|           Warm Node            | 일정 기간이 지난 데이터를 저장 및 검색하기 위한 노드<br/>- Hot Node 비해 리소스가 적게 들기 때문에 최소한 하드웨어 스펙으로 구성 |
|        Cold<br>(Amazon S3)         | 오랜 기간이 지난 데이터를 저장하기 위한 노드<br/>- Amazon S3 버킷에 데이터가 저장되어, 최소한의 비용으로 수집<br/>- Warm Node으로 데이터 전달하여 검색 사용 |
| Kibana<br/>(자동 구성) | 데이터를 검색 및 분석, 알람 등의 보안 운영으로 활용<br/>- 오픈소스 기반으로 Security, Alerting, Report 등의 상용 수준의 기능 제공<br/>- OpenSearch(구 OpenDistro For ElasticSearch) 제품명으로 온프라미스 구성 가능 |

- ElasticSearch(Amazon OpenSearch Service) 생성하기
    1.  배포 유형 선택<br/>
    \- 배포유형 : 프로덕션, 개발 및 테스트 등 사용 용도에 따른 배포 유형<br/>
    \- 버전 : 서비스 버전을 선택(최근 라이센스 이슈로 OpenSearch 라는 이름으로 Fork 된 버전도 확인)<br/>
        ![](/files/post/2021-10-01-aws-security1/elasticsearch_setting1.png){: width="800"} 

    2. 도메인 구성<br/>
    \- 도메인 구성 : 도메인 이름을 설정<br/>
    \- 사용자 지정 엔트 포인트 : 기본적으로는 엔드포인트 도메인을 제공하지만, 쉽게 참조 가능하도록 별도 생성<br/>
    \- 자동 튜닝 : 클러스터 성능을 모니터링 하고 권장 및 자동으로 변경<br/>
    \- 데이터 노드 : 데이터 사이즈 및 수집 기간, 복제본 등 고려하여 Hot Node 수를 지정<br/>
    \- 데이터 노드 스토리지 : Hot Node 스토리지 구성<br/>
    \- 전용 마스터 노드: 전용 마스터 노드를 설정(3개, 5개 노드 구성 가능)<br/>
    \- UltraWarm 데이터 노드 : Warm Node 구성(2가지 하드웨어 스펙 노드 지원)<br/>
    \- 콜드 스토리지 : S3 Cold 를 사용할지 여부를 설정<br/>
        ![](/files/post/2021-10-01-aws-security1/elasticsearch_setting2.png){: width="800"} 

    3. 엑세스 및 보안 구성<br/>
    \- 네트워크 구성 : VPC 액세스, 퍼블릭 엑세스 설정을 통해 접근 구성<br/>
    \- 세분화된 엑세스 제어 : IAM, 마스터 계정 생성을 통해 세부적인 접근 방식 설정<br/>
    \- SAML 인증 : AD 연동을 위한 SAML 인증 설정<br/>
    \- Amazon Cognito 인증 : Amazon Cognito 서비스를 통한 인증 설정<br/>
    \- 엑세스 정책 : 도메인 접근 시 엑세스 정책 생성<br/>
    \- 암호화 : 도메인(HTTPS), 노드 간 암호화(TLS) 등 설정<br/>
    \- 고급 클러스터 파라미터 : Heap 사이즈, 쿼리 요청 사이즈 등 설정<br/>
        ![](/files/post/2021-10-01-aws-security1/elasticsearch_setting3.png){: width="800"} 

- ElasticSearch 로그인 접속 화면

![](/files/post/2021-10-01-aws-security1/elasticsearch_login.png)

이로써 보안 모니터링에 필요한 데이터 수집 및 활용이 가능한 기본적인 SIEM 환경이 구성 되었습니다.

![Splunk 로그 수집 화면](/files/post/2021-10-01-aws-security1/splunk_log.png)![ElasticSearch 로그 수집 화면](/files/post/2021-10-01-aws-security1/elasticsearch_log.png)



## 4. Splunk Custom Search Command 만들기<a id='aws-security-customcommand-splunk'></a>

Splunk에서 ElasticSearch 데이터를 가져와 활용할 수 있도록 Splunk Custom Search Command를 만들어 구성해보도록 하겠습니다.

- App 생성
    1. Splunk → 앱 관리 → 앱 만들기 → 앱 이름 설정 및 템플릿 barebones 로 생성<br/>
    \- $SPLUNK_HOME/etc/app 하단에 위에서 만든 빈 App 생성

- 생성한 App 내 파일 생성
    1. local/commands.conf
    \- Splunk 에서 사용 가능한 Command 명령어 생성
        ```
        [elasticsearch] // "elasticsearch" Custom Search Command 생성
        disabled = 0 // 활성화
        chunked=true  // Custom Search Command v2 protocol 버전 사용
        filename=elasticsearch.py   // App 내 bin/elasticsearch.py 실행
        ```

    2. (옵션) local/searchbnf.conf
    \- Command 명령어 사용 방법 내용 생성
        ```
        [elasticsearch-command]
        syntax = | elasticsearch index=<인덱스> query=<쿼리조건> size=<결과수> start_time=시작시간 end_time=종료시간(시간 미지정 시 Splunk 시간 적용)
        shortdesc = Elasticsearch 로그 검색
        usage = public
        example1 = | elasticsearchindex="*cloudtrail*" query="*" size=10000 start_time=now-30m end_time=now
        example2 = | elasticsearchindex="*cloudtrail*" query="*" size=10000 (Splunk시간 적용)
        ```

    3. bin/elasticsearch.py
    \- ElasticSearch API 통해서 요청한 데이터를 가져 올 수 있도록 Python 코드 작성
        ```
        import os,sys
        import json
        import requests
        from splunklib.searchcommands import dispatch, GeneratingCommand, Configuration, Option, validators
        from datetime import datetime, timezone
        from time import time
        from urllib import parse
         
        @Configuration()
        class GenerateELKCommand(GeneratingCommand):
         
            index = Option(require=True)
            query = Option(require=True)
            size = Option(require=True)
            start_time = Option(require=False)
            end_time = Option(require=False)
         
            def generate(self):
         
                if (self.start_time) and (self.end_time):
                    url='ElasticSearch_URL'+ self.index + '/_search?q=' + str(self.query) + ' AND @timestamp:[' + str(self.start_time) + ' TO ' + str(self.end_time) + ']&sort=@timestamp:desc&size=' + str(self.size)
         
                else:
                    latestTime=self.search_results_info.search_lt
                    earliestTime=self.search_results_info.search_et
         
                    search_lt = datetime.utcfromtimestamp(int(latestTime)).isoformat()
                    search_et = datetime.utcfromtimestamp(int(earliestTime)).isoformat()
         
                    url='ElasticSearch_URL'+ self.index + '/_search?q=' + str(self.query) + ' AND @timestamp:[' + str(search_et) + ' TO ' + str(search_lt) + ']&sort=@timestamp:desc&size=' + str(self.size)
           
                headers = {
                    'Authorization': 'ElasticSearch_Authorization',
                    'Content-type': 'application/json'
                    }
         
                response = requests.get(url,headers=headers)
                r = json.loads(response.text)
         
                if response.status_code == 200:
                    thing = r['hits']['hits']
                    results = []
         
                    for row in thing:
                        tmp = {}
                        tmp['_raw'] = json.dumps(row['_source'])
                        tmp['sourcetype'] = "elasticsearch"
                        tmp['index'] = row['_index']
                        tmp['_time'] = int(datetime.strptime(row['_source']['@timestamp'], "%Y-%m-%dT%H:%M:%S.%fZ").strftime("%s")) + 32400
         
                        for key in row['_source'].keys():
                            tmp[key] = row['_source'][key]
                        results.append(tmp)
         
                    for row in results:
                        yield row
         
         
        if __name__ == "__main__":
            dispatch(GenerateELKCommand, sys.argv, sys.stdin, sys.stdout, __name__)
        ```

    4. App 권한 설정
    Splunk → 앱 관리 → 생성 했던 앱 권한 → 모든 앱(시스템) 설정<br/>
    \- 다른 앱 에서 생성한 Splunk Custom Search Command 사용하기 위한 권한 설정 

    ![](/files/post/2021-10-01-aws-security1/splunk_custom1.png){: width="300"} 

별도 생성한 Splunk Custom Search Command를 사용하여 ElasticSearch에서 수집된 데이터도 SPL 쿼리를 활용하여 데이터 검색 및 이벤트 알람, 상관 분석 등으로 활용 가능하도록 구성하였습니다.

- Splunk Custom Search Command를 사용하여 ElasticSearch 데이터 분석 화면

    ![](/files/post/2021-10-01-aws-security1/splunk_custom2.png)


## 5. 마치며...<a id='aws-security-end'></a>

AWS 클라우드 환경의 보안 모니터링 구성 했던 경험에서 전체 구성도 및 보안 데이터 수집을 위한 기본적인 SIEM 구성에 대해서 간단히 정리해 보았습니다.
2부 내용에서는 수집하고 있는 보안 데이터 수집 및 유형, 분석 방법, 자산 모니터링 환경 구성 등의 내용을 공유드리고자 합니다.<br/>
지금까지의 내용이 AWS 보안 모니터링 환경을 구성하는데 조금이라도 도움이 되셨길 바라며, "AWS 보안 모니터링 환경 구성 하기-1부" 내용은 마치겠습니다.<br/>
감사합니다.

