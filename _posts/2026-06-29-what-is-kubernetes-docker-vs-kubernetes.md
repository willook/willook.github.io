---
title: "쿠버네티스란 무엇일까? Docker와 Kubernetes의 차이"
description: "Docker, Docker Compose, Kubernetes, k3s의 차이를 처음 공부하는 관점에서 정리합니다."
date: 2026-06-29 09:00:00 +0900
categories: [Infra, Kubernetes]
tags: [kubernetes, docker, container, k8s, k3s]
---

## 한 줄 요약

Kubernetes는 컨테이너를 직접 만드는 기술이라기보다, 여러 서버 위에서 컨테이너들이 원하는 상태로 계속 떠 있도록 배치하고 운영해주는 시스템이다.

Docker가 "컨테이너 하나를 만들고 실행하는 도구"에 가깝다면, Kubernetes는 "많은 컨테이너를 여러 서버 위에서 안정적으로 운영하는 관리자"에 가깝다.

## 왜 헷갈리는가

처음에는 Kubernetes가 마치 "한 컴퓨터 안에 여러 개의 격리된 서버를 만들어주는 기술"처럼 느껴질 수 있다.

그 감각이 완전히 틀린 것은 아니다. 하지만 정확히는 격리 자체를 Kubernetes가 직접 하는 것이 아니다.

```text
Linux kernel
  namespace: 프로세스, 네트워크, 파일시스템 격리
  cgroup: CPU, RAM 같은 리소스 제한

container runtime
  containerd, Docker 등
  실제 컨테이너 실행

Kubernetes
  컨테이너를 어느 서버에 띄울지 결정
  죽으면 다시 띄움
  서비스 이름, 네트워크, 설정, 볼륨, 배포를 관리
```

즉 격리된 실행 공간은 컨테이너 런타임과 Linux 커널이 만들고, Kubernetes는 그 컨테이너들을 클러스터 단위로 운영한다.

## Docker란 무엇인가

Docker는 애플리케이션과 실행 환경을 이미지로 묶고, 그 이미지를 컨테이너로 실행하게 해주는 도구다.

예를 들어 FastAPI 앱을 Docker 이미지로 만들면 다음을 함께 담을 수 있다.

- Python 버전
- pip 패키지
- 실행 명령
- 앱 코드
- OS 레벨 의존성

Docker의 핵심 질문은 이것이다.

```text
이 앱을 어떤 환경에서 어떻게 실행할 것인가?
```

Docker Compose는 여기서 한 단계 더 나아가, 한 서버 안에서 여러 컨테이너를 같이 띄우는 도구다.

예를 들어:

```text
FastAPI
PostgreSQL
Redis
Grafana
Prometheus
```

이 정도 구성은 Docker Compose로도 충분히 관리할 수 있다.

## Kubernetes란 무엇인가

Kubernetes는 여러 서버를 하나의 클러스터로 보고, 그 위에 컨테이너 기반 애플리케이션을 배치하고 유지하는 시스템이다.

Kubernetes의 핵심 질문은 이것이다.

```text
이 클러스터 전체에서 원하는 서비스 상태를 어떻게 계속 유지할 것인가?
```

예를 들어 사용자가 이렇게 선언한다고 생각할 수 있다.

```text
FastAPI 모델 서버를 3개 띄워줘.
각 서버는 CPU 2개, RAM 4GB까지만 쓰게 해줘.
GPU가 있는 노드에만 배치해줘.
죽으면 자동으로 다시 띄워줘.
서비스 이름은 model-api로 고정해줘.
새 버전 배포 시 점진적으로 교체해줘.
```

Kubernetes는 이 원하는 상태를 계속 맞추려고 한다.

## Pod와 Container

Kubernetes에서 가장 작은 실행 단위는 Container가 아니라 Pod다.

보통은 Pod 하나 안에 컨테이너 하나가 들어간다.

```text
Pod
  app container
```

필요하면 여러 컨테이너를 같은 Pod 안에 넣을 수도 있다.

```text
Pod
  FastAPI model server
  log collector sidecar
```

같은 Pod 안의 컨테이너들은 다음을 공유한다.

- 같은 네트워크 IP
- 같은 localhost
- 같은 볼륨
- 비슷한 생명주기

즉 Kubernetes가 관리하는 단위는 Pod이고, 실제 실행되는 것은 컨테이너다.

## Kubernetes가 Docker 컨테이너를 띄우는가?

감각적으로는 "Pod 안에 Docker 컨테이너가 뜬다"고 말해도 어느 정도 맞다.

하지만 요즘 기준으로 더 정확히 말하면 다음과 같다.

```text
Kubernetes
  kubelet
    container runtime, 보통 containerd
      container
```

예전에는 Docker Engine을 런타임으로 많이 썼지만, 요즘 Kubernetes는 보통 containerd를 사용한다.

다만 Docker 이미지 형식은 여전히 그대로 많이 쓴다.

```text
nginx:latest
python:3.11
my-model-server:v1
```

이런 이미지를 Kubernetes에서도 그대로 실행할 수 있다.

## Docker Compose와 Kubernetes의 차이

| 구분 | Docker Compose | Kubernetes |
|---|---|---|
| 기본 단위 | 한 서버 | 여러 서버를 묶은 클러스터 |
| 실행 단위 | 컨테이너 | Pod |
| 목적 | 간단한 멀티 컨테이너 실행 | 컨테이너 운영 자동화 |
| 장애 대응 | 같은 서버에서 재시작 | 클러스터 상태를 보고 재배치 가능 |
| 배포 | 비교적 단순 | 롤링 업데이트, 롤백, 스케일링 |
| 네트워크 | 한 서버 중심 | 서비스 디스커버리, Ingress, LoadBalancer |
| 적합한 경우 | 개인 서버, 작은 서비스 | 여러 서비스, 여러 서버, 팀 단위 운영 |

감각적으로는 이렇게 볼 수 있다.

```text
Docker Compose = 이 서버에서 이 컨테이너들을 띄워줘.
Kubernetes = 이 클러스터 어딘가에서 이 상태가 항상 유지되게 해줘.
```

## 서버를 Docker로 관리할지 Kubernetes로 관리할지

실무에서는 정말로 이 선택을 자주 하게 된다.

작은 서비스라면 Docker Compose가 더 낫다.

- 설정이 단순하다.
- 학습 비용이 낮다.
- 장애 지점이 적다.
- 개인 서버나 작은 API 운영에 충분하다.

Kubernetes는 다음 상황에서 의미가 커진다.

- 서비스가 많다.
- 여러 서버나 GPU 노드를 묶어야 한다.
- 배포와 롤백을 체계적으로 하고 싶다.
- 서비스별 리소스 제한이 중요하다.
- Secret, Config, Volume, Ingress를 표준 방식으로 관리하고 싶다.
- 팀 여러 명이 같은 인프라를 쓴다.
- 나중에 EKS, GKE, AKS 같은 클라우드 Kubernetes로 옮길 가능성이 있다.

## MLE 관점에서 Kubernetes가 나오는 곳

Machine Learning Engineer 입장에서는 Kubernetes가 특히 이런 곳에서 자주 등장한다.

- 모델 서빙
- GPU 노드 관리
- batch job, training job 실행
- Triton, vLLM, Ray Serve 운영
- Kubeflow, Argo Workflows
- feature store, vector DB, inference service 운영
- Prometheus/Grafana 기반 모니터링
- 사내 ML platform 위에서 서비스 배포

즉 Kubernetes는 모델을 만드는 기술이라기보다, 모델과 관련 서비스를 안정적으로 운영하는 기술에 가깝다.

## k8s와 k3s

k8s는 Kubernetes의 약자다.

k3s는 Kubernetes를 가볍게 패키징한 배포판이다.

```text
k8s = Kubernetes 자체
k3s = 가벼운 Kubernetes 배포판
```

k3s도 진짜 Kubernetes다. `kubectl`, `Deployment`, `Service`, `Ingress`, `ConfigMap`, `Secret`, `PVC`, `Helm` 같은 개념을 그대로 쓴다.

개인 서버나 온프레미스 학습 환경에서는 k3s가 시작하기 좋다.

- 설치가 쉽다.
- 단일 서버에서도 잘 돈다.
- Kubernetes 핵심 개념을 그대로 배울 수 있다.
- 나중에 EKS/GKE/AKS로 넘어가기 쉽다.

## Cloud와 Managed Kubernetes

클라우드에서도 Kubernetes는 많이 쓴다.

클라우드에서 서버 자원을 빌리는 것과, 그 자원 위에서 많은 컨테이너를 안정적으로 운영하는 것은 다른 문제이기 때문이다.

대표적인 Managed Kubernetes는 다음과 같다.

```text
AWS  -> EKS
GCP  -> GKE
Azure -> AKS
```

Managed Kubernetes는 Kubernetes API와 생태계는 그대로 쓰되, control plane 관리 일부를 클라우드 회사에 맡기는 방식이다.

직접 관리해야 하는 것:

- 앱 배포
- worker node 크기와 개수
- GPU node pool
- Ingress, LoadBalancer
- StorageClass, PVC
- Secret, Config
- 비용과 권한

클라우드가 주로 관리해주는 것:

- Kubernetes control plane
- API server
- scheduler
- controller manager
- control plane 업그레이드와 가용성 일부

## On-prem Kubernetes를 클라우드에서 관리할 수 있을까?

가능하다. 다만 Temporal Cloud처럼 worker만 내 서버에서 돌고 control plane이 완전히 SaaS에 있는 형태와는 조금 다르다.

Kubernetes에서는 보통 클러스터 자체는 온프레미스에 두고, 관리 화면과 정책, GitOps, 모니터링을 클라우드나 중앙 관리 도구에 연결하는 식이다.

대표적인 선택지는 다음과 같다.

- Azure Arc-enabled Kubernetes
- Google Distributed Cloud / GKE Enterprise
- Amazon EKS Anywhere
- Rancher Manager

## 현재 이해 정리

Docker는 앱을 컨테이너로 포장하고 실행하는 도구다.

Docker Compose는 한 서버에서 여러 컨테이너를 편하게 띄우는 도구다.

Kubernetes는 여러 서버를 하나의 클러스터처럼 보고, 그 위에 Pod와 Service를 원하는 상태로 유지하는 운영 시스템이다.

따라서 처음 공부할 때는 이렇게 잡으면 된다.

```text
컨테이너를 이해하려면 Docker
작은 서버 운영은 Docker Compose
운영 자동화와 클러스터 감각은 Kubernetes
개인 서버 실습은 k3s
클라우드 실무 감각은 EKS/GKE/AKS 또는 hybrid management
```
