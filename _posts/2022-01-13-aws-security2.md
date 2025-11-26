---
layout: post
title: 'AWS 보안 모니터링 환경 구성 하기-2부'
author: 이상옥
date: 2021-11-01 13:00
tags: [aws,security]
---

안녕하세요!!
11번가 정보보안팀 이상옥 입니다.<br/>
"AWS 보안 모니터링 환경 구성 시작하기-1부"에서 AWS 환경에서의 사용할 수 있도록 SIEM 솔루션을 구성 하였습니다.
AWS 환경에서 보안 서비스를 직접 구성하고, 데이터를 수집 및 상관 분석, 알람을 통해 지속적으로 모니터링 및 대응 조치를 진행하고 있습니다.<br/>
아직 고도화 해야할 부분도 많지만, 현재까지 11번가에서 구성하고 있는 AWS 보안 모니터링 환경에 대해서 알아보도록 하겠습니다.<br/>

### 목차

- [1. 보안 로그 수집 구성도](#map)<br/>
- [2. 보안 서비스](#service)<br/>
  - [2.1 Compliance Check](#com0)<br/>
    - [- 데이터 수집 구성](#com1)<br/>
	- [- SecurityHub(보안표준)](#com2)<br/>
	- [- Prowler](#com3)<br/>
   - [2.2 Threat Detect](#thr0)<br/>
	 - [- 데이터 수집 구성](#thr1)<br/>
	 - [- GuardDuty](#thr2)<br/>
	 - [- AccessAnalyzer](#thr3)<br/>
   - [2.3 Network Logs](#net0)<br/>
	 - [- 데이터 수집 구성](#net1)<br/>
	 - [- VPC Flow(Version 5)](#net2)<br/>
	 - [- DNS Resolver](#net3)<br/>
- [3. 작성을 마치며...](#end)<br/>

# 1. 보안 로그 수집 구성도<a id='map'></a>

보안 위협 및 취약점을 모니터링을 하기 위한 AWS 에서 제공하는 보안 서비스가 있으며, 분석에 필요한 데이터를 제공하고 있습니다.<br/> 
11번가 에서는 AWS 제공하는 보안 서비스를 최대한 활용하고 있으며, 최소한으로 필요한 보안 로그를 수집하여 모니터링 및 분석 활동에 활용하고 있습니다. 보안 모니터링 환경을 개선하고 있지만, 현재까지 구성된 수집 구성도 및 보안 로그 현황에 대해서 간단히 말씀드리도록 하겠습니다.<br/>
현재 구성 되어 있는 로그 수집 현황에 대한 전체 구성도와 수집 데이터 내용을 보도록 하겠습니다.<br/>

\# 로그 수집 전체 구성도
![](/files/post/2022-01-13-aws-security2/map.png){: width="1200"}  

\# 수집 데이터 내용
아래 표와 같이 데이터를 수집하여 탐지 및 분석 작업을 진행하고 있으며, 수집 데이터 내역을 간단하게 정리하였습니다.

|            로그 수집 플랫폼            |        수집 대상         |                   데이터 유형                   |                             설명                             |
| :------------------------------------: | :----------------------: | :---------------------------------------------: | :----------------------------------------------------------: |
|     Splunk<br />(탐지 데이터 기준)     |        GuardDuty         |               GaurdDuty findings                |    AWS 클라우드 환경 내부에서 발생하는 보안 위협 모니터링    |
|                                        |      AWS Inspector       |               Inspector findings                |      EC2/ ECR 대상에 대한 워크로드 취약점 분석 모니터링      |
|                                        | AWS IAM Access Analyzer  |            Access Analyzer findings             | 신뢰하지 않는 외부에서 IAM 신뢰 정책, S3 버킷 정책이 오픈 되어있는 리소스 모니터링 |
|                                        | Security Hub (보안 표준) |      Config Rule<br />(CIS, Best Practice)      |       AWS 리소스 정보를 통한 Compliance 준수여부 체크        |
|                                        |   Prolwer (OpneSource)   |                Prolwer findings                 |       AWS 리소스 정보를 통한 Compliance 준수여부 체크        |
| ElasticSearch<br /> (로우 데이터 기준) |      AWS CloudTrail      |           Management<br />S3(Object)            | AWS 클라우드 환경에서 발생하는 모든 행위에 대한 요청,응답 데이터 수집 |
|                                        |        Amazon VPC        |            VPC Flow<br />(Version 5)            | AWS 클라우드 환경에서 발생하는 네트워크 통신에 대한 연결 정보를 제공 |
|                                        | Amazon Route 53 Resolver |                  DNS query log                  |     AWS 클라우드 환경에서 발생하는 DNS 요청 데이터 수집      |
|                                        |         AWS WAF          |                 Web Access Logs                 | AWS 에서 제공하는 웹방화벽(Web Application Firewall, WAF) 보안 서비스 |
|                  기타                  |       SecurityHub        |                        -                        |     AWS 보안 서비스에서 탐지되는 이벤트를 중앙에서 관리      |
|                                        |        AWS Shield        |        Standard 서비스는 제공 로그 없음         | AWS 기본으로 제공하는 DDOS 대응 서비스 <br />→ AWS Shield Advanced 서비스 적용은 검토 진행<br /> → AWS WAF 로그를 활용한 DDOS 탐지 패턴 기능 추가(RCF 알고리즘 기반 머신러닝 탐지 적용) |
|                                        |        AWS Macie         |              Amazon Macie findings              |      개인 정보 모니터링 보안 서비스 → 연동 검토 진행 중      |
|                                        |   Firebeat Winlogbeat    | Linux : /var/log/* Windows: WinEventLog, Sysmon | 엔드포인트(서버)에 대한 Security, Audit 로그 수집 → EDR 도입 대체 검토 중 |

# 2. 보안 서비스<a id='service'></a>
AWS 환경에서 필요한 보안 모니터링 기준을 정하고, 보안 서비스 설명, 데이터 수집 구성, 활용방안 등에 대해서 알아보도록 하겠습니다. 보안 서비스에 대한 기능 설명 주제로 인해, Multi Account 환경에서 구성 내용은 제외 하도록 하겠습니다.<br/>

## 2.1 Compliance Check<a id='com0'></a>
클라우드 환경인 AWS 에서는 인프라, 서비스 등 모든 환경이 가상화로 이루어져 있으며, 온프레미스 환경과 달리 관리자,개발자 모두 같은 환경에서 서비스를 구성할 수 있게 되었습니다. 제공하는 서비스들 중에서는 IP 및 Port 의 접근을 제어하는 SecurityGroup, 스토리지처럼 활용 가능한 S3 Bucket, AWS 서비스 권한 관리를 위한 IAM 등등..다양하게  존재 합니다.<br/>
모든 관리자, 개발자들이 개별 권한을 가지고 다양하게 클라우드 환경을 구성할 수 있다보니, 이로 인한 취약한 설정이 발생할 수 있으며, 그것이 보안 이슈로 이어질 수 있습니다.예를 들면 SecurityGroup 에서 SSH 관리 포트가 외부 모두 오픈, S3 Bucket 이 외부에 노출 되도록 Public 오픈, SSRF 공격이 가능한 취약한 환경에서 IMDSv1를 통한 비정상적인 IAM 권한 획득 등의 취약한 환경을 통해 보안 이슈가 발생할 수 있는 상황이 만들어질 수 있습니다.<br/>
별도 CSPM 솔루션을 도입하여 관리하는 것도 방법이지만, 단일 클라우드 환경과 효율적인 비용 구성을 하기 위해 SecurityHub(보안표준)과 Prowler(오픈소스)로 구성하여 Compliance Check 모니터링을 통해 관리하고 있습니다.<br/>



### - 데이터 수집 구성<a id='com1'></a>
![](/files/post/2022-01-13-aws-security2/com01.png){: width="1000"}  

SecurityHub(보안표준), Prowler 는 SecurityHub 통해 중앙에서 수집하여 관리하고 있습니다. Kinesis Data Streams 를 통해 실시간으로 데이터를 받으면, Splunk Forwarder 에서 주기적으로 데이터를 가져오고 있습니다.<br/>
Kinesis Data Streams를 활용하여 데이터가 최소한의 Latency로 받을 수 있도록 구성하였습니다.

#### - SecurityHub(보안표준)<a id='com2'></a>
SecurityHub(보안표준)는 Config 서비스와 기본적으로 연동이 되어 있기 때문에 보안표준 메뉴에서 활성화를 해주면 자동 적용 됩니다.(Config 서비스 별도 활성화 필요)

\#SecurityHub(보안표준) 활성화
![](/files/post/2022-01-13-aws-security2/com02.png){: width="1000"}  



SecurityHub(보안표준)이 활성화되면 Config(AWS 관리형 규칙) 을 통해 AWS 환경에 Compliance Check를 진행하고, 규정 준수 상태를 확인하고 이벤트로 발생하게 됩니다.

\#SecurityHub 이벤트
![](/files/post/2022-01-13-aws-security2/com03.png){: width="1000"}  



SecurityHub(보안표준) 이벤트는 SIEM(Splunk)으로 데이터를 전달하고, Slack 을 통해 실시간 알람을 받아 처리 하고 있습니다.<br/>
조치 상황에 따라 별도 Jira 티켓을 만들어서 조치 할 수 있도록 대응하고 있습니다.

\# SecurityHub(보안표준) Slack 을 통한 알람 발생 모니터링
![](/files/post/2022-01-13-aws-security2/com04.png){: width="800"}  

탐지 규칙에 대한 내용은 AWS Doc 문서에 기재 되어 있으며, 참조하여 검토 및 조치를 진행 하고 있습니다.<br/>
- CIS 기반 탐지 규칙 : https://docs.aws.amazon.com/ko_kr/securityhub/latest/userguide/securityhub-cis-controls.html<br/>
- PCI DSS 기반 탐지 규칙 : https://docs.aws.amazon.com/ko_kr/securityhub/latest/userguide/securityhub-pci-controls.html<br/>
- 보안 모범 사례 규칙 : https://docs.aws.amazon.com/ko_kr/securityhub/latest/userguide/securityhub-standards-fsbp-controls.html<br/>

### - Prowler<a id='com3'></a>

Prowler 는 OpenSource 기반의 Compliance Check를 검증하는 솔루션으로 SecurityHub(보안표준) 탐지 규칙을 보강하기 위해 구성하게 되었습니다. CIS, GDPR, HIPAA, ISO-27001 등의 관련된 다양한 지침에 대한 탐지 규칙을 지원하여 조금 더 많은 범위에 Compliance Check 모니터링이 가능합니다. SecurityHub 통합 기능을 통해 공식적으로 연동이 가능하여 중앙에서 관리 할 수 있는 이점으로 구성하게 되었습니다.<br/>

Prowler 를 처음 사용하기 위해서는 내부 인프라 정보 수집 및 SecurityHub 로그 연동을 위한 권한 설정이 필요합니다.

```
#Prowler 분석에 필요한 정책
arn:aws:iam::aws:policy/SecurityAudit
arn:aws:iam::aws:policy/job-function/ViewOnlyAccess

#SecurityHub 연동을 위한 정책
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "securityhub:BatchImportFindings",
                "securityhub:GetFindings"
            ],
            "Effect": "Allow",
            "Resource": "*"
        }
    ]
}
```
AccessKey, InstanceProfile, ConsoleMe(OpenSource)권한 가져오기 등등 방식으로 권한을 받을 수 있으며, 보안 정책으로 AccessKey 최소화하는 방향으로 가고 있기 때문에 instanceProfile 을 통해 구성하게 되었습니다.

\# 인스턴스 프로파일 Role 할당
<center><img src="/files/post/2022-01-13-aws-security2/com05.png" width="600"></center> 

\# 인스턴스 내 할당 권한 확인
![](/files/post/2022-01-13-aws-security2/com04.png){: width="400"}  



SecurityHub 분석 데이터를 연동하기 위해서 통합 메뉴에서 Prowler 분석 결과를 받을 수 있도록 수집 설정을 활성화 합니다.

\# SecurityHub 통합 메뉴에서 Prowler 분석 결과 수락 활성화
<center><img src="/files/post/2022-01-13-aws-security2/com07.png" width="400"></center>



prowler을 실행하면 aws cli 명령어를 통한 AWS 인프라 구성에 대한 Compliance Check 를 진행하게 되며, 분석된 데이터는 SecurityHub 에 전달됩니다.

\# Prowler 실행 화면
![](/files/post/2022-01-13-aws-security2/com08.png){: width="800"}  

\# SecurityHub 서비스의 Prowler 로그 수집 화면
![](/files/post/2022-01-13-aws-security2/com09.png){: width="1000"}  



Prowler 이벤트는 SIEM(Splunk)으로 데이터를 전달하고, Slack 을 통해 실시간 알람을 받아 처리 하고 있습니다.<br/>
조치 상황에 따라 별도 Jira 티켓을 만들어서 조치 하고 있습니다.

\# Prowler 탐지 이벤트 Slack 알람 발생 확인
![](/files/post/2022-01-13-aws-security2/com10.png){: width="800"}  

## 2.2 Threat Detect<a id='thr0'></a>

### - 데이터 수집 구성<a id='thr1'></a>
![](/files/post/2022-01-13-aws-security2/thr1.png){: width="1000"}  

GuardDuty, Access Analyzer 는 SecurityHub 통해 중앙에서 수집하여 관리하고 있습니다. Kinesis Data Streams 를 통해 실시간으로 데이터를 받으면, Splunk Forwarder 에서 주기적으로 데이터를 가져오고 있습니다. Kinesis Data Streams를 활용하여 데이터가 최소한의 Latency로 받을 수 있도록 구성하였습니다.

### - GuardDuty<a id='thr2'></a>
GuardDuty 보안 서비스는 AWS 클라우드 환경 내에서 발생하는 보안 위협을 지속적으로 모니터링 해주는 서비스 입니다.
VPC Flow Logs, AWS CloudTrail 관리 이벤트 로그, CloudTrail S3 데이터 이벤트 로그 및 DNS 로그의 데이터 원본을 통해 위협 인텔리전스 정보(IP, Domain) 과 머신러닝 기법(DGA 등), EKS 감사로그를 활용하여 클라우드 내 보안 위협을 탐지합니다.
외부 비정상적인 보안 위협 공격 및 클라우드 환경 취약점에 대한 보안 위협을 탐지 이벤트로 제공하고 있어, AWS 환경에서의 보안 모니터링을 위한 필수 서비스로 사용하고 있습니다.

GuardDuty 보안 서비스는 계정 내 서비스 접근 후 활성화를 통해 사용이 가능합니다.

\# GuardDuty 활성화
![](/files/post/2022-01-13-aws-security2/thr2.png){: width="800"}  



서비스 활성화 후 적용 되어 있는 탐지 패턴을 통해 분석하게 되며, 탐지된 이벤트는 결과에 내용이 보여지게 됩니다.

\# 탐지 이벤트 화면
![](/files/post/2022-01-13-aws-security2/thr3.png){: width="800"}  



위협 목록 기능을 통해 TI(Threat Intelligence) 에서 제공하는 위협 IP 정보를 통해 VPC Flow 데이터와 연관해서 탐지 할 수 있는 구성이 가능합니다.<br/>
TI(Threat Intelligence)정보에서 주요한 위협IP 산정하고, S3 버킷에 업데이트를 자동화를 통해 추가 위협을 탐지 할 수 있도록 구성하였습니다.<br/>

\# 위협목록을 통한 Threat Intelligence 적용
![](/files/post/2022-01-13-aws-security2/thr4.png){: width="800"}  



S3 보호에 대한 활성화를 별도 설정을 두고 있으며, 모든 S3 보호가 가능할 수 있도록 구성하여 사용하고 있습니다.<br/>
S3 객체 수준의 API 작업을 모니터링하여, S3 버킷 내 데이터에 대한 잠재적인 보안 위협을 탐지 할 수 있습니다.<br/>

\# S3 보호 설정 적용
![](/files/post/2022-01-13-aws-security2/thr5.png){: width="800"}  

GuardDuty 보안 서비스는 EC2, S3, IAM 기준으로 탐지 패턴을 제공 하고 있습니다.<br/>
EC2는 VPC Flow, DNS 로그 기반을 통해 평판 연관 정보, 머신러닝 기법등을 통해 비정상적인 행위를 탐지합니다.<br/>
S3는 CloudTrail S3 데이터를 통해서 평판 연관 정보, 비정상적인 권한 설정 등에 대한 이벤트를 탐지 합니다.<br/>
IAM는 CloudTrail 관리 데이터를 통해서 비정상적인 권한 사용 및 행위 패턴 등에 대한 이벤트를 탐지 합니다.<br/>
EKS는 EKS Audit 로그 데이터를 통해서 비정상적인 행위 이벤트를 탐지 합니다.

자세한 패턴 내역은 공식 문석에 기재 되어 있으며, AWS 에서 탐지 패턴들이 관리되고 있습니다.<br/>
EC2: https://docs.aws.amazon.com/ko_kr/guardduty/latest/ug/guardduty_finding-types-ec2.html<br/>
S3: https://docs.aws.amazon.com/ko_kr/guardduty/latest/ug/guardduty_finding-types-s3.html<br/>
IAM: https://docs.aws.amazon.com/ko_kr/guardduty/latest/ug/guardduty_finding-types-iam.html<br/>EKS: https://docs.aws.amazon.com/ko_kr/guardduty/latest/ug/guardduty_finding-types-kubernetes.html

탐지하는 위협 이벤트는 SIEM(Splunk)으로 데이터를 받아서, Slack 을 통해 실시간 알람을 받아서 처리 하고 있습니다.<br/>
추가 조치 상황에 따라 별도 Jira 티켓을 만들어서 조치 할 수 있도록 대응하고 있습니다.

\# S3 버킷 퍼블릭 오픈으로 인한 이벤트 탐지
![](/files/post/2022-01-13-aws-security2/thr6.png){: width="1000"}  

\# IMDSv1 SSRF 공격 취약점을 이용하여 EC2 할당 되어있는 IAM 권한 획득 시 이벤트 탐지
![](/files/post/2022-01-13-aws-security2/thr7.png){: width="1000"}  

### - AccessAnalyzer<a id='thr3'></a>

AWS IAM Access Analyzer 는 신뢰영역(Organizational Unit) 내 없는 외부 보안 주체에 엑세스 권한을 부여하게 되었을 경우 탐지하는 서비스 입니다. 
S3 버킷 정책, IAM 신뢰정책, KMS, Lambda함수, SQS, Secrets Manager 등 서비스에 대한 리소스를 검토 합니다.
예를 들어 자신 클라우드 영역 내 S3 버킷 정책을 구성 할 때, Principal 을 * 또는 타사 계정으로 연결할 경우 해당 리소스를  탐지하게 됩니다.<br/>
이를 통해 신뢰 영역이 아닌 외부에서 접근을 인지함으로써, 비정상적인 행위에 대한 모니터링을 하고 있습니다.

AccessAnalyzer 는 IAM 서비스의 Access Analyzer 메뉴의 분석기 생성을 통해 서비스를 활성화 할 수 있습니다.

\# Access Analyzer 생성 화면
![](/files/post/2022-01-13-aws-security2/thr8.png){: width="800"}  

\# Access Analyzer 탐지 화면
![](/files/post/2022-01-13-aws-security2/thr9.png){: width="800"}  

신뢰영역(자신 클라우드 내부) 외에서 접근 가능한 리소스에 대해서 모니터링하고, Slack 알람을 통해 비정상적으로 외부에 노출되는 리소스인지 분석하여 대응하고 있습니다.

\# S3 버킷에서 타사 계정 CloudFront 서비스 접근 정책으로 인한 이벤트 탐지
![](/files/post/2022-01-13-aws-security2/thr10.png){: width="1000"}  

## 2.3 Network Logs<a id='net0'></a>

### - 데이터 수집 구성<a id='net1'></a>
![](/files/post/2022-01-13-aws-security2/net1.png){: width="1000"}  

VPC Flow, DNS Resolver 데이터는 S3 버킷으로 전달하고, Logstash 를 통해 주기적으로 S3 버킷 내용을 가져오도록 구성 하였습니다.<br/>
보안 분석에 필요한 Flag 정보나 트래픽 방향성에 대한 로그를 추가 수집하기 위해 커스텀 필드 설정을 통해 Version 5 로그 유형으로 변경해서 수집하고 있습니다.<br/>

\# 사용자 지정 커스텀 필드를 추가하여 Version5 유형 로그 받을 수 있도록 설정
![](/files/post/2022-01-13-aws-security2/net2.png){: width="1000"} 

### - VPC Flow(Version 5)<a id='net2'></a>

VPC Flow 는 플로우 로그와 방화벽 로그 처럼 네트워크 통신에 대한 연결 정보를 제공 합니다.<br/>
보안 분석을 하기 위해서 기본적으로 필요한 로그이며, 모든 구간에 대해서 VPC Flow 를 수집 할 수 있도록 구성 하였습니다.

\# VPC Flow  검색 화면
![](/files/post/2022-01-13-aws-security2/net3.png){: width="1000"} 

\# Version5 유형 로그 수집을 통한 Flag 및 방향성 정보 확인
![](/files/post/2022-01-13-aws-security2/net4.png){: width="1000"} 

VPC Flow 를 통해 세션 검증, 데이터 통신량 검토, 통신 이력 검토 등 침해 위협에 대한 보안 분석 활동에 활용 하고 있습니다.

\# VPC Flow 분석 대시보드
![](/files/post/2022-01-13-aws-security2/net5.png){: width="1000"} 



### - DNS Resolver<a id='net3'></a>

AWS 클라우드 환경 내에서 요청하는 DNS 요청 데이터를 수집할 수 있는 서비스 입니다.<br/>
Route53에서 Resolver - Query Logging 활성화를 통해 쉽게 로그를 수집 할 수 있도록 기능으로 제공하고 있습니다.

\# DNS Resolver 로그 검색 화면
![](/files/post/2022-01-13-aws-security2/net6.png){: width="1000"} 

\# DNS Resolver 로그 수집 유형
<center><img src="/files/post/2022-01-13-aws-security2/net7.png" width="400"></center>

DNS Resolver 로그를 활용하여, Threat Intelligence IOC(DNS) 연동 이벤트 탐지 및 보안 위협 분석 활동 등에 활용하고 있습니다.<br/>

\# DNS 요청 Query 길이를 통한 DNS 터널링 공격 패턴 분석
![](/files/post/2022-01-13-aws-security2/net8.png){: width="1000"} 

# 3. 작성을 마치며....<a id='end'></a>

부족한 내용이지만 AWS 클라우드 환경에서 보안 모니터링을 위해 구성했던 보안 서비스 내용들을 작성하였습니다.<br/>
3부에서는 WorkLoad(Inspector, CWPP[3rd]), AWSWAF, CloudTrail 등의 보안 서비스를 구성하고, 활용 했던 내용을 공유 하고자 합니다.<br/>
지금까지 내용이 AWS 보안 모니터링 환경을 구성하는데 조금이라도 도움이 되셨길 바라며, 2부 내용은 마치도록 하겠습니다.<br/>
감사합니다.<br/>
