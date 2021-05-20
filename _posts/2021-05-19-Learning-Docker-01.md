---
layout: article
title: "[ #1 ] Docker 기본 익히기 "
tags: Docker
---

## Image vs Container

### 도커 이미지
도커 컨테이너를 구성하는 파일 시스템과 실행할 어플리케이션 설정을 하나로 합친 것으로, 컨테이너를 생성하는 템플릿 역할을 합니다. 

### 도커 컨테이너
도커 이미지를 실행 함으로써 생성되며, 파일 시스템과 어플리케이션이 구체화되어 실행되는 상태입니다.
컨테이너 내부 에플리케이션과 소통하기위해 5000번 Port로 바인딩 돼어 있습니다.

## 허브로부터 이미지 받아오기
<code>docker pull</code> 커맨드를 사용하여 도커허브에서 이미지를 받아올 수 있습니다.
~~~shell
$ docker pull <image>
~~~


---
## 이미지 실행하기

<code>docker run</code> 커맨드를 사용하여 원하는 이미지를 실행시킬 수 있습니다.

~~~shell
$ docker run <image>
~~~

먼저 해당 이미지가 로컬머신에 있는지 확인합니다. 만약에있다면 해당이미지를 실행시켜주고, 없다면 Docker hub 에서 pull 해와서 실행 시켜줍니다.

+ 디폴트로 attatched모드(해당 쉘이 닫히면 서버도 종료되는)로 실행됩니다.
+ -d 옵션을 추가함으로써 백그라운드에서(쉘이 종료되도 서버는 유지되게) 실행되게 할 수 있습니다.
+ --rm 옵션을 추가함으로써 해당 컨테이너를 멈춤과동시에 삭제시킬 수 있습니다.

## 현재 실행중인 컨테이너 리스트
<code>docker ps</code>커맨드를 사용하면 현재 실행중인 컨테이너 목록을 보여줍니다.
~~~shell
$ docker ps
~~~

-a 옵션을 추가하면, 실행되고 있는 모든 컨테이너들의 리스트를 보여줍니다. 

~~~shell
$ docker ps -a
~~~

## 로컬머신과 네트워크 바인딩하기
~~~shell
$ docker run -p 6000:6379 redis
~~~
-p옵션으로 로컬머신과 컨테이너를 바인딩 할 수 있습니다.
~~~shell
$ docker ps    
CONTAINER ID   IMAGE     COMMAND                  CREATED              STATUS              PORTS                                       NAMES
14eb691f51ad   redis     "docker-entrypoint.s…"   About a minute ago   Up About a minute   0.0.0.0:6000->6379/tcp, :::6000->6379/tcp   flamboyant_khorana
~~~

<code>docker ps</code> 커맨드를 이용하여 실행되고있는 컨테이너를 보면
redis컨테이너는 6379포트로 liten중이고 해당 컨테이너의 6379포트는 로컬머신의 6000포트와 바인딩 되어있음을 확인 할 수 있습니다.


## 컨테이너에 이름부여하기

컨테이너 생성시 이름을 --name "원하는이름" 을 적어주면 원하는 이름으로 컨테이너 생성이 가능합니다.<br>
이름을 지정하지 않을경우 랜덤으로 만들어집니다.

~~~shell
$ docker run -p 6000:6379 --name redis-named redis:4.0
1:C 19 May 10:11:04.418 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
1:C 19 May 10:11:04.418 # Redis version=4.0.14, bits=64, commit=00000000, modified=0, pid=1, just started
1:C 19 May 10:11:04.418 # Warning: no config file specified, using the default config. In order to specify a config file use redis-server /path/to/redis.conf
1:M 19 May 10:11:04.420 * Running mode=standalone, port=6379.
1:M 19 May 10:11:04.420 # Server initialized
1:M 19 May 10:11:04.420 # WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' as root, and add it to your /etc/rc.local in order to retain the setting after a reboot. Redis must be restarted after THP is disabled.
1:M 19 May 10:11:04.420 * Ready to accept connections
~~~
<code>docker ps</code>커맨드로 로 원하는 이름으로 컨테이너가 생성되었는지 확인해 봅니다.

~~~shell
$ docker ps
CONTAINER ID   IMAGE       COMMAND                  CREATED         STATUS         PORTS                                       NAMES
a5846e3006f2   redis:4.0   "docker-entrypoint.s…"   3 minutes ago   Up 3 minutes   0.0.0.0:6000->6379/tcp, :::6000->6379/tcp   redis-named
~~~
---

## 컨테이너 실행하기
<code>docker start</code>커맨드를 이용하여 원하는 컨테이너를 실행할 수 있습니다.
~~~shell
$ docker start <idOrNameOfContainer>
~~~


## 컨테이너 종료하기
<code>docker stop</code>커맨드를 이용하여 원하는 컨테이너를 실행할 수 있습니다.
~~~shell
$ docker stop <idOrNameOfContainer>
~~~


---





## 컨테이너의 쉘에 접근하기
<code> docker exec -it [idOrNameOfContainer] /bin/bash </code>
원하는 컨테이너의 쉘에 접속하기 위해 쓰입니다. <br>

다른 방법으로는 docker desktop에서 원하는 컨테이너를 클릭해서 해당 컨테이너의 쉘에 접근 가능합니다. <br>

다음과 같이 해당 컨테이너에 root권한으로 원으며 원하는 작업을 할 수가 있겠습니다.
~~~shell
$ docker exec -it a5846e /bin/bash
root@a5846e3006f2:/data# ls
dump.rdb
root@a5846e3006f2:/data# cd /
root@a5846e3006f2:/# ls
bin  boot  data  dev  etc  home  lib  media  mnt  opt  proc  root  run	sbin  srv  sys	tmp  usr  var
root@a5846e3006f2:/# env
HOSTNAME=a5846e3006f2
REDIS_DOWNLOAD_SHA=1e1e18420a86cfb285933123b04a82e1ebda20bfb0a289472745a087587e93a7
PWD=/
HOME=/root
REDIS_VERSION=4.0.14
GOSU_VERSION=1.12
TERM=xterm
REDIS_DOWNLOAD_URL=http://download.redis.io/releases/redis-4.0.14.tar.gz
SHLVL=1
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
_=/usr/bin/env
OLDPWD=/data
~~~

쉘접속을 종료하기위해 exit을 입력하거나, ctrl+D로 빠져나올 수 있습니다.


---

## 컨테이너의 로그 확인하기

<code>docker logs</code>커맨드로 원하는 컨테이너의 로그를 확인할 수 있습니다.

~~~shell
$ docker logs <idOrNameOfConainer>
~~~
원하는 컨테이너의 로그를 보고 싶을때 쓸 수 있습니다.

~~~shell
$ docker ps -a
CONTAINER ID   IMAGE       COMMAND                  CREATED             STATUS             PORTS                                       NAMES
95424043babd   redis:4.0   "docker-entrypoint.s…"   About an hour ago   Up About an hour   0.0.0.0:6001->6379/tcp, :::6001->6379/tcp   epic_wiles
14eb691f51ad   redis       "docker-entrypoint.s…"   About an hour ago   Up About an hour   0.0.0.0:6000->6379/tcp, :::6000->6379/tcp   flamboyant_khorana
$ docker logs epic_wiles
1:C 19 May 08:54:29.491 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
1:C 19 May 08:54:29.493 # Redis version=4.0.14, bits=64, commit=00000000, modified=0, pid=1, just started
1:C 19 May 08:54:29.493 # Warning: no config file specified, using the default config. In order to specify a config file use redis-server /path/to/redis.conf
1:M 19 May 08:54:29.494 * Running mode=standalone, port=6379.
1:M 19 May 08:54:29.494 # Server initialized
1:M 19 May 08:54:29.494 # WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' as root, and add it to your /etc/rc.local in order to retain the setting after a reboot. Redis must be restarted after THP is disabled.
1:M 19 May 08:54:29.494 * Ready to accept connections
~~~

