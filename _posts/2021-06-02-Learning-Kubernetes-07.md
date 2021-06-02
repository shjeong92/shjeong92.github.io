---
layout: article
title: "[ #7 ] 배포 전략"
tags: Docker Kubernetes Replicasets Deployment static pod strategy 쿠버네티스 rolling update
---


쿠버네티스에는아래 그림과 같이 두가지 배포방법이 있습니다

![deplstrategy](https://user-images.githubusercontent.com/75003424/120428975-9582c300-c3af-11eb-95cf-ba8825576440.png)


## Recreate
Recreate 방식은 이전버전 A를 종료시킨후 신규버전 B를 롤아웃 시키는 방식입니다.

### pros
+ 가장 쉬운 배포방법
+ 클라우드 리소스 비요이 적음

### cons
+ A가 중지되고 B가 배포되기전까지 서비스가 중단되어 사용자에게 악영향을 줌

## Rolling Update
+ 기존서버를 순차적으로 중지시키며 순차적으로 업그레이드 시키는 방식으로, kubernetes에서 spec.strategy.type을 지정하지 않으면 기본적으로 RollingUpdate를 사용합니다.


+ 처음 쿠버네티스에 입문하였을때 replicaset과, deployment의 차이점은 yaml파일로만 놓고 봤을때는 다른점이 kind 부분 말고는 없었습니다. 똑같이 파드를 적정갯수 만큼 생성해주고, crash되면 다시 복구시켜주는 점에서는 말이죠.

오늘 배포방식에 대해 공부하면서 왜 deployment를 사용하여 pod을 배포하는가에 대한 궁금증을 풀 수 있었습니다.


Deployment를 사용하여 pod을 배포하게되면 롤아웃을 생성합니다. 그리고 이 롤아웃은 새로운 배포 버전(revision)을 생성합니다. 이말이 잘 이해가 안 되실수도 있는데

자세한 설명은 실습을 진행하면서 하도록 하겠습니다.

아래와같이 deployment definition을 작성한 yaml파일이 있습니다.
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
        image: nginx
  replicas: 3
  selector:
    matchLabels:
      type: front-end
~~~

아래의 명령어를 사용하여 deployment를 생성하게되면 
~~~sh
kubectl apply -f depl.yaml
~~~

myapp-deployment라는 이름을가진 deployment와, myapp-deployment-hash값 을 가진 replicaset, 그리고 deployment를 통해 생성된 replicaset에 의해 생성된 pod을 확인할 수 있습니다.

~~~sh
kubectl get all
NAME                                    READY   STATUS    RESTARTS   AGE
pod/myapp-deployment-7df67f74c5-4nkmq   1/1     Running   0          5m27s
pod/myapp-deployment-7df67f74c5-bnl9r   1/1     Running   0          5m27s
pod/myapp-deployment-7df67f74c5-jsd9p   1/1     Running   0          5m27s

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   5d2h

NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/myapp-deployment   3/3     3            3           5m27s

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/myapp-deployment-7df67f74c5   3         3         3       5m27s
~~~

아직까지는 별 차이가 없지요, deployment가 생겼다는것 외에는 말입니다. 

하지만 사실은 이외에도 <code>revision</code>이라는 것을 생성합니다. 

이는 해당 deployment를 업데이트 할때마다 생기고, 이전 <code>revision</code>의 정보도 가집니다. 또한, 새로운 버전의 <code>replicaset</code>을 생성하고, 이전버전의 <code>replicaset</code>의 정보를 남겨놓습니다. 

만약에 업데이트후에 문제가 생겼을시에 롤백을 가능케 하기 위해서 입니다. 그리고 이러한점이 **deployment**를 사용해서 pod을 배포했을때랑 **replicaset**을 사용했을때의 가장 큰 차이점이 아닐까 싶습니다.

## Rollout 상태확인

deployment를 사용하여 pod을 배포하게되면 rollout을 생성한다고 했었는데 
추가로 사용할 수 있는 커맨드도 생깁니다. <code>rollout</code> 이라는 커맨드인데요. 

rollout 커맨드를 통하여 deployment의 revision및 history를 확인할 수 있습니다


<code> kubectl rollout status deployment </code> 커맨드를 사용하여 다음과 같이 롤아웃 생성의 상태를 확인할 수 있습니다.


~~~sh
$ kubectl rollout status deployment/myapp-deployment
Waiting for deployment "myapp-deployment" rollout to finish: 0 of 3 updated replicas are available...
Waiting for deployment "myapp-deployment" rollout to finish: 1 of 3 updated replicas are available...
Waiting for deployment "myapp-deployment" rollout to finish: 2 of 3 updated replicas are available...
deployment "myapp-deployment" successfully rolled out
~~~

## Rollout history 확인

또한 <code> kubectl rollout history deployment </code> 커맨드를 사용하면 아래와 같이 리비전 정보를 확인할 수 있는데요

~~~sh
$ kubectl rollout history deploy/myapp-deployment
deployment.apps/myapp-deployment 
REVISION  CHANGE-CAUSE
1         <none>
~~~

하나의 revision 밖에 보이지않습니다. 그 이유는 depl.yaml을 이용하여 처음으로 배포한 버전이기 때문입니다.

## Rollout update
해당 deployment를 업데이트하면 어떤일이 생기는지 보도록 하겠습니다.

우선 yaml파일을 수정해줍니다

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
        image: nginx:1.20-alpine       <----- 바뀐부분
  replicas: 3
  selector:
    matchLabels:
      type: front-end
~~~
이미지를 바꿀때에는 간단하게 set image를 통해서도 가능하지만 개인적으로 변경상황을 이력으로 남기는게 좋다고 생각하여 해당 방법을 사용합니다.

변경사항을 적용시켜줍니다.
~~~sh
$ kubectl apply -f depl.yaml
deployment.apps/myapp-deployment configured
~~~

그다음 replicaset과 revision을 확인해봅시다
~~~sh
$ kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
myapp-deployment-79896f8f68   3         3         3       5m14s
myapp-deployment-7df67f74c5   0         0         0       5m42s
~~~
replicaset이 하나더 생겼으며,

revision도 하나더 생긴것을 확인할 수 있습니다. 
~~~
$ kubectl rollout history deploy myapp-deployment
deployment.apps/myapp-deployment 
REVISION  CHANGE-CAUSE
2         <none>
1         <none>
~~~

하지만 change-cause가 None 으로 되있으면 무엇이 바뀌었는지 알 수 가없는데, 아래 커맨드를 이용하면 따로 설정이 가능합니다.

<code>kubectl annotate deployment.v1.apps/[deployment name] kubernetes.io/change-cause="reason of update"</code>

해당커맨드를 사용하여 주석을 달고 다시 확인해봅시다

~~~sh
$ kubectl annotate deployment.v1.apps/myapp-deployment kubernetes.io/change-cause="image updated to 1.20"
deployment.apps/myapp-deployment annotated


$ kubectl rollout history deployment myapp-deployment
deployment.apps/myapp-deployment 
REVISION  CHANGE-CAUSE
1         <none>
2         image updated to 1.20
~~~
주석이 잘 달린것을 확인할 수 있습니다.

## Rollback
만약에 새로운 버전으로 업데이트를 했는데 문제가 생겼을때 이전 버전 으로 돌아갈 수 있는 기능입니다.

<code> kubeclt rollout undo deployment </code>커맨드를 사용하여 롤백 시킬 수 있습니다.

직접 실습해봅시다.

~~~sh
$ kubectl rollout undo deployment myapp-deployment
deployment.apps/myapp-deployment rolled back
~~~
성공적으로 롤백 되었다고 뜹니다. 

우선 image가 다시 원래대로 돌아왔는지 확인해봅니다

~~~sh
$ kubectl describe deployment myapp-deployment
Name:                   myapp-deployment
Namespace:              default
CreationTimestamp:      Wed, 02 Jun 2021 07:05:06 +0000
Labels:                 app=myapp
                        type=front-end
Annotations:            deployment.kubernetes.io/revision: 3
                        kubernetes.io/change-cause: 
Selector:               type=front-end
Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=myapp
           type=front-end
  Containers:
   nginx-container:
    Image:        nginx          <------- 1.20 에서 다시 원래대로 돌아온것을 확인할 수 있습니다.
    Port:         <none>
    Host Port:    <none>
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   myapp-deployment-7df67f74c5 (3/3 replicas created)
Events:
  Type    Reason             Age                  From                   Message
  ----    ------             ----                 ----                   -------
  Normal  ScalingReplicaSet  23m                  deployment-controller  Scaled up replica set myapp-deployment-79896f8f68 to 1
  Normal  ScalingReplicaSet  23m                  deployment-controller  Scaled down replica set myapp-deployment-7df67f74c5 to 2
  Normal  ScalingReplicaSet  23m                  deployment-controller  Scaled up replica set myapp-deployment-79896f8f68 to 2
  Normal  ScalingReplicaSet  23m                  deployment-controller  Scaled down replica set myapp-deployment-7df67f74c5 to 1
  Normal  ScalingReplicaSet  23m                  deployment-controller  Scaled up replica set myapp-deployment-79896f8f68 to 3
  Normal  ScalingReplicaSet  23m                  deployment-controller  Scaled down replica set myapp-deployment-7df67f74c5 to 0
  Normal  ScalingReplicaSet  4m28s                deployment-controller  Scaled up replica set myapp-deployment-7df67f74c5 to 1
  Normal  ScalingReplicaSet  4m24s                deployment-controller  Scaled down replica set myapp-deployment-79896f8f68 to 2
  Normal  ScalingReplicaSet  4m24s                deployment-controller  Scaled up replica set myapp-deployment-7df67f74c5 to 2
  Normal  ScalingReplicaSet  4m20s (x2 over 24m)  deployment-controller  Scaled up replica set myapp-deployment-7df67f74c5 to 3
  Normal  ScalingReplicaSet  4m20s                deployment-controller  Scaled down replica set myapp-deployment-79896f8f68 to 1
  Normal  ScalingReplicaSet  4m16s                deployment-controller  Scaled down replica set myapp-deployment-79896f8f68 to 0
  ~~~






그럼 replicaset과 revision은 어떨까요?

~~~sh
#이미지 1.20으로 업데이트후 replicaset 상태
$ kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
myapp-deployment-79896f8f68   3         3         3       5m14s
myapp-deployment-7df67f74c5   0         0         0       5m42s
~~~

~~~sh
#롤백후 replicaset상태
$ kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
myapp-deployment-79896f8f68   0         0         0       25m
myapp-deployment-7df67f74c5   3         3         3       26m
~~~

**revision**
~~~sh
$ kubectl rollout history deployment myapp-deployment
deployment.apps/myapp-deployment 
REVISION  CHANGE-CAUSE
2         image updated to 1.20
3         <none>
~~~



replicaset은 이전 버전의 replicaset으로 바뀐것을 확인할 수가 있는데 revision은 1로돌아가는것이아닌 1이 증가한 것을 확인할 수 있습니다.
여기서 궁금한점이 생겼습니다.

**undo를 여러번하면 어떻게될까요?**
~~~sh
$ kubectl rollout history deployment myapp-deployment
deployment.apps/myapp-deployment 
REVISION  CHANGE-CAUSE
3         <none>
4         image updated to 1.20
~~~


### 결과
~~~sh
$ kubectl describe deployment myapp-deployment
Name:                   myapp-deployment
Namespace:              default
CreationTimestamp:      Wed, 02 Jun 2021 07:05:06 +0000
Labels:                 app=myapp
                        type=front-end
Annotations:            deployment.kubernetes.io/revision: 4
                        kubernetes.io/change-cause: image updated to 1.20
Selector:               type=front-end
Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=myapp
           type=front-end
  Containers:
   nginx-container:
    Image:        nginx:1.20
    Port:         <none>
    Host Port:    <none>
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   myapp-deployment-79896f8f68 (3/3 replicas created)
Events:
  Type    Reason             Age                  From                   Message
  ----    ------             ----                 ----                   -------
  Normal  ScalingReplicaSet  14m                  deployment-controller  Scaled up replica set myapp-deployment-7df67f74c5 to 1
  Normal  ScalingReplicaSet  14m                  deployment-controller  Scaled down replica set myapp-deployment-79896f8f68 to 2
  Normal  ScalingReplicaSet  14m                  deployment-controller  Scaled up replica set myapp-deployment-7df67f74c5 to 2
  Normal  ScalingReplicaSet  14m (x2 over 33m)    deployment-controller  Scaled up replica set myapp-deployment-7df67f74c5 to 3
  Normal  ScalingReplicaSet  14m                  deployment-controller  Scaled down replica set myapp-deployment-79896f8f68 to 1
  Normal  ScalingReplicaSet  13m                  deployment-controller  Scaled down replica set myapp-deployment-79896f8f68 to 0
  Normal  ScalingReplicaSet  2m13s (x2 over 33m)  deployment-controller  Scaled up replica set myapp-deployment-79896f8f68 to 1
  Normal  ScalingReplicaSet  2m9s (x2 over 33m)   deployment-controller  Scaled down replica set myapp-deployment-7df67f74c5 to 2
  Normal  ScalingReplicaSet  2m9s (x2 over 33m)   deployment-controller  Scaled up replica set myapp-deployment-79896f8f68 to 2
  Normal  ScalingReplicaSet  2m5s (x2 over 33m)   deployment-controller  Scaled down replica set myapp-deployment-7df67f74c5 to 1
  Normal  ScalingReplicaSet  2m5s (x2 over 33m)   deployment-controller  Scaled up replica set myapp-deployment-79896f8f68 to 3
  Normal  ScalingReplicaSet  2m1s (x2 over 33m)   deployment-controller  Scaled down replica set myapp-deployment-7df67f74c5 to 0

~~~

바로 직전단계의 버전만을 기억하고 있었고, 다시 undo 하자 image가 1.20으로 바뀐것을 확인할 수 있었습니다.

따라서 undo를 여러번 한다고 더 과거의 버전으로 돌아갈 수는 없는것을 확인할 수 있었습니다.
