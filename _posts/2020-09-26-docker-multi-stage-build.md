---
title: Docker multi-stage 빌드
category: 
- DevOps
tags:
- docker
summary: Docker multi stage build
thumbnail: "/assets/img/thumbnail/docker.png"
---
Docker 컨테이너의 이미지를 만들 때 사이즈를 경량화 하는 것은 매우 중요하다. 
Docker 이미지의 크기가 GB 단위 이거나, 부팅하는데 많은 시간이 걸린다면 Dockerized를 해서 얻는 이점이 많이 줄어들기 때문이다.
때문에 많은 사람들이 이미지의 사이즈를 줄이기 위해 Ubuntu가 아닌 alpine과 같은 작은 사이즈의 이미지를 base image로 사용하고 있다.  
Docker 17.05 이상의 버전부터는 multi-stage 빌드를 이용하여 컨테이너 이미지의 사이즈를 더욱 경량화 시킬 수 있다.
Multi-stage build를 사용하면 애플리케이션을 Dockerize할 때에는 필요하지만 애플리케이션의 실행에는 필요없는 부분을 최종 이미지에서 제외할 수 있기에 이미지의 크기를 줄일 수 있다.  

```go
package main

import(
    "fmt"
    "time"
)

func main() {
    for {
        fmt.Println("Hello, world!")
        time.Sleep(5 * time.Second)
    }
}
```

```dockerfile
FROM golang:1.15

WORKDIR /usr/src/donghoon-khan
COPY app.go .
RUN go build -o app app.go
CMD ["./app"]
```

위의 코드를 보면 5초마다 "Hello, World!"를 출력하는 간단한 Go언어 애플리케이션을 도커 이미지로 만들고 있다. 하지만 위와 같이 이미지를 만들 경우 이미지의 크기가 너무 커지게 된다.
애플리케이션의 빌드를 위해 사용한 base image인 golang의 크기가 크기 때문이다.

```bash
$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
example             latest              a8f49e6fae7f        30 minutes ago      841MB
golang              1.15                05c8f6d2538a        2 weeks ago         839MB

# ls -lh
total 2.0M
-rwxr-xr-x 1 root root 2.0M Oct  1 14:14 app
-rw-r--r-- 1 root root  151 Oct  1 13:17 app.go
```

2MB의 바이너리파일을 실행하기 위해 841MB의 이미지를 사용하는 것은 매우 비효율적이라는 생긱이 들 것이다.
이제 빌드와 같이 Dockerize할 때에는 필요하지만 애플리케이션의 실행에는 필요없는 부분을 최종 이미지에서 제외하여 경량화된 이미지를 만드는 방법에 대해 알아보자.

## Unused multi-stage builds
Multi-stage를 사용하지 않고도 최종 이미지를 경량화 할 수 있다. 
빌드 이미지에서는 빌드만을 수행하고 빌드의 결과물을 로컬파일시스템으로 가져온다.
그리고 최종 이미지에에는 애플리케이션 실행에 필요한 것들만 카피하여 실행하면 된다.

```dockerfile
# On this Dockerfile.build
FROM golang:1.15

WORKDIR /usr/src/donghoon-khan
COPY app.go .
RUN go build -o app app.go
```

```dockerfile
# On this Dockerfile
FROM alpine:latest
WORKDIR /donghoon-khan/
COPY app .
CMD ["./app"]
```

```bash
#!/bin/sh
# On this build.sh
echo Building donghoon-khan/multi-stage:build

docker build -t donghoon-khan/multi-stage:build . -f Dockerfile.build

docker container create --name extract donghoon-khan/multi-stage:build
docker container cp extract:/usr/src/donghoon-khan/app ./app
docker container rm -f extract

echo echo Building donghoon-khan/multi-stage:release

docker build --no-cache -t donghoon-khan/multi-stage:release .
rm ./app
```

```bash
$ docker images
REPOSITORY                  TAG                 IMAGE ID            CREATED             SIZE
donghoon-khan/multi-stage   release             24dfa47c06aa        12 seconds ago      7.61MB
```

최종 release 이미지의 크기를 보면 7.61MB로 줄어든 것을 볼 수 있다.
최종 이미지의 크기는 확연히 줄어들었지만, 도커파일을 여러개 관리해야 한다는 단점이 있다.
빌드 스테이지를 여러개 가져가야 할 경우 도커 파일은 계속 늘어날 수 밖에 없을 것이다.
뿐만 아니라, 빌드 스크립트 역시 관리가 필요하다.

## Use multi-stage builds
Multi-stage 빌드는 여러개의 FROM문을 사용해 stage를 구분하고 한 stage에서 다른 stage로 아티팩트를 선택적으로 복사하여 최종 이미지에서 원하지 않는 것들을 제외하는 방식이다.
앞에서 사용한 애플리케이션을 multi-stage 빌드를 사용하여 빌드해보자.

```dockerfile
# On this Dockerfile
FROM golang:1.7.3 AS build
WORKDIR /usr/src/donghoon-khan
COPY app.go .
RUN go build -o app app.go

FROM alpine:latest
WORKDIR /donghoon-khan/
COPY --from=build /usr/src/donghoon-khan/app .
CMD ["./app"]
```

```bash
$ docker images
REPOSITORY                  TAG                 IMAGE ID            CREATED             SIZE
donghoon-khan/multi-stage   release             0482a0eb2ecb        11 minutes ago      7.21MB
```
Dockerfile을 보면 여러 FROM이 등장하는 것을 볼 수 있다.
처음 FROM에서 `AS` 키워드를 이용해 `build`라는 별칭을 부여하고, `--from=build`을 통해 빌드 스테이지의 아티팩트를 최종 릴리즈 이미지에 복사하였다.
따로 별칭을 추가하지 않아도 `--from=0`와 같은 방법으로 이전 스테이지의 아티팩트를 참조할 수 있으며, `COPY --from=nginx:latest /etc/nginx/nginx.conf /nginx.conf`와 같이 외부 이미지애서의 참조도 가능하다.
또한 빌드 시 `--target` 옵션을 통해 특정 스테이지를 빌드하는 것도 가능하며, `FROM build AS release`와 같이 이전 단계를 참조하는 것도 가능하다.  
Docker multi-stage 빌드를 적극 사용한다면 최종 이미지의 크기를 줄일 수 있을 뿐만 아니라 debug stage, testing stage, production stage와 같이 stage를 구분 하여 더욱 도커스러운 운영을 할 수 있을 것이다.