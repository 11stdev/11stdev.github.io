---
layout: post
title: 'AWS Serverless 웹서비스 구축기'
author: 손명선
date: 2021-10-08
tags: [aws, cf, s3, code-build]
---

많은 사용자들이 이용하는 웹사이트 및 모바일 앱에는 이미지, 동영상 또는 음악 같은 파일이나 .css 또는 .js 같은 정적 파일을 가지고 있습니다.
콘텐츠 배포 네트워크(CDN) 서비스가 등장하기 전에는 특정 지역에 서버를 일일이 두고 서비스를 해야 했고, 이와 더불어 DevOps 바람이 불면서
CI/CD에 대한 복잡한 구성까지 진행을 해야 했으며, Terraform이나 AWS Cloudformation과 같은 다양한 자동화 코드를 통해 이를 구성하고 이해해야 했습니다.
이번 포스팅에서는 이런 복잡한 구조를 단순화하면서 비용까지 절감하는 AWS Serverless한 구성으로 웹서비스 하는 방법을 포스팅하려 합니다.

#### 목차
* [1.구성 요건](#01mig)  
* [2.웹 서비스 구성 요구 사항](#02task)  
* [3.웹 서비스 구성](#03firewall)  
* [4.배포 구성](#04instance)  
* [5.배포 구성 시 유의할 점](#05task)  


# 1.구성 요건<a id='01mig'></a>
Cloud 기반으로 인프라를 구축하면서  기존에는 <그림1>의 IDC 구성과 같이 앞단에 CDN Saas를 두고, L4를 통해 Web Server에서 스테틱 컨텐츠를 
제공해 주는 구조로 구성을 했었는 데, view.js와 같이 간단한 웹 서비스를 제공할 때 구성을 단순화할 수 있지 않을까 고민을 하게 됩니다.
이를 고려하여 구성한 부분이 아래 그림의 AWS 구성입니다.  

![](/files/post/2021-10-08-aws-Serverless-webservice/1-1.png){: width="100" height="200"}
                        

위 그림의 AWS 구성을 IDC 구성과 기능을 비교하면 아래와 같습니다.

|구분|기능|IDC구성|AWS 구성|
|-|----|---|---|
|CDN.     |Cache                          |CDN|AWS CloudFront|
|L4       |SSL Offload                    |L4|AWS CloudFront|
|L4       |LB                             |L4|AWS S3(HDFS)|
|WebServer|Redirect/Rewrite|WebServer     |Cloudfront(or S3)|
|WebServer|Content Cache-Control|WebServer|Cloudfront/S3|
|WebServer|대용량 Request 처리               |WebServer 혹은 증설|S3|
|보안|index 설정 제거 & 400/404에러페이지 설정등  |Apache|Cloudfront|
|WebServer|Source type                    |View.js|View.js|
|NAS 기능|Source type                      |NAS|S3|
|빌드/배포|구성 방안                           |Jenkins+스크립트|Jenkins + S3 + Codebuild|

* CF+S3 구성은 Backend가 아닌 Frontend단의 구성 → Backend에 WAS가 있는 경우에는 요구 사항을 고려 후 구성


위 비교표만 보면 모든 웹서버를 AWS 구성으로 가야하는 게 좋은 거 아냐라는 생각을 하게 됩니다. 하지만 AWS 구성에도 한계는 존재합니다.
S3 혹은 CloudFront의 Redirect/Rewrite 기능은 Webserver(apache/nginx)와 같이 다이나믹한 기능을 제공하지는 않습니다.
아래 표와 같이 라우팅 규칙 요소를 잘 조합하여 구현해야 하기 때문에 복잡도가 높은 룰은 많은 테스트가 필요한 편입니다.

|이름|설명|
|---|----|
|RoutingRules |RoutingRule 요소 모음용 컨테이너입니다.|
|RoutingRule  |조건 및 조건이 충족되었을 때 적용되는 리디렉션을 식별하는 규칙입 조건:RoutingRules 컨테이너에는 적어도 1개 이상의 라우팅 규칙이 포함되어 있어야 합니다.|
|Condition    |지정된 리디렉션이 적용되려면 충족되어야 할 조건을 설명하기 위한 컨테이너입니다. 라우팅 규칙이 조건을 포함하지 않는 경우 해당 규칙은 모든 요청에 적용됩니다.|
|KeyPrefixEquals|리디렉션된 요청을 보내는 객체 키 이름 접두사입니다.|
|KeyPrefixEquals|HttpErrorCodeReturnedEquals를 지정하지 않을 경우 가 필요합니다. KeyPrefixEquals 및 HttpErrorCodeReturnedEquals가 모두 지정되는 경우 두 항목 모두 true로 설정되어야 조건이 충족됩니다.|
|HttpErrorCodeReturnedEquals|리디렉션 적용 조건과 일치하는 HTTP 오류 코드입니다. 오류가 발생하고 오류 코드가 이 값에 해당하는 경우, 지정된 리디렉션이 적용됩니다.|
|HttpErrorCodeReturnedEquals|KeyPrefixEquals를 지정하지 않을 경우 가 필요합니다. KeyPrefixEquals 및 HttpErrorCodeReturnedEquals가 모두 지정되는 경우 두 항목 모두 true로 설정되어야 조건이 충족됩니다.Redirect요청에 대한 리디렉션 지침을 제공하는 컨테이너 요소입니다. 다른 호스트나 다른 페이지로 요청을 리디렉션할 수 있으며, 사용할 다른 프로토콜을 지정할 수 있습니다. RoutingRule에는 Redirect 요소가 있어야 합니다. Redirect 요소는 Protocol, HostName, ReplaceKeyPrefixWith, ReplaceKeyWith 또는 HttpRedirectCode 중 한 개 이상의 형제 요소를 포함해야 합니다.|
|Protocol|응답에서 반환된 http 헤더에 사용할 https 또는 Location 프로토콜입니다.해당 형제 요소 중 하나가 제공되는 경우 Protocol는 필요하지 않습니다.|
|HostName|응답에서 반환된 Location 헤더에 사용할 호스트 이름입니다.해당 형제 요소 중 하나가 제공되는 경우 HostName는 필요하지 않습니다.|
|ReplaceKeyPrefixWith|리디렉션 요청의 KeyPrefixEquals 값을 대체하는 객체 키 이름의 접두사입니다.해당 형제 요소 중 하나가 제공되는 경우 ReplaceKeyPrefixWith는 필요하지 않습니다. ReplaceKeyWith가 제공되지 않는 경우에만 제공 가능한 파라미터입니다.|
|ReplaceKeyWith|	응답에서 반환된 Location 헤더에 사용할 객체 키입니다.해당 형제 요소 중 하나가 제공되는 경우 ReplaceKeyWith는 필요하지 않습니다. ReplaceKeyPrefixWith가 제공되지 않는 경우에만 제공 가능한 파라미터입니다.|
|HttpRedirectCode|응답에서 반환된 Location 헤더에 사용할 HTTP 리디렉션 코드입니다.해당 형제 요소 중 하나가 제공되는 경우 HttpRedirectCode는 필요하지 않습니다.|

* Link URL : https://docs.aws.amazon.com/ko_kr/AmazonS3/latest/userguide/how-to-page-redirect.html

S3 비용은 저렴하나 CF는 계약에 따라 저렴하기도 하고 비싸질 수 있습니다.


# 2. 웹 서비스 요구 사항<a id='02task'></a>
  
자 이제 제가 구성한 내용을 실제 구성하는 내용을 설명 드리겠습니다.
요구 사항은 아래 표와 같이 js,html.css를 웹서비스 해주는 과정에서 기본적인 redirect와 Cache 정책에 대한 부분입니다.

|구분|요구사항|
|---|---|
|Source|View JS|
|Redirect|파일이 없거나~ 디렉토리가 없는 경우(HTTP Code 404 Error)가 발생할 경우 /index.html로 Redirect|
|Redirect|Bad protocol Error(HTTP Code 400 Error)가 발생할 경우 /404.html로 Redirect| 
|Cache|index.html은 no cache, /static. : TTL 3600, /js : TTL 3600|



# 3. 웹 서비스 구성<a id='03firewall'></a>

먼저 컨텐츠가 저장되는 S3를 구성해 보도록 하겠습니다.
버킷은 아래 그림과 같이 Private 버킷으로 구성하며
![](/files/post/2021-10-08-aws-Serverless-webservice/3-2.png){: width="100" height="200"}
CF에서 접근할 수 있도록 Bucket Policy에 권한을 부여해 줍니다.
![](/files/post/2021-10-08-aws-Serverless-webservice/3-1.png){: width="100" height="200"}
* 버킷의 권한에서 CF OAI가 있는 Distribution ID만 접근이 가능하도록 설정(OAI 생성은 아래 CF에서 다루겠습니다.)
* S3에서 다양한 리다이렉트 요건이 없어 Static Web Site Hosting 기능은 사용하지 않았습니다.


추가로 샘플 소스 Upload 후 아래 그림과 같이 Object의 Content Type을 확인할 수 있습니다.
![](/files/post/2021-10-08-aws-Serverless-webservice/3-3.png){: width="100" height="200"}
* CSS/JS 등 단일 포맷 이름을 가지고 있는 경우 자동으로 타입이 맞춰지나 js.map 과 같이 포맷이름이 2개이상 들어가는 경우 소스 배포 시 맞춰줘야 합니다.


CF 구성은 OAI를 사용하여 S3에서 특정 OAI 값을 가진 CF Distribution만 Access할 수 있도록 설정하며, Domain 이하 URL별 Cache 정책이 다를 경우
아래와 같이 생성하여 적용합니다.

* CF>원본ID 생성을 통해 OAI를 생성합니다.

![](/files/post/2021-10-08-aws-Serverless-webservice/3-4.png){: width="50" height="100"}

* Distribution 생성을 위해 아래와 같이 설정해 줍니다.

![](/files/post/2021-10-08-aws-Serverless-webservice/3-5.png){: width="100" height="200"}

* Cache 설정 시 아래 그림과 같이 URL별로 정책이 다르다면 URL을 분리하여 적용해야 합니다.

![](/files/post/2021-10-08-aws-Serverless-webservice/3-6.png){: width="100" height="200"}

* No cache 설정을 하려면 아래 그림과 같이 최소/최대/기본 TTL을 0으로 적용해 줍니다.(No-Store로 저장 시에는 원본에서 해당 캐시 설정이 들어가야 합니다.)

![](/files/post/2021-10-08-aws-Serverless-webservice/3-7.png){: width="100" height="200"}

* Cache 설정을 하려면 아래와 같이 최소/최대/기본 TTL을 설정해 줍니다.

![](/files/post/2021-10-08-aws-Serverless-webservice/3-8.png){: width="100" height="200"}

보안 취약점 권고 사항에 400/404 Error Page 설정이 필요한 경우 아래와 같이 설정을 진행합니다.

![](/files/post/2021-10-08-aws-Serverless-webservice/3-9.png){: width="100" height="200"}

 * TTL 0으로 설정  필요 : React 웹서비스인 경우 아래와 같이 동작 시 배포 후 장애 발생 할 수 있음
  

# 4. 배포 구성하기<a id='04instance'></a>
위와 같이 구성후 배포 구성을 아래와 같이 할 수 있습니다.

![](/files/post/2021-10-08-aws-Serverless-webservice/4-1.png){: width="100" height="200"}

 * Jenkins(혹은 Lambda)가 Bitbuket의 Source를 S3에 업로드하고, S3의 Put object 이벤트를 Codebuild로 전달받아 빋드후 서비스 버킷으로 업로드하는 구성입니다.

# 5. 배포 구성 시 유의할 점<a id='05task'></a>
Source 빌드 후 S3 배포 시 서비스 유형이나 TTL 정책에 따라 URL 별로 배포를 순차적으로 해야 할 수 있으니 이를 고려해서 배포를 진행해야 합니다.

S3에 배포 시 content type 변경 : Apache의 경우 mime 파일이 있어 여기에 추가를 해주면 되지만 S3는 자주 사용하는 타입을 자동 맵핑(확장자를 보고 맵핑을 하는 것 같습니다.)을 해 줍니다.다만 js.map과 같이 json 타입으로 선언되어야 하는 Content type이 S3에서는 바이너리 파일의 기본 포맷인 octet-stream으로 지정이 되므로 배포 후 타입 변경이 필요합니다.

S3 배포시 Sync 명령으로 하게되면 기존 파일이 남아 있어 혼선이 있을 수 있어 삭제 후 배포를 진행했습니다.(서비스 특성에 따라 선택하시면 됩니다.)
위의 내용을 정리하면 아래와 같은 스크립트가 만들어 집니다.

![](/files/post/2021-10-08-aws-Serverless-webservice/5-1.png){: width="100" height="200"}


위와 같이 간단하게 웹서비스를 만드는 방법을 포스팅해 봤습니다. 

제일 중요한 부분은 구성하려는 서비스가 CF나 S3에서 제공하는 기능으로 가능한 지 충분한 테스트를 통해 구성해야 합니다.
이 부분만 충분히 테스트가 된다면 부하가 몰렸을 때나 비용적인 부분 그리고 관리적인 부분에서 매력적인 구성인 것 같습니다.

감사합니다.

