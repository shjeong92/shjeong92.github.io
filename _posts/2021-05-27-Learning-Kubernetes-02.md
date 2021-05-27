---
layout: article
title: "[ #2 ] Kubernetes 설치 및 환경 구성하기 "
tags: Docker Kubernetes CKA 도커 컨테이너 container etcd Google cloud
---

minikube를 이용하면 정말 간단하게 로컬머신에 테스트환경을 구성할 수 있었습니다 또한 <code>kubectl</code> 커맨드에 친숙해지는데 잘 활용했었지요,
 하지만 이는 싱글 노드 클러스트이고, 다른 노드를 추가할 수 없습니다. 그 말은 즉 진또배기 쿠버네티스를 체험할 수 없는것이며 모는 기능을 제대로 체험할 수 없다는 것이지요.

<br>
따라서 kubeadm을 이용하여 마스터노드와 워커노드들을 구성해보기로 했습니다.
로컬 환경에서 vagrant와 virtualbox를 통해 실험환경을 구축하고싶었지만 m1 맥에 그런것은 없습니다 ㅠㅠ
어쩌다보니 구글 클라우드를 처음시작하면 무료크레딧으로 300$를 제공하는것을 발견하였고 구글클라우드를 이용하여 쿠버네티스 환경을 구축해보기로 결정했습니다.

<br>
이번 포스트에서는 쿠버네티스 설치부터 기본적인 환경까지 구성할것입니다.

---

쿠버네티스는 기본적으로 <code>마스터 노드</code> 와 <code>워커 노드</code>로 구성됩니다.

<code>마스터 노드</code>는 <code>워커 노드</code>에 Pod를 할당하고 Pod 안에 컨테이너를 띄우게 하는 역할을 합니다.
또한 쿠버네티스의 상태를 관리하고 여러 Pod 들의 스케줄링도 하는 등 쿠버네티스에서 중추적인 역할을 합니다.
<code>워커 노드</code>는 <code>마스터 노드</code>와 통신하면서 Pod를 할당 받고 그 안에 컨테이너를 띄워 유지 및 관리하는 역할을 합니다. 
또한 네트워크나 볼륨에 대한 기능도 컨트롤도 합니다.

마스터 노드1개와 와 워커 노드 2개를 생성할 것이며, 
운영체제는 Ubuntu 18.04 LTS 를 사용하여 구성하도록 하겠습니다.
<br>
그리고 마스터 및 모든 워커노드의 스펙은 아래와 같습니다.

+ CPU : 2 core
+ RAM : 2 GB
+ Storage : 30 GB

## kubeadm을 활용한 환경구축

<code>kubeadm</code>이란, kubernetes에서 제공하는 기본적인 도구이며, 
kubernetes 클러스터를 빨리 구축하기 위한 다양한 기능을 제공합니다.

<code>kubeadm</code>은 아래와같이 다양한 커맨드명령어를 사용하여 클러스터를 구축할 수 있습니다.

+ **kubeadm init**
  - Kubernetes 컨트롤 플레인 노드를 초기화합니다.
  - 즉, 마스터 노드를 초기화합니다.

+ **kubeadm join**
  - Kubernetes 워커 노드를 초기화하고 클러스터에 연결합니다.
+ **kubeadm upgrade**
  - Kubernetes 클러스터를 업그레이드 합니다.
+ **kubeadm token**
  - 부트 스트랩 토큰을 사용한 인증에 설명된대로 부트 스트랩 토큰은 클러스터에 참여하는 노드와 제어 평면 노드 사이에 양방향 신뢰를 설정하는 데 사용됩니다.
+ **kubeadm reset**
  - kubeadm init 혹은 kubeadm join의 변경사항을 최대한 복구합니다.
+ **kubeadm version**
  - kubeadm 버젼을 보여줍니다.
+ **kubeadm alpha**
  - 정식으로 배포된 기능은 아니지만 kubernetes측에서 사용자 피드백을 얻기 위해 인증서 갱신, 인증서 만료 확인, 사용자 생성, kubelet 설정 등 다양한 기능을 제공한다고 합니다.


## 시작하기전에 확인할 것

+ 2GB 또는 그 이상의 메모리(각노드별)
+ 2CPUs 또는 그이상의 시피유
+ 클러스터의 모든 시스템 간 완벽한 네트워크 연결되었는지 확인
+ 모든 노드에 대한 고유한 호스트 이름, 고유한 MAC 주소, 고유한 product_uuid
+ 각노드의 컨트롤러인 kubelet이 제대로 동작하기위해선 swap이 disabled 되어야 합니다.

~~~sh
#swap 비활성화하기
swapoff -a
~~~

## 모든 노드의 Mac address 및 product_uuid 확인

로컬 머신의경우 하드웨어 기기들은 각각의 유니크한 주소를 가지는 반면에 가상 머신은 같은 id를 가질 수도 있습니다.
만약에 같은 Mac address 및 product_uuid를 가지고 있으면 설치에 실패 할 수 있기에 확인하여야 합니다.

~~~sh
#MAC address 확인하기
ifconfig -a

#product_uuid 확인하기
sudo cat /sys/class/dmi/id/product_uuid
~~~
위 쉘 커맨드를 각각의노드에 적용하여 중복되는 맥주소나 제품아이디가 있는지 확인합니다.


##iptables 설정하기
~~~sh
#br_netfilter모듈 로드되었는지 확인하기
$ lsmod | grep br_netfilter
# 아무정보도 뜨지않는다.
$ 

#명시적으로 br_netfilter 로드하기
$ sudo modprobe br_netfilter
#다시 확인하면 br_netfilter이 로드된걸 볼수 있어요.
$ lsmod | grep br_netfilter
br_netfilter           28672  0
bridge                176128  1 br_netfilter

~~~






리눅스 노드의 iptables가 브리지된 트래픽을 올바르게 보기 위한 요구 사항으로, sysctl 구성에서 net.bridge.bridge-nf-call-iptables 가 1로 설정되어 있는지 확인해야 합니다. 
다음3개의 스크립트를 실행해주시면,


~~~sh
$ cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

$ cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

$ sudo sysctl --system
~~~

아래와같이 뜰거에요.

~~~
* Applying /etc/sysctl.d/10-console-messages.conf ...
kernel.printk = 4 4 1 7
* Applying /etc/sysctl.d/10-ipv6-privacy.conf ...
net.ipv6.conf.all.use_tempaddr = 2
net.ipv6.conf.default.use_tempaddr = 2
* Applying /etc/sysctl.d/10-kernel-hardening.conf ...
kernel.kptr_restrict = 1
* Applying /etc/sysctl.d/10-link-restrictions.conf ...
fs.protected_hardlinks = 1
fs.protected_symlinks = 1
* Applying /etc/sysctl.d/10-lxd-inotify.conf ...
fs.inotify.max_user_instances = 1024
* Applying /etc/sysctl.d/10-magic-sysrq.conf ...
kernel.sysrq = 176
* Applying /etc/sysctl.d/10-network-security.conf ...
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.all.rp_filter = 1
net.ipv4.tcp_syncookies = 1
* Applying /etc/sysctl.d/10-ptrace.conf ...
kernel.yama.ptrace_scope = 1
* Applying /etc/sysctl.d/10-zeropage.conf ...
vm.mmap_min_addr = 65536
* Applying /usr/lib/sysctl.d/50-default.conf ...
net.ipv4.conf.all.promote_secondaries = 1
net.core.default_qdisc = fq_codel
* Applying /etc/sysctl.d/60-gce-network-security.conf ...
net.ipv4.tcp_syncookies = 1
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.secure_redirects = 1
net.ipv4.conf.default.secure_redirects = 1
net.ipv4.ip_forward = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.icmp_ignore_bogus_error_responses = 1
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1
kernel.randomize_va_space = 2
kernel.panic = 10
* Applying /etc/sysctl.d/99-cloudimg-ipv6.conf ...
net.ipv6.conf.all.use_tempaddr = 0
net.ipv6.conf.default.use_tempaddr = 0
* Applying /etc/sysctl.d/99-sysctl.conf ...
* Applying /etc/sysctl.d/k8s.conf ...
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
* Applying /etc/sysctl.conf ...

~~~



## 필수 포트 확인하기
**마스터 노드에서 필요한 필수 노트**
+ 6443 포트 : Kubernetes API Server / Used By All
+ 2379~2380 포트 : etcd server client API / Used By kube-apiserver, etcd
+ 10250 포트 : Kubelet API / Used By Self, Control plane
+ 10251 포트 : kube-scheduler / Used By Self
+ 10252 포트 : kube-controller-manager / Used By Self


**워커 노드에서 필요한 필수 포트**
+ 10250 포트 : Kubelet API / Used By Self, Control plane
+ 30000~32767 포트 : NodePort Services / Used By All


## 컨테이너 런타임 설치하기
저는 docker를 설치할 것입니다..

~~~sh
# 구버전의 도커 설치되어있다면 삭제하기.
$  sudo apt-get remove docker docker-engine docker.io containerd runc
~~~

도커 공홈에가면 도커 설치하는 방법이 여러가지있는데 저는 그중에서 가장 간단한 방법으로 설치하겠습니다.

~~~sh
#도커를 설치해주는 스크립트 get-docker.sh 로저장하기
$ curl -fsSL https://get.docker.com -o get-docker.sh
#스크립트 실행하기
$ sudo sh get-docker.sh
~~~

아래와같이 뜨면 성공입니다.

~~~sh
# Executing docker install script, commit: 7cae5f8b0decc17d6571f9f52eb840fbc13b2737
+ sh -c apt-get update -qq >/dev/null
+ sh -c DEBIAN_FRONTEND=noninteractive apt-get install -y -qq apt-transport-https ca-certificates curl >/dev/null
+ sh -c curl -fsSL "https://download.docker.com/linux/ubuntu/gpg" | apt-key add -qq - >/dev/null
Warning: apt-key output should not be parsed (stdout is not a terminal)
+ sh -c echo "deb [arch=amd64] https://download.docker.com/linux/ubuntu bionic stable" > /etc/apt/sources.list.d/docker.list
+ sh -c apt-get update -qq >/dev/null
+ [ -n  ]
+ sh -c apt-get install -y -qq --no-install-recommends docker-ce >/dev/null
+ [ -n 1 ]
+ sh -c DEBIAN_FRONTEND=noninteractive apt-get install -y -qq docker-ce-rootless-extras >/dev/null
+ sh -c docker version
Client: Docker Engine - Community
 Version:           20.10.6
 API version:       1.41
 Go version:        go1.13.15
 Git commit:        370c289
 Built:             Fri Apr  9 22:46:01 2021
 OS/Arch:           linux/amd64
 Context:           default
 Experimental:      true

Server: Docker Engine - Community
 Engine:
  Version:          20.10.6
  API version:      1.41 (minimum version 1.12)
  Go version:       go1.13.15
  Git commit:       8728dd2
  Built:            Fri Apr  9 22:44:13 2021
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          1.4.4
  GitCommit:        05f951a3781f4f2c1911b05e61c160e9c30eaa8e
 runc:
  Version:          1.0.0-rc93
  GitCommit:        12644e614e25b05da6fd08a38ffa0cfe1903fdec
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad0

================================================================================

To run Docker as a non-privileged user, consider setting up the
Docker daemon in rootless mode for your user:

    dockerd-rootless-setuptool.sh install

Visit https://docs.docker.com/go/rootless/ to learn about rootless mode.


To run the Docker daemon as a fully privileged service, but granting non-root
users access, refer to https://docs.docker.com/go/daemon-access/

WARNING: Access to the remote API on a privileged Docker daemon is equivalent
         to root access on the host. Refer to the 'Docker daemon attack surface'
         documentation for details: https://docs.docker.com/go/attack-surface/

================================================================================
~~~

혹시 모르니 도커 커맨드를 써서 설치가 잘되었는지 확인해봅니다
~~~sh
$ sudo docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
~~~

Docker 설치가 완료 되었으면, Docker 데몬이 사용하는 드라이버를 cgroupfs 대신 systemd를 사용하도록 설정합니다
왜냐하면 kubernetes에서 권장하는 Docker 데몬의 드라이버는 systemd 이기 때문이고, systemd를 사용하면 kubernetes가 클러스터 노드에서 사용 가능한 자원을 쉽게 알 수 있도록 해주기 때문이다.
또한, 시스템에 두개의 cgroup관리자가 있으면 리소스가 부족할 때 불안정해지는 사례가 있다고 하니 무조건적으로 설정해줍시다.
아래 명령어를 통해 Docker 데몬의 드라이버를 교체합니다.

~~~sh
$ sudo mkdir /etc/docker

$ cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

#도커 재시작과 부팅시 실행되게 설정
$ sudo systemctl enable docker
$ sudo systemctl daemon-reload
$ sudo systemctl restart docker
~~~


## kubeadm, kubelet 및 kubectl 설치하기

**kubeadm은 kubelet 또는 kubectl 을 설치하거나 관리하지 않으므로, kubeadm이 설치하려는 쿠버네티스 컨트롤 플레인의 버전과 일치하는지 확인해야 합니다**
따라서 kubeadm 및 쿠버네티스를 업그레이드 할 때에는 버전에 신경쓰셔야 합니다.

아래에서는 kubeadm, kubelet, 및 kubectl 설치후에 업데이트를 홀드 시킴으로써 실수로 <code>sudo apt-get update</code>
실행한 커맨드가 해당 패키지에 영향이 가지 않도록 할것입니다.

~~~sh
# 패키지 색인을 최신으로 업데이트하고, kubeadm, kubelet, 및 kubectl 를 설치하는데 필요한 패키지 설치하기.
$ sudo apt-get update
$ sudo apt-get install -y apt-transport-https ca-certificates curl

# 구글 클라우드의 공개 사이닝 키를 다운로드 하기.
$ sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

# 쿠버네티스 apt 리포지터리를 추가하기.
# *apt리포는 apt 도구로 읽을 수있는 deb 패키지 및 메타 데이터 파일이 포함 된 네트워크 서버 또는 로컬 디렉토리입니다*
$ echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

# apt 패키지 색인을 업데이트 후 kubeadm, kubelet 및 kubectl다운로드 한다음, 버전 고정시키기.
$ sudo apt-get update
$ sudo apt-get install -y kubelet kubeadm kubectl
$ sudo apt-mark hold kubelet kubeadm kubectl
~~~

# 처음부터 이 줄 윗부분까지는 마스터노드와 워커노드 모두에 적용되어야 할 내용입니다.

--- 

---

## 마스터 노드 세팅

마스터 노드를 실행시키려면 <code>kubeadm init</code> 커맨드를 사용해야합니다.
해당명령어에는 여러가지 옵션이 들어가는데, 여기서 필요한 옵션값은 다음과 같습니다.

+ --pod-network-cidr : Pod 네트워크를 설정할 때 사용
+ --apiserver-advertise-address : 특정 마스터 노드의 API Server 주소를 설정할 때 사용

## Pod 네트워크 설정
우선 세팅할 클러스터에서 Pod가 서로 통신할 수 있도록 Pod 네트워크 애드온을 설치 해야 합니다.
kubeadm을 통해 만들어진 클러스터는 CNI(Container Network Interface) 기반의 애드온이 필요합니다.

기본적으로 k8s에서 제공해주는 kubenet이라는 네트워크 플러그인이 있지만, 
매우 기본적이고 간단한 기능만 제공하는 네트워크 플러그인이기 때문에 이 자체로는 크로스 노드 네트워킹이나 네트워크 정책과 같은 기능들은 구현되어 있지 않습니다.
따라서 kubeadm은 kubernetes가 기본적으로 지원해주는 네트워크 플러그인인 kubenet을 지원하지 않고, CNI 기반 네트워크만 지원합니다.
여기서는 Flannel이라는 Pod 네트워크 애드온을 설치하여 사용할 것입니다.
<br>
Flannel을 사용하려면 kubeadm init 명령어에 --pod-network-cidr=10.244.0.0/16 이라는 인자를 추가해서 실행해야 하며,
 10.244.0.0/16 이라는 네트워크는 Flannel에서 기본적으로 권장하는 네트워크 대역입니다.
<br>
Pod 네트워크가 호스트 네트워크와 겹치면 문제가 발생할 수 있기 때문에 일부로 호스트 네트워크로 잘 사용하지 않을 것 같은 네트워크 대역을 권장하는 것입니다.

## API Server 주소 설정
~~~sh
#마스터노드입니다.
$ ifconfig 

cni0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1410
        inet 10.244.0.1  netmask 255.255.255.0  broadcast 10.244.0.255
        inet6 fe80::44f7:e3ff:fe0e:7d15  prefixlen 64  scopeid 0x20<link>
        ether 46:f7:e3:0e:7d:15  txqueuelen 1000  (Ethernet)
        RX packets 48997  bytes 4021971 (4.0 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 54114  bytes 4910238 (4.9 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

docker0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 172.17.255.255
        ether 02:42:29:4d:09:4f  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

ens4: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1460
        inet 10.178.0.2  netmask 255.255.255.255  broadcast 0.0.0.0
        inet6 fe80::4001:aff:feb2:2  prefixlen 64  scopeid 0x20<link>
        ether 42:01:0a:b2:00:02  txqueuelen 1000  (Ethernet)
        RX packets 456862  bytes 219264003 (219.2 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 291537  bytes 122783503 (122.7 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
~~~

위 <code>ifconfig</code> 커맨드의 출력에서 해당 마스터 노드의 IPv4 주소는 ens4 이더넷의 10.178.0.2 인것을 확인할 수 있고, 
<br>
<code>--apiserver-advertise-address</code> 옵션을 통해 해당 주소를 kubeadm 에게 전달할 수 있습니다.

## :rocket:마스터 노드 생성 및 실행

최종적으로 실행할 <code> kubeadm init </code>는 아래와 같습니다.

~~~sh
$ sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=10.178.0.2
~~~

해당 명령어를 실행했을때 뜨는 결과 창인데요 여기서 짚고가야할 두가지가 있습니다

~~~sh
$ sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=10.178.0.2
[init] Using Kubernetes version: v1.21.1
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local master] and IPs [10.96.0.1 10.178.0.2]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [localhost master] and IPs [10.178.0.2 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [localhost master] and IPs [10.178.0.2 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 12.003615 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.21" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node master as control-plane by adding the labels: [node-role.kubernetes.io/master(deprecated) node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node master as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: ol9g7l.d1f7klw0uh9kmsm5
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.178.0.2:6443 --token ol9g7l.d1f7klw0uh9kmsm5 \
	--discovery-token-ca-cert-hash sha256:723e45876b6726f23ff39566d6ae9046423edae440fd76bd328b40ab9b0a8b7d 
~~~


---

~~~sh
# 첫째로는 마스터노드에서 실행해야할 커맨드에요
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
~~~
해당 명령어는 Root 계정이 아닌 다른 사용자 계정에서 kubectl 커맨드 명령어를 사용하여 클러스터를 제어하기 위해 사용하는 명령어입니다.
기본적으로 kubernetes에서는 /etc/kubernetes/admin.conf 파일을 가지고 kubernetes 관리자 Role의 인증 및 인가 처리를 하며, 
위 명령어는 사용자 계정의 $HOME/.kube/config 디렉터리에 admin.conf 파일을 복사함으로써 사용자 계정이 kubectl을 사용하면서 관리자 Role의 인증 및 인가 처리를 받을 수 있도록 하는 것입니다.
여기서 말한 사용자 계정이란, 마스터 노드의 Shell에 접속한 계정입니다.




~~~sh
#두번째로는 각각의 워커노드에 실행시켜줄 커맨드에요 메모장에 복사붙여놓기 해놓으시길 추천드립니다!

$ kubeadm join 10.178.0.2:6443 --token ol9g7l.d1f7klw0uh9kmsm5 \
	--discovery-token-ca-cert-hash sha256:723e45876b6726f23ff39566d6ae9046423edae440fd76bd328b40ab9b0a8b7d 
~~~
잃어버리면 토큰 다시받아야하고 귀찮아집니다
물론 24시간지나면 토큰 다시 받아야하긴 하지만요 ㅎㅎ

## Flanner 사용을 위해 Pod 네트워크 배포하기

~~~sh
$ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
~~~
이 명령어를 실행하면 아래와 같은 메세지를 확인할 수 있습니다.

~~~sh
podsecuritypolicy.policy/psp.flannel.unprivileged created
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.apps/kube-flannel-ds created
~~~

이제 마스터 노드가 잘 세팅되었는지 아래 명령어를 통해 확인해볼 수 있습니다.

~~~sh
노드 확인하기
$ kubectl get nodes
NAME         STATUS   ROLES                  AGE   VERSION
master       Ready    control-plane,master   49m   v1.21.1

마스터 노드 내의 모든 pod 확인하기
$ kubectl get pods --all-namespaces
NAMESPACE     NAME                             READY   STATUS    RESTARTS   AGE
kube-system   coredns-558bd4d5db-8mclv         1/1     Running   0          49m
kube-system   coredns-558bd4d5db-qbs2t         1/1     Running   0          49m
kube-system   etcd-master                      1/1     Running   0          50m
kube-system   kube-apiserver-master            1/1     Running   0          50m
kube-system   kube-controller-manager-master   1/1     Running   0          50m
kube-system   kube-flannel-ds-dmbnj            1/1     Running   0          45m
kube-system   kube-flannel-ds-gmhfd            1/1     Running   0          44m
kube-system   kube-flannel-ds-h27rp            1/1     Running   0          46m
kube-system   kube-flannel-ds-smnhd            1/1     Running   0          45m
kube-system   kube-proxy-2hxsg                 1/1     Running   0          49m
kube-system   kube-proxy-dhgfh                 1/1     Running   0          44m
kube-system   kube-proxy-q9kwk                 1/1     Running   0          45m
kube-system   kube-proxy-x7m4d                 1/1     Running   0          45m
kube-system   kube-scheduler-master            1/1     Running   0          50m

~~~

## 워커 노드 세팅

마스터 노드 생성후 출력된 메세지 맨아래에서 저장 해놓으라고 했던부분을 쓸 시간이 왔습니다.


**연결하고싶은 모든 워커노드에 아래 커맨드를 입력해주세요**
~~~sh

$ kubeadm join 10.178.0.2:6443 --token ol9g7l.d1f7klw0uh9kmsm5 \
	--discovery-token-ca-cert-hash sha256:723e45876b6726f23ff39566d6ae9046423edae440fd76bd328b40ab9b0a8b7d 
~~~

## 마스터노드에서 확인하기

원래 두개 등록되어있었는데 포스트 작성하면서 하나 더만들어졌네요
두번째 만들때는 그래도 수월하게 진행한것 같습니다.
~~~sh
$ kubectl get nodes
NAME         STATUS   ROLES                  AGE   VERSION
instance-1   Ready    <none>                 44m   v1.21.1
master       Ready    control-plane,master   49m   v1.21.1
worker-1     Ready    <none>                 44m   v1.21.1
worker-2     Ready    <none>                 44m   v1.21.1
~~~

긴 글 보시느라 고생많으셨습니다.