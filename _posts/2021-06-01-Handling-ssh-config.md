---
layout: article
title: "ssh config 설정으로 간편하게 ssh접속하기"
tags: gcloud ssh key-gen rsa config 
---

최근 몇일간 kubernetes를 공부하면서 구글 클라우드에 ssh접속을 하는일이 잦았는데요, 접속 할 때마다 구글 클라우드에 접속해서 인스턴스의 외부 IP를 확인 하여야하고 긴 ssh 접속문을 쓰는게 귀찮았는데 알고보니 이를 간편하게 해주는 설정이 있더군요.

이 글에선 맥에서 GCP에 접속하기 위한 <code>RSA key pair</code>를 생성하는 부분부터 <code>ssh config</code> 설정하는법을 다루겠습니다.

## key pair 생성하기

~~~
#-t 암호화타입 -f key pair 저장할 위치 -C 주석(보통 사용자 로그인ID를 적음)
$ssh-keygen -t rsa -f ~/.ssh/<KEY_FILE_NAME> -C "account@gmail.com"
~~~

위와같이 입력하면 RSA key pair가 생성됩니다.

~~~sh
ssh-keygen -t rsa -f ~/.ssh/test -C "test@gmail.com"
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /Users/sanghyukjeong/.ssh/test.
Your public key has been saved in /Users/sanghyukjeong/.ssh/test.pub.
The key fingerprint is:
SHA256:6TjvkM1IFHeI5ywCVgt8DFJpqiK+K4SZO4eTuBcEHkQ test@gmail.com
The key's randomart image is:
+---[RSA 3072]----+
|+E+=. .....      |
|.o*.o..oo.       |
|.=.o. .+         |
|... ... o.       |
|o+   ...S        |
|B..  . B         |
|*+ .  * +        |
|O.o    +         |
|oOo    .o        |
+----[SHA256]-----+
~~~

이후 cat명령어로 rsa 공개키를 확인할 수 있는데
구글 클라우드 인스턴스에 이 키를 등록하기위해 복사 해놓습니다.


~~~sh
$ cat ~/.ssh/test.pub
ssh-rsa AAAB3NzaC1yc2EAAAADAQABAAABgQDIlHiLTIM6oCQtjILfmIaTZv6j37J0kZNNjIgK0dFOxSYnMgXirh8gbRqFQY7kMWXZwAo82akxyTAP65GcjsqR+L1RarkKlYA7/lj7rvAf2VMJXy0X6yCbGo3yHcPGEoWLlsOgeYZDAk3Wld/hGdBWKXX4404iUQygjWbIircQ6BNZBb14nbhJK+pLWXMB7TmaRvtDsumBAko0shkA5g6y1oNGdpGBsUTQHk+rqQhqZElP9hgXp71qGhTO0V11kH41n0beReBoi9YngrpJtu6b4h1Ttm4N7CpCU5rUW38i5s71aInLuPolCDXlA8b6qVtBqg6dQ6/IgNQgQFRpwIkcf4v0EHKygcdm+QM25x3VAx1XC51gqHxhdU71/24EiUw0EW3cIvER6150aZF3PiAe5odstk9vx4L/1Vtu1GySQ2Qz8LcK1J3lawh3i8yaobVPUOIOxGDhme+tRMuyy16PhumtxSI5SeSwoIhYEIZtHKHFHKztNGV6m5aNSLZwzbk= test@gmail.com
~~~

## key 등록하기

![gcp](https://user-images.githubusercontent.com/75003424/120263396-9136a700-c2d6-11eb-8c97-65769591c66c.png)

compute엔진 -> 메타데이터 -> SSH 키로 이동후에

항목추가를 클릭하여 공개키를 추가합니다.

키등록이 끝났으니 ssh 접속을 해봅시다

~~~sh

$ ssh -i ~/.ssh/test yourID@your.gcloudExternal.ip

Welcome to Ubuntu 18.04.5 LTS (GNU/Linux 5.4.0-1043-gcp x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Tue Jun  1 03:44:22 UTC 2021

  System load:  0.27               Users logged in:        0
  Usage of /:   14.2% of 28.90GB   IP address for ens4:    10.178.0.2
  Memory usage: 44%                IP address for docker0: 172.17.0.1
  Swap usage:   0%                 IP address for cni0:    10.244.0.1
  Processes:    143

 * Super-optimized for small spaces - read how we shrank the memory
   footprint of MicroK8s to make it the smallest full K8s around.

   https://ubuntu.com/blog/microk8s-memory-optimisation
~~~

잘 접속 되는것을 확인할 수 있습니다.

그러나 접속할때마다 저 커맨드를 다 입력하는것은 참 귀찮은일입니다. ip정도는 한번보면 외우시는분이 아니라면요 ㅎㅎ

이제 ssh config파일 설정에대해 알아봅시다




## ssh config 설정하여 ssh 접속 간편하게 하기

Linux/Unix서버 접속 계정들을 여러 가지 관리하는 경우, 별도로 메모나 파일에 접속 계정 / 비밀번호 등을 관리하거나 alias에 접속 명령어를 설정하는 번거로움이 있습니다.

하지만 .ssh 경로 밑 config 파일에 계정 정보들을 저장하여 관리하면 정말 편리하게 접속들을 관리할 수 있습니다.

우선 사용자 홈 디렉토리에서 .ssh 폴더로 접근합니다. .ssh 폴더는 ssh 접속을 한 번이라도 했다면, rsa 파일이나 known_hosts 등이 생성되기 때문에 자동으로 생성되는 디렉토리입니다.

홈디렉토리에 있는 .ssh폴더에 들어가서 config 파일을 생성해줍니다.
~~~sh
$ cd ~/.ssh

$ vim config
~~~


~~~
Host gcloud
        HostName 10.178.0.3
        User shjeong920522
        IdentityFile: ~/.ssh/test
        Port 22
~~~

+ Host: 접속할 정보의 이름입니다.
+ HostName: 접속할 서버의 외부 IP 주소를 입력하세요
+ User: 접속할 서버의 user 정보입니다.
+ IdentityFile: 인증 키파일이 필요한 경우, 키파일 경로를 설정합니다. 이전 단계에 rsa 키페어 만든거 기억나시죠? 
+ Port: ssh 접속 포트입니다. 생략 가능하며, 생략할 경우에 디폴트로 22번 포트로 실생됩니다.

모든 설정이 끝났으니 google cloud에 접속해보겠습니다.

~~~sh
# config 에서 Host에 등록한 접속할 정보만 적어주면 접속이되는 마법....
$ ssh gcloud
Welcome to Ubuntu 18.04.5 LTS (GNU/Linux 5.4.0-1043-gcp x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Tue Jun  1 04:05:52 UTC 2021

  System load:  0.65               Users logged in:        0
  Usage of /:   14.2% of 28.90GB   IP address for ens4:    12.178.0.2
  Memory usage: 44%                IP address for docker0: 172.17.0.1
  Swap usage:   0%                 IP address for cni0:    12.244.0.1
  Processes:    144

 * Super-optimized for small spaces - read how we shrank the memory
   footprint of MicroK8s to make it the smallest full K8s around.

   https://ubuntu.com/blog/microk8s-memory-optimisation

11 updates can be applied immediately.
To see these additional updates run: apt list --upgradable

New release '20.04.2 LTS' available.
Run 'do-release-upgrade' to upgrade to it.


Last login: Tue Jun  1 03:44:23 2021 from 122.32.103.55
shjeong920522@master:~$ 
~~~

이렇게 한번 설정해 놓으면 외부 ip포트 확인하러 gcloud 콘솔에 접속하지 않아도되고, 정말 간편하게 ssh 접속을 할 수 있습니다.