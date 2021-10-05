---
title: 엔진엑스 + Configuration
date: 2021-10-01 12:00:00 +0900
categories: [DevOps,nginx]
tags: [nginx,webServer,configuration,proxy]
---

## Nginx

### Nginx로 할 수 있는 것들
1. WS(web-server)의 역할
  - 정적인 파일을 처리하는 HTTP서버로서의 역할을 할 수 있다. 
  - 이번 프로젝트에서는 이 역할이 주다. (Spring Boot - tomcat을 통한 WAS이전의 WS를 두는 용도)
  - 또한 Nginx는 비동기 처리 방식 (event-driven)을 채택해 사용중이다.

2. 리버스 프록시
  - 클라이언트가 가짜 서버에 요청을 하면, Nginx가 배후 서버로부터 데이터를 가져오는 것이다.
  - 웹 응용프로그램 서버에 리버스 프록시를 두는 이유는 요청에 대한 버퍼링 때문이다. 클라이언트가 직접 APP 서버에 요청하는 경우, 프로세스 1개가 응답 대기 상태가 되어야만 한다.
  - 따라서 프록시 서버를 둠으로써 요청을 배분하는 역할을 한다.
  - 이전에, 한번 사용을 해본적이 있다. (conf 파일에서 location 지시어를 이용해 요청을 배분한다)

3. 로드 밸런싱

### Configuration(nginx.conf)
1. 디렉토리 위치
  - 기본 값으로 /etc/nginx/ 아래에 위치하고 있다.
  - find / -name nginx.conf 명령어를 통해 추적할 수도 있다.
  - 이 외에 include 시켜 사용하는 conf(주로 서버정보들)은 /etc/nginx/conf.d 아래에 위치하고 있다.
> 기본값인 default.conf가 존재한다.

2. 기본적인 튜닝
> nginx.conf 파일의 경우 root계정만 수정이 가능하다.

- 최상단 (CORE)
  - user : Nginx 프로세스가 실행되는 권한을 의미한다.
  - worker_process : Nginx에서 실행 가능한 프로세스 수이다. (auto로 설정해도 무방하다)
  > nginx는 마스터와 워커 프로세스로 나뉘는데, 이중 워커 프로세스가 실질적인 웹서버 역할을 수행하게 된다.
  - error_log : 에러로그가 찍히는 위치와 수준을 정해 둘 수 있다.
  > levels: debug, info, notice, warn, error, crit, alert, emerg
  - pid : 실행되는 Nginx의 process id가 저장되는 위치에 해당한다.(/var/run/nginx.pid)

- 이벤트 블락 = events {}
  - 앞서 이야기 하였듯, event-driven방식으로 동작하기에 이에 대한 옵션을 설정할 수 있다.
  - worker_connections : 하나의 프로세스가 처리할 수 있는 커넥션 수를 의미한다.
  > 앞선 worker_process에 곱하면 허용 가능한 최대 접속자수에 해당한다.

- Http 블락 = http {}
  - keepalive_requests : nginx에서 캐싱할 커넥션 수에 해당한다. 초과한 연결 요청이 들어오면 LRU에 따라서 과거 커넥션 부터 해제한다.
  > 당연히 많이 설정할 수록 부담이 될 수 있다.
  - keepalive_timeout : 접속시에 커넥션을 몇 초동안 유지할 것인지에 대한 설정값이다.
  > 이 값이 높게 되면 클라이언트 입장에서는 좋지만, 서버에서는 요청도 없으면서 커넥션을 너무 많이 맺게 된다.
  > 한 페이지를 로딩하는 데 걸리는 시간 보다 조금 길게 잡으면된다. 나는 default값인 10초로 설정해 두었다.
  - server-token : nginx의 버전을 드러낼 것인가에 대한 항목이다. 나는 off로 설정해 두었다.
  - types_hash_max_size, server_names_hash_bucket_size : 호스트의 도메인 이름에 대한 공간을 설정하는 곳이다.
  > 이 값이 낮을 경우 가상 호스트 도메인을 등록한다거나, 도메인 이름이 길 경우 bucket공간의 에러가 날 수 있으므로 넉넉하게 설정한다.
  > 전자의 경우 기본값이 512, 후자의 경우 32에 해당한다.
  - access_log : error_log와 마찬가지의 값에 해당한다.
  - sendfile : nginx에서 정적 파일을 보내도록 설정하는 옵션이다 (on/off)
  - tcp_nopush : 클라이언트로 패킷이 전송되기 전에 버퍼가 가득 찼는지 확인하여, 다 찼으면 패킷을 전송하도록 하여 네트워크 오버헤드를 줄이도록 한다 (on/off)
  > 실제 진행하면 성능 향상이 있을 수 있지만, 때로는 오히려 저하가 발생할 수 있어서 테스트가 꼭 필요하다. 고민하기 좋은 옵션이다.
  - tcp_nodelay: 활성화 하면 소켓이 패킷 크기에 상관없이 버퍼에 데이터를 보내도록 한다.
  > [이글](https://thoughts.t37.net/nginx-optimization-understanding-sendfile-tcp-nodelay-and-tcp-nopush-c55cdd276765)에서는 sendfile과 함께 위의 두가지 항목에 대해 권장하고 있다.
  - gzip: 웹서버에서 문서 요청에 대한 응답을 gzip 압축 전송에 대해 설정을 한다.

- 서버 블락 = http{ server{ } }
  - listen: 서버의 리스너 포트를 지정한다.
  - server_name: 서버의 이름을 설정하는 곳이다. 주로 도메인이 등록된다.
  - location (path) : 해당 path로 들어오는 요청에 대한 설정에 해당한다. 주로 프록시 패턴에 사용된다.
  > root는 linux file system에서 디렉토리 경로를 명시하며, index로 해당 위치의 index문서를 설정할 수 있다. 
  - error_page (number) [page.html] : 해당 에러num에 대한 페이지를 설정해 줄 수 있다.

3. 프록시 설정하기

NGINX가 request들을 프록시 할때, 요청을 proxied server로 보내게 되며 response를 fetch해오게 된다. 그리고 클라이언트에게 이 응답값을 전송해준다. 프록시 서버가 굳이 HTTP 서버가 아니라도 응답값을 내려준다면 프록시서버로 사용할 수 있다.
> 언제 사용하는 것이 좋을까? 를 생각해보았는데 만약 example.com 이라는 도메인이 A라는 서버에 물려있고 example.com/api 가 B 서버에 물려있다면 리버스 프록시 기능을 사용하는 것이 좋을 것 같다.

- location /some/path { properties value }
  - proxy_pass : 프록시 서버의 link를 전달해야 한다. 만약 부트의 톰캣이 내부의 8080포트에서 구동중이라면 http://127.0.0.1:8080으로 작성하면 된다.
  > 만약 /some/path/request로 접속을 하게 되면 앞의 /some/path는 http://127.0.0.1:8080 로 대체되며 이후 http://127.0.0.1:8080/request로 이동하게 된다.
  - proxy_set_header : 기본값으로 NGINX는 두가지 필드에 대해 재정의한다. (Host - $proxy_host, Connection - close)
  > proxy_set_header Host $host 로 바꾸게 되면 실제 호스트의 값을 헤더로 삼아 req를 진행할 수 있게 된다.
  > proxy_set_header X-Real-IP $remote_addr 처럼 실제 접속자의 IP를 X-Real-IP 헤더에 입혀서 전송도 가능하다.
  - 이외에 proxy_set_header foo foo 처럼 사용자 정의 프록시 헤더를 집어 넣어줄 수 있다.
  > 기존 헤더를 지우거나 할땐 foo "" 처럼 empty string을 넘겨주면 된다.

4. 로드밸런싱 = http { upstream { } }

upstream이라는 말 그대로 상류로부터 여러 서버에 요청을 분산시켜 내려줌을 뜻한다. 기본 값으로는 라운드로빈으로 동작한다. 
> 자주 쓰이는 방식으로, ip_hash 방식도 있다고 한다. ip값을 해시시켜 나누어주는데, 한번 접속했던 아이피는 계속 같은 서버를 쓰게 된다.

- server (ipaddr + port) [option] 으로 서버를 설정해 줄 수 있다.
> ex) server 127.0.0.1:8080 weight=3 max_fails=5 fail_timeout=30s

    - 앞서 location의 proxy 설정에서 proxy_pass를 이 upstream의 이름으로 전달해주면 알아서 적어준 방식대로 lb를 실행하게 된다.
    - 옵션을 풀어보자면 다른 서버 대비 3배만큼 빈도수를 할당해 준다는 것이며 30초 동안 응답하지 않은 상태가 5번 지속되면 죽은 것으로 간주한다.

[사용 예시 - nginx official docs](https://www.nginx.com/resources/wiki/start/topics/examples/loadbalanceexample/)



#### 참고 - [출처](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/)
