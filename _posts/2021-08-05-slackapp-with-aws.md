---
layout: post
title: 'AWS를 활용한 Slack 공개 App 제작 - Luckydraw'
author: 유광엽
date: 2021-08-05
tags: [aws, slack, luckydraw]
---

안녕하세요 !  
사내에서 무작위 추첨(뽑기)을 사용할 일이 있으신가요?   
그럴 때 Slack에서 간단히 추첨할 수 있도록 App을 만들어봤습니다.   
추첨 App을 만들면서 AWS Lambda를 처음 사용해봤는데요, AWS서비스 이용부터 Slack에 공개런칭까지 고군분투한 과정을 이야기해보려 합니다. 

## [Slack 공개 App `Luckydraw` 바로가기](https://d2uashsimr8af7.cloudfront.net/){:target="_blank"}
#### 목차 - [App 런칭 방법](https://api.slack.com/tutorials/submitting-apps-to-the-directory){:target="_blank"}

* [1.추첨 Slack APP 만들기](#bulid-slack-app)  
    * 1-1. 새로운 App 만들기  
    * 1-2. 새로운 App 공개배포 활성화 시키기  
    * 1-3. App제출 진행하기  
   
* [2.추첨서비스 구축하기](#bulid-luckydraw)  
    * 2-1. 사전준비  
    * 2-2. Slack 슬래시커맨드 호출 흐름도  
        * 2-2-1. 공통 Lambda 생성  
        * 2-2-2. 추첨서비스 Lambda 생성  
        * 2-2-3. API Gateway 생성  
    * 2-3. 트러블슈팅 사항  
* [3.소개용 랜딩페이지 구축하기](#bulid-landing-page)  
    * 3-1. S3 버킷 생성하기  
    * 3-2. CloudFront 배포하기  
* [4.OAuth 인증 서버 구성하기](#bulid-oauth-server)  
    * 4-1. AWS RDS 구성 시 참고사항  
    * 4-2. AWS Dynamodb 구성 시 참고사항  
* [5.앱 제출 후 Slack팀 Review 처리내용](#review-after-launching)  
* [6.최종 결과](#complete-result)


---
> **목표:** Slack의 Slash command 기능을 가진 추첨 App을 만들어서 공개 런칭해보기   
> **필요사항:** Slack 및 AWS 계정  
> **사용 Spec:** Python3, Slack API, AWS Lambda, API Gateway, S3, CloudFront, Dynamodb    
> **참조문서**  
> - **Slack App 디렉터리:** 
>   - https://[회사slack주소]/apps  
> - **Slack App 런칭 공식 가이드 문서:** 
>   - [바로가기](https://api.slack.com/start/distributing/guidelines){:target="_blank"}
> - **Slack App 런칭 공식 체크리스트 문서:** 
>   - [바로가기](https://api.slack.com/reference/slack-apps/directory-submission-checklist#new-slack-app-directory-checklist__app-submission-checklist__help-us-review-your-app){:target="_blank"}
> - **Slack API 공식 문서** 
>   - [바로가기](https://api.slack.com/){:target="_blank"}

---

# 1. 추첨 Slack APP 만들기 <a id='bulid-slack-app'></a>
## 1-1. 새로운 App 만들기
1. https://api.slack.com/apps 접속해서  Create New App 버튼 클릭
2. App Name 및 Workspace 설정
3. App Display Information 설정

## 1-2. 새로운 App 공개배포 활성화 시키기
1. Manage Distribution(배포관리) 메뉴 접속해서 체크리스트 확인
    - Enable Features & Functionality 내 Slash Commands 설정
        - 새로운 슬래시 커맨드 생성(Create New Command 버튼)

    - Add OAuth Redirect URLs 설정
        - Install to Workspace
        - Add New Redirect URL 셋팅

    - Remove Hard Coded Information 확인

2. 공개 배포 활성화 (Activate Public Distribution)

## 1-3. App제출 진행하기
1. 좌측 Sumbit to App Directory 메뉴 접속해서 체크리스트 확인
    1. Review your permission scopes 검토
    2. Review your user experience 검토
    3. Prepare your App Directory Listing 검토
        - 추첨서비스 구축 진행 - 아래 2번 항목 참고(추첨서비스 구축하기)
    4. Review your installation links 검토
        - Landing Page 구축 진행 - 아래 3번 항목 참고(소개용 랜딩페이지 구축하기)
    5. ..등등 review사항 전체 검토


# 2. 추첨서비스 구축하기 <a id='bulid-luckydraw'></a>
> 목적: Slack App 공개를 위해서 외부에서 호출 가능한 서비스 구성  
> 기술 Spec: AWS Lambda + API Gateway  

## 2-1. 사전준비
- 추첨봇이 추첨서비스를 정상적으로 수행하기 위한 Slack Bot 권한 scope 추가 (ex. 유저정보 조회, 채널정보 조회, 메세지 전송 등)

## 2-2. Slack 슬래시커맨드 호출 흐름도
![](/files/post/2021-06-28-slackapp-with-aws/image01.png)

### 2-2-1. 공통 Lambda 생성
- 목적: Slash command에 의해서 호출되면 3초이내 response 응답 + 비동기로 서비스 람다 호출
- 함수명: lambda-slack-command    

```python
import json
import boto3
 
def lambda_handler(event, context):
    request_uri = event['path']
    client = boto3.client('lambda')
    response = client.invoke_async(
        FunctionName='lambda-slack-command-'+request_uri[1:].lower(),
        InvokeArgs=json.dumps(event, ensure_ascii=False)
    )
    return {
        'statusCode': 200,
        'body': ''
    }
```

### 2-2-2. 추첨서비스 Lambda 생성
- 목적: 추첨 로직 수행해서 endpoint로 결과메세지 전송
- 함수명: lambda-slack-command-luckydraw
- 외부 라이브러리: python slack_sdk 추가

### 2-2-3. API Gateway 생성
- 목적: 클라이언트가 slash command 요청 시, Lambda실행을 위한 rest api 역할 수행
- API명: api-slack-command
- 생성방법  
    1) API생성 클릭  
    ![](/files/post/2021-06-28-slackapp-with-aws/api-gateway01.png){: width="100%" height="100%"}  
    
    2) REST API 구축 클릭   
    ![](/files/post/2021-06-28-slackapp-with-aws/api-gateway02.png){: width="100%" height="100%"}
    
    3) 내용 입력 후 생성  
    ![](/files/post/2021-06-28-slackapp-with-aws/api-gateway03.png){: width="100%" height="100%"}
    
    4) 리소스 경로 생성 (리소스 이름(ex. luckydraw)은 lambda 소스에서 공통으로 활용되도록 작업했으므로 참고바람.)   
    ![](/files/post/2021-06-28-slackapp-with-aws/api-gateway04.png){: width="100%" height="100%"}
      
    5) 메서드 생성 후(POST) 통합될 포인트 설정(ex. 공통 lambda로 지정함)  
    ![](/files/post/2021-06-28-slackapp-with-aws/api-gateway05.png){: width="100%" height="100%"}
    
    6) 설정이 완료됐으면 API 배포  
    ![](/files/post/2021-06-28-slackapp-with-aws/api-gateway06.png){: width="100%" height="100%"}
    
    7) 스테이지에서 호출 URL 확인가능  
    ![](/files/post/2021-06-28-slackapp-with-aws/api-gateway07.png){: width="100%" height="100%"}
    
## 2-3. 트러블슈팅 사항
- slack에서 서비스lambda 바로 호출 시, 3초 이내 응답불가로 인해서 공통lambda 앞단에 위치시킴
- 공통lambda에서 다른 lambda호출 시, IAM 역할 설정 필요 (공통lambda & 사용자 IAM)  
![](/files/post/2021-06-28-slackapp-with-aws/iam-role.png){: width="100%" height="100%"} 
- API gateway생성 후 POST 설정 시, "Lambda 프록시 통합 사용" 항목 체크


# 3. 소개용 랜딩페이지 구축하기 <a id='bulid-landing-page'></a>
> 목적: 추첨서비스 소개 홈페이지 구성  
> 기술 Spec: AWS S3 + CloudFront
  
## 3-1. S3 버킷 생성하기
- 주요 참고사항 - (S3 정적웹페이지 접근 시)모든 퍼블릭 엑세스 차단 해제 및 버킷 정책 추가 설정
        - 버킷정책
        
        {
            "Version": "2012-10-17",
            "Id": "Policy1612765023063",
            "Statement": [
                {
                    "Sid": "Stmt1612765021033",
                    "Effect": "Allow",
                    "Principal": "*",
                    "Action": "s3:GetObject",
                    "Resource": "arn:aws:s3:::s3-luckydraw/*"
                }
            ]
        }

## 3-2. CloudFront 배포하기
- CloudFront 관련 [공식문서 바로가기](https://docs.aws.amazon.com/ko_kr/AmazonCloudFront/latest/DeveloperGuide/GettingStarted.SimpleDistribution.html){:target="_blank"} 
- 트러블슈팅 사항
    - Default Root Object 항목에 index.html 설정
    - S3 접근이 퍼블릭 차단이 됐다면 Restrict Bucket Access는 Yes로 설정 후, OAI(Origin Access Identity)를 새로 만들어서 S3정책에서 사용
        - Restrict Bucket Access 관련 [공식문서 바로가기](https://docs.aws.amazon.com/ko_kr/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-s3.html){:target="_blank"}


# 4. OAuth 인증 서버 구성하기 <a id='bulid-oauth-server'></a>
> 목적: Slack App을 workspace에 install 하기 위해서 oauth 인증을 통해서 access token을 발급 받고 기록하기 위함  
> 기술Spec: AWS Lambda, API Gateway, DynamoDB  
> 공식설명 URL: https://api.slack.com/legacy/oauth  
1. Client가 Slack App 설치 요청
2. 설정해놓은 RedirectURL로 인증 요청 (*OAuth 인증서버)
3. Slack 공식 인증서버로 요청(https://slack.com/api/oauth.v2.access 필수값: clientId, clientSecret, 요청code)
4. 전달받은 access token 저장


## AWS RDS 구성 시 참고사항
1. VPC 보안그룹을 새로 만들어서 방화벽 통제 필요
2. 파라미터 그룹을 새로 만들어서 문자열 set 및 timezone 설정 필요
3. 테이블 예시
    - table명: access_token
    - column명: team_id(*), app_id(*), team_name, access_token, reg_date, update_date (*는 PK)


## AWS Dynamodb 구성 시 참고사항
1. lambda에서 접근 허용 정책 추가 (resource는 dynamodb 테이블 생성 후 arn참조)
    - IAM 인라인 정책 - dynamodb Put/Get
    ```
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "VisualEditor0",
                "Effect": "Allow",
                "Action": [
                    "dynamodb:PutItem",
                    "dynamodb:GetItem"
                ],
                "Resource": "arn:aws:dynamodb:ap-northeast-2:610564112701:table/*"
            }
        ]
    }
   ```

2. 예상 요금
- 쓰기 100만 건당 $1.3556
- 읽기 100만 건당 $0.271
- 데이터 1GB당 $0.27075  
→ 위 내용으로 가정 시.. Total: 월 $2 예상 (약 2,400원)
![](/files/post/2021-06-28-slackapp-with-aws/expect-bill.png){: width="100%" height="100%"}

# 5. 앱 제출 후 Slack팀 Review 처리내용 <a id='review-after-launching'></a>
1. It looks like the app's icon is a bit blurry, especially when viewed at larger sizes. Could it be improved?
    > App Icon 이미지 고화질로 재 설정
2. Unfortunately we need a privacy policy in English for review that can't be machine translated. There is no need to host this on your site however, you can provide that to us by email or other means.
    > 개인정보 보호정책 영어로 작성하여 이메일로 전달
3. Could the "Data archival/removal policy" be filled out for the app?
    > 보존&저장 정책과 동일하게 수정
4. Could some information about the use of the Slack token be added to the privacy policy, as well as any information about how data is shared with third party service providers, such as hosting or data processing services?
    > Landing page에 영어로 AccessToken 및 Team정보 저장관련 내용과 Third party에 데이터 공유관련 정책 영어로 작성


# 6. 최종결과 - 공개 App 런칭완료 <a id='complete-result'></a>
- Luckydraw 공식 소개 페이지: 
    - <https://d2uashsimr8af7.cloudfront.net/>{:target="_blank"}
![](/files/post/2021-06-28-slackapp-with-aws/complete.png){: width="100%" height="100%"}

