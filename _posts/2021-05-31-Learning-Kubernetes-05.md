---
layout: article
title: "[ #5 ] pod 스케쥴링-2 "
tags: Docker Kubernetes Replicasets Deployment Google cloud
---

## nodeSelector
nodeSelector 는 파드를 특정 레이블이 있는 노드로 제한하는 매우 간단한 방법을 제공합니다.
### 1. 노드에 레이블 붙이기

노드에 레이블을 붙여서 추후 pod을 원하는 노드에 할당할때 사용할 수 있습니다.
~~~sh
kubectl label nodes <노드 이름> <레이블 키>=<레이블 값>
~~~



### 2. pod설정에 nodeSelector 필드 추가하기
실행하고자 하는 파드의 설정 파일을 가져오고, 이처럼 nodeSelector 섹션을 추가합니다. 아래의 예를 들어보면

~~~yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
~~~

spec아래에 nodeSelector을 추가해줍니다.

~~~yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  nodeSelector:
    #<레이블 키>: <레이블 값>
    disktype: ssd
~~~

## node affinity

**nodeSelector** 을 사용했을때는 매칭 조건이 레이블에 key=value, key와 value가 일치했을때 pod을 노드에 있었습니다.<br>

**nodeAffinity** 는 operator로 Equal, In, NotIn, Exists, DoseNotExist 를 사용할 수 있어, 더 여러가지 제약을 정의 할 수 있습니다.

또한 **nodeAffinity** 에는 아래와 같이 두가지 타입이 있습니다.

+ requiredDuringSchedulingIgnoredDuringExecution
+ preferredDuringSchedulingIgnoredDuringExecution


첫번째 타입은 파드가 노드에 스케줄되려면 규칙을 만족해야만 하는것이고, 없다면 pod는 pending 상태가 됩니다
두번째 타입은 선호하는 선호하는 또다른 키와 벨류, 연산자를 추가할 수 있는데요, 첫번째 조건을 만족하는 노드가 여러개 있을경우에, 두번째 조건의 weight(선호도) 점수에따라
pod이 스케쥴되는 형태입니다.

미래에는 requiredDuringSchedulingRequiredDuringExecution 라는 새로운 타입이 나올것이라고 합니다, 앞서 소개드린 두 타입은 이미 pod이 스케줄 되었을때 node의 label이 바뀌거나 없어지면서
affinity의 조건과 맞지 않더라도 이미 배정되었음으로 무시합니다. 하지만 미래에 나올 새로운 타입은 이미 배정받은 pod이라도 노드의 label이 삭제,변경되면 조건에 맞지 않는 pod을 중지 시키는 타입 이라고합니다.


### 예제
~~~yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/e2e-az-name
            operator: In
            values:
            - e2e-az1
            - e2e-az2
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: another-node-label-key
            operator: In
            values:
            - another-node-label-value
  containers:
  - name: with-node-affinity
    image: k8s.gcr.io/pause:2.0
~~~


Node affinity는 여러 affinity를 동시에 적용할 수 있는데요 저는 아래와 같은 경우에 대해 실험해 보았습니다.

+ **requiredDuringSchedulingIgnoredDuringExecution 만 적용했을 경우**:
  - 조건을 만족하는 노드가 없을경우 pod는 스케쥴되지않고 pending상태가 됩니다.


+ **requiredDuringSchedulingIgnoredDuringExecution 과 preferredDuringSchedulingIgnoredDuringExecution 를 동시에 적용할경우**:
  - 첫번째 조건을 을 만족하는 노드가 여러개 있을때 두번째 조건을 만족하는 노드가 있다면 두번째 조건을 만족하는 노드에 pod가 배치됩니다.
  - 두번째 조건만을 만족하는 경우에는 pod는 스케쥴링되지 않고 pending 상태가 됩니다.

+ **preferredDuringSchedulingIgnoredDuringExecution 만 적용할 경우**:
  - 조건을 만족하는 노드가 있을경우 해당 노드에 스케쥴링되고, 조건을 만족하는 노드가 없다면 랜덤으로 스케쥴 되었습니다.



위의 예제에서는 두 개의 Affinity가 정의되어 있는데요 만약 노드들중 **requiredDuringSchedulingIgnoredDuringExecution** 을 만족하는 노드가 없다면 해당 pod은 pending 상태가
될것이며, 해당 조건을 만족하는 노드가 여러개일 경우 **preferredDuringSchedulingIgnoredDuringExecution** 조건이 맞는 노드로 배정 될 것입니다. <br>

<code>weight</code> 는 선호도입니다 만약 여러개의 **preferredDuringSchedulingIgnoredDuringExecution** 을 적용하였을때에는 weight이 높은 노드에 스케쥴됩니다.
