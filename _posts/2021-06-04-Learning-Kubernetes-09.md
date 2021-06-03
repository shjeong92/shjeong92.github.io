---
layout: article
title: "[ #9 ] 클러스터 백업 & 복구"
tags: Kubernetes maintenance static pod strategy 쿠버네티스 update
---

## 백업과 복구 방법

백업해야할 정보들
+ Resource Configuration
+ ETCD Cluster
+ Persistence Volumes





### Backup - Resource configs

~~~sh
kubectl get all -all-namespaces -o yaml > all-deploy-services.yaml
~~~

를 이용하면 대부분의 리소스를 yaml파일에 담을수는 있지만 모두를 다 담을 수 있는것은 아닙니다

그래서 나온툴은 예전에 ARK라고 불리었고 현재는 VELERO로 불리는 툴을 이용하면됩니다.

### Backup - ETCD

ETCD는 클러스터의 상태를 key=value형식으로 저장하고 있습니다. 

그리고 ETCD는 각 마스터노드에 존재하고, key=value의 데이터는 마스터 노드의 /var/lib/etcd 에 저장됩니다.

ETCD는 built in  스냅샷 solution을 가지고 있는데요 