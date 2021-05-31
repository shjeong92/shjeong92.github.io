---
layout: article
title: "[ #4 ] pod 스케쥴링-1 "
tags: Docker Kubernetes Replicasets Deployment Google cloud
---

## Taint & Toleration

+ taint: 노드마다 설정가능
+ toleration: taint를 무시할 수 있음


주로 노드를 지정된 역할만 하게할때 사용합니다.
예를들어 gpu잇는 노드에는 다른 pod들은 올라가지않고 gpu쓰는 pod들만 올라가게 하는등의 상황에 사용할 수 있습니다.

<br>
taint에는 사용할 수 있는 3가지 옵션이 있습니다.

+ NoSchedule : toleration이 없으면 pod이 스케쥴되지 않음, 기존 실행되던 pod에는 적용 안됨
+ PreferNoSchedule : toleration이 없으면 pod을 스케줄링안하려고 하지만 필수는 아님, 클러스터내에 자원이 부족하거나 하면 taint가 걸려있는 노드에서도 pod이 스케줄링될 수 있음
+ NoExecute : toleration이 없으면 pod이 스케줄되지 않으며 기존에 실행되던 pod도 toleration이 없으면 종료시킴.


taint 형식은 다음과 같습니다.
~~~sh
$ kubectl taint node {nodename} {key}={value}:{option}
~~~
하나의 노드에 taint 적용하기
~~~sh
$ kubectl taint node worker-1 key=value:NoSchedule
node/worker-1 tainted
~~~

describe로 잘 적용 되었는지 확인해 봅시다.
~~~sh
$ kubectl describe node worker-1 | grep -i taint
Taints:             key=value:NoSchedule
~~~
잘 적용이 되었군요.

taint의 옵션이 <code>NoSchedule</code> 이므로 pod에 toleration 필드에 같은 key, value가 없다면 생성되지 않겠죠?
확인해봅시다.

### :rocket:테스트해보기
Deployment로 pod을 띄어보겠습니다.

**환경**:<br>
실습에 사용한 클러스터는 마스터노드1개 워커노드3개로 구성되었으며,
<br>
이름은 다음과 같고, 워커노드들의 cpu와 ram스팩은 모두 같게 설정하였습니다. <br>
taint는 worker-1 노드에 설정하였습니다.
master
- worker-1
- worker-2
- instance-1

우선 pod을 세개 생성하는 test-taint.yaml이라는 deployment를 만들어줍니다
~~~sh
$ kubectl create deployment test-taint --image=nginx --replicas=3 --dry-run=client -o yaml > test-taint.yaml
~~~

yaml파일을 열어보면 이렇습니다.

~~~yaml
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: test-taint
  name: test-taint
spec:
  replicas: 3
  selector:
    matchLabels:
      app: test-taint
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: test-taint
    spec:
      containers:
      - image: nginx
        name: nginx
        resources: {}
status: {}
~~~


만들어진 yaml파일을 apply 해봅시다.
~~~sh
$ kubectl apply -f test-taint.yaml
deployment.apps/test-taint created
~~~
pod을 확인해보면 taint를 설정한 node에는 pod이 배정되지 않은것을 확인 할 수 있습니다.
~~~sh
$ kubectl get pods -o wide
NAME                          READY   STATUS    RESTARTS   AGE    IP           NODE         NOMINATED NODE   READINESS GATES
test-taint-6bd4977499-2thmv   1/1     Running   0          106s   10.244.3.7   instance-1   <none>           <none>
test-taint-6bd4977499-4sccs   1/1     Running   0          106s   10.244.1.6   worker-2     <none>           <none>
test-taint-6bd4977499-h4rt8   1/1     Running   0          106s   10.244.3.8   instance-1   <none>           <none>
~~~

이제 이 deployment를 삭제하고 toleration을 적용시킨후 다시 deployment를 apply 해보겠습니다.
~~~sh
$ kubectl delete deployment test-taint
deployment.apps "test-taint" deleted

$ vim test-taint.yaml
~~~
spec.template.spec 밑에 toleration옵션을 추가해줍니다.
~~~yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: test-taint
  name: test-taint
spec:
  replicas: 3
  selector:
    matchLabels:
      app: test-taint
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: test-taint
    spec:
      tolerations:
      - key: key
        operator: Equal
        value: value
        effect: NoSchedule
      containers:
      - image: nginx
        name: nginx
        resources: {}
status: {}
~~~

수정된 yaml파일 다시 배포하기:

~~~sh
$ kubectl apply -f test-taint.yaml
deployment.apps/test-taint created
~~~

pod 리스트를 다시 확인해보면 각 1개의노드에 1개의 pod이 생성된걸 확인할 수 있습니다.
~~~
$ kubectl get pods -o wide
NAME                          READY   STATUS    RESTARTS   AGE   IP           NODE         NOMINATED NODE   READINESS GATES
test-taint-77b85b88c6-8hq5l   1/1     Running   0          43s   10.244.3.9   instance-1   <none>           <none>
test-taint-77b85b88c6-957bv   1/1     Running   0          43s   10.244.1.7   worker-2     <none>           <none>
test-taint-77b85b88c6-f6xn2   1/1     Running   0          43s   10.244.2.8   worker-1     <none>           <none>
~~~