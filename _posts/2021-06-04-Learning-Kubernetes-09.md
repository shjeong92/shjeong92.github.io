---
layout: article
title: "[ #9 ] 클러스터 백업 & 복구"
tags: Kubernetes maintenance static pod strategy 쿠버네티스 update
---

이번 포스트에서는 ETCD database를 이용한 클러스터 백업 및 복구 방법에 대해서 다뤄보겠습니다.

## etcd란

etcd는 모든 클러스터 데이터에 대한 Kubernetes의 백업 저장소로 사용되는 일관되고 가용성이 높은 키 값 저장소입니다

그리고 ETCD는 각 마스터노드에 존재하고, key=value의 데이터는 마스터 노드의 /var/lib/etcd 에 저장됩니다.

ETCD는 built in 스냅샷 solution을 가지고 있는데요 이를통해 현재 실행중인 모든 리소스의 정보를 백업하고, 복구할 수 있습니다.

### STEP1 - etcd가 설치되어 있지 않을경우 설치합니다.

제 쿠버네티 클러스터는 1 개의 마스터노드, 3개의 워커노드로 구성되어 있습니다. googld cloud compute engine 인스턴스를 사용하여 구성하였으며 모든 노드는 Ubuntu 18.04.5 LTS 버전입니다.

아래 커맨드를 이용해 etcd-client를 설치해줍니다
~~~sh
$ sudo apt update

$ sudo apt install etcd-client
~~~



### STEP2 - 복구시킬 환경 구성


~~~yaml
#depl.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
  labels:
    app: myapp
    type: front-end
spec:
  template:
    metadata:
      name: myapp-pod
      labels:
        app: myapp
        type: front-end
    spec:
      containers:
      - name: nginx-container
        image: nginx:1.19
  replicas: 3
  selector:
    matchLabels:
      type: front-end
~~~


간단한 yaml파일을 통해 3개의 nginx 파드를 배포합니다

~~~sh
$ kubectl apply -f depl.yml 
deployment.apps/myapp-deployment created
~~~

그리고 현재 어떤 리소스들이 돌아가고있는지 확인해주고 이 상황의 스냅샷을 생성할겁니다
~~~
$ kubectl get all
NAME                                    READY   STATUS    RESTARTS   AGE
pod/myapp-deployment-7f6679cc7d-5qwxx   1/1     Running   0          32s
pod/myapp-deployment-7f6679cc7d-855lp   1/1     Running   0          32s
pod/myapp-deployment-7f6679cc7d-bxxjx   1/1     Running   0          32s

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   8d

NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/myapp-deployment   3/3     3            3           32s

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/myapp-deployment-7f6679cc7d   3         3         3       32s
shjeong920522@master:~/Pod$ 

~~~

## 스냅샷 생성하기

etcdctl의 스냅샷을 생성하기위해선 아래와같은 옵션이 필요합니다.

~~~sh
etcdctl --cacert=옵션값 \
--cert=옵션값 \
--key=옵션값 \
snapshot save 백업경로 및 파일이름
~~~

**그렇다면 이 옵션값은 어디서 확인가능할까요?**

ps명령어 와 grep을 를 활용하면 된답니다

아래 명령어를 사용하여 다 찾아서 값을 가져와도되지만
~~~sh
$ ps -ef | grep etcd
root      2863  2764  1 05:36 ?        00:00:23 etcd --advertise-client-urls=https://10.178.0.2:2379 --cert-file=/etc/kubernetes/pki/etcd/server.crt --client-cert-auth=true --data-dir=/var/lib/etcd --initial-advertise-peer-urls=https://10.178.0.2:2380 --initial-cluster=master=https://10.178.0.2:2380 --key-file=/etc/kubernetes/pki/etcd/server.key --listen-client-urls=https://127.0.0.1:2379,https://10.178.0.2:2379 --listen-metrics-urls=http://127.0.0.1:2381 --listen-peer-urls=https://10.178.0.2:2380 --name=master --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt --peer-client-cert-auth=true --peer-key-file=/etc/kubernetes/pki/etcd/peer.key --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt --snapshot-count=10000 --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
~~~

표에 들어간 커맨드로 원하는 부분을 강조시켜 찾는방법을 추천드립니다. 눈알굴려서 찾는것보다 훨씬 빠르더라구요

|<center>etcdctl 명령 옵션</center>|<center>etcd <br>프로세스 실행옵션</center>| <center>값</center>|<center>커맨드로 빨리찾기</center>
|:---|:---|:----------|
| \--cacert | \--trusted-ca-file | /etc/kubernetes/pki/etcd/ca.crt | `ps -ef | grep etcd | grep trusted`
| \--cert | \--cert-file | /etc/kubernetes/pki/etcd/server.crt | `ps -ef | grep etcd | grep "\--cert"`
| \--key | \--key-file | /etc/kubernetes/pki/etcd/server.key | `ps -ef | grep etcd | grep "\--key"`

그리고 값 부분은 항상 같은것이 아닐 수도 있기에 커맨드로 직접 확인하셔야 합니다.



자 이제 스냅샷을 생성하기위하여 우선 루트계정으로 전환해줍니다
~~~sh
$ sudo -i
~~~

아래와 같이 환경변수를 등록해줍니다
ETCDCTL_API 3버전을 사용하기 위함입니다.

~~~sh 
root# export ETCDCTL_API=3
~~~

~~~sh
root# etcdctl --cacert=/etc/kubernetes/pki/etcd/ca.crt \
> --cert=/etc/kubernetes/pki/etcd/server.crt \
> --key=/etc/kubernetes/pki/etcd/server.key \
> snapshot save /opt/backup
Snapshot saved at /opt/backup
~~~

스냅샷을 생성하였으니 아까 배포하였던 deployment 를 delete하고 etcdctl 복구를하여
정말 동작하던 deployment를 다시 배포해주는지 확인해봅시다 

우선 루트계정에서 로그아웃 해주시구요
~~~sh
root# exit
logout
~~~

deployment를 삭제시켜줍니다.
~~~sh
$ kubectl delete -f depl.yml 
deployment.apps "myapp-deployment" deleted
~~~

복구하기로 넘어갑니다.

## 스냅샷으로 복구하기

복구할때는 스냅샷을 만들때 넣었던 옵션에서 네가지의 옵션이 더 추가됩니다.

|<center>etcdctl 명령 옵션</center>|<center>etcd <br>프로세스 실행옵션</center>| <center>값</center>|<center>커맨드로 빨리찾기</center>
|:---|:---|:----------|
| \--cacert | \--trusted-ca-file | /etc/kubernetes/pki/etcd/ca.crt | `ps -ef | grep etcd | grep trusted`
| \--cert | \--cert-file | /etc/kubernetes/pki/etcd/server.crt | `ps -ef | grep etcd | grep "\--cert"`
| \--key | \--key-file | /etc/kubernetes/pki/etcd/server.key | `ps -ef | grep etcd | grep "\--key"`
| \--initial-advertise-peer-urls |\--initial-advertise-peer-urls | https://10.178.0.2:2380 | `ps -ef | grep etcd | grep initial`
| \--initial-cluster=master | \--initial-cluster=master  | https://10.178.0.2:2380 | `ps -ef | grep etcd | grep initial`
| \--data-dir | \--data-dir | /var/lib/etcd_backup | `ps -ef | grep etcd | grep "\--data-dir" `
| \--name | \--name | master(마스터노드이름) | `ps -ef | grep etcd | grep "\--name"`

자 여기까지가 restore 할때 필요한 모든 옵션입니다.

이제 아까 찍어놨던 스냅샷으로 복구되는지 확인해봅시다.

우선 루트 계정으로 전환하구요
~~~sh 
$ sudo -i
~~~

restore 명령어를 입력해주고 로그아웃해줍니다
~~~sh
root# etcdctl --cacert=/etc/kubernetes/pki/etcd/ca.crt \
> --cert=/etc/kubernetes/pki/etcd/server.crt \
> --key=/etc/kubernetes/pki/etcd/server.key \
> --initial-advertise-peer-urls=https://10.178.0.2:2380 \
> --initial-cluster=master=https://10.178.0.2:2380 \
> --data-dir=/var/lib/etcd_backup \
> --name=master \ 
> snapshot restore /opt/backup

root# exit
logout
~~~

/opt/backup 에 생성해뒀던 스냅샷을이용해서 /var/lib 위치에 etcd_backup 파일을 생성합니다
staticpod에서 실행되는 etcd가 이를 바라보도록 해줘야하겠지요

staticpod의 위치인 /etc/kubernetes/manifest로 이동후 etcd.yaml을 수정해야합니다
~~~sh
$ cd /etc/kubernetes/manifest

$ sudo vim etcd.yaml

~~~yaml
#etcd.yaml
...중략

volumes:
- hostPath:
    path: /etc/kubernetes/pki/etcd
    type: DirectoryOrCreate
  name: etcd-certs
- hostPath:
    path: /var/lib/etcd_backup     <-----------  backup한 스냅샷을 바라보게 수정해줍니다
    type: DirectoryOrCreate
  name: etcd-data

...
~~~

변경사항을 저장해주면 새로 로드될때까지 시간이 조금걸립니다 

잠시후 정말 실행 중이던 deployment가 실행중인지 확인해봅시다

~~~sh
$ kubectl get deploy
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
myapp-deployment   3/3     3            3           20s
~~~

짜잔.. 잘 복구된것을 확인할 수 있었습니다 