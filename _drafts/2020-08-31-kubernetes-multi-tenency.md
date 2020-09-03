---
title: Kubernetes Multi-Tenancy
category: 
- DevOps
tags:
- kubernetes
summary: Kubernetes Multi-Tenancy
thumbnail: "/assets/img/thumbnail/2020-08-05-k8s-package-management-helm.png"
---
최근 KubeCon 또는 쿠버네티스 관련 자료를 찾아 보면 Multi-Tenancy를 주제로 많이 다루고 있다. 쿠버네티스를 여러 Tenant들이 함께 사용가능하게끔 제공하고 운영하는 것은 어려운 일이다. security, resource isolation 등의 이유로 대부분이 팀, 프로젝트 또는 환경 별로 쿠버네티스 클러스터를 만들어서 사용하고 있다고 한다.  
Kubernetes에서 multi-tenancy를 제공하는 방법에 대해 알아보자.

**Multi-Tenancy:** 소프트웨어 아키텍처의 하나를 가리키며, 여기에서 하나의 소프트웨어 인스턴스가 한 대의 서버 위에서 동작하면서 여러 개의 `테넌트(tenant)`를 서비스한다. 여기에서 테넌트란 소프트웨어 인스턴스에 대해 공통이 되는 특정 접근 권한을 공유하는 사용자들의 그룹이다. 멀티테넌트 구조에서 응용 소프트웨어는 `데이터`, `구성`, `사용자 관리`, `테넌트 개별 기능 및 비기능 속성`을 포함하여, 모든 테넌트에게 인스턴스의 일부분을 단독적으로 제공하기 위해 설계되어 있다. 멀티테넌시는 개개의 소프트웨어 인스턴스들이 각기 다른 테넌트를 위해 운영되는 멀티인스턴스 구조와는 상반된다. [- WIKIPEDIA](https://ko.wikipedia.org/wiki/%EB%A9%80%ED%8B%B0%ED%85%8C%EB%84%8C%EC%8B%9C)
{: .notice--warning}

