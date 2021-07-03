---
title: 스프링 mvc 프로젝트 + AWS EC2에 빌드까지(9) 배포시작
date: 2021-06-09 11:00:00 +0900
categories: [Spring, spring_project]
tags: [spring,spring mvc,springboot,springframework,java]
---

# 이전 글

[이전글](https://vitriol95.github.io/posts/mvcproject8/) 에서는 최종 코드를 정리해 보았고, 이번시간부터는 직접 AWS EC2에 빌드해보도록 하겠다. AMI는 아마존 Linux 2를 사용하였다.

---

# EC2 생성 및 접속
Amazon Linux 2 를 인스턴스 AMI로 사용하였고, 필자의 OS는 윈도우 이므로(맥을 꼭 사야겠다..) SSH로 원격 터미널 접속을 도와주는 putty프로그램을 이용해 조작할 예정이다.

<br>
접속에 성공하였다면, 가장 먼저 리눅스 내부에 자바를 설치해주어야 한다. 프로젝트의 jdk 버전은 11에 해당하므로 이에 맞는 버전을 설치해주어야 한다. 따라서 리눅스 콘솔창에 다음과 같이 입력해준다.

```
sudo amazon-linux-extras install open-jdk11
```

이후에, 혹시 amazon linux에서 제공하는 다른 jdk가 깔려있는지 확인하기 위해 자바 버전을 살펴보도록 하자. (java 환경 변수를 지금 설치한 jdk11으로 바꾸어놓자.)

```
sudo /usr/sbin/alternatives --config java
```

다음과 같이 창이 뜬다. 아까 설치한 1개의 Java만이 깔려있었고, 이는 이미 현재 버전값으로 사용하고 있으므로 넘어가면 된다.

<img src="/assets/img/mvcproject/37.JPG">

마지막으로 버전을 확인해 보자

```
java -version
```

<img src="/assets/img/mvcproject/38.JPG">

성공적으로 설치및 적용이 되었다.

## 타임존 설정

이후, EC2 서버의 타임존을 한국으로 변경해야 한다. 현재 Ec2내 시각을 확인하고 싶다면 `date`명령어만 입력하면 바로 알 수 있다. 이를 변경하기 위해 다음과 같은 작업을 실시하였다.

```
sudo vi /etc/sysconfig/clock

ZONE="Asia/Seoul" 로 변경
```
> 관리자 권한으로 들어가야 하는 이유는 clock이라는 파일은 readonly이기 때문이다.

이후 표준 시간대 파일을 찾을 수 있도록 원본 대상파일을 대상 디렉토리에 링크파일을 생성해 둔다. 바로가기 같은 개념이라 볼 수 있다. 마지막으로 적용을 위해 재부팅 해준다.
```
sudo ln -sf /usr/share/zoneinfo/Asia/Seoul /etc/localtime 
sudo reboot
```
> 다시 켜서 date명령어를 이용해 확인하면 시간대가 변경된 것을 확인할 수 있다.

---

## 호스트 네임 변경

[AWS EC2 공식유저가이드](https://docs.aws.amazon.com/ko_kr/AWSEC2/latest/UserGuide/set-hostname.html) 를 참조하였다.

<br>
가장 먼저 호스트 이름 업데이트를 유지하기 위해 다음과 같은 명령어를 수행해준다.

```
sudo vi /etc/cloud/cloud.cfg

preserve_hostname: true << 추가
```

이후 실제 호스트네임을 변경해 보겠다. 이때 아마존 리눅스2에서 제공하는 hostnamectl 명령을 이용하여 진행한다.

```
sudo hostnamectl set-hostname 쓰고싶은이름.localdomain

sudo vi /etc/hosts

127.0.0.1 쓰고싶은이름.localdomain 쓰고싶은이름 localhost4 localhost4.localdomain4

sudo reboot
```
> etc/hosts를 에디터로 열고 항목을 수정해주어야 한다. 이후 재부팅한다. 필자의 경우는 이름을 mvcservice라고 하였다.

<br>
재부팅이 완료되었다면 hostname을 이용해 정규화된 도메인 이름이 표시되는지 까지 확인해 보자

<img src="/assets/img/mvcproject/39.JPG">

잘 적용이 되었다.

---

# RDS 인스턴스 생성 및 적용

RDS인스턴스의 엔진 옵션으로 MariaDB를 적용하도록 하겠다. 이는 Amazon Aurora와도 호환이 잘 되기에 MySQL보다 훨씬 범용적이다.

<img src="/assets/img/mvcproject/40.JPG">

생성이 되었다면 파라미터 그룹을 할당해준다. 필자는 전에 사용하던 파라미터 그룹을 재사용하겠다. 중요한것은 character 항목들을 utf8mb4로 맞추어 주어야 한다는 것이다. (mariaDB는 항상 이게 골칫거리이긴 하다.)

<br>
또한 보안그룹을 편집하여 현재 IP와 EC2에서 접근이 가능하게 끔 설정해둔다.
이후, 로컬에서 잘 열리는지 테스트 해보겠다. 이때는 RDS의 엔드포인트로 접근해야한다.

<br>
필자는 인텔리제이의 database 플러그인을 통해 접속해 보았고, 커넥션이 잘 되고있었다. 이후 새로운 데이터 베이스인 mvcservice를 만들고 이를 use해준다.

<br>
`show variables like 'c%'`을 해보면 아직도 char set이 적용되지 않았다.. 따라서 필자는 다음과 같이 세부적인 수정을 해주었다.

```sql
alter database mvcservice
CHARACTER SET = 'utf8mb4'
COLLATE = 'utf8mb4_general_ci';

show variables like 'c%';

SET character_set_filesystem = 'utf8mb4';
SET character_set_server = 'utf8mb4';
SET collation_server = 'utf8mb4_general_ci';
```

<img src="/assets/img/mvcproject/42.JPG">

이제야 모두 설정되었다..

---

## EC2에서 RDS접근

가장 먼저 EC2에 mysql을 설치해 주어야 한다. 이후 접근해보자.

```
sudo yum install mysql

mysql -u 계정 -p -h 엔드포인트
```

<img src="/assets/img/mvcproject/43.JPG">

접근이 잘되고 있었으며, 앞서 생성한 mvcservice라는 스키마도 존재한다.

---

# 프로젝트 배포하기

필자는 현재 git을 이용해 버전관리를 하고 있으므로, git pull로 파일을 받기 위해 EC2에 git 설치가 필요하다.

```
sudo yum install git
```

이후 잘 받아졌는지, 버전확인을 진행한다.

```
git --version
```

모두 완료가 되었다면, 프로젝트파일이 위치할 디렉토리를 만들어두자. 필자의 경우에는 /app/vanilla 로 진행하였다.

```
mkdir ~/app && mkdir ~/app/vanilla
cd ~/app/vanilla
```

이후 git clone을 따오면 된다.

```
git clone https://github.com/vitriol95/mvc-service.git
```

이제 배포용 스크립트를 만들어보도록 하자. 여기서는 .sh형식의 쉘스크립트 파일을 vim에디터를 이용해 작성해 보겠다.

```
vim ~/app/vanilla/deploy.sh
```

```sh

REPOSITORY=/home/ec2-user/app/vanilla
PROJECT_NAME=mvc-service

cd $REPOSITORY/$PROJECT_NAME/
git pull origin master
./gradlew build

cd $REPOSITORY
cp $REPOSITORY/$PROJECT_NAME/build/libs/*.jar $REPOSITORY/

CURRENT_PID=$(pgrep -f ${PROJECT_NAME}.*.jar)

if [ -z "$CURRENT_PID" ]; then
        echo "어플리케이션 시작"
else
        kill -15 $CURRENT_PID
        sleep 5
fi

JAR_NAME=$(ls -tr $REPOSITORY/ | grep jar | tail -n 1)
nohup java -jar $REPOSITORY/$JAR_NAME 2>&1 &
```
이 배포 스크립트의 흐름은 다음과 같다.

1. 가장 먼저, 경로와 프로젝트 파일이름을 변수에 저장한다. 
2. 다음 폴더로 이동한뒤 git pull origin master 명령을 수행한다.
3. 이후 jar파일로 build를 실시한다. 만들어진 jar파일을 프로젝트 경로 최상단에 복사하여 접근이 쉽도록 만들어 두었다.
4. 이후 현재 다른 어플리케이션이 작동하고 있는지 pgrep 명령을 통해 확인해 본다. 이는 process id를 가져오게 되며 -f 옵션을 주게 되면 프로세스 이름으로 검색한다.
5. 분기문으로 들어가게 된다. if 안에 -z 라는 옵션을 주어 현재 스트링이 비어있는 값이면 true에 해당한다. 즉, true가 되는 상황은 "CURRENT_PID"가 없는 상황이므로 바로 실행시키면 된다.
> else인 경우 kill 명령어를 이용해 현재 구동중인 프로세스를 종료시키고 실행하도록 한다.
6. 다음으로는 $()를 이용해 정의된 명령을 수행하고 리턴값을 JAR_NAME에 담아오게 된다. 이때 정의된 명령은 ls이며 -t와 -r옵션을 줌으로써 생성(수정)시간의 역순으로 출력하게 한다.
> 이후 grep 명령어를 통해 jar이라는 문자가 포함된 파일을 가져온다. 이어서 tail명령어를 통해 뒤에서 1개줄을 가져오게 된다.
7. 위의 과정을 통해 얻어진 jar 파일을 java명령어를 이용해 실행한다. 터미널세션 연결이 끊어져도 지속적인 동작을 위해 nohup명령어도 넣어주었다. 이때 로그는 nohup.out이라는 파일에 찍히게 된다.
> 따라서 맨 뒤에 & 명령을 넣어 background 실행을 할 수 있게 해야하며, 2>&1 옵션을 주어 표준에러(2)를 로그의 표준 출력(1)에 넣어준다. 이렇게 되면 에러메세지는 로그파일에만 남게된다.

<br>
이후 application-real-db.yml 파일을 vim에디터를 이용해 작성해준다. 최상위 디렉토리인 /app에 만들어 주었다.

```yaml
// application-real-db.yml
spring:
  datasource: 
    url: 엔드포인트 및 포트, DB
    username: 
    password: 
    driver-class-name: org.mariadb.jdbc.Driver
  jpa:
    hibernate:
      ddl-auto: none
    properties:
      hibernate:
        default_batch_fetch_size: 1000

server:
  tomcat:
    max-http-form-post-size: 5MB

```

그리고 deploy.sh 파일을 다음과 같이 수정해 준다.

```sh
vim deploy.sh

# previous code
nohup java -jar -Dsping.config.location=classpath:/application.yml,/home/ec2-user/app/application-real-db.yml -Dspring.profiles.active=real-db -jar $REPOSITORY/$JAR_NAME 2>&1 &
```
config 파일에 원래있던 application.yml과 함께 vim 에디터를 이용해 만든 yml파일을 추가해준다. 이후, active를 후자로 설정해 둔다.
> 이 안에는 엔드포인트나, RDS의 이름과 패스워드가 들어가있는 비밀정보에 해당하므로 노출시켜선 안된다.

<br>
이후 ./deploy.sh를 실행하여 실제 프로그램이 EC2 환경에서 잘 돌아가는지 확인하면 된다.

<img src="/assets/img/mvcproject/44.JPG">

잘 된다!. (개발자가 가장 만족도를 느끼는 순간이 아닐까...)
<br>
다음시간에는 Travis CI를 이용하여 자동 배포를 해보고, 이후 Nginx를 이용하여 중단없는 배포까지 한번 실행해 볼 예정이다. 