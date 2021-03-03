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

HPA는 metric(CPU, Memory, custom metric)을 관찰하여 RC, Deployment, RS, StatefulSet의 파드 개수를 `Scale Up` 또는 `Scale Down`한다. 요구하는 metric에 비해 실행되고 있는 파드의 수가 작다면 파드의 수를 늘리고(Scale Up), 반대의 경우 파드의 수를 줄인다(Scale Down). DaemonSet과 같이 크기를 조정할 수 없는 경우에는 적용되지 않는다.  
다음과 같이 작성된 CPU에 부하를 주는 애플리케이션으로 HPA를 테스트 해보자.

**index.php**를 다음과 같이 작성하자.

```php
<?php
  $x = 0.0001;
  for ($i = 0; $i <= 1000000; $i++) {
    $x += sqrt($x);
  }
  echo "OK!";
?>
```

**Dockerfile**을 다음과 같이 작성하자.

```dockerfile
FROM php:5-apache
COPY index.php /var/www/html/index.php
RUN chmod a+rx index.php
```

k8s.gcr.io에 빌드 된 이미지가 있기에 따로 빌드하거나 푸쉬할 필요는 없다. 위 애플리케이션을 쿠버네티스 클러스터에 배포한다.

```bash
$ cat << EOF > application.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  selector:
    matchLabels:
      run: php-apache
  replicas: 2
  template:
    metadata:
      labels:
        run: php-apache
    spec:
      containers:
      - name: php-apache
        image: k8s.gcr.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 200m

---
apiVersion: v1
kind: Service
metadata:
  name: php-apache
  labels:
    run: php-apache
spec:
  ports:
  - port: 80
  selector:
    run: php-apache
EOF

$ kubectl apply -f application.yaml
deployment.apps/php-apache created
service/php-apache created
```

CPU 사용량을 기준으로 Scale Up/Down을 하게끔 HPA를 생성해보자.

```bash
$ cat << EOF > hpa.yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80
EOF

$ kubectl describe deploy php-apache
Name:                   php-apache
Namespace:              default
CreationTimestamp:      Wed, 18 Nov 2020 13:43:24 +0900
Labels:                 <none>
Annotations:            deployment.kubernetes.io/revision: 1
                        kubectl.kubernetes.io/last-applied-configuration:
...
Replicas:               2 desired | 2 updated | 2 total | 2 available | 0 unavailable
...

$ kubectl apply -f hpa.yaml
horizontalpodautoscaler.autoscaling/php-apache created

$ kubectl top pod
NAME                          CPU(cores)   MEMORY(bytes)   
php-apache-79b85969bb-5vs98   1m           9Mi             
php-apache-79b85969bb-vvmh6   1m           9Mi

$ kubectl get hpa
NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   0%/80%    1         10        2          20s
```

애플리케이션에 부하를 증가시키는 컨테이너를 배포해서 HPA를 테스트 해보자.

```bash
$ kubectl run -i --tty load-generator --rm --image=busybox --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"

$ kubectl get hpa --watch
NAME         REFERENCE               TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   215%/80%   1         10        2          89s
php-apache   Deployment/php-apache   215%/80%   1         10        4          92s
php-apache   Deployment/php-apache   215%/80%   1         10        6          107s
php-apache   Deployment/php-apache   69%/80%    1         10        6          2m18s

$ kubectl top pod
NAME                          CPU(cores)   MEMORY(bytes)   
load-generator                7m           1Mi             
php-apache-79b85969bb-5vs98   239m         12Mi            
php-apache-79b85969bb-bhpr2   109m         12Mi            
php-apache-79b85969bb-cg9hh   51m          14Mi            
php-apache-79b85969bb-mhrjx   132m         12Mi            
php-apache-79b85969bb-v8w9n   206m         14Mi            
php-apache-79b85969bb-vvmh6   126m         12Mi

$ kubectl get deploy
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
php-apache   6/6     6            6           4m37s
```

HPA 설정에서 scaleTargetRef로 지정한 php-apache deployment의 targetCPUUtilizationPercentage를 80%이하로 유지하기 위해 Scale Up하는 것을 확인할 수 있다. 이제 load-generator를 삭제하고 Scale Down이 되는지 확인해 보자.

```bash
$ kubectl delete pod load-generator
pod "load-generator" deleted

$ kubectl get hpa --watch
NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   64%/80%   1         10        6          3m40s
php-apache   Deployment/php-apache   0%/80%    1         10        6          4m20s
php-apache   Deployment/php-apache   0%/80%    1         10        6          7m8s
php-apache   Deployment/php-apache   0%/80%    1         10        5          7m23s
php-apache   Deployment/php-apache   0%/80%    1         10        5          9m8s
php-apache   Deployment/php-apache   0%/80%    1         10        1          9m23s

$ kubectl top pod
NAME                          CPU(cores)   MEMORY(bytes)   
php-apache-79b85969bb-vvmh6   1m           12Mi

$ kubectl describe deploy php-apache
php-apache
Namespace:              default
CreationTimestamp:      Wed, 18 Nov 2020 13:28:36 +0900
Labels:                 <none>
Annotations:            deployment.kubernetes.io/revision: 1
                        kubectl.kubernetes.io/last-applied-configuration:
...
Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
...
```

부하가 줄어들면서 HPA가 Scale Down하는 모습을 확인할 수 있다.

 **NOTE:**  
 레플리카의 수는 다음 식에 의해 결정된다.  
 desiredReplicas = ceil[currentReplicas * ( currentMetricValue / desiredMetricValue )]
{: .notice--info}

## 수직형 Pod 자동 확장(Vertical Pod Autoscaler)

![VPA-workflow](https://github.com/donghoon-khan/drawIO/blob/master/app/k8s-autoscaling-workflow/VPA-workflow.png?raw=true)

HPA가 파드의 수를 늘리거나 줄여서 스케일링 했다면, VPA는 파드의 요청 리소스를 늘리거나 줄여서 스케일링 하는 방식이다. 파드에 할당된 리소스는 실행 중에 다시 할당할 수 없기에 VPA는 스케일링 시 파드를 재시작하게 된다. VPA는 현재 베타단계이므로 VPA를 클러스터에 인스톨 한 후 진행해야 하며
CPU or memory metrics에 대해서는 HPA와 VPA를 함께 사용할 수 없으니, HPA를 삭제한 후 VPA를 생성해야 한다.

```bash
$ kubectl delete hpa php-apache
horizontalpodautoscaler.autoscaling "php-apache" deleted
```

```bash
$ git clone https://github.com/kubernetes/autoscaler.git
$ ./autoscaler/vertical-pod-autoscaler/hack/vpa-up.sh 
$ kubectl get pod -n kube-system | grep vpa
vpa-admission-controller-85466984bf-vg2z4   1/1     Running   0          75s
vpa-recommender-864db687f6-648xg            1/1     Running   0          76s
vpa-updater-79d9958696-m75xq                1/1     Running   0          78s
```

VPA 관련 파드가 정상적으로 배포된 것을 확인할 수 있다. HPA 예제에서 사용한 애플리케이션에 VPA를 설정해서 테스트 해보자.

```bash

$ kubectl get pod
NAME                          READY   STATUS    RESTARTS   AGE
php-apache-79b85969bb-676jp   1/1     Running   0          31s
php-apache-79b85969bb-gs5md   1/1     Running   0          31s

$ kubectl describe pod php-apache-79b85969bb-gs5md
...
Containers:
  php-apache:
    Container ID:   docker://b733c97425da680120a719e629c7364078e9f86228f3de28b99496b79bf6980f
    Image:          k8s.gcr.io/hpa-example
    Image ID:       docker-pullable://k8s.gcr.io/hpa-example@sha256:581697a37f0e136db86d6b30392f0db40ce99c8248a7044c770012f4e8491544
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Wed, 18 Nov 2020 14:18:37 +0900
    Ready:          True
    Restart Count:  0
    Requests:
      cpu:        200m
    Environment:  <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-vv7bl (ro)
...
```

VPA를 설정하고 애플리케이션에 부하를 증가시켜보자.

```bash
$ cat << EOF > vpa.yaml
apiVersion: "autoscaling.k8s.io/v1"
kind: VerticalPodAutoscaler
metadata:
  name: php-apache
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: php-apache
  resourcePolicy:
    containerPolicies:
      - containerName: '*'
        minAllowed:
          cpu: 200m
          memory: 50Mi
        maxAllowed:
          cpu: 1
          memory: 250Mi
        controlledResources: ["cpu", "memory"]
EOF

$ kubectl apply -f vpa.yaml
verticalpodautoscaler.autoscaling.k8s.io/php-apache created

$ kubectl get vpa
NAME         AGE
php-apache   16s

$ kubectl run -i --tty load-generator --rm --image=busybox --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"

$ kubectl get --watch pod
NAME                          READY   STATUS    RESTARTS   AGE
load-generator                1/1     Running   0          5s
php-apache-79b85969bb-676jp   1/1     Running   0          2m57s
php-apache-79b85969bb-gs5md   1/1     Running   0          2m57s
php-apache-79b85969bb-gs5md   1/1     Terminating   0          3m49s
php-apache-79b85969bb-n5wcm   0/1     Pending       0          0s
php-apache-79b85969bb-n5wcm   0/1     Pending       0          0s
php-apache-79b85969bb-n5wcm   0/1     ContainerCreating   0          0s
php-apache-79b85969bb-gs5md   0/1     Terminating         0          3m51s
php-apache-79b85969bb-gs5md   0/1     Terminating         0          3m52s
php-apache-79b85969bb-gs5md   0/1     Terminating         0          3m52s
php-apache-79b85969bb-n5wcm   1/1     Running             0          3s
php-apache-79b85969bb-676jp   1/1     Terminating         0          4m49s
php-apache-79b85969bb-p74nt   0/1     Pending             0          0s
php-apache-79b85969bb-p74nt   0/1     Pending             0          0s
php-apache-79b85969bb-p74nt   0/1     ContainerCreating   0          0s
php-apache-79b85969bb-676jp   0/1     Terminating         0          4m51s
php-apache-79b85969bb-p74nt   1/1     Running             0          4s
php-apache-79b85969bb-676jp   0/1     Terminating         0          4m57s
php-apache-79b85969bb-676jp   0/1     Terminating         0          4m57s

$ kubectl describe pod php-apache-79b85969bb-n5wcm
...
Containers:
  php-apache:
    Container ID:   docker://4bab2fc008b7ed0f62c50662580615627b496f0edb6da6d6dacc073058992974
    Image:          k8s.gcr.io/hpa-example
    Image ID:       docker-pullable://k8s.gcr.io/hpa-example@sha256:581697a37f0e136db86d6b30392f0db40ce99c8248a7044c770012f4e8491544
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Wed, 18 Nov 2020 14:22:26 +0900
    Ready:          True
    Restart Count:  0
    Requests:
      cpu:        296m
      memory:     262144k
    Environment:  <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-vv7bl (ro)
...
```

배포한 애플리케이션이 재시작하면서 리소스 요청값이 증가한 것을 확인할 수 있다. 파드의 리소스가 부족하므로 VPA가 원래 요청한 리소스 보다 더 적절한 값으로 수정한 것을 확인할 수 있다.

## 클러스터 자동 확장(Cluster Autoscaler)

![CA-workflow](https://github.com/donghoon-khan/drawIO/blob/master/app/k8s-autoscaling-workflow/CA-workflow.png?raw=true)

CA는 리소스가 부족해서 Pending 상태에 있는 파드를 감지해서 클러스터에 노드를 추가하고 충분할 경우 노드를 삭제하는 방식이다. 이 때 클라우드 프로바이더에게 노드를 추가하고 삭제하는 요청을 하게 되는데, 사용 중인 클라우드 별로 설정이 다르므로 [Cluster Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler)을 참고해서 설정하도록 하자.

## 참고자료

[https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)  
[https://github.com/kubernetes/autoscaler](https://github.com/kubernetes/autoscaler)  
[https://levelup.gitconnected.com/kubernetes-autoscaling-101-cluster-autoscaler-horizontal-pod-autoscaler-and-vertical-pod-2a441d9ad231](https://levelup.gitconnected.com/kubernetes-autoscaling-101-cluster-autoscaler-horizontal-pod-autoscaler-and-vertical-pod-2a441d9ad231)