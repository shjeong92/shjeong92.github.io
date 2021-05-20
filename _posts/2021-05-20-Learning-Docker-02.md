---
layout: article
title: "[ #2 ] Docker 네트워크 사용하기 "
tags: Docker network 도커 네트워크 컨테이너
---


Docker 컨테이너는 서로 격리되어 있기 때문에 따로 설정을 하지 않는다면 컨테이너간 통신을 할 수 없습니다. 하지만 네트워크 연결을 통해서 서로 통신 할 수 있게 됩니다. 이번 글에서는 컨테이너간 통신을 도와주는 Docker 네트워크에 대하여 작성하도록 하겠습니다.

## 네트워크 조회


<code >docker network ls</code> 커맨드를 사용하면 현재 생성되어 있는 Docker 네트워크 목록을 조회할 수 있습니다.
~~~shell
$ docker network ls
NETWORK ID     NAME             DRIVER    SCOPE
b1545defe462   bridge           bridge    local
2fff1f6ef4f0   core_default     bridge    local
95295abe6cbb   docker_default   bridge    local
a152a103d15b   host             host      local
7cf51facc103   mongo-network    bridge    local
f984d450d39e   none             null      local
~~~

<code>bridge</code>, <code>host</code>, <code>none</code> 은 Docker 데몬이 실행되면서 기본적으로 생성되는 네트워크인데, 사용자가 직접 네트워크를 생성해서 사용하는게 권장된다고 합니다.

## 네트워크 종류
Docker 네트워크는 <code>bridge</code>, <code>host</code>, <code>overlay</code>, <code>none</code>, <code>container</code> 가 있습니다.
+ bridge 네트워크는 하나의 호스트 컴퓨터 내에서 여러 컨테이너들이 서로 소통할 수 있도록 해줍니다.
+ host 네트워크는 컨터이너를 호스트 컴퓨터와 동일한 네트워크에서 컨테이너를 돌리기 위해서 사용됩니다.
+ overlay 네트워크는 여러 호스트에 분산되어 돌아가는 컨테이너들 간에 네트워킹을 위해서 사용됩니다.
+ none 네트워크는 네트워크를 사용하지 않는다는 것입니다.
+ container 네트워크는 다른 컨테이너의 네트워크 환경 공유를위해 쓰입니다.
<code>--net container: (containerIdOrName)  </code>


## 네트워크 생성
<code>docker network create (name)</code> 커맨드를 사용하면 새로운 Docker 네트워크를 생성할 수 있습니다.

~~~zsh
$ docker network create new
56aabbc93d48529886e50a1c86bdd328c8ccbe770630711abe14babaac50eff7
$ docker network ls
NETWORK ID     NAME             DRIVER    SCOPE
b1545defe462   bridge           bridge    local
95295abe6cbb   docker_default   bridge    local
a152a103d15b   host             host      local
7cf51facc103   mongo-network    bridge    local
56aabbc93d48   new              bridge    local <-- 새로생긴 네트워크
f984d450d39e   none             null      local
~~~

-d 옵션을 추가하면 네트워크의 종류를 정의할 수도 있습니다. 기본값은 bridge 입니다.

## 네트워크 디테일 확인하기

<code>docker network inspect</code>커맨드를 사용하면 다음과 같이 네트워크의 상세정보를 확인할 수 있습니다.

~~~ shell 
$ techworld-js-docker-demo-app docker network inspect new
[
    {
        "Name": "new",
        "Id": "56aabbc93d48529886e50a1c86bdd328c8ccbe770630711abe14babaac50eff7",
        "Created": "2021-05-20T11:38:23.716924633Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "192.168.96.0/20",
                    "Gateway": "192.168.96.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {},
        "Options": {},
        "Labels": {}
    }
]
~~~

## 디폴트 네트워크

컨테이너 생성시 --net 태그를 사용하여 원하는 네트워크에 연결시킨 상태로 만들 수 있습니다 만약 --net 옵션을 사용하지 않으면 기본적으로 bridge라는 디폴트 네트워크에 붙습니다.

자그럼 <code>sample</code>이라는 컨테이너를 하나 실행해보겠습니다.

~~~shell
docker run -itd --name sample busybox
c5f06e4ce41c43262d1743a7505bcb6715c316047fe9c5d1c7ed0702f3111473
~~~

--net 옵션을 사용하지 않았으니 bridge 네트워크에 붙었겠지요? 한번 확인해봅시다.

~~~shell
$ docker network inspect bridge

[ 
  {
  ...중략
    "Containers": {
      "c5f06e4ce41c43262d1743a7505bcb6715c316047fe9c5d1c7ed0702f3111473": {
          "Name": "sample",
          "EndpointID": "8e3585999080257814d644b79484041fd7b517fe467dcd21c55175922cce69cf",
          "MacAddress": "02:42:ac:11:00:02",
          "IPv4Address": "172.17.0.2/16",
          "IPv6Address": ""
      }
    },
  ...중략

  }
]
~~~
containers 안에 sample이 잘 들어가있는것을 확인할 수 있습니다.

## 네트워크에 컨테이너 연결
<code>docker network connect</code>커맨드를 사용하면 원하는 네트워크에 컨테이너를 연결할 수 있습니다.
sample컨테이너를 new 네트워크에 연결후 확인해 보겠습니다.
~~~shell
$ docker network connect new sample

$ docker network inspect new
[
    ...중략
        "Containers": {
            "c5f06e4ce41c43262d1743a7505bcb6715c316047fe9c5d1c7ed0702f3111473": {
                "Name": "sample",
                "EndpointID": "6c538f7675a1552238adc0a36ec659aa9adc4312404db46cb66160fad2404080",
                "MacAddress": "02:42:c0:a8:60:02",
                "IPv4Address": "192.168.96.2/20",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {}
    ...중략
]
~~~
이와같이 잘 연결된 것을 확인 할 수 있습니다.

## 네트워크로부터 컨테이너 연결 해제
<code>docker network disconnect</code> 커맨드를 이용하여 디폴트 네트워크인 bridge 과 sample의 연결을 해재 하겠습니다.

~~~shell
$ docker network disconnect bridge sample
~~~

잘 해제된것을 확인해 봅시다.
~~~shell
$ docker network inspect bridge

[ 
  {
  ...중략
    "Containers": {}
  ...중략

  }
]
~~~

## 두번째 컨테이너 연결
같은 네트워크에 다른 서로 다른 컨테이너들을 연결시키면 통신 가능하다고 했었죠? 한번 직접 확인해봅시다.
우선 두번째 sample을 만들어주고,
~~~shell
docker run -itd --name sample2 --net new busybox
6db6d2deb04abc6c5c15e85159c9525ec602951640b1df10e655709301f4583c
~~~
네트워크에 잘 붙었는지 확인합니다

~~~shell
$ docker network inspect new

  ...중략
    "Containers": {
        "6db6d2deb04abc6c5c15e85159c9525ec602951640b1df10e655709301f4583c": {
            "Name": "sample2",
            "EndpointID": "68c44b54a118ab3f54ea29aff5a6bbb1431668d40b4f1cc22807525592ca362b",
            "MacAddress": "02:42:c0:a8:60:03",
            "IPv4Address": "192.168.96.3/20",
            "IPv6Address": ""
        },
        "c5f06e4ce41c43262d1743a7505bcb6715c316047fe9c5d1c7ed0702f3111473": {
            "Name": "sample",
            "EndpointID": "6c538f7675a1552238adc0a36ec659aa9adc4312404db46cb66160fad2404080",
            "MacAddress": "02:42:c0:a8:60:02",
            "IPv4Address": "192.168.96.2/20",
            "IPv6Address": ""
        }
    }
  ...중략
~~~
new라는 네트워크에 sample과 sample2가 잘 연결되었습니다.

## 컨테이너 간 네트워킹

이제 두 컨테이너가 네트워크를 통해 통신 가능한지 테스트 해보겠습니다.

우선 <code>sample</code> 컨테이너에서 <code>sample2</code> 컨테이너로 <code>ping</code> 을 날려봅니다 컨테이너 이름대신 ip로 입력할 수도 있지만 이름을 넣는게 편하겠지요.

~~~shell
docker exec sample ping sample2
PING sample2 (192.168.96.3): 56 data bytes
64 bytes from 192.168.96.3: seq=0 ttl=64 time=0.109 ms
64 bytes from 192.168.96.3: seq=1 ttl=64 time=0.232 ms
64 bytes from 192.168.96.3: seq=2 ttl=64 time=0.196 ms
64 bytes from 192.168.96.3: seq=3 ttl=64 time=0.198 ms
64 bytes from 192.168.96.3: seq=4 ttl=64 time=0.164 ms
64 bytes from 192.168.96.3: seq=5 ttl=64 time=0.199 ms
~~~

## 네트워크 삭제

마지막으로 <code>docker network rm</code>커맨드를 사용하여 <code>new</code> 네트워크를 삭제해보겠습니다.
<br><br>
에러가 뜨면서 삭제가 되질 않습니다
~~~shell
$ docker network rm new
Error response from daemon: error while removing network: network new id 56aabbc93d48529886e50a1c86bdd328c8ccbe770630711abe14babaac50eff7 has active endpoints
~~~

네트워크를 삭제하기위해선 해당 네트워크상에서 실행중인 컨테이너가 있으면 안됩니다.
따라서 
<code>docker stop</code>커맨드로 컨테이너들을 멈춰주고 다시 지워보겠습니다.
~~~shell
$ docker stop sample sample2
sample
sample2
$ docker network rm new
new
$ docker network ls
NETWORK ID     NAME             DRIVER    SCOPE
b1545defe462   bridge           bridge    local
95295abe6cbb   docker_default   bridge    local
a152a103d15b   host             host      local
7cf51facc103   mongo-network    bridge    local
f984d450d39e   none             null      local
~~~
성공적으로 삭제되었습니다.

