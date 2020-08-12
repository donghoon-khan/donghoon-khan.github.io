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

