---
title: On-premise HA Kubernetes cluster 구축
category: 
- DevOps
tags:
- kubernetes
summary: Creating higily available kubernetes cluster with haproxy and keepalived
thumbnail: "/assets/img/thumbnail/kubernetes.png"
---
쿠버네티스 클러스터는 `마스터`와 `노드`로 구성된다.

* `마스터` - 클러스터 상태를 관리하는 프로세스의 묶음(kube-apiserver, kube-controller-manager, kube-scheduler)으로 노드를 관리하고, 클러스터의 상태를 유지할 책임을 진다.
* `노드` - 애플리케이션과 워크플로우를 구동시키는 머신(VM, 물리 서버 등)이며, 마스터와 통신하는 kubelet, 네트워킹 서비스를 반영하는 kube-proxy 프로세스로 구성된다.

사용자는 마스터 내의 API Server로 원하는 상태(컨테이너 이미지, 레플리카 수, 네트워크, 디스크 등)를 요청하고, 마스터는 노드와 통신하면서 클러스터를 사용자가 원하는 상태로 관리한다. 사용자는 노드와 직접 인터페이스 할 일은 거의 없고, kubectl 또는 쿠버네티스 API를 이용해 API Server와 인터페이스 하게 된다. 클러스터를 관리하는 마스터가 shuwdown 된다면 쿠버네티스 클러스터 전체가 마비되기에 마스터의 HA는 필수이다.  
AWS-EKS, GCP-GKE 등 클라우드 서비스를 이용한다면, 클라우드 프로바이더가 마스터의 HA를 제공하므로 사용자가 HA를 고민하지 않아도 된다. 하지만 On-premise 환경에서 쿠버네티스 클러스터를 구축한다면 HA를 고려하여 구축해야 한다. HA Proxy + Keppalived + Kubernetes 조합으로 3개의 마스터와 3개의 노드를 가지는 고가용성 쿠버네티스 클러스터를 구축해보자.

 **NOTE:**  
 마스터의 수는 1,3,5,7과 같이 홀수개를 유지하는 것이 좋다. [참고](https://stackoverflow.com/questions/53843195/why-kubernetes-ha-need-odd-number-of-master)
{: .notice--info}

---

## Prerequisite

* 6대의 물리머신 또는 가상머신 (Ubuntu 18.04)
* Master1 - 192.168.0.3
* Master2 - 192.168.0.4
* Master3 - 192.168.0.5
* Node1   - 192.168.0.6
* Node2   - 192.168.0.7
* Node3   - 192.168.0.8
* 쿠버네티스 클러스터의 엔드포인트로 사용할 가상 IP - 192.168.0.244

## 필요 패키지 설치

HA Proxy, Keepalived, docker, kubelet 등 필요 패키지를 설치하자.

### HAProxy, Keepalived 설치/설정(All Masters)

```bash
$ apt-get install keepalived haproxy
$ sudo vi /etc/keepalived/keepalived.conf
```

```bash
vrrp_instance VI_1 {
    state MASTER
    interface eth0 ${eth}
    virtual_router_id ${router_id}
    priority ${priority} #Master 간 우선 순위 조정, 값이 클 수록 높은 우선 순위
    advert_int 1
    authentication {
        auth_type AH
        auth_pass iech6peeBu6Thoo8xaih
    }
    virtual_ipaddress {
        ${vip}
    }
}
```

```bash
$ sudo vi /etc/haproxy/haproxy.cfg
```

```bash
global
...
 
defaults
...
 
frontend k8s-api
  bind ${host ip}:8443
  bind 0.0.0.0:8443
  bind 127.0.0.1:8443
  mode tcp
  option tcplog
  default_backend k8s-api
 
backend k8s-api
  mode tcp
  option tcplog
  option tcp-check
  balance roundrobin
  default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
  server master1 ${Master1_ip}:6443 check
  server master2 ${Master2_ip}:6443 check
  server master3 ${Master3_ip}:6443 check
```

* [설정 참고](https://github.com/donghoon-khan/kubernetes-demo/tree/master/cluster/on-premise)

### Docker, Kubernetes 설치(All Masters, All Nodes)

```bash
$ sudo apt-get remove docker docker-engine docker.io containerd runc
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
$ sudo apt-key fingerprint 0EBFCD88
$ sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
$ sudo curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
$ sudo apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
$ sudo apt-get update
$ sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
$ sudo apt-get install docker-ce docker-ce-cli containerd.io
$ sudo apt-get install kubeadm kubelet kubectl
$ sudo apt-mark hold kubeadm kubelet kubectl docker-ce docker-ce-cli containerd.io
```

* [설치 스크립트 참고](https://github.com/dhkang22/kubernetes-cluster/blob/master/on-premise/setup.sh)

## Kubernetes 클러스터 생성

이제 쿠버네티스 클러스터를 생성하고 Master와 Node를 Join하면 된다.

### All Masters

```bash
$ sudo swapoff -a
$ vi /etc/ssh/sshd_config #PermitRootLogint yes
$ sudo systemctl stop keepalived
$ sudo systemctl stop haproxy
```

### Master1

```bash
$ sudo systemctl start keepalived
$ sudo systemctl enable keepalived
$ vi kubeadm-config.yaml
```

``` bash
apiVersion: kubeadm.k8s.io/v1beta1
kind: ClusterConfiguration
kubernetesVersion: stable
apiServer:
  certSANs:
  - "${vip}"
networking:
  #cidr=https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/#pod-network 참조
  podSubnet: ${cidr} #10.244.0.0/16 (flannel)
controlPlaneEndpoint: "{$vip}"
```

```bash
$ vi deploy_certs_conf.sh
```

```bash
#!/bin/bash
set -e

EXITCODE=0
MASTERS="Master2 Master3"
CERTS=$(find /etc/kubernetes/pki/ -maxdepth 1 -name '*ca.*' -o -name '*sa.*')
ETCD_CERTS=$(find /etc/kubernetes/pki/etcd/ -maxdepth 1 -name '*ca.*')
for MASTER in $MASTERS; do
  ssh $MASTER mkdir -p /etc/kubernetes/pki/etcd
  scp $CERTS $MASTER:/etc/kubernetes/pki/
  scp $ETCD_CERTS $MASTER:/etc/kubernetes/pki/etcd/
  scp /etc/kubernetes/admin.conf $MASTER:/etc/kubernetes
done
 
exit $EXITCODE
```

```bash
$ kubeadm init --config=kubeadm-config.yaml #Token, hash 값 저장
$ chmod +x deploy_certs_conf.sh
$ bash deploy_certs_conf.sh
```

### Master2, Master3

``` bash
$ kubeadm join ${vip}:6443 --token ${token} --discovery-token-ca-cert-hash ${hash} --experimental-control-plane
```

### All Nodes

``` bash
$ kubeadm join ${vip}:6443 --token ${token} --discovery-token-ca-cert-hash ${hash}
```

### All Masters

``` bash
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
$ kubectl get nodes
$ kubectl get pods --all-namespaces
$ sudo systemctl start keepalived
```

### Master1

```bash
$ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
$ kubectl get nodes
$ kubectl get pod --all-namespaces
```

### All Master

``` bash
$ sudo systemctl start haproxy
```

## Conclusion

모든 단계를 마치면 아래의 그림과 같은 구성을 가지게 된다.
![on-premise ha kubernetes cluster](/assets/img/posts/2020-08-10-block-diagram.png)  

keepalived의 active상태의 Master가 죽으면 stanby상태의 Master가 active상태가 되어 VIP로 들어오는 트래픽을 ha-proxy로 전달한다. 동기화를 위해 ha-proxy로 들어온 트래픽을 모든 마스터의 apiserver로 전달함으로써 HA Master구조를 갖는 Kubernetes cluster를 구축할 수 있다.
이제 다음 작업으로 Ceph 또는 GlusterFS를 사용해 클러스터에서 PV로 사용할 파일시스템을 구축 해 보자.