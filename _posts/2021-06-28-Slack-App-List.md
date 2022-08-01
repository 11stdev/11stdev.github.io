---
layout: post
title: 'Slack User App 리스트 추출'
author: 엄재도
date: 2021-06-28 13:27
tags: [slack]
---

 Slack은 다양한 API를 제공하며, 이를 통해 기본 통계 화면에서 제공하지 않는 입맛에 맞는 Slack 사용현황 분석과 확장 기능을 구현할 수 있다.
<br/>API를 통하여 업무에 필요한 데이터를 얻고 활용한 사례를 소개해 보고자 한다.


## 로그 추출 히스토리

11번가 Slack Workspace의 `유저 생성 앱`과 `Incoming-webhook`, [`Jenkins-CI`](https://11st-pf.slack.com/apps/A0F7VRFKN-jenkins-ci)와 같은 `사용자 지정 통합 앱` 의 리스트를 추출하여 분석함으로서, 사용자 계정이 비활성 처리될 때 문제가 될만한 앱을 찾아보고자 하였다. 

슬랙의 앱 관리 WEB-UI는 검색 기능이나 Export  기능을 제공하지 않으므로 수백개의 설정이 존재하는 Incoming-Webhook의 리스트만 확인하려 해도 수십 페이지에 걸친 정보를 취합하는 반복 작업이 필요하기에 엄두가 나지 않는다. 

Slack 측에 추출방법에 대해 문의해 보니 아래와 같은 답변을 들을 수 있었다. 

> You can call the team.integrationLogs ([https://api.slack.com/methods/team.integrationLogs](https://api.slack.com/methods/team.integrationLogs)) API to get the integration activity logs for a team and then filter that by user by passing the user ID as the user value. You will need to navigate through the results as they go back all the way to the start of your workspace, but hopefully this would be of use.
(te)


[team.integrationLogs](https://api.slack.com/methods/team.integrationLogs) API를 호출해 확인해 보니, 앱 리스트는 6158건이 나왔다. 이것은 생성/삭제/업데이트 내역이 모두 카운트 된 횟수로서 숫자 자체로서의 의미는 없다. 

![App Activity Log를 통한 Workspace의 앱 히스토리 확인](/files/post/2021-05-05-Slack-App-List/app_activity_log.png)

Activity Log 화면에서도 동일한 정보를 확인할 수 있지만, 설치/확장/업데이트 등의 다양항 타입이 구분없이 리스팅되므로 유의미한 값을 확인하는데는 어려움이 있다.

더군다나, 유료버전 Slack을 최대한 활용한다는 측면에서 멤버의 슬랙앱 개발이나 외부 앱 설치를 제한하지 않고 있으므로 더 많은 수의 설치 기록이 존재하는 듯 하다.


추출된 리스트에서 의미있는 값만 필요했으므로, 단순하게 리스트를 종류별로 추출해 보는 코드를 작성하였다. 

## 로그 분석을 위한 로그추출 버전

```python
import requests
import json, time

# document: https://api.slack.com/methods/team.integrationLogs
#USER_TOKEN = ""  # Basic Auth - User Token

def callSlackTeamIntegratinLogs(page):
    url = "https://slack.com/api/team.integrationLogs?count=100&page=" + str(page)
    headers = {
        "Accept": "application/json", 
        "Authorization": "Bearer " + USER_TOKEN 
    }

    response = requests.request(
        "GET",
        url,
        headers=headers
    )

    return response.text

response = json.loads(callSlackTeamIntegratinLogs(1))

logs = response.get("logs")
paging = response.get("paging")

total = paging.get("total")
pages = int(paging.get("pages"))
app_list = []
service_list = []

for log in logs:

    if log.get("service_type"):
        print("[Service]==>", log)

    elif log.get("app_type"):
        print("[APP]==>", log)
    else:
        print("other==>", log)

print(json.dumps(app_list, indent=4)) # json beautify print

```


## 단순 호출을 통한 앱 타입별 데이터 형태 구분

team.integrationLogs API를 통하여 확인해 보니, 설치앱은 APP과 Service 타입으로 구분할 수 있었고 종류에 따라 일부 다른 필드가 반환됨을 확인할 수 있었다.

### Workspace에 종속 설정된 App (Service)
```
* 업데이트 
> [Service] ==> `{'user_id': 'UEDL4TBT4', 'user_name': 'jaedo.aum', 'date': '1617038581', 'change_type': 'updated', 'service_id': 1894205218199, 'service_type': '수신 웹후크', 'scope': 'incoming-webhook', 'channel': '**private**'}`

* 추가
> [Service]==> `{'user_id': 'UEDL4TBT4', 'user_name': 'jaedo.aum', 'date': '1617038562', 'change_type': 'added', 'service_id': 1894205218199, 'service_type': '수신 웹후크', 'scope': 'incoming-webhook', 'channel': 'private'}`

* 삭제: 
> [Service]==> `{'user_id': 'UEDL4TBT4', 'user_name': 'jaedo.aum', 'date': '1617038950', 'change_type': 'removed', 'service_id': 1894205218199, 'service_type': '수신 웹후크', 'scope': 'incoming-webhook', 'channel': 'private'}`
```

### User Created App 

> User Create APP은 Workspace 전용앱과 마찬가지로 Slash Command, Bot, Incoming-webhook 등을 모두 설정할 수 있다. 

![slack user app functionality](/files/post/2021-05-05-Slack-App-List/2021-04-05-Slack-1.png)



아래 샘플로그는 유저앱 이벤트에 따른 로그 분석을 위하여, User Created App을 Install(1)하고,  Incoming-webhook을 추가 (2)하고, App을 삭제(3)하는 일련의 과정을 테스트하며 로그 상태를 체크.

(1) 유저앱 설치시
``` 
[APP]==> {'user_id': 'UEDL4TBT4', 'user_name': 'jaedo.aum', 'date': '1617039190', 'change_type': 'added', 'app_type': 'PriTest', 'app_id': 'A01TEPT5C9W'}
```

(2) 유저앱에 incoming-webhook 추가후 재인스톨
```
[APP]==> {'user_id': 'UEDL4TBT4', 'user_name': 'jaedo.aum', 'date': '1617039492', 'change_type': 'expanded', 'app_type': 'PriTest', 'app_id': 'A01TEPT5C9W'}
```

!! 앱에서 설정된는 웹훅은 삭제시 히스토리 로그가 남지 않는다. (워크스페이스 종속앱의 경우 히스토리가 남음)

- Private, Public 채널 관계없이 "added, 설치됨" 혹은 "expanded, 확장됨"으로만 기록됨
- webhook 삭제시, 기록이 남지 않음
- 즉, **User APP에서 설정되는 incoming-webhook은 관리 불가능**


(3) 유저앱 삭제
```
 [APP]==> {'user_id': 0, 'user_name': 'slack', 'date': '1617039941', 'change_type': 'removed', 'reason': 'developer_uninstalled_app', 'app_type': 'PriTest', 'app_id': 'A01TEPT5C9W'}
```


## CSV 추출을 위한 메인 버전

로그 분석을 통해 유의미한 통계값을 바로 확인 가능할 것이라 생각하고 접근했지만, 코딩을 통한 완벽한 통계 데이터 추출은 ROI가 나오지 않는다고 판단되었기 때문에 엑셀을 통해 추가 분석을 진행하였다. 


```python
import requests
import json, time

# document: https://api.slack.com/methods/team.integrationLogs
#USER_TOKEN = ""  # Basic Auth - User Token

def callSlackTeamIntegratinLogs(page):
    url = "https://slack.com/api/team.integrationLogs?count=100&page=" + str(page)
    headers = {
        "Accept": "application/json", 
        "Authorization": "Bearer " + USER_TOKEN 
    }

    response = requests.request(
        "GET",
        url,
        headers=headers
    )

    return response.text

app_list = []
service_list = []
pcnt = 1
while True:
    # break
    if(pcnt > pages):
        break

    response = json.loads(callSlackTeamIntegratinLogs(pcnt))

    if pcnt == 1:
        paging = response.get("paging")

        total = paging.get("total")
        pages = int(paging.get("pages"))

        print(paging, total, pages)

    logs = response.get("logs")

    for log in logs:
        app_list.append(log)

    print(len(app_list), len(service_list))

    # keys = app_list[0].keys()
    keys = ['app_id','app_type','service_id','service_type','channel','user_id', 'user_name', 'date', 'change_type', 'reason', 'scope', 'rss_feed_url', 'rss_feed', 'rss_feed_change_type', 'rss_feed_title']
    
    with open('slack-app-list.csv', 'w', newline='') as cf:
        dict_writer = csv.DictWriter(cf, keys)
        dict_writer.writeheader()
        dict_writer.writerows(app_list)

    pcnt += 1
    time.sleep(5)   # rate limit 2 이므로 5초씩 쉰다. 그냥 추출하면, 1500건 추출된 후 retelimit

```

### excel을 통한 분석

앱의 목록을 추출한 결과는 app type, 상태 등에 의해 리턴값이 다르다.
![excel all](/files/post/2021-05-05-Slack-App-List/excel_all.png)

사전에 분류한대로 App과 Service 타입으로 나누고 문제점을 찾아 보았다.
User App의 Incoming-webhook은 히스토리가 추출되지 않으므로, 검출 가능한 문제점은 Workspace 종속 앱(service) 중 Private channel에 대해 설정된 incoming-webhook 이었다.

![excel all](/files/post/2021-05-05-Slack-App-List/excel_problem.png)


## 로그 분석을 통해 도출된 결론

수신 웹훅은 사용자 지정 통합 앱(Workspace에 종속설정) 방식일때만 존재 여부를 모니터링 할 수 있다. 
모니터링 가능한 수신웹훅 중, Private 채널에 설정된 앱은 다른 사용자(관리자포함)가 컨트롤 불가능하고, 담당자 퇴사시 Deactive되며 복구 불가능한 상태가 된다. 

사용자 구축 앱으로 설정된 수신 웹훅은, Private/Public 채널 여부와는 관계 없이 Incoming-webhook 용도라는 사실을 알수가 없다. 채널에 설정된 웹훅을 삭제해도 히스토리에 남지 않는다. 사실상 관리가 불가능하다. 

Incoming-webhook을 Private 채널에 설정할 때는 비활성화 될 염려가 없는 공용시스템계정(@11stslackbot) 을 통해 설정되어야 안전하다. 
사용자 지정통합웹 방식으로 설정하고자 할때, 시스템관리계정을 채널에 초대하고 Incoming-webhook 생성요청을 한다. 시스템계정은 설정후 채널을 나가도 된다.

사용자 앱을 만들땐, 공용시스템계정을 Collaborator로 초대한다. 해당 앱에서 Incoming-webhook을 설정할 땐 공용시스템계정을 해당 채널에 초대한 뒤, 웹훅 URL 생성을 요청한다. 설정 후 공용계정은 채널을 떠나도 설정된 웹훅은 만료되지 않는다.