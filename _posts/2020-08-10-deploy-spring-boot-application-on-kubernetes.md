---
title: Spring Boot 애플리케이션 쿠버네티스로 배포하기
category: 
- DevOps
- web
tags:
- spring
- kubernetes
- docker
summary: Deploy Spring-Boot application to Kubernetes cluster
thumbnail: /assets/img/thumbnail/spring+docker+kubernetes.png
---
스프링 프레임워크는 자바 플랫폼을 위한 오픈소스 애플리케이션 프레임워크로, 따로 설명하지 않아도 웹개발자라면 한번씩은 경험해본 프레임워크일 것이다. Spring은 MSA를 위해 [Spring-Cloud](https://spring.io/projects/spring-cloud)라는 라이브러리를 제공하는데 라우팅, 로드밸런싱, 서킷브레이커 등 많은 기능을 제공한다. Spring cloud만 사용해도 쿠버네티스 못지 않은 강력한 MSA환경을 구축할 수 있을 것 이다. 아쉬운 점은 Auto-scaling이나 Self-Healing과 같은 기능을 프레임워크 단에서 제공하지 않는다는 것이다. 
이 글에서는 Spring-boot 애플리케이션을 쿠버네티스 클러스터에 배포하는 방법에 대해서 설명한다.

## Create Spring-boot Application

Spring-boot 애플리케이션 프로젝트를 생성하고, 필요한 디펜던시를 추가한 후 테스트 용으로 Rest controller를 하나 만들자. 만약 기존의 Spring 프로젝트가 존재한다면 그대로 사용해도 된다.

### Create Spring-boot project

[Spring initializr](https://start.spring.io/)에서 dependencies로 `webflux`, `actuator`를 추가하고 스프링 프로젝트를 생성하자.
![spring initializr](/assets/img/posts/2020-08-10-spring-initializr.png)
또는 curl을 이용해 프로젝트를 생성하자.

```bash
$ mkdir kubernetes-spring && cd kubernetes-spring
$ curl https://start.spring.io/starter.tgz -d dependencies=webflux,actuator | tar -xzvf -
```

### Create Rest controller

`RestController`를 하나 추가하자.

```java
package com.example.demo;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class DemoController {
    
    @GetMapping("/")
    public String DemoRestApi() {
        return "Hi, I'm demo application";
    }
}
```

### Build and Run

프로젝트의 루트 경로에서 애플리케이션을 빌드하고 실행해 보자.

```bash
$ ./mvnw install
$ java -jar target/*.jar
...
2020-07-13 10:05:43.964  INFO 8422 --- [           main] o.s.b.a.e.web.EndpointLinksResolver      : Exposing 2 endpoint(s) beneath base path '/actuator'
2020-07-13 10:05:44.569  INFO 8422 --- [           main] o.s.b.web.embedded.netty.NettyWebServer  : Netty started on port(s): 8080
2020-07-13 10:05:44.591  INFO 8422 --- [           main] com.example.demo.DemoApplication         : Started DemoApplication in 3.496 seconds (JVM running for 4.098)
...
$ curl localhost:8080/
Hi, I'm demo application
```

### NOTE - Actuator

Dependency로 추가한 [actuator](https://docs.spring.io/spring-boot/docs/current/reference/html/production-ready-features.html) 모듈을 통해 애플리케이션의 상태를 모니터링할 수 있다. actuator는 다양한 endpoints를 제공하는데 그 중 `health`를 조회해 보자.

```bash
$ curl localhost:8080/actuator
{"_links":{"self":{"href":"http://localhost:8080/actuator","templated":false},"health":{"href":"http://localhost:8080/actuator/health","templated":false},"health-path":{"href":"http://localhost:8080/actuator/health/{*path}","templated":true},"info":{"href":"http://localhost:8080/actuator/info","templated":false}}}
$ curl localhost:8080/actuator/health
{"status":"UP"}
```

actuator를 사용하면 애플리케이션이 의존하는 시스템의 Health check를 할 수 있다. 예를들면 DB, Rabbitmq와 같은 애플리케이션이 의존하는 인프라의 상태를 파악할 수 있다. 아래는 MariaDB와 Rabbit MQ를 사용한 애플리케이션의 actuator 모듈의 Health endpoint 예시이다.

```json
{
  "status": "UP",
  "components": {
    "db": {
      "status": "UP",
      "details": {
        "database": "MariaDB",
        "validationQuery": "isValid()"
      }
    },
    "diskSpace": {
      "status": "UP",
      "details": {
        "total": 21462233088,
        "free": 12606222336,
        "threshold": 10485760,
        "exists": true
      }
    },
    "livenessState": {
      "status": "UP"
    },
    "ping": {
      "status": "UP"
    },
    "rabbit": {
      "status": "UP",
      "details": {
        "version": "3.8.2"
      }
    },
    "readinessState": {
      "status": "UP"
    }
  },
  "groups": [
    "liveness",
    "readiness"
  ]
}
```

뿐만 아니라, 사용자가 정의한 Health Indicator상태를 파악할 수 있다. actuator를 이용해서 쿠버네티스 배포 시 컨테이너가 동작 중인지, 트래픽을 받을 준비가 되어 있는지를 kubelet에게 알려 줄 것이다.

## Containerize

이제 쿠버네티스로 배포 할 애플리케이션은 준비가 끝났다. 애플리케이션을 [docker](https://www.docker.com/)를 이용해 컨테이너화 한 후 [docker hub](https://hub.docker.com/)로 Push하자.

### Create Dockerfile

이제 애플리케이션을 도커를 이용해 컨테이너화 하자. 프로젝트의 루트 경로에 `Dockerfile`을 추가한다.

```bash
FROM openjdk:8-jdk-alpine
ARG JAR_FILE=target/*.jar
COPY ${JAR_FILE} app.jar
ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
```

### Docker build

spring-boot 애플리케이션을 Dockerfile을 이용해 docker 이미지로 만들어 보자.

```bash
$ docker build --help                      

Usage:  docker build [OPTIONS] PATH | URL | -

Build an image from a Dockerfile

Options:
      --add-host list           Add a custom host-to-IP mapping (host:ip)
      --build-arg list          Set build-time variables
      --cache-from strings      Images to consider as cache sources
      --cgroup-parent string    Optional parent cgroup for the container
      --compress                Compress the build context using gzip
      --cpu-period int          Limit the CPU CFS (Completely Fair Scheduler) period
      --cpu-quota int           Limit the CPU CFS (Completely Fair Scheduler) quota
  -c, --cpu-shares int          CPU shares (relative weight)
      --cpuset-cpus string      CPUs in which to allow execution (0-3, 0,1)
      --cpuset-mems string      MEMs in which to allow execution (0-3, 0,1)
      --disable-content-trust   Skip image verification (default true)
  -f, --file string             Name of the Dockerfile (Default is 'PATH/Dockerfile')
      --force-rm                Always remove intermediate containers
      --iidfile string          Write the image ID to the file
      --isolation string        Container isolation technology
      --label list              Set metadata for an image
  -m, --memory bytes            Memory limit
      --memory-swap bytes       Swap limit equal to memory plus swap: '-1' to enable unlimited swap
      --network string          Set the networking mode for the RUN instructions during build (default "default")
      --no-cache                Do not use cache when building the image
      --platform string         Set platform if server is multi-platform capable
      --pull                    Always attempt to pull a newer version of the image
  -q, --quiet                   Suppress the build output and print image ID on success
      --rm                      Remove intermediate containers after a successful build (default true)
      --security-opt strings    Security options
      --shm-size bytes          Size of /dev/shm
      --squash                  Squash newly built layers into a single new layer
      --stream                  Stream attaches to server to negotiate build context
  -t, --tag list                Name and optionally a tag in the 'name:tag' format
      --target string           Set the target build stage to build.
      --ulimit ulimit           Ulimit options (default [])
```

docker build 시 -t 옵션으로 tag를 설정할 건데, `[DOCKER_REGISTRY_IP]:[DOCKER_REGISTRY_PORT]/[REPOSITORY]/[IMAGE_NAME]:[TAG]`형식이다. 이 때 DOCKER_REGISTRY는 쿠버네티스에서 접근이 가능해야 한다. 이 글에서는 [Docker hub](https://hub.docker.com/repository/docker/dhkang222/kubernetes-spring)로 image를 push할 것이기 때문에 `[USERNAME]/[REPOSITORY]:tagname`으로 빌드 한다.

```bash
$ docker build -t dhkang222/kubernetes-spring:demo ./
```

### Docker run

docker build명령을 이용해 docker image가 성공적으로 만들어 졌으면, 이 이미지를 사용해 container를 하나 만들어서 테스트해보자.

```bash
$ docker images
REPOSITORY                    TAG                 IMAGE ID            CREATED             SIZE
dhkang222/kubernetes-spring   demo                ca4bfbb41b1f        15 minutes ago      125MB
...
$ docker run -p 9090:8080 dhkang222/kubernetes-spring:demo
...
2020-07-13 01:49:49.112  INFO 1 --- [           main] o.s.b.a.e.web.EndpointLinksResolver      : Exposing 2 endpoint(s) beneath base path '/actuator'
2020-07-13 01:49:49.878  INFO 1 --- [           main] o.s.b.web.embedded.netty.NettyWebServer  : Netty started on port(s): 8080
2020-07-13 01:49:49.909  INFO 1 --- [           main] com.example.demo.DemoApplication         : Started DemoApplication in 3.869 seconds (JVM running for 4.525)
...
$ curl localhost:9090
Hi, I'm demo application
```

### Docker push

만들어진 docker image를 docker registry로 push하면 kubernetes에서 사용할 컨테이너에 대한 준비는 끝난다.

```bash
$ docker login --username=${DOCKER_REGISTRY_USER_NAME}
$ docker push dhkang222/kubernetes-spring:demo
```

## Deploy Spring-Boot application to Kubernetes cluster

컨테이너화한 애플리케이션을 쿠버네티스에 배포하자. Deployment와 Service 설정을 정의한 demo.yaml을 만들자. 
서비스를 외부로 노출하는 방법은 ingress, LoadBalancer type의 Service, ClusterIP type Service + Proxy 등 다양하다. 이 글에서는 NodePort를 이용해서 Service를 외부로 노출한다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: demo
  name: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demo
  strategy: {}
  template:
    metadata:
      labels:
        app: demo
    spec:
      containers:
      - image: dhkang222/kubernetes-spring:demo
        name: kubernetes-spring
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: demo
  name: demo
spec:
  ports:
  - name: demo-svc-8080
    port: 8080
    protocol: TCP
    nodePort: 31111
    targetPort: 8080
  selector:
    app: demo
  type: NodePort
```

```bash
$ kubectl apply -f demo.yaml
deployment.apps/demo created
service/demo created

$ kubectl get po
NAME                   READY   STATUS    RESTARTS   AGE
demo-6db4c4577-hvb2t   1/1     Running   0          2s

$ k describe po demo-6db4c4577-hvb2t
...
Containers:
  kubernetes-spring:
    Container ID:   containerd://207e4541d80ed6c11de0fff0eb247eead111b45f57112a41adf98a6297e96235
    Image:          dhkang222/kubernetes-spring:demo
    Image ID:       docker.io/dhkang222/kubernetes-spring@sha256:7e70ebd2fd7d38e48a90d6d64e48ace45f73fbe5de752a92d40b2170402b4cde
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Tue, 14 Jul 2020 12:45:21 +0900
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-l79fd (ro)
...
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  7s    default-scheduler  Successfully assigned default/demo-64774b95f5-p5l2l to node1
  Normal  Pulling    6s    kubelet, sonm-pc   Pulling image "dhkang222/kubernetes-spring:demo"
  Normal  Pulled     4s    kubelet, sonm-pc   Successfully pulled image "dhkang222/kubernetes-spring:demo"
  Normal  Created    3s    kubelet, sonm-pc   Created container kubernetes-spring
  Normal  Started    3s    kubelet, sonm-pc   Started container kubernetes-spring
```

하나의 파드 내에는 여러 컨테이너가 존재할 수 있는데, 컨테이너 이미지들의 정보를 Containers 탭에서 확인할 수 있다. Events 탭을 보면 pod가 스케줄러에 의해 적절한 노드에 할당 된 것을 알 수 있고 이후 이미지 pull, 컨테이너 생성이 차례대로 실행된 것을 알 수 있다. 이 외에도 Volume과 같은 정보를 describe 명령으로 볼 수 있는데, 쿠버네티스 오브젝트 배포에 실패한다면 describe 명령어가 큰 도움이 될 것이다. 이제 kubernetes에 배포된 애플리케이션을 테스트 해보자.

```bash
$ curl ${NODE_IP}:31111
Hi, I'm demo application
```

## livenessProbe, readinessProbe, startupProbe

애플리케이션에 추가한 actuator 모듈을 livenessProbe, readinessProbe, startupProbe에서 사용해 보자. 프로브는 kubelet에 의해 주기적으로 실행되는 진단점 이라고 보면 된다.
kubelet은 컨테이너에 의해 구현된 핸들러를 호출하는데 HTTP GET, TCP Socket, Exec Action이 있다.  즉 이 컨테이너가 동작 중인지, 트래픽을 받을 준비가 되어 있는지, 컨테이너가 시작 됬는지를 HTTP GET 방식, TCP Socket 방식, script와 같은 Exec action을 통해서 검사한다. 이 글에서는 웹 애플리케이션을 배포했으므로 http endpoint로 actuator 모듈의 health를 이용한다. [Kubernetes Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)에서 프로브가 어떤 역할을 하는지, 언제 어떤 프로브를 사용해야 하는지 친절하게 설명하고 있다.

- livenessProbe: 컨테이너가 동작 중인지에 대한 여부를 나타낸다. livenessProbe가 실패라면 kubelet은 컨테이너를 재시작한다. livenessProbe를 설정하지 않는다면 기본 상태는 Success이다.
- readinessProbe: 컨테이너가 트래픽을 받을 준비가 되어 있는지 여부를 나타낸다. 만약 readinessProbe가 실패라면 컨트롤러는 파드에 연관된 모든 서비스들의 엔드 포인트에서 파드의 IP 주소를 제거한다. 즉 deployment가 4개의 pod를 관리하고 있다고 가정할 때 하나의 파드가 readinessProbe 실패라면 다른 3개의 파드로만 트래픽을 보내게 된다. readinessProbe를 하는설정하지 않는다면 기본 상태는 Success이다.
- startupProbe: 컨테이너 내의 애플리케이션이 시작되었는지를 나타낸다. 이 프로브를 설정할 경우 startupProbe가 success가 될 때 까지 다른 프로브는 검사하지 않는다. 만약 startupProbe가 실패하면 kubelet은 컨테이너를 재시작한다. startupProbe 역시 설정하지 않는다면 기본상태는 Success이다.

### Deployment - containers probe 추가

demo.yaml을 수정해서 livenessProbe, readinessProbe, startupProbe를 설정하자.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: demo
  name: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demo
  strategy: {}
  template:
    metadata:
      labels:
        app: demo
    spec:
      containers:
      - image: dhkang222/kubernetes-spring:demo
        name: kubernetes-spring
        ports:
          - name: demo-svc-8080
            containerPort: 8080
            protocol: TCP
        livenessProbe:
          httpGet:
            path: /actuator/health
            port: demo-svc-8080
        readinessProbe:
          httpGet:
            path: /actuator/health
            port: demo-svc-8080
        startupProbe:
          httpGet:
            path: /actuator/health
            port: demo-svc-8080
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: demo
  name: demo
spec:
  ports:
  - name: demo-svc-8080
    port: 8080
    protocol: TCP
    nodePort: 31111
    targetPort: 8080
  selector:
    app: demo
  type: NodePort
```

```bash
$ kubectl apply -f demo.yaml
deployment.apps/demo configured
service/demo unchanged

$ kubectl get po
NAME                   READY   STATUS    RESTARTS   AGE
demo-dc9cd888f-wbqx4   1/1     Running   0          21s

$ kubectl describe po demo-dc9cd888f-wbqx4
...
Containers:
  kubernetes-spring:
    Container ID:   containerd://857cd87d4b2654e86ca229b0f02c0553e4d901e6d14933816a733e00dd700b40
    Image:          dhkang222/kubernetes-spring:demo
    Image ID:       docker.io/dhkang222/kubernetes-spring@sha256:7e70ebd2fd7d38e48a90d6d64e48ace45f73fbe5de752a92d40b2170402b4cde
    Port:           8080/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Tue, 14 Jul 2020 13:49:46 +0900
    Ready:          True
    Restart Count:  0
    Liveness:       http-get http://:demo-svc-8080/actuator/health delay=0s timeout=1s period=10s #success=1 #failure=3
    Readiness:      http-get http://:demo-svc-8080/actuator/health delay=0s timeout=1s period=10s #success=1 #failure=3
    Startup:        http-get http://:demo-svc-8080/actuator/health delay=0s timeout=1s period=10s #success=1 #failure=3
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-l79fd (ro)
...
```

Container 생성에 관련된 정보를 보면 Liveness, Readiness, Startup이 추가 된 것을 확인할 수 있다. 이제 kubelet은 세개의 probe를 검사하고 컨테이너를 재시작 하거나, 컨트롤러는 트래픽을 받을 준비가 되어 있지 않은 파드에게 트래픽을 보내지 않는다.

## Conclustion

몇 가지 설정과 코드의 추가 만으로 Spring-boot application을 컨테이너로 만들고 kubernetes가 관리할 수 있게끔 할 수 있다. 이 과정에서 docker build, docker push, kubernetes object 생성 등 귀찮은 작업이 생기지만, Jenkins or Bamboo와 같은 Continuous Integration 솔루션과 [Helm](https://helm.sh/)을 이용한다면 코드에만 집중할 수 있을 것이다.  
애플리케이션 소스와 demo.yaml은 [github](https://github.com/donghoon-khan/kubernetes-demo/tree/master/app/spring-demo)을 참고하면 된다.
