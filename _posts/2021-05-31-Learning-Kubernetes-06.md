---
layout: article
title: "[ #6 ] static pod"
tags: Docker Kubernetes Replicasets Deployment static pod
---

## Static pods이란

제일 처음에 kubernetes의 구조에서 공부했을때 마스터노드에 <code>kube-apiserver</code>가 있고, 각 워커노드에는 <code>kubelet</code> 이 존재하며 마스터노드의 kube-apiserver의 명령에따라 pod을 지우거나 삭제하거나 했었죠. <br>
이번 시간에는 마스터 노드의 kube-scheduler의 영향을 받지 않는 Static pod 에 대해서 알아보겠습니다.

각 워커노드에 존재하는 kubelet 또한 pod이 죽거나 에러가 발생했을떄 아래의 방법으로 다시 살릴 수 있습니다.

우선 static pod을 생성할 노드를 선택하여 ssh 접속해줍니다. 저는 worker-1에 접속하도록 하겠습니다



~~~sh
$ kubectl get nodes -o wide
NAME         STATUS   ROLES                  AGE    VERSION
instance-1   Ready    <none>                 3d3h   v1.21.1
master       Ready    control-plane,master   3d3h   v1.21.1
worker-1     Ready    <none>                 3d3h   v1.21.1
worker-2     Ready    <none>                 3d3h   v1.21.1

#worker 에 접속하기
$ ssh -i ~/.ssh/rsa-gcp-worker-1 shjeong920522@10.178.0.5
~~~

### 특징 1

+ **kubelet** 은 기본적으로 /etc/kubernetes/manifests 파일안의 pod.yaml 을 바라보는데, 이는 kubelet.service 의 config.yaml파일안에 staticPodPath에 정의되어있습니다.

직접 확인해봅시다

우선 아래 커맨드를 입력하여 config.yaml파일의 위치를 알아냅니다
~~~sh
# 모든서비스 확장 | kubelet 정보중에서 | --config 포함하는줄 가져오기
$ ps -ef | grep kubelet | grep "\--config"
root     18270     1  1 May28 ?        01:31:14 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml --network-plugin=cni --pod-infra-container-image=k8s.gcr.io/pause:3.4.1
~~~
해당 파일을 확인해 봅시다
~~~yaml
$ sudo vi config=/var/lib/kubelet/config.yaml

apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 0s
    enabled: true
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 0s
    cacheUnauthorizedTTL: 0s
cgroupDriver: systemd
clusterDNS:
- 10.96.0.10
clusterDomain: cluster.local
cpuManagerReconcilePeriod: 0s
evictionPressureTransitionPeriod: 0s
fileCheckFrequency: 0s
healthzBindAddress: 127.0.0.1
healthzPort: 10248
httpCheckFrequency: 0s
imageMinimumGCAge: 0s
kind: KubeletConfiguration
logging: {}
nodeStatusReportFrequency: 0s
nodeStatusUpdateFrequency: 0s
resolvConf: /run/systemd/resolve/resolv.conf
rotateCertificates: true
runtimeRequestTimeout: 0s
shutdownGracePeriod: 0s
shutdownGracePeriodCriticalPods: 0s
staticPodPath: /etc/kubernetes/manifests        <------------------------------ staticPodPath 가 설정되어 있는것을 확인할 수 있습니다.
streamingConnectionIdleTimeout: 0s
syncFrequency: 0s
volumeStatsAggPeriod: 0s
~~~
위와 같이 파일을 통으로 확인해보아도 되지만 grep명령어를 사용하면 더욱 편히 찾을 수도 있습니다.

~~~sh
$ sudo grep static /var/lib/kubelet/config.yaml
staticPodPath: /etc/kubernetes/manifests
~~~

### 특징 2
+ 만약 해동 폴더내에 kind: Pod인 yaml 파일이있다면 kubelet이 자동으로 이를 생성하며 static pod의 이름에는 자동으로 해당 노드의 이름이 suffix로 붙습니다.

잘 생성되나 확인해 봅시다.
~~~sh
$ cd /etc/kubernetes/manifests
$ sudo vi static.yaml
~~~

간단한 pod을 해당 볼더에 생성해줍니다.
~~~yaml
#static.yaml
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

**마스터** 노드로 이동하여 확인해봅시다.
~~~sh
$ kubectl get pods
NAME                 READY   STATUS    RESTARTS   AGE
myapp-pod-worker-1   1/1     Running   0          42m
~~~
static pod를 생성한 노드의 이름인 worker-1 가 Pod이름의 suffix로 붙어있는것을 확인할 수 있습니다.

### 특징 3

+ 마스터노드에서 kubectl delete 명령어를 사용하더라도 워커노드의 kubelet이 다시 살려냅니다. static pod는 해당 워커노드의 kubelet이 관리하기 떄문입니다.
따라서 해당 pod의 image를 변경하고싶다면 manifest 폴더안의 해당 yaml파일의 이미지를 수정해야하고, 지우려면 해당 yaml파일을 삭제하면 됩니다.

그럼 정말로 다시 시작되는지 마스터노드에서 **myapp-pod-worker-1**을 삭제해 봅시다

~~~sh
$ kubectl delete pod myapp-pod-worker-1
pod "myapp-pod-worker-1" deleted
~~~
지운다음에 바로 확인해보면

~~~sh
$ kubectl get pods
NAME                 READY   STATUS    RESTARTS   AGE
myapp-pod-worker-1   0/1     Pending   0          1s

kubectl get pods
NAME                 READY   STATUS    RESTARTS   AGE
myapp-pod-worker-1   1/1     Running   0   
~~~
pending상태에 있다가 running으로 바뀌는 것을 확인할 수 있습니다.
따라서 static pod을 지우고싶으면 해당 노드로 가서 해당 yaml파일을 삭제하는 방법뿐입니다.