---
title: Kubernetes auto scaling
category: 
- DevOps
tags:
- kuberntes
summary: kubernetes auto scaling
thumbnail: "/assets/img/thumbnail/kubernetes.png"
---
서비스를 운영하다 보면 갑작스런 리퀘스트의 증가와 같이 예상치 못한 변수로 인해 CPU, Memory와 같은 리소스가 부족해 원할한 서비스를 할 수 없는 상황이 발생할 수 있다.
쿠버네티스는 이러한 문제를 해결하기 위해 애플리케이션을 확장하는 방법을 제공하는데,
쿠버네티스가 관리하는 가장 작은 컴퓨팅 단위인 파드를 `수직형` 또는 `수평형`으로 확장하거나 `노드를 확장`하여 리소스 부족현상을 처리할 수 있다.

prometheus + kube-apiserver의 custom API, external API와 같은 방식으로 커스텀 및 외부 측정항목으로도 HPA, VPA 구축이 가능하지만 이 글에서는 [metrics-server](https://github.com/kubernetes-sigs/metrics-server)를 이용하여 CPU, memory에 대한 metrics를 수집하여 HPA, VPA를 구축할 것이다.
metircs-server와 autoscaler는 kube-apiserver의 resource API를 이용해 metric을 주고 받는다. 

 **NOTE:**  
 kubectl top 명령어를 이용해 node or pod의 리소스 사용량을 확인 할 수 있는데, 이는 kube-apiserver의 resource API를 이용하는 것이다.
{: .notice--info}

metrics-server는 kubectl을 이용해 간단하게 설치가 가능하다.
```bash
$ kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
serviceaccount/metrics-server created
clusterrole.rbac.authorization.k8s.io/system:aggregated-metrics-reader created
clusterrole.rbac.authorization.k8s.io/system:metrics-server created
rolebinding.rbac.authorization.k8s.io/metrics-server-auth-reader created
clusterrolebinding.rbac.authorization.k8s.io/metrics-server:system:auth-delegator created
clusterrolebinding.rbac.authorization.k8s.io/system:metrics-server created
service/metrics-server created
deployment.apps/metrics-server created
apiservice.apiregistration.k8s.io/v1beta1.metrics.k8s.io created
```

## 수평형 Pod 자동 확장(Horizontal Pod Autoscaler)

![HPA-workflow](https://github.com/donghoon-khan/drawIO/blob/master/app/k8s-autoscaling-workflow/HPA-workflow.png?raw=true)

## 수직형 Pod 자동 확장
쿠버네티스 [VPA(Vertical Pod Autoscaler)](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)는 pod에 대한 CPU or Memory를 자동으로 조정하여 애플리케이션의 리소스를 `적절히 조정`할 수 있게 지원한다.

![VPA-workflow](https://github.com/donghoon-khan/drawIO/blob/master/app/k8s-autoscaling-workflow/VPA-workflow.png?raw=true)


## 클러스터 자동 확장(Cluster Autoscaler)