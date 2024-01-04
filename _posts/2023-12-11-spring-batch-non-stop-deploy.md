---
layout: post
title: 심볼릭 링크로 스프링 배치 무중단 배포하기
author: 박지훈
date: 2023-12-11
tags: [spring-batch, deploy, linux]
---

안녕하세요. 11번가 클레임개발팀 박지훈입니다.

11번가에서는 전사 배치 서버가 있고, 각 팀별로 팀 전용 배치 서버를 추가로 관리하기도 합니다.<br/>
(최종 목표는 모든 팀이 함께 관리하는 레거시 배치를 각 팀 전용 배치로 이관하는 것입니다.)

클레임개발팀에서는 한 대의 서버로 운영되는 팀 배치 서버를 추가로 관리하고 있고,<br/>
Spring Batch Job(이하 Job) 스케줄러는 Jenkins 툴을 사용하여 Job 들을 주기적으로 실행시켜 주고 있습니다.

평화롭던 어느 날..🌞

![출처: https://medium.com/rta902/kermit-the-frog-from-muppets-to-memes-f6fea5be3cf1](/files/post/2023-12-11-spring-batch-non-stop-deploy/kermit-01.jpeg)

팀 배치 서버에서 한 가지 문제를 발견하게 되었습니다.<br/>
Job 수행을 위해 jar 파일을 실행하는 도중 배포가 진행될 경우, **jar 파일이 변경(업데이트, 제거)되면서 에러가 발생**하는 문제입니다.

이러한 이슈를 해결하기 위해 스프링 배치 무중단 배포를 적용하게 된 과정을 공유해 드리고자 합니다.

## Contents

- [Contents](#contents)
- [문제 상황](#문제-상황)
- [기존 배포 방식](#기존-배포-방식)
- [아이디어](#아이디어)
- [Symbolic link](#symbolic-link)
- [적용](#적용)
  - [배포 이후 단계](#배포-이후-단계)
  - [Job 실행 단계](#job-실행-단계)
  - [적용 결과](#적용-결과)
- [마무리](#마무리)

---

## 문제 상황

아래와 같은 상황에 처하게 되면 `java.lang.NoClassDefFoundError` 또는 `java.lang.ClassNotFoundException` 예외가 터지면서 비정상적으로 배치가 실패하거나 중단되는 현상이 발생하였습니다.
- Job 실행 중일 때 배포 진행
- 빌드&배포 중일 때 Job 실행

마주했던 문제의 로그 일부입니다.

```shell
Exception in thread "main" java.lang.reflect.InvocationTargetException
...
Caused by: java.lang.NoClassDefFoundError: ch/qos/logback/classic/spi/ThrowableProxy
...
Exception in thread "SpringApplicationShutdownHook" java.lang.NoClassDefFoundError: ch/qos/logback/classic/spi/ThrowableProxy

...
```

이제 열심히 서칭해야 할 시간입니다.  

![출처: https://medium.com/rta902/kermit-the-frog-from-muppets-to-memes-f6fea5be3cf1](/files/post/2023-12-11-spring-batch-non-stop-deploy/kermit-02.gif)

[stack overflow](https://stackoverflow.com/questions/32477145/java-lang-classnotfoundexception-ch-qos-logback-classic-spi-throwableproxy) 에서 유사한 사례를 발견하게 되었는데, 서론에서 언급했듯이 Job 수행을 위해 jar 파일을 실행하는 도중 배포가 진행될 경우, **jar 파일이 변경(업데이트, 제거)되면서 에러가 발생**한다는 것을 알게 되었습니다.

---

## 기존 배포 방식

개선 방법을 공유하기 전에 팀 배치 서버의 배포 방식을 간단하게 소개하고 가면 좋을 것 같습니다.

> **1) 빌드&배포**
> - 클레임개발팀의 배치 서버 빌드, 배포는 **사내 배포 시스템**을 사용하고 있습니다.
> - 사내 배포 시스템을 통해 **특정 브랜치를 빌드**하고, **특정 경로(Deploy Path)에 빌드된 파일을 배포**하게 됩니다.
> 
> **2) Job 실행**
> - Jenkins 툴의 `Build periodically > Schedule` 설정에 따라 주기적으로 `Execute Shell` 명령으로 Job을 실행시키고 있습니다.
> 
> ![Jenkins Execute Shell](/files/post/2023-12-11-spring-batch-non-stop-deploy/image01.png)

`run_job.sh` 파일을 살짝 보면 단순하게 `Execute Shell` 에 명시된 jobName, jobParameter 정보를 가져와서 Job을 실행하는 역할을 하고 있습니다.
 
**/app/batch/shell/run_job.sh**

```shell
#!/bin/bash

# Read jobName, jobParameter
jobName=$1
jobParameters=""
args=("$@")
for arg in "$@"; do
  if [[ $arg == $1 ]]; then
    continue
  fi
  jobParameters+=" $arg"
done

# Run batch job
PROFILE="prod"
JAVA_OPTS="-Xms512m -Xmx1024m"
$JAVA_HOME/bin/java -jar -Dspring.profiles.active=$PROFILE /app/deploy/batch/batch-0.0.1-SNAPSHOT.jar $JAVA_OPTS --job.name=$jobName $jobParameters
```

---

## 아이디어

본론으로 돌아와서, 기존 배포 방식에서 어떤 아이디어로 개선을 진행하게 되었는지 살펴보겠습니다.

스프링 배치 무중단 배포는 **심볼릭 링크**를 활용하였습니다.<br/>
(심볼릭 링크 아이디어는 향로님의 [Spring Batch 공통 설정 관리하기](https://jojoldu.tistory.com/445) 글을 읽으면서 얻게 되었습니다.)

사내 배포 시스템을 사용하다 보니 빌드&배포는 기존 방식과 동일하고 **배포 이후**와 **Job 실행** 단계에 심볼릭 링크를 활용하여 스프링 배치 무중단 배포를 적용하는 전략을 세우게 되었습니다. 

> **1) 빌드&배포 (기존 방식과 동일)**
> - 사내 배포 시스템을 통해 **특정 브랜치를 빌드**하고, **Deploy Path에 빌드된 파일 배포**하기.
> 
> **2) 배포 이후 단계**
> - Deploy Path에 배포된 jar 파일을 **새로운 디렉토리로 복사**하기.
> - 기존 링크를 해제하고 **새로운 디렉토리 경로에 복사된 jar 파일로 링크 연결**하기.
> 
> **3) Job 실행 단계**
> - 심볼릭 링크가 연결되어 있는 원본 파일명을 가져오는 **readlink 명령어를 활용하여 새로 배포된 jar 파일로 Job 실행**하기
> - **기존 jar 파일은 변경(업데이트, 제거)되지 않고 유지**되므로 문제의 상황 해결 기대

![Idea](/files/post/2023-12-11-spring-batch-non-stop-deploy/image05.png)

---

## Symbolic link

리눅스에서 [ln](https://www.ibm.com/docs/en/aix/7.2?topic=l-ln-command) 커맨드는 파일/디렉토리 링크를 생성하는 기능을 가지고 있습니다.<br/>
기본적으로 `ln` 커맨드는 하드 링크(Hard Link)를 생성하고, `-s` 옵션으로 심볼릭 링크(Symbolic Link, Soft Link)를 생성할 수 있습니다.

```shell
ln [ -s ] [대상 파일/디렉토리 경로] [링크 파일/디렉토리 경로]
```

하드 링크와 심볼릭 링크를 간략하게 살펴보겠습니다.

> **심볼릭 링크**
> - 윈도우의 바로가기와 유사한 기능
> - 링크 파일은 대상 파일에 대한 참조를 가지고 있어서 링크 파일을 대상 파일처럼 사용 가능
> - 대상 파일이 삭제될 경우 링크 파일 사용 불가
>
> **하드 링크**
> - 파일 복사와 유사한 개념
> - 원본 파일과 동일한 inode
> - 원본 파일이 삭제되어도 링크 파일 사용 가능

심볼릭 링크도 간략히 알아보았으니 이제 적용해 보겠습니다.

---

## 적용

### 배포 이후 단계

**switch-link.sh**

```shell
#!/bin/bash

DEPLOY_PATH=/app/deploy/batch
DIRECTORY_NAME=batch-$(/bin/date +%Y%m%d%H%M%S)
# 1) 새로운 디렉토리 생성
mkdir /app/deploy/batch/${DIRECTORY_NAME}
# Deploy Path에 배포된 jar 파일을 새로운 디렉토리로 복사하기.
cp -f ${DEPLOY_PATH}/batch-0.0.1-SNAPSHOT.jar ${DEPLOY_PATH}/${DIRECTORY_NAME}/ 

echo "> $DIRECTORY_NAME Directory has been created."
echo "> new jar file was copied to a new directory."

BEFORE_JAR_PATH=$(readlink /app/batch/shell/application.jar)
# 2) 새로운 디렉토리 경로에 복사된 jar 파일로 링크 변경하기.
ln -Tfs ${DEPLOY_PATH}/${DIRECTORY_NAME}/batch-0.0.1-SNAPSHOT.jar /app/batch/shell/application.jar

echo "> Link switched from $BEFORE_JAR_PATH to $DEPLOY_PATH/$DIRECTORY_NAME."

# 이후 추가될 쉘 파일
sh /app/batch/shell/remove-old-directories.sh
```

참고 1. 새로운 디렉토리 생성

> `mkdir batch-$(/bin/date +%Y%m%d%H%M%S)` 명령으로 아래와 같이 날짜 정보로 디렉토리를 생성할 수 있습니다. 
> 
> ![Result mkdir command](/files/post/2023-12-11-spring-batch-non-stop-deploy/image02.png)

참고 2. 심볼릭 링크 변경

> `ln -Tfs TARGET LINK` 명령으로 링크를 변경할 수 있습니다.
> - `-T option`: --no-target-directory  treat LINK_NAME as a normal file
>   - 링크 파일을 일반 파일처럼 다루는 옵션
> - `-f option`: --force remove existing destination files
>   - 심볼릭 링크가 이미 존재할 경우 덮어쓰는 옵션
> - `-s option`: --symbolic make symbolic links instead of hard links
>   - 심볼릭 링크 파일 생성 옵션

참고 3. 실행 결과

> `./switch-link.sh` 명령으로 위에 작성한 쉘 파일을 실행해 보면 아래와 같이 디렉토리 생성, jar 파일 복사, 링크 스위칭이 정상적으로 동작하는 것을 확인할 수 있습니다.
> 
> ![Result Execute Shell](/files/post/2023-12-11-spring-batch-non-stop-deploy/image03.png)

...

여기서 잠깐! ✋🏼<br/>
배포할 때마다 새로운 디렉토리와 jar 파일이 계속 쌓이게 될 텐데요.<br/>
계속 생성되는 jar 파일로 서버 용량이 초과하는 문제를 방지하기 위해 최근 배포된 10개의 디렉토리만 남기고 전부 삭제해 주려고 합니다.
   
**remove-old-directories.sh**

```shell
#!/bin/bash

# 1) 배포 경로에 생성된 디렉토리 개수
DIRECTORY_COUNT=$(ls -d /app/deploy/batch/*/ | wc -l)

# 디렉토리가 10개보다 많이 존재할 경우
if [ $DIRECTORY_COUNT -gt 10 ]
then
  # 2) 제거할 디렉토리 개수 카운트
  REMOVE_TARGET_COUNT=$(( ${DIRECTORY_COUNT} - 10))
  # 3) 오래된 디렉토리부터 제거할 디렉토리 개수만큼 추출
  REMOVE_TARGET_LIST=$(ls -dltr /app/deploy/batch/*/ | head -$REMOVE_TARGET_COUNT | awk '{print $9}')

  # 제거 대상 디렉토리 제거
  for file in ${REMOVE_TARGET_LIST}
  do
    echo "remove $file"
    /usr/bin/rm -rf ${file}
  done
fi
```

흐름은 아래와 같습니다.

> 1) 배포 경로에 존재하는 디렉토리(jar 파일이 담긴) 개수 카운팅
>
> 2) 10개의 디렉토리를 제외하고 제거할 디렉토리의 개수 카운팅
>
> 3) 제거할 디렉토리 목록 추출
> - 오래된 순으로 제거하기 위해 ls 명령어의 `-t`, `-r` 옵션 사용
> - `-t option`: 파일과 디렉토리를 최근 시간 기준 내림차순 정렬
> - `-r option`: 정렬된 데이터의 순서를 오름차순으로

위에서 생성한 `remove-old-directories.sh` 파일을 실행하기 전과 후를 비교해 보면 최근 10개의 디렉토리를 제외한, 오래된 디렉토리들이 삭제된 것을 확인할 수 있습니다.

![Result Execute Shell](/files/post/2023-12-11-spring-batch-non-stop-deploy/image04.png)

배포 이후 **remove-old-directories.sh** 쉘도 동작할 수 있도록 **switch-link.sh** 쉘 마지막 줄에 실행 커맨드를 추가해 줍니다.

```shell
sh /app/batch/shell/remove-old-directories.sh
```

이제 마지막으로 Job 실행 단계만 남았습니다.

### Job 실행 단계

![Jenkins Execute Shell](/files/post/2023-12-11-spring-batch-non-stop-deploy/image01.png)

`Jenkins Execute Shell`에서 입력된 jobName, jobParameter를 읽는 부분은 기존과 동일하고, 

readlink 명령어만 추가해 주면 심볼릭 링크가 연결되어 있는 jar 파일 경로를 가져올 수 있게 됩니다.

**/app/batch/shell/run_job.sh**

```shell
#!/bin/bash

# Read jobName, jobParameter
jobName=$1
jobParameters=""
args=("$@")
for arg in "$@"; do
  if [[ $arg == $1 ]]; then
    continue
  fi
  jobParameters+=" $arg"
done

# Run batch job
PROFILE="prod"
JAVA_OPTS="-Xms512m -Xmx1024m"
# 심볼릭 링크가 연결되어 있는 jar 파일 경로 가져오기
ORIGIN_JAR=$(readlink /app/batch/shell/application.jar)

echo "> ORIGIN_JAR_PATH: ${ORIGIN_JAR}"

$JAVA_HOME/bin/java -jar -Dspring.profiles.active=$PROFILE ${ORIGIN_JAR} $JAVA_OPTS --job.name=$jobName $jobParameters
```

### 적용 결과 

![Jenkins Execute Shell](/files/post/2023-12-11-spring-batch-non-stop-deploy/image06.png)

> 1) 배포된 jar 파일을 보관해 둘 디렉토리 생성
> 
> 2) 생성한 디렉토리에 배포된 jar 파일 복사
> 
> 3) 심볼릭 링크를 배포된 jar 파일 경로로 스위칭
> 
> 4) jar 파일이 담긴 오래된 디렉토리 제거

---

## 마무리

심블릭 링크를 활용하여 스프링 배치 무중단 배포를 적용하면서 쉘 스크립트와 장난도 치면서 즐겁고 유익한 시간을 가질 수 있었습니다.<br/>
나름의 여러 고민과 탐색 끝에 적용한 방식이지만, 분명 더 좋은 개선 방법도 있을 것으로 생각합니다.<br/>
읽으시면서 궁금하신 사항이나 개선 사항이 있다면 언제든 아래 코멘트 부탁드립니다.<br/>
글을 읽어주신 모든 분께 감사드립니다. 🙇🏻‍

## Reference

- [https://jojoldu.tistory.com/445](https://jojoldu.tistory.com/445)
- [https://www.ibm.com/docs/en/aix/7.2?topic=l-ln-command](https://www.ibm.com/docs/en/aix/7.2?topic=l-ln-command)
- [https://madplay.github.io/post/what-is-a-symbolic-link-in-linux](https://madplay.github.io/post/what-is-a-symbolic-link-in-linux)