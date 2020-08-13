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
[Spring initializr](https://start.spring.io/)에서 dependencies로 `webflux`, `actuator`를 추가하고 스프링 프로젝트를 생성하자.
또는 curl을 이용해 프로젝트를 생성하자.
```bash
$ mkdir kubernetes-spring && cd kubernetes-spring
$ curl https://start.spring.io/starter.tgz -d dependencies=webflux,actuator | tar -xzvf -
```

### InfluxDB2 auto-configuration