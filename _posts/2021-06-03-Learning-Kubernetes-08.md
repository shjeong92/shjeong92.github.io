---
layout: article
title: "[ #8 ] 클러스터 유지보수"
tags: Kubernetes maintenance static pod strategy 쿠버네티스 update
---

오늘은 클러스터의 유지보수에 관련한 포스트를 써볼까 합니다.

쿠버네티스를 이용하여 서비스를 구축하고나서 언젠가는 시스템을 업데이트 시켜야할 일이 생길 것이며, 오류로 인하여 노드가 먹통이 되는일도 생길 것입니다.

그렇다면 노드의 작동을 중지시키면, 또는 오류로 중지된다면 어떻게 될까요?

+ master node의 kube-contoller-manager에는 pod-eviction-timeout이 존재합니다


## 컨디션

각 노드에는 아래와 같이 여러가지 condition이 존재합니다

|-+-
|노드 컨디션|설명
|:-|:-
|**Ready**|노드가 상태 양호하며 파드를 수용할 준비가 되어 있는 경우 True, 노드의 상태가 불량하여 파드를 수용하지 못할 경우 False, 그리고 노드 컨트롤러가 마지막 node-monitor-grace-period (기본값 40 기간 동안 노드로부터 응답을 받지 못한 경우) Unknown
|**DiskPressure**|디스크 사이즈 상에 압박이 있는 경우, 즉 디스크 용량이 넉넉치 않은 경우 True, 반대의 경우 False
|**MemoryPressure**|노드 메모리 상에 압박이 있는 경우, 즉 노드 메모리가 넉넉치 않은 경우 True, 반대의 경우 False
|**PIDPressure**|프로세스 상에 압박이 있는 경우, 즉 노드 상에 많은 프로세스들이 존재하는 경우 True, 반대의 경우 False
|**NetworkUnavailable**|노드에 대해 네트워크가 올바르게 구성되지 않은 경우 True, 반대의 경우 False


또한 노드 lifecycle controller는 컨디션을 아래와 같은 ***built-in taint***를 자동으로 생성합니다.

### built-in taints
  + node.kubernetes.io/not-ready: 노드가 준비되지 않음. 이는 NodeCondition Ready 가 "False"로 됨에 해당.
  + node.kubernetes.io/unreachable: 노드가 노드 컨트롤러에서 도달할 수 없음. 이는  NodeCondition Ready 가 "Unknown"로 됨에 해당.
  + node.kubernetes.io/memory-pressure: 노드에 메모리 할당 압박이 있음.
  + node.kubernetes.io/disk-pressure: 노드에 디스크 할당 압박이 있음.
  + node.kubernetes.io/pid-pressure: 노드에 PID 할당 압박이 있음.
  + node.kubernetes.io/network-unavailable: 노드의 네트워크를 사용할 수 없음.
  + node.kubernetes.io/unschedulable: 노드를 스케줄할 수 없음.
  + node.cloudprovider.kubernetes.io/uninitialized: "외부" 클라우드 공급자로 kubelet을 시작하면, 이 테인트가 노드에서 사용 불가능으로 표시되도록 설정됨. 


그 중에서도 <code>ready</code> 컨디션은 좀 특별한데요. 
ready 컨디션의 상태가 pod-eviction-timeout (kube-controller-manager에 전달된 인수) 보다 더 길게 Unknown 또는 False로 유지되는 경우, 노드 상에 모든 파드는 노드 컨트롤러에 의해 삭제되도록 스케줄 됩니다.
 



클라우드-컨트롤러-관리자의 컨트롤러가 이 노드를 초기화하면, kubelet이 이 테인트를 제거합니다 즉 다시 pod가 배정될 수 있는 상태가 되는것이죠. 기본 축출 타임아웃 기간은 기본 5분으로 설정되어 있습니다. 만약 노드가 다운되었다가 5분안에 복구가 되었다면 노드 내의 pod들은 그대로 존재하지만, 설정된 타임아웃 시간을 초과한다면 그안의 파드들은 모두 중단 되는것입니다.


만약 수정하고 싶다면 마스터 노드의 kube-system 네임스페이스에 있는 kube-controller-manager-master을 edit해주면됩니다.


그런데 말이죠 이 pod는 static pod입니다. 
[6번째 포스트](https://shjeong92.github.io/2021/05/31/Learning-Kubernetes-06.html)에서 설명 했었는데 static pod 를 수정하려면 
static pod을 생성한 yaml파일 자체를 수정한다고 했었죠?

static pod의 위치를 찾는것 부터 복습해봅시다

우선 config.yaml파일의 위치를 찾아줍니다.
~~~sh 
$ ps -ef | grep kubelet | grep "\--config"

root     10476     1  3 May28 ?        05:09:26 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml --network-plugin=cni --pod-infra-container-image=k8s.gcr.io/pause:3.4.1
~~~

config.yaml파일 내에서 staticpath를 찾아냅시다 
~~~sh
$ sudo grep -i static /var/lib/kubelet/config.yaml
staticPodPath: /etc/kubernetes/manifests
~~~

staticPodPath로 이동후 뭐가있나 확인해봅니다.
~~~sh
$ cd /etc/kubernetes/manifests

$ ls
etcd.yaml  kube-apiserver.yaml  kube-controller-manager.yaml  kube-scheduler.yaml
~~~

자 이제 이 yaml파일을 수정하면 됩니다.
저는 이 기본 5분인 eviction-timeout을 60초로 바꾸어 보겠습니다.

command에 <code>--pod-eviction-timeout=60s</code> 옵션을 추가해 줌으로써 말이죠.

~~~yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    kubernetes.io/config.hash: 47cace34a635d6f3e305eee20e0e7b30
    kubernetes.io/config.mirror: 47cace34a635d6f3e305eee20e0e7b30
    kubernetes.io/config.seen: "2021-05-28T03:40:42.961803243Z"
    kubernetes.io/config.source: file
  creationTimestamp: "2021-05-28T03:40:55Z"
  labels:
    component: kube-controller-manager
    tier: control-plane
  name: kube-controller-manager-master
  namespace: kube-system
  ownerReferences:
  - apiVersion: v1
    controller: true
    kind: Node
    name: master
    uid: 69f4d243-9420-45a9-928e-672a1d3b977a
  resourceVersion: "478"
  uid: 4cfad35d-8664-42cf-9ed9-5fd5b4d771a4
spec:
  containers:
  - command:
    - kube-controller-manager
    - --allocate-node-cidrs=true
    - --pod-eviction-timeout=60s      #<----------- 해당라인을 추가하면 이제 5분이아닌 60초로 변경됩니다.
    - --authentication-kubeconfig=/etc/kubernetes/controller-manager.conf
    - --authorization-kubeconfig=/etc/kubernetes/controller-manager.conf
    - --bind-address=127.0.0.1
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    - --cluster-cidr=10.244.0.0/16
    - --cluster-name=kubernetes
    - --cluster-signing-cert-file=/etc/kubernetes/pki/ca.crt
    - --cluster-signing-key-file=/etc/kubernetes/pki/ca.key
    - --controllers=*,bootstrapsigner,tokencleaner
    - --kubeconfig=/etc/kubernetes/controller-manager.conf
    - --leader-elect=true
    - --port=0
    - --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt
    - --root-ca-file=/etc/kubernetes/pki/ca.crt
    - --service-account-private-key-file=/etc/kubernetes/pki/sa.key
    - --service-cluster-ip-range=10.96.0.0/12
    - --use-service-account-credentials=true
    image: k8s.gcr.io/kube-controller-manager:v1.21.1

...중략

~~~
저장해주면 멈췄다가
~~~sh
kube-controller-manager-master    0/1     Running   0          3s
~~~
 옵션을 적용시켜 pod시 재시동됩니다~
~~~sh
kube-controller-manager-master    1/1     Running   0          3s
~~~

## eviction timeout의 존재이유?
위에서 eviction timeout의 값을 바꾸는 실습을 해보았는데요 이 timeout은 무엇과 연관이 있는지 두가지 케이스를 통해서 알아보겠습니다.

+ **case1: node가 먹통이 되었다가 설정해놓은 eviction-timeout보다 짦은시간안에 복구된경우**
  - 노드내에 있던 pod들이 kubectl에 의해 다시 해당노드에서 재시작됩니다.
+ **case2: node가 먹통이 된후 eviction-timeout을 초과한경우**
  - 해당 노드에있는 pod들을 모두 축출한 후에 노드가 재시동됩니다.
  - 해당 노드에있던 pod이 replicaset을 통해 생성되었다면 다시 다른노드에 잘 생성이 되겠지만, 일반 pod이라면 다시 살아나지 못할겁니다.




## 노드관련 명령어
+ <code> kubectl drain node </code>
~~~sh
$ kubectl drain node
~~~
  + 해당 노드에있는 pod들을 종료시키고, 다른노드에 재시작시킵니다 (graceful node shutdown) 또한 해당 노드를 unschedulable하게 만든다.

  + kubernetes의 버전 또는 리눅스 커널등의 업그레이드할 때 사용할 수 있겠습니다(kubeadm, kubelet, kubectl, ...)

+ <code> kubectl cordon node </code>
~~~sh
$ kubectl cordon node
~~~
  + 해당 노드를 unschedulable하게 만듭니다
  + **실행 중인 Pod를 축출하지는 않습니다**

+ <code> kubectl uncordon node </code>
~~~sh
$ kubectl uncordon node
~~~
  + 해당 노드를 schedulable하게 만든다
  + 업그레이드를 마친 노드를 스케쥴러블하게 만듭니다 (taint를 제거)
  + **drain명령에 의해서 집나간 pod들이 다시 집을 찾아오진 않습니다.**



## 쿠버네티스 버전 및 버전차이 지원

### 버전
쿠버네티스 버전은 x.y.z로 표현되는데, 여기서 x는 메이저 버전, y는 마이너 버전, z는 시맨틱 버전 용어에 따른 패치 버전입니다.
kubernetes에는 여러가지 컴포넌트가 있습니다.
각각의 버전은 전부 동일해야할까요? 아닙니다.

kube-apiserver의 버전이 주축이되고, 나머지 컴포넌트는 각각 다르게 한 두단계 버전이 낮거나 같아도 상관이 없습니다. 특별하게 **kubectl**의 경우는 kube-apiserver의 버전보다 한단계 높은경우도 지원이 됩니다.

더 자세한 내용은 [공식문서](https://kubernetes.io/ko/docs/setup/release/version-skew-policy/)에서 확인가능 합니다




## 클러스터 업그레이드

kubernets는 latest버전부터 두단계 낮은버전까지 서비스를 지원하는데요, 만약에 현재 1.19버전을 사용하고 있는데 1.22버전이 출시 된다면 업그레이드를 해야겠죠. 

그렇다면 1.19버전에서 한번에 1.22번으로 업그레이드 시키면될까요? 추천되는 방법에 의하면
한단계 한단계씩 차례차례로 업그레이드 해야된다고 합니다.

### 업그레이드 방법

쿠버네티스 환경을 어떻게 구성하였는가에 따라 업그레이드 방법이 다양합니다.

+ **구글의 GKE, AWS의 EKS등의 kubernete를 지원하는 cloud의 경우**
  + 간단한 클릭 몇번으로 업그레이드 가능

+ **The hard way로 설치한경우**
  + 한땀한땀 설치한 것처럼 업그레이드 또한 한땀한땀 해야합니다


저는 **kubeadm**을 이용하여 kubernetes환경을 구축하였고, **kubeadm**을 통한 업그레이드 방법에 대해 자세히 설명해 보겠습니다.

저의 쿠버네티스 환경은 한개의 master노드, 3개의 worker노드로 구성되어있습니다. 1.20버전을 이용하고 있는데 1.21버전으로 업그레이드 해보겠습니다.

### STEP1 - 마스터 노드에 kubeadm 업그레이드 시키기.


마스터 노드에 접속합니다
~~~sh
$ ssh gcloud
~~~
* **ssh config**에 설정을 해놨기에 이렇게 간단하게 접속가능합니다. **ssh config 설정방법**이 궁금하시면 [여기](https://shjeong92.github.io/2021/06/01/Handling-ssh-config.html)로 이동하세요.



그리고  kubernetes환경을 구축할때 hold시켜놨었던 kubeadm을 unhold시켜주고, apt-get 색인 업데이트후, kubeadm을 업데이트 한다음 다시 kubeadm을 hold시켜 줄겁니다. 
~~~sh
# 1.21.x-00에서 x를 최신 패치 버전으로 바꿉니다
apt-mark unhold kubeadm && \
apt-get update && apt-get install -y kubeadm=1.21.x-00 && \
apt-mark hold kubeadm

#버전 확인해주기
$ kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"21", GitVersion:"v1.21.1", GitCommit:"5e58841cce77d4bc13713ad2b91fa0d961e69192", GitTreeState:"clean", BuildDate:"2021-05-12T14:17:27Z", GoVersion:"go1.16.4", Compiler:"gc", Platform:"linux/amd64"}
~~~

**apt-mark unhold**와 **apt-mark hold** 해주는 이유는 kubeadm을 업그레이드하면 설치시 kubelet과 같은 다른 구성 요소가 기본적으로 최신 버전 으로 자동으로 업그레이드되기 때문입니다. (kubernetes는 여러버전을 한꺼번에 업그레이드 권장하지 않기때문입니다!) 이를 해결하기 위해 보류를 사용하여 패키지를 보류 된 것으로 표시하여 패키지가 자동으로 설치, 업그레이드 또는 제거되지 않도록합니다.

### STEP2 - 업그레이드 정보 확인하기

~~~sh
$ kubeadm upgrade plan

...

COMPONENT            CURRENT AVAILABLE

API Server           v1.20.0 v1.21.1

Controller Manager   v1.20.0 v1.21.1

Scheduler            v1.20.0 v1.21.1

Kube Proxy           v1.20.0 v1.21.1

...

~~~


### STEP3 - 업그레이드 적용시키기
~~~sh
$ kubeadm upgrade plan apply v1.21.1
(조금 오래걸릴 수도 있습니다)

아래와같이뜨면 성공
[upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.21.1". Enjoy!

[upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.
~~~


### STEP4 - node drain후 kubelet, kubectl 업데이트후 재시작하기

~~~sh
#노드 드레인하기(모든 pod 종료후 다른노드에 새로 실행하기 및 해당노드 unschedulable하게 만듬)
$ kubectl drain <node-to-drain> --ignore-daemonsets

$ apt-mark unhold kubelet kubectl && \
$ apt-get update && apt-get install -y kubelet=1.21.1-00 kubectl=1.21.1-00 && \
$ apt-mark hold kubelet kubectl

#설정 수정사항 재로딩
$ sudo systemctl daemon-reload
#kubelet 재시작
$ sudo systemctl restart kubelet

#업데이트가 끝났으므로 다시 schedulable하게 만들어주기
$ kubectl uncordon <node-to-uncordon>

~~~

여기까지가 마스터노드의 업그레이드 방법이었습니다.

---
# :rocket: 여기서부터는 워커노드들을 업그레이드 합니다.



### STEP5 - 워커 노드에 kubeadm 설치하기

우선 3개의 워커노드중 하나인 worker-1에 접속합니다.
~~~sh
ssh worker-1 
~~~
그 다음, 마스터노드에 설치했던과 같은방법으로 kubeadm를 설치합니다.
~~~sh
$ apt-mark unhold kubeadm && \
$ apt-get update && apt-get install -y kubeadm=1.21.x-00 && \
$ apt-mark hold kubeadm
~~~

아까 마스터노드에서 아래의 커맨드를 이용해 upgrade가능 버전을 확인하고 v1.21.1로 apply했었습니다.
~~~sh
$ kubeadm upgrade plan
$ kubeadm upgrade apply v1.21.1
~~~
하지만 워커노드에서는 아래의 커맨드를 이용하여 클러스터에서 kubeadm ClusterConfiguration 을 가져오며,
이 노드의 kubelet 구성을 업그레이드 합니다.
~~~sh
$ kubeadm upgrade node
~~~



그리고 마스터 노드로 이동하여 업그레이드 할 워커노드를 드레인 해줍니다. (마스터 노드에서 워커노드로 접속했었기에 접속 종료하면 마스터노드로 이동합니다.)
~~~sh
$ exit
logout
Connection to 10.128.0.5 closed.

$ kubectl drain worker-1 --ignore-daemonsets
~~~

### STEP6 - 워커노드의 kubelet 및 kubectl 업그레이드

~~~sh
# 다시 워커노드로 접속하기
$ ssh worker-1
 

# kubectl 과 kubelet 업데이트하기
$ apt-mark unhold kubectl kubelet && \
$ apt-get update && apt-get install -y kubectl=1.21.1-00 kubelet=1.21.1-00 && \
$ apt-mark hold kubectl kubelet
~~~

### STEP7 - kubelet 다시 시작하기

~~~sh
$ sudo systemctl daemon-reload
$ sudo systemctl restart kubelet
~~~

### STEP8 - cordon 해재하기

~~~sh
#마스터노드로 이동하기
$ exit

$ kubectl uncordon worker-1
~~~


### STEP9 - 반복하기

업그레이드 할 워커노드에
STEP5 ~ STEP8을 똑같이 진행해줍니다

마지막으로 업데이트가 잘 되었는지 확인해줍니다

~~~sh
$ kubectl get nodes
NAME         STATUS   ROLES                  AGE    VERSION
master       Ready    control-plane,master   6d8h   v1.21.1
worker-1     Ready    <none>                 6d8h   v1.21.1
worker-2     Ready    <none>                 6d8h   v1.21.1
worker-3     Ready    <none>                 6d8h   v1.21.1
~~~

버전이 v1.21.1로 잘 업그레이드 된것을 확인할 수 있습니다.



