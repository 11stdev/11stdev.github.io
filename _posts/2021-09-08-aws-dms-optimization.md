---
layout: post
title: 'AWS 데이터이관도구 DMS Tips'
author: 강세훈
date: 2021-09-08
tags: [aws, data]
---

클라우드!  
저는 10년 전만 해도 클라우드 하면, 사진 저장할 때 쓰는 인터넷 외장하드인줄 알았습니다.  
그런데 이제는 전 산업에서 클라우드 전환이 트렌드가 되며, IT인프라도 클라우드를 적극 활용하는 시대가 되었네요!  
기술트렌드를 빠르게 캐치하고 있는 11번가도 클라우드의 여러 장점을 활용하기 위한 노력이 계속되어 왔습니다.  
본 포스팅에서는 그 중, 데이터베이스를 AWS 클라우드로 이관한 내용입니다.  

#### 목차
* [1.마이그레이션요건](#01mig)  
* [2.이관을 수행하는 요소 인스턴스, 테스크](#02task)  
* [3.인스턴스와 방화벽](#03firewall)  
* [4.인스턴스 늘리기](#04instance)  
* [5.테스크 늘리기](#05task)  
* [6.히든기능 parallel-load](#06parallel)
* [7.CLI](#07cli)
* [8.노가다를 줄여주는 매크로](#08macro)


# DMS!
Data Migration Service(이하 DMS)는 AWS에서 자랑하고 있는 데이터 마이그레이션 서비스입니다.  
AWS 웹콘솔을 통해 GUI환경으로도 쉽게 접할 수 있고, 숨겨진 Hidden 설정 기능도 있어서 세부적으로 조절도 가능합니다.  
DMS를 사용했던 주요 경험을 풀어보고자 합니다.  

# 1.마이그레이션 요건<a id='01mig'></a>
On-premise 환경에 구축이 되어 있는 데이터베이스를 AWS 서비스로 옮기는 요건이 대표적이었습니다.  

|#|소스|타겟|
|-|---|---|
|1|On-prem Oracle	|AWS RDS Aurora MySQL|
|2|On-prem SQL Server	|AWS RDS SQL Server|
|3|On-prem Exadata	|AWS S3|


# 2. 이관을 수행하는 요소 인스턴스, 테스크<a id='02task'></a>
![](/files/post/2021-09-08-aws-dms-optimization/image01.png){: width="100" height="200"}  

DMS는 내부적으로 EC2 인스턴스를 활용해서 이관하므로 복제인스턴스가 필요합니다.  
그리고 인스턴스에는 하나 또는 여러개의 테스크를 할당 시킬 수 있는데요, 하나의 테스크에는 단일 또는 여러 테이블을 지정할 수 있습니다.  

# 3. 인스턴스와 방화벽<a id='03firewall'></a>

아무래도 이관이다보니 네트워크 연결이 중요합니다. 이를 위해 방화벽 설정이 선행되어야 합니다.  
DMS는 복제인스턴스가 source와 target을 이어주도록 엔드포인트 기능을 제공합니다.  
대부분의 기업에서는 보안을 위해 IP단위로 방화벽을 통제할텐데요, 복제인스턴스의 IP로 방화벽을 열게 됩니다.  

![](/files/post/2021-09-08-aws-dms-optimization/image02.png){: width="100" height="200"}  

그런데, 복제인스턴스의 IP로만 방화벽을 열었다면, 인스턴스를 삭제한 후 다시 생성할 때 IP가 바뀌어 다시 방화벽 설정이 필요해져버립니다.

![](/files/post/2021-09-08-aws-dms-optimization/image03.png){: width="100" height="200"}  

따라서 좀 더 효율적으로 관리하기 위해서는 서브넷 그룹을 사용하여 사용할 IP대역대를 파악하고 방화벽을 설정하는 것이 좋습니다.

![](/files/post/2021-09-08-aws-dms-optimization/image04.png){: width="100" height="200"}  


# 4. 인스턴스 늘리기<a id='04instance'></a>
전략적으로 복제 인스턴스를 여러 대 사용해서, 마이그레이션 단위를 나누어 관리하는 것도 좋습니다.  
특히 복제인스턴스 자체를 모니터링 할 때 유용합니다.  

![](/files/post/2021-09-08-aws-dms-optimization/image05.png){: width="100" height="200"}  

On-premise Exadata에서 AWS S3로 이관할 때에는 소스 데이터 사이즈가 커서 테스크를 120개로 쪼개었는데요,  
보다 효율적인 모니터링과 관리를 위해서 복제인스턴스를 3대로 나누어 사용했습니다.

# 5. 테스크 늘리기<a id='05task'></a>
복제 인스턴스에 여러 개의 테스크를 할당시킨 후, 전략적으로 이관의 순서를 정하는 것도 고려해볼 수 있습니다.  
On-premise Oracle 데이터베이스에서 AWS RDS Aurora MySQL으로 이관할 때에는, 데이터 성격에 따라 이관 우선 순위를 나누었습니다.

|우선순위|데이터성격|후행작업|LOB처리
|-|---|---|---|
|1|메인 데이터|있음|없음|
|2|메인 데이터|있음|없음|
|3|로그 데이터|없음|없음|
|4|로그 데이터|없음|있음|

# 6. 히든기능 parallel-load<a id='06parallel'></a>
만약 한 테이블이 너무 커서 특정 테스크가 너무 오래 걸릴 때에는 parallel-load 기능을 쓸 수 있습니다.  
이 기능은 숨겨져 있는데요, DMS 테스크 설정 시 아래처럼 json 모드로 바꾼 후 수동으로 입력하여 설정을 줄 수 있습니다.  

![](/files/post/2021-09-08-aws-dms-optimization/image06.png){: width="100" height="200"}  

On-premise SQL Server에서 AWS RDS SQL Server로 이관할 때에는 특정 테이블의 사이즈가 컸습니다.  
그래서 테스크를 잘 나누어도 속도가 최적화 되지 않았는데요, 이 옵션을 통해 전반적인 이관 속도를 향상시켰습니다.  

이 외에도 줄 수 있는 옵션들은 많이 있지만, 기본적으로 DMS는 1:1로 이관하는 것이 유리합니다.  
예를 들어서 이기종DB간 이관 시 date타입을 varchar로 넣고 싶은 경우,  
DMS는 내부적으로 데이터타입을 캐스팅 하기 때문에, 타겟DB의 컬럼사이즈를 초과할 수 있습니다.  

mysql이 소스이고 다음 테이블이 있다고 가정합니다.  
``` sql
create table mysql.source_table ( column_1 datetime; )
```
oracle이 타겟이고 테이블의 구조는 다음과 같습니다.  
``` sql
create table oracle.target_table ( col_1 varchar(20) ) ;
```
이 상태에서 DMS로 이관하면 다음과 같은 오류가 발생합니다.  
```
value too large for column col_1 (actual: 29)
```

여기서 간단히 DMS세팅으로 소스에서 'substr(column_name, 14)'를 읽게 하고 싶지만, 변형할 수 있는 기능은 없습니다.  
대신 "rule-action": "add-column" 옵션으로 타겟DB에 새 컬럼을 만들어서 넣어줄 수는 있습니다.  

그러나 이미 표준화를 시켜놓은 테이블이고, 테이블에 데이터가 이미 많다면 컬럼추가가 부담이 될 수 있는데요,  
이를 위해서 소스 데이터베이스에서 substr(column_name, 14) 컬럼이 포함된 임시테이블을 만든 후, 이관 소스로 활용하는게 좋습니다.  

# 7. CLI<a id='07cli'></a>

테스크를 연속적으로 생성해야할 때에는 웹콘솔 GUI 환경에서 JSON형태의 정책을 복사해서 쓰면 편합니다.  

그런데 더 많은 테스크를 연속적으로 생성해야 할 때에는 이 방법도 너무 노가다가 되는데요,  
이럴 때에는 CLI를 사용하면 보다 편리하게 사용할 수 있습니다.  

우선 DMS테스크정책과 매핑규칙 정책파일을 생성합니다.  
AWS웹콘솔에서 복사해서 .json 파일로 저장할 수도 있습니다.  

![](/files/post/2021-09-08-aws-dms-optimization/image07.png){: width="100" height="200"}  
![](/files/post/2021-09-08-aws-dms-optimization/image08.png){: width="100" height="200"}  

정책파일을 적용한 테스크를 CLI로 생성합니다.  

``` bash
aws dms create-replication-task \
    --replication-task-identifier task-name-01 \
    --resource-identifier reousrces \
    --source-endpoint-arn arn:aws:dms:ap-northeast-2:account_number:endpoint: \
    --target-endpoint-arn arn:aws:dms:ap-northeast-2:account_number:endpoint: \
    --replication-instance-arn arn:aws:dms:ap-northeast-2:account_number:rep:instance_name \
    --migration-type full-load \
    --table-mappings file://folder/dmstask.json \
    --replication-task-settings file://folder/dmstasksetting.json
```    
참조로, CLI로 테스크를 생성할 때,  
클라우드와치로그 활성화를 위해 파라미터에 기본값을 직접 지정하면 버그로 인해서 아래와 같이 오류가 발생했었는데요,  
![](/files/post/2021-09-08-aws-dms-optimization/image09.png){: width="100" height="200"}  

기본값으로 사용할 것이면, 아예 클라우드와치로그 관련된 파라미터를 빼면 됩니다.  
``` bash
"enableLogging":true
  "CloudWatchLogGroup":null, -> 삭제
  "CloudWatchLogStream":null, -> 삭제
```
(이 내용은 곧 업데이트 되어 해결될 예정이라고 답변을 받았었습니다)

# 8. 노가다를 줄여주는 매크로<a id='08macro'></a>
만약 소스DB에서 이관 단위를 결정하기 위해 쿼리하였다면 엑셀메크로를 활용하면 편하게 테스크를 만들 수 있습니다.  
CLI에서 쓰일 테스크 분할 파일을 자동으로 만들어줄 수 있도록 엑셀 메크로를 개발하여 사용했습니다.  

![](/files/post/2021-09-08-aws-dms-optimization/image10.png){: width="100" height="200"}

파일 명, 테이블 명, 컬럼명, 데이터의 범위 값을 입력해주면 .json파일이 생성됩니다.

실행버튼을 누르면 파일이 생성됩니다.  

![](/files/post/2021-09-08-aws-dms-optimization/image11.png){: width="100" height="200"}

매크로에 지정된 소스는 다음과 같습니다.

``` vb
Sub textwrite()

    Dim filename As String
    Dim ruleformat As String
    Dim rng As Range
    Dim cellfilename As Variant, cellid As Variant, cellschema As Variant, celltable As Variant, cellcolumn As Variant, cellstart As Variant, cellend As Variant
    Dim i As Integer

'#1 시트의 범위 파악
    Set rng = ActiveSheet.Range("A3").CurrentRegion

    For i = 4 To rng.Rows.Count + 2
        '#2 셀항목 읽기
        cellfilename = Cells(i, 1).Value
        cellid = Cells(i, 2).Value
        cellschema = Cells(i, 3).Value
        celltable = Cells(i, 4).Value
        cellcolumn = Cells(i, 5).Value
        cellstart = Cells(i, 6).Value
        cellend = Cells(i, 7).Value

        '#3 파일저장위치
        filename = "/Users/tasksetting/" & cellfilename

    '#4 json파일 만들기

        ' chr(13) 줄 바꿈
        ' chr(34) "
        ' chr(58) :
        ' chr(91) [
        ' chr(123) {
        ' chr(125) }
        ' chr(93) ]

        ruleformat = "{" & Chr(10)
        ruleformat = ruleformat & "    " & Chr(34) & "rules" & Chr(34) & Chr(58) & " " & Chr(91) & Chr(10)
        ruleformat = ruleformat & "        " & Chr(123) & Chr(10)
        ruleformat = ruleformat & "            " & Chr(34) & "rule-type" & Chr(34) & Chr(58) & " " & Chr(34) & "selection" & Chr(34) & "," & Chr(10)
        ruleformat = ruleformat & "            " & Chr(34) & "rule-id" & Chr(34) & Chr(58) & " " & Chr(34) & cellid & Chr(34) & "," & Chr(10)
        ruleformat = ruleformat & "            " & Chr(34) & "rule-name" & Chr(34) & Chr(58) & " " & Chr(34) & cellid & Chr(34) & "," & Chr(10)
        ruleformat = ruleformat & "            " & Chr(34) & "object-locator" & Chr(34) & Chr(58) & " " & Chr(123) & Chr(10)
        ruleformat = ruleformat & "                " & Chr(34) & "schema-name" & Chr(34) & Chr(58) & " " & Chr(34) & cellschema & Chr(34) & "," & Chr(10)
        ruleformat = ruleformat & "                " & Chr(34) & "table-name" & Chr(34) & Chr(58) & " " & Chr(34) & celltable & Chr(34) & Chr(10)
        ruleformat = ruleformat & "            " & Chr(125) & "," & Chr(10)
        ruleformat = ruleformat & "            " & Chr(34) & "rule-action" & Chr(34) & Chr(58) & " " & Chr(34) & "include" & Chr(34) & "," & Chr(10)
        ruleformat = ruleformat & "            " & Chr(34) & "filters" & Chr(34) & Chr(58) & " " & Chr(91) & Chr(10)
        ruleformat = ruleformat & "                " & Chr(123) & Chr(10)
        ruleformat = ruleformat & "                    " & Chr(34) & "filter-type" & Chr(34) & Chr(58) & " " & Chr(34) & "source" & Chr(34) & "," & Chr(10)
        ruleformat = ruleformat & "                    " & Chr(34) & "column-name" & Chr(34) & Chr(58) & " " & Chr(34) & cellcolumn & Chr(34) & "," & Chr(10)
        ruleformat = ruleformat & "                    " & Chr(34) & "filter-conditions" & Chr(34) & Chr(58) & " " & Chr(91) & Chr(10)
        ruleformat = ruleformat & "                        " & Chr(123) & Chr(10)
        ruleformat = ruleformat & "                            " & Chr(34) & "filter-operator" & Chr(34) & Chr(58) & " " & Chr(34) & "between" & Chr(34) & "," & Chr(10)
        ruleformat = ruleformat & "                            " & Chr(34) & "value" & Chr(34) & Chr(58) & " " & Chr(34) & Chr(34) & "," & Chr(10)
        ruleformat = ruleformat & "                            " & Chr(34) & "start-value" & Chr(34) & Chr(58) & " " & Chr(34) & cellstart & Chr(34) & "," & Chr(10)
        ruleformat = ruleformat & "                            " & Chr(34) & "end-value" & Chr(34) & Chr(58) & " " & Chr(34) & cellend & Chr(34) & Chr(10)
        ruleformat = ruleformat & "                        " & Chr(125) & Chr(10)
        ruleformat = ruleformat & "                    " & Chr(93) & Chr(10)
        ruleformat = ruleformat & "                " & Chr(125) & Chr(10)
        ruleformat = ruleformat & "            " & Chr(93) & Chr(10)
        ruleformat = ruleformat & "        " & Chr(125) & Chr(10)
        ruleformat = ruleformat & "    " & Chr(93) & Chr(10)
        ruleformat = ruleformat & Chr(125)

        '#5 파일저장
        Open filename For Output As #1

            Print #1, ruleformat

        Close #1

    Next i

  MsgBox rng.Rows.Count - 1 & "개 파일이 생성되었습니다 "

End Sub
```
