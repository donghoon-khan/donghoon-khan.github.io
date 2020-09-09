---
title: Kubernetes Multi-Tenancy
category: 
- DevOps
tags:
- kubernetes
summary: Kubernetes Multi-Tenancy
thumbnail: "/assets/img/thumbnail/2020-08-31-kubernetes-multi-tenency-thumbnail.png"
---
쿠버네티스 환경을 여러 Tenant들이 사용하게끔 하기 위해선 어떻게 해야 할까? 이와 관련되어 최근 CNCF or 쿠버네티스 관련 자료를 찾아 보면 많은 곳에서 Multi-Tenancy를 주제로 다루고 있다. Multi-Tenancy 환경에서는 Single-Tenant에서는 고려하지 않아도 되는 Security, Resource isolation 등의 문제들이 발생한다.  
대부분의 팀, 프로젝트 또는 환경 별로 쿠버네티스 클러스터를 만들어서 사용하고 있다고 한다. 가장 Simple하게 Multi-Tenancy 환경을 구성할 수 있으며 Security, Resource isolation과 같은 문제를 쉽게 해결할 수 있다. 하지만 낮은 Resource utilization을 가질 수 밖에 없다.  
반면 Tenant별로 Namespace를 제공하는 방법을 사용할 경우 Resource utilization은 높지만 Resource isolation, security 측면에서 문제가 발생할 수 있다. Multi-tenancy 모델 별로 상황에 맞게 제공해야 할 것인데 쿠버네티스에서 Multi-tenancy를 제공하는 방법에 대해 자세히 알아보자.

**Multi-Tenancy:** 소프트웨어 아키텍처의 하나를 가리키며, 여기에서 하나의 소프트웨어 인스턴스가 한 대의 서버 위에서 동작하면서 여러 개의 `테넌트(tenant)`를 서비스한다. 여기에서 테넌트란 소프트웨어 인스턴스에 대해 공통이 되는 특정 접근 권한을 공유하는 사용자들의 그룹이다. 멀티테넌트 구조에서 응용 소프트웨어는 `데이터`, `구성`, `사용자 관리`, `테넌트 개별 기능 및 비기능 속성`을 포함하여, 모든 테넌트에게 인스턴스의 일부분을 단독적으로 제공하기 위해 설계되어 있다. 멀티테넌시는 개개의 소프트웨어 인스턴스들이 각기 다른 테넌트를 위해 운영되는 멀티인스턴스 구조와는 상반된다. [- WIKIPEDIA](https://ko.wikipedia.org/wiki/%EB%A9%80%ED%8B%B0%ED%85%8C%EB%84%8C%EC%8B%9C)
{: .notice--info}

# What is multi-tenant Kubernetes?
쿠버네티스는 cluster, node, namespace, pod 등 여러 레이어의 자원을 가지고 있는데, 테넌트 간 어느 정도의 isolation이 필요한 가에 따라 다음과 같이 멀티 테넌시 모델을 나눌 수 있을 것이다.
- `Soft multi-tenancy model`: 소프트 멀티 테넌시 모델은 테넌트들이 옳은 행동을 하고 악의가 없다고 가정하는 모델이다. 사고들을 방지하고 사고 발생 시 결과 관리에 중점을 두는 모델이다. (Ex. 테넌트들이 같은 회사의 서로 다른 팀일 경우)
- `Hard multi-tenancy model`: 하드 멀티 테넌시 모델은 테넌트들이 악의적인 행동을 할 수 있다고 가정하는 모델이다. 때문에 테넌트들 사이의 신뢰가 존재하지 않는다고 가정한다. 테넌트들의 자원은 격리되고 다른 테넌트와의 접근을 허용하지 않는다. 클러스터는 테넌트들의 자원을 격리하고 다른 테넌트와의 접근을 차단하는 방식으로 구성되어야 한다. (Ex. 테넌트들이 독립적인 서로 다른 회사일 경우)

쿠버네티스는 멀티 테넌시 환경을 위해 다양한 기능을 제공한다.
하나의 쿠버네티스 환경에 여러 테넌트 들이 접근을 하기에 Access control 기능이 필요하며 RBAC, Network policy, Admission control과 같은 기능을 제공한다. 또한 테넌트들에게 적절하게 자원을 할당 및 스케줄링 해야 하기에 Resource quota, Limit range, Pod affinity와 같은 기능을 제공한다.

# Namespace per tenant

# Nodes per tenant

# Cluster per tenant

# Reference
- [https://platform9.com/blog/kubernetes-multi-tenancy-best-practices/](https://platform9.com/blog/kubernetes-multi-tenancy-best-practices/)
- [https://www.replex.io/blog/kubernetes-in-production-best-practices-and-checklist-for-multi-tenancy](https://www.replex.io/blog/kubernetes-in-production-best-practices-and-checklist-for-multi-tenancy)
- [Multi-Tenancy in Kubernetes: Best Practices Today, and Future Directions - David Oppenheimer](https://www.youtube.com/watch?v=xygE8DbwJ7c)
- [https://www.cncf.io/webinars/things-to-consider-to-operate-a-multi-tenant-kubernetes-cluster-multi-tenant-kubernetes-cluster](https://www.cncf.io/webinars/things-to-consider-to-operate-a-multi-tenant-kubernetes-cluster-multi-tenant-kubernetes-cluster%eb%a5%bc-%ec%9a%b4%ec%98%81%ed%95%98%ea%b8%b0-%ec%9c%84%ed%95%b4-%ea%b3%a0%eb%a0%a4%ed%95%a0/)
- [https://www.infoq.com/presentations/multi-tenancy-kubernetes/](https://www.infoq.com/presentations/multi-tenancy-kubernetes/)