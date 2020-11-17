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
다음과 같이 작성된 CPU에 부하를 주는 애플리케이션으로 HPA를 테스트 해보자. k8s.gcr.io에 빌드 된 이미지가 있기에 따로 빌드하거나 푸쉬할 필요는 없다.

*index.php*
```php
<?php
  $x = 0.0001;
  for ($i = 0; $i <= 1000000; $i++) {
    $x += sqrt($x);
  }
  echo "OK!";
?>
```

*Dockerfile*
```dockerfile
FROM php:5-apache
COPY index.php /var/www/html/index.php
RUN chmod a+rx index.php
```

위 애플리케이션을 쿠버네티스 클러스터에 배포한다.
```bash
$ cat << EOF > hpa-application.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  selector:
    matchLabels:
      run: php-apache
  replicas: 1
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
          limits:
            cpu: 500m
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

$ kubectl apply -f hpa-application.yaml
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

$ kubectl apply -f hpa.yaml
horizontalpodautoscaler.autoscaling/php-apache created

$ kubectl top pod
NAME                          CPU(cores)   MEMORY(bytes)   
php-apache-79544c9bd9-gqpbj   1m           9Mi             

$ kubectl get hpa
NAME         REFERENCE               TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   0%/80%   1         10        0          60s
```

애플리케이션에 부하를 증가시키는 컨테이너를 배포해서 HPA를 테스트 해보자.
```bash
$ kubectl run -i --tty load-generator --rm --image=busybox --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"

$ kubectl get hpa
NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   72%/80%   1         10        4          5m

$ kubectl top pod
NAME                          CPU(cores)   MEMORY(bytes)   
load-generator                5m           0Mi             
php-apache-79544c9bd9-9dshs   174m         14Mi            
php-apache-79544c9bd9-gqpbj   206m         12Mi            
php-apache-79544c9bd9-w4h7c   71m          14Mi            
php-apache-79544c9bd9-wjhd2   90m          12Mi

$ kubectl get deployment
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
php-apache   4/4     4            4           16m
```

HPA 설정에서 scaleTargetRef로 지정한 php-apache deployment의 targetCPUUtilizationPercentage를 80%이하로 유지하기 위해 Scale Up하는 것을 확인할 수 있다. 이제 load-generator를 삭제해보자.
```bash
$ kubectl delete pod load-generator
pod "load-generator" deleted

$ kubectl get hpa
NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   0%/80%    1         10        1          56m

$ kubectl top pod
NAME                          CPU(cores)   MEMORY(bytes)   
php-apache-79544c9bd9-gqpbj   1m           12Mi

$ kubectl get deployment
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
php-apache   1/1     1            1           63m
```

부하가 줄어들면서 HPA가 파드의 레플리카를 줄이는 모습을 확인할 수 있다.

 **NOTE:**  
 레플리카의 수는 다음 식에 의해 결정된다.  
 desiredReplicas = ceil[currentReplicas * ( currentMetricValue / desiredMetricValue )]
{: .notice--info}

## 수직형 Pod 자동 확장(Vertical Pod Autoscaler)
쿠버네티스 [VPA](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)는 pod에 대한 CPU or Memory를 자동으로 조정하여 애플리케이션의 리소스를 `적절히 조정`할 수 있게 지원한다.

![VPA-workflow](https://github.com/donghoon-khan/drawIO/blob/master/app/k8s-autoscaling-workflow/VPA-workflow.png?raw=true)

## 클러스터 자동 확장(Cluster Autoscaler)