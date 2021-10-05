---
title: 도커의 Cmd vs EntryPoint
date: 2021-10-04 13:00:00 +0900
categories: [DevOps,docker]
tags: [docker,linux,dockerfile]
---

## DockerFile 

Dockerfile을 쓰며 가장 헷갈렸던 점은 CMD와 ENTRYPOINT의 차이에 해당했다. 이 차이점을 느껴보자.

### CMD vs ENTRYPOINT
둘다 docker container가 실행될때 실행되어야 할 커맨드들을 지정할 수 있다. 

<br>

먼저 요약해보자면, ENTRYPOINT는 컨테이너가 시작될때 항상 실행될 명령어를 의미한다. CMD는 이 ENTRYPOINT에 넣을 arguments와 같은 역할을 해줄 수 있다.

#### 예를 들어보자면

`ENTRYPOINT ["/path/dedicated_command"]` 로 지정한 것과 `CMD ["/path/dedicated_command"]` 로 지정한 것의 차이를 보면 된다. 여기서 `docker run -it program command` 를 실행하면 어떻게 될까

- 전자의 경우는 "/path/dedicated_command"뒤의 argument로 command가 붙어 실행된다.
- 후자의 경우는 "command"가 실행된다. (실제로는 default ENTRYPOINT인 /bin/sh -c 뒤의 argument로 실행된다.)
> 즉, CMD의 경우는 재정의가 가능하고 dockerfile에 적힌 CMD는 default argument의 역할만을 하는 것이다.

__The ENTRYPOINT specifies a command that will always be executed when the container starts. The CMD specifies arguments that will be fed to the ENTRYPOINT__ 이문장이 정말 와닿았다. [출처](https://stackoverflow.com/questions/21553353/what-is-the-difference-between-cmd-and-entrypoint-in-a-dockerfile)
<br>

그럼 이 두가지를 이용해 다음과 같은 문장을 짤 수 있다.

```
FROM ubuntu:latest
ENTRYPOINT ["/bin/ping"]
CMD ["localhost"]
```

여기서 `docker run -it ubuntu` 라는 명령어를 주었을때, 자동으로 localhost의 핑테스트를 실행한다. 하지만 `docker run -it ubuntu google.com` 으로 실행을 한다면, CMD에 정의된 localhost는 google.com으로 재정의 되는 것이다.