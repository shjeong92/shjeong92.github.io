---
layout: article
title: "[ #3 ] Dockerfiled에서 자주 쓰이는 명령어"
tags: Docker network 도커 네트워크 컨테이너 명령어 
---

<br>
## Dockerfile 포멧

하나의 Dockerfile은 기본적으로 다음과 같은 구조를 가진 여러 개의 명령문으로 구성되어 있습니다.

~~~docker
# 주석(Comment)
명령어(INSTRUCTION) 인자(arguments)
~~~

각 구문은 명령어와 인자로 이루워져있고, 명령어는 소문자로 쓸수도 있지만 네이밍 컨벤션을 따라서 명령어는 대문자로만 작성하는게 좋습니다. 

--- 

## FROM 명령문

~~~docker
FROM <이미지>
FROM <이미지>:<태그>
~~~

<code>FROM</code>명령문은 base image를 지정해주기 위해서 사용합니다. 하나의 Docker image는 base 이미지로부터 시작해서 새로운 이미지를 중첩해서 여러 단계의 이미지 층을 쌓아가면서 만들어지는 것입니다.

+ python 최신버전을 base 이미지로 사용

~~~docker
FROM python:latest
~~~


+ NodeJS 11버전을 base 이미지로 사용

~~~docker
FROM node:latest
~~~

---

## WORKDIR 명령문
<code>WORKDIR</code> 명령문은 쉘의 <code>cd</code> 와 같습니다.
<code>WORKDIR</code> 명령문을 사용하여 디렉토리 위치를 이동시키면 그이후에 등장하는 모든 커멘드(<code>RUN</code>, <code>CMD</code>, <code>ENTRYPOINT</code>, <code>COPY</code>, <code>ADD</code>)는 해당 디렉토리 위치 기준으로 실행됩니다


~~~docker
WORKDIR <이동할경로>
~~~

+ /usr/test로 작헙할 디렉토리위치 전환

~~~docker
WORKDIR /usr/test
~~~

--- 

## RUN 명령문

~~~docker
RUN ["<커맨드>", "<파라미터1>", "<파라미터2>" ...]
RUN <전체 커맨드>
~~~

+ npm 패키지 설치

~~~docker
RUN npm install --silent
~~~


+ pip 패키지 설치

~~~docker
RUN pip install -r requirements.txt
~~~


~~~docker
#Dockerfile
FROM ubuntu:latest
CMD ["echo", "CMD test"]
~~~
위와 같은 도커파일이 있을 때, 이미지를 빌드하고 실행해보도록 하겠습니다.

먼저, 아래 명령어로 이미지를 빌드해줍니다.
~~~sh
docker build -t cmd_test .
~~~

--- 

## ENTRYPOINT 명령문


~~~docker
ENTRYPOINT ["<커맨드>", "<파라미터1>", "<파라미터2>" ...]
ENTRYPOINT <전체 커맨드>
~~~
<code>ENTRYPOINT</code> 명령문은 컨테이너가 실행될때 항상 실행되야하는 커멘드를 지정할 때 사용합니다.


+ Django 서버 실행하기

~~~docker
ENTRYPOINT ["python", "manage.py", "runserver"]
~~~

## CMD 명령문

~~~docker
CMD ["<커맨드>","<파라미터1>","<파라미터2>"]
CMD ["<파라미터1>","<파라미터2>"]
CMD <전체 커맨드>
~~~

<code>CMD</code> 명령문은 컨테이너가 시작될때 디폴트로 실행할 커맨드나, 디폴트로 파라미터를 넘길때 사용됩니다.
<code>CMD</code> 명령문은 Dockerfile의 마지막 라인에 쓰곤하는데요, 이 명령문은 Dockerfile당 한줄만 사용할 수 있습니다. 여러줄을 쓴다면
가장 마지막 줄만 적용됩니다. 또한, <code>docker run</code>뒤에 인자가 주어진다면 도커파일내의 <code>CMD</code>명령문은 무시됩니다.


## EXPOSE 명령문

~~~docker
EXPOSE <포트>
EXPOSE <포트>/<프로토콜>
~~~

<code>EXPOSE</code>  명령문은 네트워크 상에서 컨테이너로 들어오는 트래픽(traffic)을 리스닝(listening)하는 포트와 프로토콜를 지정하기 위해서 사용됩니다. 프로토콜은 TCP와 UDP 중 선택할 수 있는데 지정하지 않으면 TCP가 기본값으로 사용됩니다.
여기서 주의해야할 점은 <code>EXPOSE</code> 명령문으로 지정된 포트는 해당 컨테이너 내부에서만 유효한데요, 호스트 컴퓨터로부터 
해당 포트로 접근을 허용하려면 <code>docker run</code> 커맨드에 <code>-p</code> 옵션을 추가하여 바인딩 해줘야 합니다.

+ 80/TCP 포트로 리스닝

~~~docker
EXPOSE 80
~~~

+ 80/UDP 포트로 리스닝

~~~docker
EXPOSE 80/udp
~~~

--- 

## COPY/ADD 명령문

~~~docker
COPY <src>... <dest>
COPY ["<src>",... "<dest>"]
~~~
<code>COPY</code> 명령문은 호스트 컴퓨터에 있는 디렉터리나 파일을 Docker 이미지의 파일 시스템으로 복사하기 위해서 사용됩니다. 절대 경로와 상대 경로를 모두 지원하며, 상대 경로를 사용할 때는 이 전에 등장하는 
<code>WORKDIR</code> 명령문으로 작업 디렉토리를 어디로 전환했는지 고려해야 합니다.

+ 이미지를 빌드한 디렉터리의 모든 파일을 컨테이너의 app/test 디렉터리로 복사

~~~docker
WORKDIR app/test
COPY . .
~~~
<code>ADD</code> 명령문은 좀 더 파워풀한 <code>COPY</code> 명령어라고 보시면되는데요, <code>ADD</code> 명령문은
일반 파일이외에도 압축파일이나 네트워크 상의 파일도 사용할 수 있습니다. 공식문서에 의하면, 이러한 특수경우가 아닐경우 <code>COPY</code> 를 사용하는것이
권장됩니다.

--- 

## ENV 명령문


~~~docker
ENV <키> <값>
ENV <키>=<값>
~~~

<code>ENV</code> 명령문은 환경 변수를 설정하기 위해서 사용합니다.
<code>ENV</code> 명령문으로 설정된 환경 변수는 이미지 빌드 시에도 사용됨은 물론이고, 해당 컨테이너에서 돌아가는 애플리케이션도 접근할 수 있습니다.

+ DB_PASS 환경 변수를 1234로 설정


~~~docker
ENV DB_PASS 1234
~~~

--- 

## ARG 명령문

~~~docker
ARG <이름>
ARG <이름>=<기본 값>
~~~

<code>ARG</code> 명령문은 <code>docker build</code> 커맨드로 이미지를 빌드 할때, <code>--build-arg</code> 옵션을 추가하여 넘길 수 있는 인자를 정의하기 위해 사용합니다.
<br>
예를 들어, Dockerfile에 다음과 같이 ARG 명령문으로 port를 인자로 선언해주면,

~~~docker
ARG port
~~~

다음과 같이 <code>docker build</code> 커맨드에 <code>--build-arg</code> 옵션에 <code>port</code> 값을
넘길 수가 있습니다.

~~~shell
$ docker build --build-arg port=8080 .
~~~

인자의 디폴트값을 지정해주면, <code>--build-arg</code> 옵션으로 해당 인자가 넘어오지 않았을 때 사용됩니다.

~~~docker
ARG port=8080
~~~

설정된 인자 값은 다음과 같이 <code>${인자명}</code> 형태로 읽어서 사용할 수 있습니다.

~~~docker
CMD start.sh -h 127.0.0.1 -p ${port}
~~~


