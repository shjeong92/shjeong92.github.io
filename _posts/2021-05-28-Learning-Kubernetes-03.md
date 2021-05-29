---
layout: article
title: "[ #3 ] Replicasets & Deployment "
tags: Docker Kubernetes Replicasets Deployment Google cloud
---

---

## Replicaset

먼저 Deployment의 개념중에서 가장 중요한것은 ReplicaSet입니다. Replication Controller의 새로운 버전으로 Label Selector를 통해 노드 상의 여러 Pod의 생성/복제/삭제 등의 라이프 싸이클을 관리합니다.

### 작동 방식
레플리카셋을 정의하는 필드는 획득 가능한 파드를 식별하는 방법이 명시된 셀렉터, 유지해야 하는 파드 개수를 명시하는 레플리카의 개수, 그리고 레플리카 수 유지를 위해 생성하는 신규 파드에 대한 데이터를 명시하는 파드 템플릿을 포함한다. 그러면 레플리카셋은 필드에 지정된 설정을 충족하기 위해 필요한 만큼 파드를 만들고 삭제합니다. 레플리카셋이 새로운 파드를 생성해야 할 경우, 명시된 파드 템플릿을 사용합니다.



~~~yaml
#replicaset.yaml
apiVersion: apps/v1
kind: ReplicaSet 
metadata:
  name: myapp-replicaset
  labels:
    app: myapp
    type: front-end

spec:
  #replicate할 pod명시하기
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
  #pod을 몇개 유지할것인지?      
  replicas: 3
  #ReplicaSet은 이미 생성되있는 같은 종류의 pod들까지 신경써가면서 pod을 생성합니다.
  selector: 
    matchLabels:
      type: front-end


~~~
Replicaset은 selector가 있는데 만약 이에 match되는 label을 가진 pod이 있다면 그것도 갯수에 포함시킵니다.
~~~yaml
#pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
    type: front-end
spec:
  containers:
    - name: nginx-container
      image: nginx
~~~

우선 labels.type 이 동일한 pod 하나를 생성해줍니다.
~~~sh
$ kubectl apply -f pod-definition.yml 
pod/myapp-pod created
$ kubectl get pods
NAME        READY   STATUS              RESTARTS   AGE
myapp-pod   0/1     ContainerCreating   0          8s
~~~

레이블이 같으니 replicaset.yaml을 apply하면 두개가 생성 되야합니다 한번 확인해봅시다.
~~~sh
$ kubectl apply -f replicaset-definition.yml 
replicaset.apps/myapp-replicaset created
$ kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
myapp-pod                1/1     Running   0          23s
myapp-replicaset-f9f87   1/1     Running   0          13s
myapp-replicaset-szrtw   1/1     Running   0          13s
~~~

딱 2개가 생성된것을 확인할 수 있습니다.
<br>
만약 pod.yaml을 통해 생성된 pod을 지우면 어떻게될까요?

~~~sh
$ kubectl delete pod myapp-pod
pod "myapp-pod" deleted

$ kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
myapp-replicaset-f9f87   1/1     Running   0          2m46s
myapp-replicaset-spvjx   1/1     Running   0          9s
myapp-replicaset-szrtw   1/1     Running   0          2m46s
~~~

ReplicaSet에 replicas를 3으로 세팅해놔서 동일한 label의 팟이 사라지자 자동으로 명시한 pod을 생선한 것을 확인할 수 있습니다.

---

## Deployment

Deployment는 Replication controller와 Replica Set의 좀더 상위 추상화 개념입니다. kubernetes docs에서는 ReplicaSet 이나 Replication Controller를 바로 사용하는 것보다, 좀 더 추상화된 Deployment를 사용하는것이 권장하고 있습니다.

yaml파일을 살펴도보면 차이점은 별로 없습니다.



위의 replicaset.yaml파일에서 한줄만 수정하면 deployment 를 위한 yaml파일을 만들 수 있습니다.

~~~yaml
apiVersion: apps/v1
kind: Deployment   <----------- 수정된부분
metadata:
  name: myapp-replicaset
  labels:
    app: myapp
    type: front-end

spec:
  #replicate할 pod명시하기
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
  #pod을 몇개 유지할것인지?      
  replicas: 3
  #ReplicaSet은 이미 생성되있는 같은 종류의 pod들까지 신경써가면서 pod을 생성합니다.
  selector: 
    matchLabels:
      type: front-end

~~~


deployment 또한 replicaset과 같이 이미 동일한 라벨의 pod이 생성되어있으면 또다른 3개의 pod을 만들지 않습니다.

위의 Replicaset.yaml을 통해 만들어진 동일한 label의 pod이 존재하기때문에 새로운 pod을 만들지 않습니다. 또한
이미 동일한 replicaset이 있기에 다른 replicaset을 만들지 않습니다.

기존의 Replicaset object를 지우면 어떻게 될까요?
~~~sh
$ kubectl delete -f replicaset-definition.yml 
replicaset.apps "myapp-replicaset" deleted

# 삭제직후 replicaset을 바로 확인해보면 Deployment.yaml에의해 새로운 replicaset이 생성된것을 확인할 수 있고
$ kubectl get replicaset
NAME                          DESIRED   CURRENT   READY   AGE
myapp-replicaset-7df67f74c5   3         3         3       3s

# 이전 replicaset에 의해 생성 된 pod들이 삭제되면서, 새로운 replicaset에의해 pod이 재 생성되는것을 볼 수 있습니다.
$ kubectl get pods
NAME                                READY   STATUS              RESTARTS   AGE
myapp-replicaset-7df67f74c5-c992z   1/1     Running             0          5s
myapp-replicaset-7df67f74c5-cwk22   0/1     ContainerCreating   0          5s
myapp-replicaset-7df67f74c5-zzx7h   1/1     Running             0          5s
myapp-replicaset-jnqgg              0/1     Terminating         0          73s
myapp-replicaset-qghpt              0/1     Terminating         0          73s
~~~