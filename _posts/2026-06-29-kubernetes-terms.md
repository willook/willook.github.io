---
title: "쿠버네티스 용어 정리"
description: "Kubernetes를 공부하면서 자주 만나는 핵심 용어를 Docker와 온프레미스와 비교하여 정리했습니다."
date: 2026-06-29 09:10:00 +0900
categories: [Infra, Kubernetes]
tags: [kubernetes, k8s, kubectl, helm, ingress, glossary]
---

## Kubernetes 용어집

이 페이지는 Kubernetes를 공부하면서 새로 만나는 용어를 짧게 정리하는 곳이다.

각 용어는 다음 관점으로 이해한다.

- 한 줄 정의
- Docker/온프레미스 경험과 비교한 감각
- 왜 중요한지

## Cluster

여러 서버를 하나의 Kubernetes 실행 환경으로 묶은 단위.

```text
서버 여러 대를 하나의 자원 풀처럼 보는 것
```

예를 들어 `control-plane-1`, `worker-1`, `worker-2` 서버가 하나의 Kubernetes cluster에 묶이면, 사용자는 각 서버에 직접 들어가 앱을 띄우기보다 cluster에 "이 앱을 띄워줘"라고 요청한다.

## Node

Kubernetes cluster에 속한 실제 서버 한 대.

Node는 물리 서버일 수도 있고, VM일 수도 있고, 클라우드 인스턴스일 수도 있다.

```text
Node = Kubernetes가 일할 수 있는 서버 한 대
```

Node 위에서 실제 Pod와 Container가 실행된다.

## Control Plane

Kubernetes cluster의 두뇌 역할을 하는 구성 요소.

Control plane은 cluster 전체 상태를 보고, 어떤 Pod를 어느 Node에 띄울지 결정하고, 원하는 상태와 실제 상태가 다르면 맞추려고 한다.

주요 역할:

- Kubernetes API 제공
- 스케줄링 결정
- cluster 상태 관리
- 원하는 상태와 실제 상태 조정

작은 k3s cluster에서는 서버 한 대가 control plane이자 worker node 역할을 동시에 할 수 있다.

## Worker Node

실제 애플리케이션 Pod가 실행되는 Node.

Control plane이 "이 Pod를 저 Node에 띄워"라고 결정하면, worker node의 kubelet이 container runtime을 통해 컨테이너를 실행한다.

## Pod

Kubernetes가 관리하는 가장 작은 실행 단위.

보통 Pod 하나 안에 컨테이너 하나가 들어간다.

```text
Pod
  app container
```

필요하면 여러 컨테이너가 같은 Pod에 들어갈 수 있다.

```text
Pod
  app container
  sidecar container
```

같은 Pod 안의 컨테이너들은 네트워크와 볼륨을 공유한다.

## Container

격리된 실행 환경에서 돌아가는 프로세스.

Docker에서 실행하던 컨테이너와 같은 개념으로 이해하면 된다. Kubernetes에서는 containerd 같은 container runtime이 실제 컨테이너를 실행한다.

## Container Runtime

컨테이너를 실제로 실행하는 프로그램.

예전에는 Docker Engine을 많이 썼고, 요즘 Kubernetes에서는 보통 containerd를 많이 쓴다.

```text
Kubernetes -> kubelet -> container runtime -> container
```

## kubelet

각 Node에서 실행되는 Kubernetes agent.

Control plane의 지시를 받아 해당 Node에 Pod를 만들고, 상태를 보고하고, 문제가 있으면 다시 맞추려고 한다.

```text
kubelet = Node 안에서 Kubernetes의 손발 역할
```

## kubectl

Kubernetes cluster를 조작하는 CLI 도구.

Docker에서 `docker ps`, `docker logs`, `docker exec`를 쓰듯이 Kubernetes에서는 `kubectl`을 쓴다.

예시:

```bash
kubectl get pods
kubectl get nodes
kubectl logs my-pod
kubectl apply -f deployment.yaml
```

## Deployment

Pod를 몇 개 띄울지, 어떤 이미지로 띄울지, 어떻게 업데이트할지를 관리하는 Kubernetes 리소스.

예를 들어:

```text
rag-api:v1 컨테이너를 Pod 3개로 유지해줘.
```

이미지를 `v2`로 바꾸면 Deployment가 rolling update를 수행한다.

## Service

Pod들이 바뀌어도 고정된 이름과 네트워크 endpoint로 접근할 수 있게 해주는 리소스.

Pod는 죽고 다시 생길 수 있어서 IP가 바뀔 수 있다. Service는 그 앞에 고정된 접근 지점을 만들어준다.

```text
model-api Service -> 현재 살아있는 model-api Pod들로 연결
```

## Ingress

외부 HTTP/HTTPS 요청을 cluster 내부 Service로 라우팅하는 리소스.

예를 들어:

```text
https://rag.example.com -> rag-api Service
https://grafana.example.com -> grafana Service
```

## ConfigMap

애플리케이션 설정값을 Kubernetes에서 관리하는 리소스.

예:

```text
LOG_LEVEL=info
MODEL_NAME=yolo12m
```

비밀값이 아닌 일반 설정에 사용한다.

## Secret

비밀번호, 토큰, API key 같은 민감한 값을 관리하는 리소스.

예:

```text
OPENAI_API_KEY
DB_PASSWORD
HF_TOKEN
```

## Volume

Pod가 사용할 수 있는 저장소.

컨테이너는 기본적으로 지워졌다가 다시 만들어질 수 있으므로, 데이터가 남아야 하면 Volume을 붙인다.

## PVC, PersistentVolumeClaim

Pod가 필요한 저장소를 요청하는 리소스.

```text
이 앱에 20GB 저장소가 필요해.
```

Kubernetes는 StorageClass와 연결해 실제 저장소를 붙인다.

## Helm

Kubernetes 앱 패키지 매니저.

여러 Kubernetes YAML을 하나의 Chart로 묶어 설치하고 업그레이드할 수 있게 해준다.

예:

```bash
helm install grafana grafana/grafana
helm upgrade rag-api ./chart
```

## Argo CD

GitOps 기반 배포 도구.

Git repository에 적힌 Kubernetes 설정을 정답 상태로 보고, cluster가 그 상태와 같아지도록 동기화한다.

```text
Git에 변경 push -> Argo CD가 cluster에 반영
```

## Rancher

여러 Kubernetes cluster를 관리하는 웹 UI/플랫폼.

Pod, Deployment, Service, Node 상태를 보고, Helm 앱 설치, 권한 관리, 여러 cluster 관리를 할 수 있다.

## k8s

Kubernetes의 줄임말.

`K`와 `s` 사이에 글자가 8개라서 k8s라고 부른다.

## k3s

가볍게 패키징된 Kubernetes 배포판.

개인 서버, edge, 온프레미스 소규모 환경, 학습용으로 시작하기 좋다.

k3s도 진짜 Kubernetes이므로 `kubectl`, Deployment, Service, Helm 같은 개념을 그대로 사용한다.

## Rolling Update

애플리케이션을 한 번에 모두 내리지 않고, Pod를 조금씩 새 버전으로 교체하는 방식.

예:

```text
v1 Pod 하나 내림
v2 Pod 하나 띄움
정상 확인
다음 Pod 교체
```

서비스 중단을 줄이고, 실패 시 롤백하기 쉽다.

## Rollback

배포가 잘못되었을 때 이전 버전으로 되돌리는 것.

Kubernetes Deployment는 배포 이력을 보고 이전 ReplicaSet으로 되돌릴 수 있다.

```bash
kubectl rollout undo deployment/rag-api
```

## State Drift

서버나 환경마다 상태가 달라지는 문제.

예를 들어 직접 SSH로 여러 서버를 업데이트하다 보면 이런 일이 생길 수 있다.

```text
server-1은 v2
server-2는 v1
server-3은 재시작 실패
```

Kubernetes는 원하는 상태를 선언하고 cluster가 그 상태를 맞추게 하므로 state drift를 줄이는 데 도움이 된다.

## Cordon

특정 Node에 새로운 Pod가 배치되지 않도록 막는 것.

서버 점검이나 업데이트 전에 사용한다.

```bash
kubectl cordon node-1
```

## Drain

특정 Node에서 실행 중인 Pod들을 다른 Node로 비우는 작업.

Node OS 업데이트나 점검 전에 사용한다.

```bash
kubectl drain node-1
```

## Uncordon

Cordon한 Node를 다시 스케줄 가능한 상태로 되돌리는 것.

```bash
kubectl uncordon node-1
```

## Traefik

k3s에 기본으로 포함되는 HTTP reverse proxy / load balancer이자 기본 Ingress Controller.

k3s에서 Ingress 리소스를 만들면 Traefik이 그 규칙을 읽고, 외부 HTTP 요청을 알맞은 Service로 보낸다.

```text
브라우저 -> Traefik -> Ingress 규칙 확인 -> Service -> Pod
```

k3s 기본 설치에서는 Traefik이 80/443 포트를 사용하려고 하므로, 기존 nginx/apache/caddy가 같은 포트를 쓰면 충돌할 수 있다.

## Ingress Controller

Kubernetes의 Ingress 리소스를 실제로 작동시키는 컨트롤러.

Ingress는 "어떤 주소로 들어온 요청을 어느 Service로 보낼지"를 적어둔 규칙이고, Ingress Controller는 그 규칙을 읽어서 실제 네트워크 프록시 설정으로 반영하는 프로그램이다.

```text
Ingress = 라우팅 규칙
Ingress Controller = 그 규칙을 실제로 적용하는 프로그램
```

대표 구현체:

- Traefik
- NGINX Ingress Controller
- HAProxy Ingress
- Kong Ingress Controller

## kubeconfig

`kubectl`이 어떤 Kubernetes cluster에 접속할지, 어떤 사용자 권한으로 접속할지, 어떤 인증 정보를 쓸지를 담고 있는 설정 파일.

```text
kubectl -> kubeconfig 확인 -> Kubernetes API server 접속
```

k3s의 기본 kubeconfig 파일은 보통 `/etc/rancher/k3s/k3s.yaml`에 있다. 이 파일은 cluster 접속 정보와 인증 정보를 포함하므로 권한 관리에 주의해야 한다.

## RBAC

Role-Based Access Control의 줄임말. Kubernetes에서 사용자나 프로그램이 어떤 리소스에 어떤 작업을 할 수 있는지 정하는 권한 모델.

예를 들어 어떤 사용자는 특정 namespace의 Pod 조회만 가능하게 하고, 다른 사용자는 Deployment 수정까지 가능하게 만들 수 있다.

```text
누가 -> 어떤 리소스에 -> 어떤 행동을 할 수 있는가
```

## Role

특정 namespace 안에서 허용할 권한 묶음.

예:

```text
이 namespace의 Pod를 get/list/watch 할 수 있다
```

## RoleBinding

Role을 실제 사용자, 그룹, ServiceAccount에 연결하는 리소스.

```text
Role = 권한 정의
RoleBinding = 그 권한을 누구에게 줄지 연결
```

## ClusterRole

cluster 전체 범위에서 사용할 수 있는 권한 묶음. namespace를 넘어서 적용되는 리소스나 여러 namespace에 공통으로 줄 권한을 정의할 때 사용한다.

## ClusterRoleBinding

ClusterRole을 사용자, 그룹, ServiceAccount에 연결하는 리소스. cluster 전체에 영향을 줄 수 있으므로 운영 환경에서는 신중하게 사용한다.

## Namespace

하나의 Kubernetes cluster 안에서 리소스를 논리적으로 나누는 공간.

예를 들어 개발용 앱은 `dev`, 운영용 앱은 `prod`, 모니터링 도구는 `monitoring` namespace에 둘 수 있다.

```text
Cluster
  namespace: dev
  namespace: prod
  namespace: monitoring
```

RBAC 권한, Service 이름, 리소스 조회 범위를 나눌 때 자주 사용한다.

## Context

`kubectl`이 사용할 cluster, 사용자, namespace 조합.

kubeconfig 안에는 여러 cluster와 사용자 정보가 들어갈 수 있고, context는 그중 어떤 조합으로 명령을 실행할지 정한다.

```text
context = 어느 cluster에 + 어떤 사용자로 + 어떤 namespace를 기본으로 볼지
```

예:

```bash
kubectl config get-contexts
kubectl config use-context my-cluster
```

## kube-system

Kubernetes 자체 운영에 필요한 시스템 리소스들이 주로 들어가는 namespace.

CoreDNS, Traefik, metrics-server 같은 기본 컴포넌트가 보통 이 namespace에서 실행된다.

## CoreDNS

Kubernetes cluster 내부 DNS 서버.

Pod나 Service 이름으로 서로를 찾을 수 있게 해준다.

```text
webapp Service 이름 -> Service IP로 해석
```

## metrics-server

Node와 Pod의 CPU/메모리 사용량 같은 기본 리소스 메트릭을 수집하는 컴포넌트.

나중에 `kubectl top nodes`, `kubectl top pods` 같은 명령에서 사용된다.

## local-path-provisioner

k3s에 기본 포함되는 로컬 디스크 기반 스토리지 provisioner.

Pod가 PVC로 저장소를 요청하면, 단일 노드 실습 환경에서는 서버 로컬 디스크에 저장 공간을 만들어 붙일 수 있게 해준다.

## ServiceLB

k3s에 기본 포함되는 간단한 LoadBalancer 구현.

클라우드 Load Balancer가 없는 온프레미스 환경에서도 `LoadBalancer` 타입 Service를 동작시킬 수 있게 도와준다.

## Job

한 번 실행되고 끝나는 작업을 표현하는 Kubernetes 리소스.

`helm-install-traefik`처럼 설치 작업을 수행한 뒤 `Completed` 상태가 되는 Pod는 정상적으로 일을 끝낸 것이다.

## Container Registry

컨테이너 이미지를 저장하고 배포하는 저장소.

Docker Hub, GitHub Container Registry, 사내 private registry 등이 여기에 해당한다.

```text
Kubernetes Node -> Registry에서 image pull -> Container 실행
```

## OCI Image

Open Container Initiative 표준을 따르는 컨테이너 이미지 형식.

Docker로 빌드한 일반적인 컨테이너 이미지는 대체로 OCI 호환이라 Kubernetes/containerd에서도 실행할 수 있다.

## imagePullSecret

private registry에서 이미지를 가져올 때 사용하는 Kubernetes Secret.

비공개 이미지에 접근하기 위한 registry 로그인 정보를 담고, Pod가 image pull을 할 때 사용한다.

## Device Plugin

Kubernetes에서 GPU 같은 특수 하드웨어를 Pod에 할당할 수 있게 해주는 확장 컴포넌트.

예를 들어 NVIDIA GPU를 Kubernetes Pod에서 쓰려면 보통 NVIDIA device plugin이 필요하다.

## Helm Chart

Kubernetes 앱을 설치하기 위한 리소스 묶음.

Deployment, Service, ConfigMap, Secret 같은 여러 YAML을 하나의 패키지처럼 관리한다.

```text
Chart = 설치 가능한 Kubernetes 앱 패키지
```

## Helm Release

Helm chart를 cluster에 실제로 설치한 결과물.

같은 chart라도 release 이름과 namespace를 다르게 해서 여러 번 설치할 수 있다.

```text
Chart = 패키지
Release = 그 패키지를 실제로 설치한 인스턴스
```

## Helm Repository

Helm chart를 저장하고 배포하는 저장소.

예를 들어 Prometheus Community chart 저장소를 추가하면 Helm이 `kube-prometheus-stack` chart를 검색하고 설치할 수 있다.

## Prometheus

metric을 수집하고 저장하는 모니터링 시스템.

Kubernetes 환경에서는 Node, Pod, Service, 애플리케이션 endpoint에서 metric을 주기적으로 가져와 저장한다.

## Grafana

metric을 dashboard로 시각화하는 도구.

Prometheus 같은 데이터 소스를 연결해 CPU, 메모리, 요청 수, 에러율 같은 지표를 그래프로 볼 수 있게 해준다.

## Prometheus Operator

Kubernetes에서 Prometheus를 Kubernetes 리소스처럼 관리하게 해주는 operator.

Prometheus, Alertmanager, ServiceMonitor 같은 custom resource를 보고 실제 모니터링 구성을 맞춘다.

## ServiceMonitor

Prometheus Operator가 사용하는 custom resource.

어떤 Service에서 metric을 수집할지 선언하는 리소스다.

```text
ServiceMonitor -> Prometheus가 이 Service를 scrape하게 해줘
```

## kube-state-metrics

Kubernetes 리소스 상태를 metric으로 노출하는 컴포넌트.

Deployment replica 수, Pod 상태, Node 상태 같은 Kubernetes 객체 정보를 Prometheus가 수집할 수 있게 해준다.

## node-exporter

Node의 CPU, 메모리, 디스크, 네트워크 같은 OS/하드웨어 metric을 노출하는 exporter.

Kubernetes에서는 보통 각 Node마다 하나씩 실행된다.

## Alertmanager

Prometheus alert을 받아서 묶고, 라우팅하고, 알림 채널로 보내는 컴포넌트.

예를 들어 특정 Pod가 계속 죽거나 Node 디스크가 부족할 때 Slack, 이메일, webhook 등으로 알릴 수 있다.

## CRD, Custom Resource Definition

Kubernetes에 새로운 리소스 종류를 추가하는 방법.

예를 들어 Prometheus Operator는 `Prometheus`, `ServiceMonitor`, `Alertmanager` 같은 리소스를 CRD로 추가해 Kubernetes API에서 다룰 수 있게 만든다.
