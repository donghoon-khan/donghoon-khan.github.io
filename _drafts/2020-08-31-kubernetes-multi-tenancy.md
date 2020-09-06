---
title: Kubernetes Multi-Tenancy
category: 
- DevOps
tags:
- kubernetes
summary: Kubernetes Multi-Tenancy
thumbnail: "/assets/img/thumbnail/2020-08-31-kubernetes-multi-tenency-thumbnail.png"
---
최근 CNCF 웨비나 or 쿠버네티스 관련 자료를 찾아 보면 Multi-Tenancy를 주제로 많이 다루고 있다. 쿠버네티스를 여러 Tenant들이 함께 사용가능하게끔 제공하고 운영하는 것은 어려운 일이다. security, resource isolation 등의 이유로 대부분이 팀, 프로젝트 또는 환경 별로 쿠버네티스 클러스터를 만들어서 사용하고 있다고 한다.  
Kubernetes에서 multi-tenancy를 제공하는 방법에 대해 알아보자.

**Multi-Tenancy:** 소프트웨어 아키텍처의 하나를 가리키며, 여기에서 하나의 소프트웨어 인스턴스가 한 대의 서버 위에서 동작하면서 여러 개의 `테넌트(tenant)`를 서비스한다. 여기에서 테넌트란 소프트웨어 인스턴스에 대해 공통이 되는 특정 접근 권한을 공유하는 사용자들의 그룹이다. 멀티테넌트 구조에서 응용 소프트웨어는 `데이터`, `구성`, `사용자 관리`, `테넌트 개별 기능 및 비기능 속성`을 포함하여, 모든 테넌트에게 인스턴스의 일부분을 단독적으로 제공하기 위해 설계되어 있다. 멀티테넌시는 개개의 소프트웨어 인스턴스들이 각기 다른 테넌트를 위해 운영되는 멀티인스턴스 구조와는 상반된다. [- WIKIPEDIA](https://ko.wikipedia.org/wiki/%EB%A9%80%ED%8B%B0%ED%85%8C%EB%84%8C%EC%8B%9C)
{: .notice--info}

# What is multi-tenant Kubernetes?
쿠버네티스는 cluster, node, namespace, pod 등 여러 레이어의 자원을 가지고 있는데, 테넌트 간 어느 정도의 isolation이 필요한 가에 따라 다음과 같이 멀티 테넌시 모델을 나눌 수 있을 것이다.
- `Soft multi-tenancy model`: 소프트 멀티 테넌시모델은 테넌트들이 악의가 없다고 가정하는 모델이다. 소프트 멀티 테넌시 모델의 포커스는 사고를 최소화하고, 만약 발생한다면 좋지 못한 결과를 manage하는 것에 있다. (Ex. 테넌트들이 같은 회사의 서로 다른 팀일 경우)
- `Hard multi-tenancy model`: 하드 멀티 테넌시 모델은 테넌트가 악의적인 것으로 가정하기에 테넌트 간의 신뢰가 전혀 없는 것을 기본으로 한다. 테넌트들의 자원이 격리되어 서로 다른 테넌트의 자원에 대한 접근이 허용되지 않아야 한다. 클러스터는 테넌트 자원을 격리하고 다른 테넌트의 자원에 대한 접근을 차단하는 방식으로 구성되어야 한다. (Ex. 테넌트들이 독립적인 서로 다른 회사일 경우)

쿠버네티스는 멀티 테넌시 환경을 위해 다양한 기능을 제공한다.
하나의 쿠버네티스 환경에 여러 테넌트 들이 접근을 하기에 Access control 기능이 필요하며 RBAC, Network policy, Admission control과 같은 기능을 제공한다. 또한 테넌트들에게 적절하게 자원을 할당 및 스케줄링 해야 하기에 Resource quota, Limit range, Pod affinity와 같은 기능을 제공한다.

# Namespace per tenant

# Nodes per tenant

# Cluster per tenant