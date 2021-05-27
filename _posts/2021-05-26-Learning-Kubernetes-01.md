---
layout: article
title: "[ #1 ] Kubernetes의 구조"
tags: Docker Kubernetes CKA 도커 컨테이너 container etcd
---

## ETCD란 무엇인가?

### Distributed Reliable Key Value Store
분산 시스템의 중요한 데이터를위한 신뢰할 수있는 분산 key/value 저장소이며 다음과 같은 특징이 있습니다.
+ 단순함: 잘 정의 된 사용자용 API
+ 보안: 선택적 클라이언트 인증을 사용하는 자동 TLS
+ 속도: 10000 쓰기 / sec

## mac에 etcd 설치하여 사용해보기

~~~sh
# brew를 이용하여 간편히 설치
$ brew install etcd
# etcd를 백그라운드 환경으로 실행시키기
$ brew services start etcd

# Key value 생성
$ etcdctl put key "I am a value"
OK

# Key를 통한 value 호출
$ etcdctl get key
key
I am a value

# Key 삭제하기
$ etcdctl del key
1

~~~





# :rocket: 마스터

## Kubernetes 안의 etcd 클러스터

+ Nodes
+ Pods
+ Configs
+ Secrets
+ Accounts
+ Roles
+ Bindings
+ Others

모든 클러스터 데이터를 담는 쿠버네티스 뒷단의 저장소로 사용되는 일관성·고가용성 키-값 저장소입니다.


## kube-apiserver
+ kubernetes의 API server이며 worker node를 관리하는 Kubelet과 통신합니다.
+ kubectl 을 통하여 kube-apiserver과 통신하는것 이외에도 post request를 통해서도 가능합니다.

  1. Authenticate User
    - 사용자가 인장된 사용자인지 확인
  2. Validate Request
    - 요청이 올바른 요청인지 확인
  3. Retrieve data
    - 요청에 대한 결과를 응답한다. (ex. Pod 생성 요청 완료)
  4. Updatae ETCD
    - 생성 요청된 POD 정보를 ETCD에 업데이트 합니다ㅣ.
  5. Scheduler
    - Scheduler에게 POD이 배치될 Worker Node를 요청합니다.
  6. Kubelet
    - Schduler에게 받은 Worker Node의 Kubelet에게 POD 생성을 요청합니다.
 
## kube-scheduler

스케줄러는 생성되었지만 배정되지 못하여 실행되지 않은 컨테이너를 감지하고 실행할 노드를 선택하는 역할을 합니다. 스케줄링 결정을 위해서 고려되는 요소는 리소스에 대한 개별 및 총체적 요구 사항, 노드의 상태, 제약 조건등을 포함.

## Kube Controller Manager

컨트롤러를 구동하는 컴포넌트로, 앞에서 설명한 control-loop를 돌면서 current state가 desired state로 되도록 바꾸어주는 역할을 담당한다. 컨트롤러의 종류에도 여러가지가 존재하는데 논리적으로 각 컨트롤러는 개별 프로세스이지만 복잡성을 낮추기 위해 모두 단일 바이너리로 컴파일되고 단일 프로세스 내에서 실행된다.

+ 노드 컨트롤러: 노드 컨트롤러: 노드가 다운되었을 때에 대응.
+ 레플리케이션 컨트롤러: 레플리케이션 오브젝트에서 알맞은 수의 Pod를 유지.
+ 엔드포인트 컨트롤러: 서비스와 Pod를 연결.
+ 서비스 어카운트 & 토큰 컨트롤러: 새로운 네임스페이스에 대한 기본 계정과 API 접근 토큰을 생성.

## Cloud-controller-manager
클라우드별 컨트롤 로직을 포함하는 쿠버네티스 컨트롤 플레인 컴포넌트. 클라우드 컨트롤러 매니저를 통해 클러스터를 클라우드에 연결하여 클라우드 플랫폼의 리소스를 추가 및 사용할 수 있습니다. kube-controller-manager와 마찬가지로 cloud-controller-manager도 논리적으로 독립적인 여러 control-loop를 단일 프로세스로 실행하는 단일 바이너리로 결합합니다.

<br>
<br>
다음 컨트롤러들은 클라우드 플랫폼의 의존성을 가질 수 있다.

+ 노드 컨트롤러: 노드가 응답을 멈춘 후 클라우드 상에서 삭제되었는지 판별하기 위해 클라우드 제공 사업자에게 확인하는 것
+ 라우트 컨트롤러: 기본 클라우드 인프라에 경로를 구성하는 것
+ 서비스 컨트롤러: 클라우드 제공 사업자 로드밸런서를 생성, 업데이트 그리고 삭제하는 것

# :rocket: 워커 노드
워커 노드는 마스터로부터 전달받은 명령에 따라 컨테이너를 실행합니다.

## kubelet(노드 관리자)
클러스터의 각 노드에서 실행되는 핵심 컴포넌트로 다양한 메커니즘을 통해 제공된 명세(Spec)의 집합을 받아서 컨테이너를 실행시킵니다. 여기서 끝나는 것이 아니라 정상적으로 동작하는지 지속적으로 모니터링하고 주기적으로 마스터 노드의 api 서버와 통신하여 정보를 주고받습니다.

## kube-proxy
클러스터의 각 노드에서 실행되는 네트워크 프록시로, 쿠버네티스의 서비스 오브젝트를 구현합니다. 즉 노드의 네트워크 규칙을 관리하여 서비스마다 개별 IP를 가질 수 있게 만들어주고 클러스터 내부나 외부의 트래픽을 Pod로 전달할 수있도록 해줍니다.

## container runtime
실제 컨테이너 실행을 담당하는 소프트웨어이다. 예를 들어 도커, CRI-O, containerd 등이 있습니다.