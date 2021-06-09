---
layout: article
title: "[ #10 ] 클러스터 보안 - 1"
tags: Kubernetes maintenance static pod strategy 쿠버네티스 update
---

기본적인 인증 수단에는 **Static Password File**을 통한 방법과, **Static Token File**을 이용한 방법이 있습니다.



## 1 - Static Password File


아래의 유저정보가 담긴 csv파일을 통하여 인증 할 수 있습니다.
~~~
#user-details.csv
password123,username1,userId1
password133,username2,userId2
password113,username3,userId3 
~~~


kubeadm으로 클러스터를 구성하였으면 
그러기 위해선 staticpodpath위치에 kube-apiserver.yaml 이 존재하는데, 아래와같이 실행 옵션을 추가해주면 됩니다
~~~yaml
...중략
  - command:
    - kube-apiserver
    - --advertise-address=10.178.0.2
    - --allow-privileged=true
    - --authorization-mode=Node,RBAC
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    - --enable-admission-plugins=NodeRestriction
    - --enable-bootstrap-token-auth=true
    - --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
    - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
    - --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
    - --etcd-servers=https://127.0.0.1:2379
    - --insecure-port=0
    - --basic-auth-file=user-details.csv        <-----추가된 실행옵션
...
~~~


또한 이 파일은 선택적으로 4번째 컬럼에 그룹명을 넣을 수도 있습니다.
~~~
#user-details.csv 
password123,username1,userId1,group1
password133,username2,userId2,group2
password113,username3,userId3,group2
~~~

## 2 - Static Token File 

두번째 기본 인증 방법으로 Password 대신에 Token을 사용하는 방법인데요

첫번째 컬럼에 비밀번호 대신에 토큰이 들어가는 차이가 있고,
~~~
user-token-details.csv
AjkfgljADSKFfjkwelkasdfKdfgkdlfga,username1,userId1,group1(optional)
RAakjkfskfgElkasdfjkERlfkglgDFFLg,username2,userId2,group2(optional)
lfkjgkqlwemdfgEaklFKSGkejsfdkgkas,username1,userId2,group2(optional)
~~~

실행 옵션이 다른 차이점이 있습니다.

~~~yaml
...중략
  - command:
    - kube-apiserver
    - --advertise-address=10.178.0.2
    - --allow-privileged=true
    - --authorization-mode=Node,RBAC
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    - --enable-admission-plugins=NodeRestriction
    - --enable-bootstrap-token-auth=true
    - --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
    - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
    - --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
    - --etcd-servers=https://127.0.0.1:2379
    - --insecure-port=0
    - --token-auth-file=user-details.csv        <-----추가된 실행옵션
...
~~~

위 두가지 방법은 추천되는 인증 방식이 아니라고합니다.
다음 포스트에서는 실제로 쓰이는 방식에 대해 다뤄보겠습니다.