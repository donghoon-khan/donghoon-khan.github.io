---
title: Spring boot에서 InfluxDB2.0 사용하기
category:
- web
tags:
- spring
summary: How to spring-boot integration for InfluxDB v2.0
---
[InfluxDB](https://www.influxdata.com/products/influxdb-overview/)는 시계열 데이터를 위한 Time series database이다. Go언어로 만들어진 오픈소스이며, OpenTSDB와 kairosDB를 합친 것 보다 많은 수의 Github star를 받을 정도로 유명한 시계열 DB이다. InfluxDB v2.0(beta) 나오면서 client library도 바꼈는데(InfluxDB v1.8 부터는 같다), Spring-boot 애플리케이션에서 InfluxDB 2.0을 사용하는 방법에 대해서 알아보자.

## InfluxDB v2.0 설치
InfluxDB v2.0의 설치는 매우 간단하다. 바이너리 파일 하나만 설치하면 된다. 현재 macOS, Linux, Docker 그리고 Kubernetes까지 지원하고 있다. [Start with influxdb](https://v2.docs.influxdata.com/v2.0/get-started/#start-with-influxdb-oss)에서 자기한테 맞는 환경으로 설치하자.

```bash
$ tar xvzf path/to/influxdb_2.0.0-beta.16_linux_amd64.tar.gz
$ sudo cp influxdb_2.0.0-beta.16_linux_amd64/{influx,influxd} /usr/local/bin/
$ influxd
```

### Set up InfluxDB2
influxd 데몬이 정상적으로 실행 되었다면, http://localhost:9999로 접속하자.
![influxdb get started](/assets/img/posts/2020-08-08-spring-boot-influxdb2-get-started.png)

Get Started 버튼을 눌러 계정과 버킷을 만들자.
![influxdb initialize](/assets/img/posts/2020-08-08-spring-boot-influxdb2-initialize.png)

설정을 마무리 하고 홈 화면으로 가면 아래와 같은 UI가 뜰 것이다.
![influxdb home](/assets/img/posts/2020-08-08-spring-boot-influxdb2-home.png)

대시보드, 쿼리, 태스크 및 알람등 다양한 기능을 사용할 수 있으니 확인해 보자. 만약 influx CLI를 사용하고자 하면 token을 확인해 환경변수로 등록하자.
```bash
export INFLUX_TOKEN=${token}
```

## Spring Integration for InfluxDB2
Spring과 InfluxDB v2.0을 통합하면 auto-configuration, Actuator micrometer registry, Actuator health 기능을 이용할 수 있는데 [Github](https://github.com/influxdata/influxdb-client-java/tree/master/spring)에서 확인할 수 있다.  
Spring-boot 애플리케이션에서 InfluxDB v2.0을 사용할 수 있는 방법에 대해 알아보자.

### Create Spring-boot project
[Spring initializr](https://start.spring.io/)에서 `web`과  `actuator`를 dependency 추가하고 스프링 프로젝트를 생성하자.
또는 curl을 이용해 프로젝트를 생성하고 pom.xml에 influxdb-spring 디펜던시를 추가하자.
```bash
$ mkdir spring-influx2 && cd spring-influx2
$ curl https://start.spring.io/starter.tgz -d dependencies=web,actuator | tar -xzvf -
$ vi pom.xml
```
```bash
<dependency>
  <groupId>com.influxdb</groupId>
  <artifactId>influxdb-spring</artifactId>
  <version>1.10.0</version>
</dependency>
```

### InfluxDB2 auto-configuration
InfluxDB의 접속 정보를 properties 파일에 추가하자.
```bash
spring:
  profiles: local
  influx2: 
    url: http://localhost:9999 # URL to connect to InfluxDB.
    username: my-user # Username to use in the basic auth.
    password: my-password # Password to use in the basic auth.
    token: my-token # Token to use for the authorization.
    org: my-org # Default destination organization for writes and queries.
    bucket: my-bucket # Default destination bucket for writes.
    logLevel: BODY # The log level for logging the HTTP request and HTTP response. (Default: NONE)
    readTimeout: 5s # Read timeout for OkHttpClient. (Default: 10s)
    writeTimeout: 5s # Write timeout for OkHttpClient. (Default: 10s)
    connectTimeout: 5s # Connection timeout for OkHttpClient. (Default: 10s)
```

### Actuator 설정
Actuator의 health endpoint를 통해 influxDB의 상태를 조회할 수 있게 properties 설정하자.
```bash
management:
  health:
    influxdb:
      enabled: true
  endpoint:
    health:
      show-details: "ALWAYS"
  endpoints:
    web:
      base-path: /actuator
      exposure:
        include: "*"
        exclude:
        - env
        - bean
```

애플리케이션을 빌드하고 테스트 해보자.
```bash
$ ./mvnw install -Dmaven.test.skip=true
$ java -jar target/*.jar
$ curl localhost:8080/actuator/health|jq #poastman or browser로 확인가능
{
  "status": "UP",
  "components": {
    "diskSpace": {
      "status": "UP",
      "details": {
        "total": 100255690752,
        "free": 63867633664,
        "threshold": 10485760,
        "exists": true
      }
    },
    "influxDb": {
      "status": "UP",
      "details": {
        "status": "PASS",
        "message": "ready for queries and writes"
      }
    },
    "ping": {
      "status": "UP"
    }
  }
}
```
influxdb의 `status`가 `UP`인 것을 확인할 수 있다. 접속 정보가 다를 경우 `DOWN`상태일 것이다.

### Micrometer registry
만약 InfluxDB의 metric을 micrometer와 연동하고 싶으면 다음과 같은 설정을 추가하면 된다.
```bash
management.metrics.export.influx2:
    bucket: my-bucket # Specifies the destination bucket for writes
    org: my-org # Specifies the destination organization for writes.
    token: my-token # Authenticate requests with this token.
    uri: http://localhost:8086/api/v2 # The URI for the Influx backend. (Default: http://localhost:8086/api/v2)
    compressed: true # Whether to enable GZIP compression of metrics batches published to Influx. (Default: true)
    autoCreateBucket: true #  Whether to create the Influx bucket if it does not exist before attempting to publish metrics to it. (Default: true)
    everySeconds: 3600 # The duration in seconds for how long data will be kept in the created bucket.
    enabled: true # Whether exporting of metrics to this backend is enabled. (Default: true)
    step: 1m # Step size (i.e. reporting frequency) to use. (Default: 1m)
    connect-timeout: 1s # Connection timeout for requests to this backend. (Default: 1s)
    read-timeout: 10s # Read timeout for requests to this backend. (Default: 10s)
    num-threads: 2 # Number of threads to use with the metrics publishing scheduler. (Default: 2)
    batch-size: 10000 # Number of measurements per request to use for this backend. If more measurements are found, then multiple requests will be made. (Default: 10000)
```

Maven dependency:
```bash
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-influx2</artifactId>
    <version>1.2.0-bonitoo-SNAPSHOT</version>
</dependency>
```