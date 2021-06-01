---
layout: article
title: "ssh config 설정으로 간편하게 ssh하기"
tags: Docker Kubernetes Replicasets Deployment static pod
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

[gcp]({{site.url}}/assets/images/gcp.png)

compute엔진 -> 메타데이터 -> SSH 키로 이동후에

항목추가를 클릭하여 공개키를 추가합니다.

키등록이 끝났으니 ssh 접속을 해봅시다