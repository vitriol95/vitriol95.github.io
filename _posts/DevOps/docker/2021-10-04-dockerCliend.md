---
title: 도커의 Host - Client 구조
date: 2021-10-04 12:00:00 +0900
categories: [DevOps,docker]
tags: [docker,linux,network]
---

## Docker 원격 이용

Docker는 Sserver-Client 구조의 어플리케이션이라고 하지만, 이를 의식하며 써본적은 드물다. 이번엔 `Server-Client가 떨어져있어도 REST API를 이용해 원격으로 이용` 을 직접 해보며 익혀보자.

<br>

사실, 기존에 사용했던 `docker ps`와 같은 명령어도 결국은 docker서버에 GET /~~/container 으로 Http요청을 보낸 것과 동일한 방식으로 작동한다. 하지만 나는 대부분 docker server(daemon)과 client가 같은 머신내에 존재하고 있어 이를 느끼지 못했었다.

#### 용어 정리하고 들어가기

<img src="/assets/img/docker/11.png">

1. docker host란 docker daemon을 가지고 있으며 실제로 image를 저장하고 container를 돌리는 머신의 의미를 갖는다. 내부에 있는 docker daemon이 클라이언트로부터 명령어를 받아 이를 직접 실행시키는 것이다. 즉 서버에 해당한다.

2. docker engine 이란, client/server application을 통틀어 지칭하는 말이라고 한다.

[출처](https://docs.docker.com/engine/docker-overview/#docker-engine)

official docker docs에 다음과 같은 그림이 나와있다. 

<img src="/assets/img/docker/12.png">

이처럼 client와 server는 같은 host에 존재할 필요도 없고 rest api를 통해 명령을 보낼 수 있다. 이전까지 이용했었던 docker command는 결국 docker Cli를 통해 rest api를 이용한 셈이다. 

> 잠깐, 제일 먼저 도커를 실행할 때 sudo service docker start 와 같은 것은 결국 docker daemon을 키는 것에 해당했다.

#### 막간을 이용한 실험 

- REST API를 사용한다는 것을 와닿게끔 도커를 다음과 같이 실행해보자 
`sudo docker -d -H tcp://0.0.0.0:4243`
- 이후 curl 명령어를 통해 다음과 같이 이미지를 받게 해보자
`curl -X POST http://127.0.0.1:4243/images/create?fromImage=nginx:latest`
- docker images 로 확인을 해보면.. nginx가 존재함을 알 수 있다.

#### 포트열어 원격 통신해보기
- Centos의 경우 /lib/systemcd/system/docker.service를 에디터로 열어 확인할 수 있다.

<img src="/assets/img/docker/13.png">

local에서는 unix 소켓을 이용해 통신하며, 여기에 -H 0.0.0.0:2375를 추가해주면 외부에서 접속할 수 있는 원격통신 호스트를 만들어 주는 것이다.
> 이렇게 하는건 좀 위험하다.. 도커daemon을 완전히 노출시켰기 때문에

- 원격으로 docker daemon에게 명령을 보내는 것도 굉장히 쉽다. 다른 머신에서 docker -H {IP:2375} images 명령어를 보내면 해당서버의 도커데몬이 가지고 있는 image리스트를 보여주게 된다. 이점은 많이 못써봤는데 기회가 있다면 사용해 보아야겠다.