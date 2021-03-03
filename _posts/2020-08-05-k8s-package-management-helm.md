---
title: Helm으로 Kubernetes 패키지 관리하기
category: 
- DevOps
tags:
- helm
- kubernetes
summary: Introduction to Helm - Package Manager for Kubernetes
thumbnail: "/assets/img/thumbnail/helm.png"
---

[Helm](https://helm.sh/)은 쿠버네티스용 패키지 매니지먼트 도구로 쿠버네티스용 apt dpkg, yum rpm, homebrew라고 생각하면 된다. Helm은 패키지 매니지먼트 도구 답게 설치, 업그레이드, 롤백, 삭제 등 강력하고 다양한 기능을 제공한다.  
웹 애플리케이션을 쿠버네티스 클러스터에 배포한다고 가정하자. 파드와 컨트롤러 외에도 DB에서 사용할 볼륨, DB 접속 정보를 저장할 컨피그 맵과 시크릿, 웹 애플리케이션을 노출 할 서비스 등 많은 오브젝트를 생성해야 한다. 전통적인 웹 애플리케이션하나에도 이렇게 많은 쿠버네티스 오브젝트들이 필요한데 MSA에서는 어떨까? 생성 뿐만 아니라 롤백, 삭제와 같은 작업을 해야 한다고 생각하면 끔찍하다. `Helm`을 도입한다면 귀찮고 번거로운 작업이 상당히 줄어드는데, `Helm`에 대해서 알아보자.

## Helm concept

`Helm`을 잘 사용하기 위해서는 다음 세가지 요소에 대해 알아야 한다.

1. `Chart`: 차트는 쿠버네티스 클러스터 내에서 필요한 모든 리소스들의 정의가 포함된 패키지이다. 타 시스템에서 apt dpkg, npm 등으로 패키지 관리 하던 것을 쿠버네티스에서는 helm chart로 할 수 있다.
2. `Release`: 쿠버네티스에서 구동되는 helm chart의 인스턴스이다. 하나의 chart는 여러 번 설치할 수 있는데, 설치할 때 마다 새로운 release가 생성된다.
3. `Repository`: helm chart를 저장하고 공유하는 장소이다. `helm repo add stable https://kubernetes-charts.storage.googleapis.com/` 명령어로 공식 저장소를 추가 할 수 있다. 이 외에도 여러 저장소를 추가하고 저장소에서 차트를 search/pull/install 해서 사용 할 수 있다.

요약하면 파드, 컨트롤러, 볼륨, 컨피그맵, 시크릿, 서비스 등 마이크로 서비스에 필요한 쿠버네티스 오브젝트들이 정의된 `Chart`로 `Release`를 생성하고 관리하며 `Chart`는 `Repository`로 공유할 수 있다.

## Helm 설치

Debian/Ubuntu의 경우 apt를 이용해 쉽게 설치 할 수 있다. 다른 OS의 경우 [설치 방법](https://helm.sh/docs/intro/install/)을 참고하자.

```bash
$ curl https://helm.baltorepo.com/organization/signing.asc | sudo apt-key add -
$ sudo apt-get install apt-transport-https --yes
$ echo "deb https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
$ sudo apt-get update
$ sudo apt-get install helm
```

Helm을 사용할 준비가 끝났다. Stable chart가 있는 공식 Helm repository를 추가하자.

```bash
$ helm repo add stable https://kubernetes-charts.storage.googleapis.com/
$ helm search repo stable
```

### NOTE - Helm2 vs Helm3

<div style="font-size: 15px; font-style: italic;">
Helm이 버전 2에서 3로 업데이트 되면서 가장 크게 달라진 것은 `tiller`가 없어진 것이다.<br/>
Helm2는 서버/클라이언트 구조로 helm 이라는 클라이언트가 tiller라는 쿠버네티스 클러스터에 배포 된 서버와 통신하고 쿠버네티스 패키지를 관리한다. 때문에 tiller의 서비스어카운트를 생성하고 tiller 네임스페이스를 생성해야 했다. 그리고 tiller가 배포된 쿠버네티스 네임스페이스를 기준으로 패키지를 관리 하기 때문에 서로 다른 네임스페이스에서 동일한 Helm 릴리즈 이름을 사용할 수 없었다.<br/>
Helm3로 업그레이드 되면서 클라이언트/라이브러리 구조로 바꼈다. 쿠버네티스 클러스터에 tiller를 배포할 필요가 없어진 것이다. kubectl로 쿠버네티스 클러스터에 접근할 수 있는 상태에서 helm 클라이언트만 있으면 Helm을 사용할 수 있다.
</div>

## Helm command

Helm은 많은 command를 제공하는데 자주 사용하는 command을 알아보자. 이 외의 command list 또는 자세한 사용법은 [Helm docs](https://helm.sh/docs/helm/)를 참고하자.

### Helm search

Helm은 repository, hub에서 검색할 수 있는 명령어를 제공한다. repository는 `helm repo add`로 추가한 저장소에서 차트를 검색하며 퍼블릭 테트워크 접속이 필요하지 않다. hub는 [Helm Hub](https://hub.helm.sh/)에서 차트를 검색한다. 
```bash
$ helm search repo mariadb
NAME             	CHART VERSION	APP VERSION	DESCRIPTION                                       
stable/mariadb   	7.3.14       	10.3.22    	DEPRECATED Fast, reliable, scalable, and easy t...
stable/phpmyadmin	4.3.5        	5.0.1      	DEPRECATED phpMyAdmin is an mysql administratio...

$ helm search hub mariadb
URL                                               	CHART VERSION	APP VERSION	DESCRIPTION                                       
https://hub.helm.sh/charts/bitnami/mariadb        	7.6.1        	10.3.23    	Fast, reliable, scalable, and easy to use open-...
https://hub.helm.sh/charts/bitnami/mariadb-galera 	4.0.0        	10.5.4     	MariaDB Galera is a multi-master database clust...
https://hub.helm.sh/charts/bitnami/phpmyadmin     	6.3.1        	5.0.2      	phpMyAdmin is an mysql administration frontend    
https://hub.helm.sh/charts/bitnami/mariadb-cluster	1.0.1        	10.2.14    	Chart to create a Highly available MariaDB cluster
https://hub.helm.sh/charts/ibm-charts/ibm-maria...	1.1.2        	           	MariaDB is developed as open source software an...
https://hub.helm.sh/charts/ibm-charts/ibm-galer...	1.1.0        	           	Galera Cluster is a multi-master solution for M..
```

### Helm pull

repository에서 로컬 디렉토리로 chart를 다운로드 할 수 있다. Helm Hub에서 chart를 가져오고 싶은 경우 `repo add`로 추가한 후 `pull`을 받을 수 있다.

``` bash
$ helm pull stable/mariadb
$ helm repo add bitnami https://charts.bitnami.com/bitnami
$ helm pull bitnami/mariadb
```

### Helm install

helm chart를 설치하려면 [helm install](https://helm.sh/docs/helm/helm_install/)명령을 사용하면 된다.

```bash
$ ls
mariadb-7.3.14.tgz

$ helm install helm-mariadb-demo ./mariadb-7.3.14.tgz
NAME: helm-mariadb-demo
LAST DEPLOYED: Thu Jul 16 15:35:14 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
...
```

설치하려는 차트를 참조하는 방법은 다음과 같다.

1. `차트 참조`: helm install helm-mariadb-demo stable/mariadb
2. `패키지 차트 경로`: helm install helm-mariadb-demo ./mariadb-7.3.14.tgz
3. `압축을 푼 차트의 디렉토리 경로`: helm install helm-mariadb-demo ./mariadb
4. `URL`: helm install helm-mariadb-demo <https://example.com/charts/mariadb-7.3.14.tgz>
5. `차트 참조 및 repository URL`: helm install --repo <https://example.com/charts/> helm-mariadb-demo mariadb

### Helm chart custom

대부분 chart를 그대로 사용하지 않고 각자의 환경에 맞게 커스텀해서 사용할 것이다. `show values`명령을 이용해 install시 구성 가능한 옵션에 대해서 조회할 수 있다.

```bash
$ helm show values mariadb-7.3.14.tgz
## Global Docker image parameters
## Please, note that this will override the image parameters, including dependencies, configured to use the global value
## Current available global Docker image parameters: imageRegistry and imagePullSecrets
##
# global:
#   imageRegistry: myRegistryName
#   imagePullSecrets:
#     - myRegistryKeySecretName
#   storageClass: myStorageClass
...

$ tar -xvzf mariadb-7.3.14.tgz
$ cat mariadb/values.yaml
## Global Docker image parameters
## Please, note that this will override the image parameters, including dependencies, configured to use the global value
## Current available global Docker image parameters: imageRegistry and imagePullSecrets
##
# global:
#   imageRegistry: myRegistryName
#   imagePullSecrets:
#     - myRegistryKeySecretName
#   storageClass: myStorageClass
```

`helm show values`명령을 통해 구성 가능한 values를 확인할 수 있다. values.yaml에 기술한 내용을 보여주는 것이다. 이 values를 install시 사용하는 것인데 설정하는 방법은 두가지 방법이 있다.

- `--values` or `-f`: override할 .yaml파일을 지정한다. 여러 개의 .yaml파일을 지정할 수 있는데 가장 오른쪽 파일이 우선시 된다.
- `--set`: 명령행에서 override를 지정한다.

```bash
$ echo '{mariadbUser: user0, mariadbDatabase: user0db}' > config.yaml
$ helm install -f config.yaml stable/mariadb --generate-name

$ helm install stable/mariadb --set mariadbUser=user0 --set mariadbDatabase=user0db
$ helm ls
```

### Helm upgrade

배포된 Release의 구성을 변경하고 싶을 때, `helm upgrade`명령을 사용하면 된다. 배포되어 있는 release를 기준으로 사용자가 입력한 정보에 따라 upgrade되는데 변경된 부분만 업데이트한다.

```bash
$ helm get values helm-mariadb-demo
USER-SUPPLIED VALUES:
null

$ helm upgrade helm-mariadb-demo stable/mariadb --set mariadbUser=user1
Release "helm-mariadb-demo" has been upgraded. Happy Helming!
NAME: helm-mariadb-demo
LAST DEPLOYED: Thu Jul 16 16:25:49 2020
NAMESPACE: default
STATUS: deployed
REVISION: 2
NOTES:
...

$ helm get values helm-mariadb-demo
USER-SUPPLIED VALUES:
mariadbUser: user1
```

### Helm rollback

Helm Release를 이전 버전으로 롤백하려면 `helm rollback [RELEASE] [REVISION]`명령을 사용하면 된다.

```bash
$ helm ls
NAME               NAMESPACE    REVISION    UPDATED                                 STATUS      CHART            APP VERSION
helm-mariadb-demo   default     2           2020-07-16 16:25:49.13778095 +0900 KST  deployed    mariadb-7.3.14   10.3.22

$ helm get values helm-mariadb-demo
USER-SUPPLIED VALUES:
mariadbUser: user1

$ helm rollback helm-mariadb-demo 1
Rollback was a success! Happy Helming!

$ helm get values helm-mariadb-demo
null

$ helm ls
NAME                 NAMESPACE  REVISION    UPDATED                                 STATUS       CHART          APP VERSION
helm-mariadb-demo    default    3           2020-07-16 17:21:05.513623447 +0900 KST deployed     mariadb-7.3.14 10.3.22
```

### Helm uninstall

배포한 Helm release를 삭제하고자 할 떄는 `helm uninstall [RELEASE]`명령을 사용하면 된다. helm 2에서는 release 및 history 삭제 시 `delete --purge`를 사용해야 했는데 이제 uninstall만으로도 release와 history가 삭제된다.

```bash
$ helm ls
NAME                NAMESPACE   REVISION    UPDATED                                 STATUS      CHART            APP VERSION
helm-mariadb-demo   default     3           2020-07-16 17:21:05.513623447 +0900 KST deployed    mariadb-7.3.14   10.3.22

$ helm uninstall helm-mariadb-demo
release "helm-mariadb-demo" uninstalled
$ helm ls
NAME                NAMESPACE   REVISION    UPDATED                                 STATUS      CHART            APP VERSION
```

### Helm chart 만들기

`helm create [CHART_NAME]`명령을 사용해서 새로운 차트를 만들 수 있다. 차트를 만든 후 [Helm template guide](https://helm.sh/docs/chart_template_guide/), [Helm chart](https://helm.sh/docs/topics/charts/), [Go Templating](https://golang.org/pkg/text/template/)을 참고해서 나만의 차트로 수정하면 된다. 공식문서를 정독하고 다른 사람이 만들어 놓은 차트를 보면 차트를 어떤 식으로 만들어야 하는지 감이 올 것이다.

```bash
$ helm create helm-chart-demo
Creating helm-chart-demo
```

## Conclusion

[github](https://github.com/helm/charts) 또는 [helmhub](https://hub.helm.sh/)를 보면 다양한 차트들을 만날 수 있다. 정말 많은 사람들이 Helm을 이용하고 있는 것 같다. Chart를 처음 만드는 것이 막막하다면 잘 만들어진 많은 차트들을 따라해 보자. 금방 내가 원하는 차트를 만들 수 있을 것이다.
