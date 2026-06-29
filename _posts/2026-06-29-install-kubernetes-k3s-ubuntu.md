---
title: "내 컴퓨터에 쿠버네티스 설치해보기: Ubuntu와 k3s"
description: "Ubuntu 서버에 k3s를 설치하고 kubectl, Deployment, Service, Ingress, 롤링 업데이트를 실습합니다."
date: 2026-06-29 09:20:00 +0900
categories: [Infra, Kubernetes]
tags: [kubernetes, k3s, ubuntu, kubectl, ingress, traefik]
---

## 목표

이 문서는 Ubuntu/Linux 온프레미스 서버에 k3s를 설치하고, Kubernetes의 기본 흐름을 직접 손으로 익히기 위한 실습 자료다.

이번 실습에서 할 일은 다음과 같다.

- 단일 노드 k3s 클러스터 설치
- `kubectl`로 클러스터 상태 확인
- nginx Deployment 배포
- Service로 Pod 앞에 고정 접근 지점 만들기
- k3s 기본 Traefik Ingress로 외부 HTTP 접근 확인
- Deployment 롤링 업데이트와 롤백 맛보기
- 실습 리소스 삭제와 k3s 제거 방법 확인

이번 문서에서는 Helm, Argo CD, Rancher는 설치하지 않고, k3s 기본 흐름만 다룬다.

## 큰 그림

Docker Compose에서는 보통 한 서버 안에서 이렇게 생각한다.

```text
이 서버에서 이 컨테이너들을 띄워줘.
```

Kubernetes에서는 이렇게 생각한다.

```text
이 클러스터에서 이 상태가 계속 유지되게 해줘.
```

예를 들어 nginx 컨테이너를 하나 띄우는 것도 Kubernetes에서는 몇 가지 개념으로 나뉜다.

```text
Deployment: nginx Pod를 몇 개 유지할지 관리
Pod: 실제 컨테이너가 들어가는 실행 단위
Service: Pod 앞에 고정된 네트워크 접근 지점 제공
Ingress: 외부 HTTP 요청을 Service로 라우팅
```

이번 실습은 이 흐름을 직접 확인하는 것이 목적이다.

## 공식 문서 기준

이 자료는 다음 공식 문서를 기준으로 작성했다.

- [k3s Quick Start](https://docs.k3s.io/quick-start)
- [k3s Requirements](https://docs.k3s.io/installation/requirements)
- [k3s Packaged Components](https://docs.k3s.io/installation/packaged-components)
- [k3s Uninstall](https://docs.k3s.io/installation/uninstall)
- [Kubernetes Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Kubernetes Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)

## 사전 요구사항

### 목적

k3s를 설치하기 전에 서버 상태를 확인한다. Kubernetes는 네트워크, 포트, container runtime, systemd 서비스와 엮이므로 설치 전 점검이 중요하다.

### 필요한 것

- Ubuntu 또는 Linux 서버
- `sudo` 권한
- 인터넷 연결
- SSH 또는 Tailscale SSH 접속
- 최소 수 GB 이상의 디스크 여유 공간
- 실습용으로 비워둘 수 있는 80/443 포트

### 서버 상태 확인

```bash
hostname    # 서버 이름 확인
uname -a    # 커널/OS 정보 확인
nproc       # CPU 코어 수 확인
free -h     # 메모리 여유 확인
df -h /     # 루트 디스크 여유 공간 확인
```

### 포트 확인

k3s 실습에서는 다음 포트가 중요하다.

```text
6443: Kubernetes API server
80: HTTP Ingress
443: HTTPS Ingress
```

현재 열려 있는 포트를 확인한다.

```bash
ss -tlnp | grep -E ':80|:443|:6443' || true  # HTTP/HTTPS/API Server 포트 사용 여부 확인
```

### 기대 결과

- `6443`은 아직 사용 중이지 않아야 한다.
- `80`, `443`을 nginx/apache/caddy 같은 기존 웹 서버가 쓰고 있다면 k3s 기본 Traefik Ingress와 충돌할 수 있다.

### 문제가 생기면 볼 것

기존 웹 서버가 80/443을 쓰고 있는지 확인한다.

```bash
systemctl is-active nginx apache2 caddy 2>/dev/null || true   # 현재 실행 중인지 확인
systemctl is-enabled nginx apache2 caddy 2>/dev/null || true  # 부팅 시 자동 실행되는지 확인
```

실습 서버에서 기존 nginx를 잠시 내려도 된다면 다음처럼 중지한다.

```bash
sudo systemctl disable --now nginx
```

Apache라면:

```bash
sudo systemctl disable --now apache2
```

Caddy라면:

```bash
sudo systemctl disable --now caddy
```

기존 웹 서버가 중요한 서비스를 담당하고 있다면 중지하지 말고, 이번 실습의 Ingress 부분은 건너뛰고 `port-forward` 방식으로만 확인한다.

## 방화벽 확인

### 목적

방화벽이 켜져 있으면 Kubernetes API server와 Pod/Service 네트워크 통신이 막힐 수 있다.

### 실행 명령

Ubuntu에서 UFW 상태를 확인한다.

```bash
sudo ufw status verbose
```

UFW를 유지한 채 k3s를 쓸 경우, k3s 공식 요구사항에 따라 API server와 Pod/Service CIDR을 허용한다.

```bash
sudo ufw allow 6443/tcp                    # Kubernetes API Server 포트 허용
sudo ufw allow from 10.42.0.0/16 to any    # k3s 기본 Pod 네트워크 대역 허용
sudo ufw allow from 10.43.0.0/16 to any    # k3s 기본 Service 네트워크 대역 허용
```

`10.42.0.0/16`은 Pod들이 받는 내부 IP 대역이고, `10.43.0.0/16`은 Service들이 받는 내부 가상 IP 대역이다.

### 기대 결과

- 외부에서 `kubectl`로 접근할 계획이 있다면 `6443/tcp` 접근이 가능해야 한다.
- 단일 노드에서 서버 내부 `kubectl`만 사용할 경우에도 Pod/Service CIDR은 막히지 않아야 한다.

### 문제가 생기면 볼 것

노드가 `Ready`가 되지 않거나 CoreDNS가 뜨지 않으면 방화벽, CNI, Pod 네트워크를 먼저 의심한다.

```bash
sudo ufw status verbose
kubectl get pods -A
```

## k3s 설치

### 목적

단일 노드 k3s 클러스터를 설치한다.

k3s는 가벼운 Kubernetes 배포판이다. 기본 설치만으로도 다음 컴포넌트를 포함한다.

- `containerd`: 컨테이너 런타임
- Flannel: Pod 네트워크
- CoreDNS: 클러스터 내부 DNS
- Traefik: 기본 Ingress Controller
- ServiceLB: LoadBalancer 타입 Service 지원
- local-path storage: 로컬 디스크 기반 기본 스토리지
- metrics-server: 리소스 메트릭 수집

### 실행 명령

기본 설치:

```bash
curl -sfL https://get.k3s.io | sh -  # 공식 설치 스크립트로 단일 노드 k3s 설치
```

설치가 끝나면 systemd 서비스가 만들어진다.

```bash
sudo systemctl status k3s --no-pager  # k3s systemd 서비스 상태 확인
```

### 기대 결과

`k3s` 서비스가 `active (running)` 상태여야 한다.

```text
Active: active (running)
```

### 문제가 생기면 볼 것

설치 로그와 서비스 로그를 확인한다.

```bash
sudo journalctl -u k3s -n 100 --no-pager  # 최근 k3s 서비스 로그 100줄 확인
```

설치 직후에는 컴포넌트가 올라오는 데 시간이 조금 걸릴 수 있다.

## kubectl 접근 확인

### 목적

설치된 k3s 클러스터를 `kubectl`로 조작할 수 있는지 확인한다.

k3s는 기본적으로 `kubectl`을 함께 제공한다. kubeconfig 파일은 다음 위치에 생성된다.

```text
/etc/rancher/k3s/k3s.yaml
```

### 실행 명령

설치 직후에는 k3s가 만든 관리자 kubeconfig로 클러스터 상태를 먼저 확인한다.

```bash
sudo k3s kubectl get nodes  # k3s에 포함된 kubectl로 노드 상태 확인
```

같은 확인을 kubeconfig 경로를 직접 지정해서 실행할 수도 있다.

```bash
sudo kubectl --kubeconfig /etc/rancher/k3s/k3s.yaml get nodes  # root 소유 관리자 kubeconfig를 직접 지정해 노드 확인
```

일상적인 실습 명령은 일반 사용자 계정에서 실행할 수 있도록 사용자 전용 kubeconfig를 만들고, `kubectl`이 그 파일을 보도록 설정한다.

```bash
mkdir -p ~/.kube                                      # kubectl 기본 설정 폴더 생성
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config      # k3s 관리자 kubeconfig를 사용자 홈으로 복사
sudo chown $USER:$USER ~/.kube/config                 # 현재 사용자가 읽을 수 있게 소유권 변경
chmod 600 ~/.kube/config                              # kubeconfig 권한을 사용자만 읽고 쓰게 제한
```

이 실습에서는 k3s 관리자 kubeconfig를 사용자 홈으로 복사하므로, 복사본도 여전히 클러스터 관리자 권한을 가진다. 실제 운영에서는 사용자나 역할별로 필요한 권한만 가진 kubeconfig를 따로 발급하는 것이 일반적이다.

현재 터미널에서 사용자 전용 kubeconfig를 사용하려면:

```bash
export KUBECONFIG=$HOME/.kube/config  # 현재 터미널에서 kubectl이 사용자 홈 kubeconfig를 보게 설정
```

`export`는 현재 터미널 세션에만 적용된다. 새 터미널에서도 계속 적용하려면 `.bashrc`에 추가한다.

```bash
echo 'export KUBECONFIG=$HOME/.kube/config' >> ~/.bashrc  # bash 시작 시 KUBECONFIG 자동 설정
source ~/.bashrc                                          # 현재 터미널에 .bashrc 설정 다시 적용
```

이후부터는 일반 사용자로 `kubectl`을 실행한다.

```bash
kubectl get nodes     # 클러스터에 등록된 노드 목록 확인
kubectl get pods -A   # 모든 namespace의 Pod 상태 확인
```

### 기대 결과

노드가 `Ready`로 보여야 한다.

```text
NAME      STATUS   ROLES                  AGE   VERSION
myserver  Ready    control-plane,master   ...   v...
```

전체 Pod 상태도 확인한다.

```bash
kubectl get pods -A
```

CoreDNS, Traefik, local-path-provisioner, metrics-server 등이 뜨는 것을 볼 수 있다.

### 문제가 생기면 볼 것

노드가 `NotReady`라면:

```bash
kubectl describe node
sudo journalctl -u k3s -n 200 --no-pager
```

Pod가 `ContainerCreating`, `CrashLoopBackOff`, `Pending`에 오래 머문다면:

```bash
kubectl get pods -A
kubectl describe pod <pod-name> -n <namespace>
```

## 첫 Deployment 만들기

### 목적

nginx 컨테이너를 Kubernetes Deployment로 띄운다.

Deployment는 "이 이미지를 사용하는 Pod를 몇 개 유지할지" 관리하는 리소스다.

### 실행 명령

```bash
kubectl create deployment webapp --image=nginx  # nginx 이미지로 webapp Deployment 생성
```

상태 확인:

```bash
kubectl get deployments  # Deployment 상태 확인
kubectl get pods         # Deployment가 만든 Pod 상태 확인
```

조금 더 넓게 확인:

```bash
kubectl get pods,svc,deployments  # Pod, Service, Deployment를 한 번에 확인
```

### 기대 결과

`webapp` Deployment가 생성되고, nginx Pod가 `Running` 상태가 된다.

```text
NAME                     READY   STATUS    RESTARTS   AGE
webapp-xxxxxxxxxx-yyyyy  1/1     Running   0          ...
```

### 문제가 생기면 볼 것

Pod 상세 상태:

```bash
kubectl describe pod <pod-name>
```

Pod 로그:

```bash
kubectl logs <pod-name>
```

이미지 다운로드 문제가 있으면 이벤트에 `ImagePullBackOff` 또는 `ErrImagePull`이 보일 수 있다.

## Service 만들기

### 목적

Pod는 죽고 다시 생성될 수 있어서 IP가 바뀔 수 있다. Service는 Pod 앞에 고정된 접근 지점을 만든다.

### 실행 명령

```bash
kubectl expose deployment webapp --port=80  # webapp Deployment 앞에 80번 Service 생성
```

상태 확인:

```bash
kubectl get service webapp       # webapp Service의 ClusterIP와 포트 확인
kubectl get pods,svc,deployments # 관련 리소스 전체 상태 확인
```

### 기대 결과

`webapp` Service가 생성된다.

```text
NAME     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
webapp   ClusterIP   10.43.x.x       <none>        80/TCP    ...
```

`ClusterIP`는 클러스터 내부에서만 접근 가능한 IP다.

### 문제가 생기면 볼 것

Service가 어떤 Pod를 바라보는지 확인한다.

```bash
kubectl describe service webapp
kubectl get endpoints webapp
```

Endpoint가 비어 있으면 Service selector와 Pod label이 맞지 않는 것이다.

## port-forward로 먼저 접속 확인

### 목적

Ingress를 만들기 전에 Service가 제대로 동작하는지 안전하게 확인한다.

`port-forward`는 로컬 포트를 Kubernetes Service나 Pod로 임시 연결한다.

### 실행 명령

서버 안에서 확인할 때:

```bash
kubectl port-forward service/webapp 8080:80  # 내 로컬 8080 포트를 webapp Service의 80번 포트로 임시 연결
```

다른 터미널에서:

```bash
curl http://127.0.0.1:8080  # port-forward가 연결한 로컬 주소로 nginx 응답 확인
```

Tailscale 또는 같은 네트워크의 다른 기기에서 임시로 접속하고 싶다면 다음처럼 바인딩 주소를 지정할 수 있다.

```bash
kubectl port-forward --address 0.0.0.0 service/webapp 8080:80  # 다른 기기에서도 8080으로 접근 가능하게 바인딩
```

그 다음 브라우저에서 접속한다.

```text
http://<server-ip>:8080
```

### 기대 결과

nginx 기본 HTML이 응답한다.

```text
Welcome to nginx!
```

### 문제가 생기면 볼 것

`port-forward`는 실행 중인 터미널이 열려 있는 동안만 유지된다. 터미널을 닫으면 연결도 끊긴다.

포트가 이미 사용 중이면 다른 포트를 쓴다.

```bash
kubectl port-forward service/webapp 18080:80
```

### 정리 명령

`Ctrl-C`로 `port-forward`를 종료한다.

## Ingress로 외부 HTTP 접근 만들기

### 목적

k3s 기본 Traefik Ingress Controller를 사용해 외부 HTTP 요청을 `webapp` Service로 보낸다.

Ingress는 외부 HTTP/HTTPS 요청을 클러스터 내부 Service로 라우팅하는 리소스다.

### 실행 명령

Ingress YAML을 만든다.

```bash
cat <<'EOF' > webapp-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webapp
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: webapp
            port:
              number: 80
EOF
```

적용한다.

```bash
kubectl apply -f webapp-ingress.yaml  # YAML에 적은 Ingress 리소스를 클러스터에 적용
```

상태 확인:

```bash
kubectl get ingress             # 생성된 Ingress 목록 확인
kubectl describe ingress webapp # webapp Ingress의 라우팅/이벤트 확인
```

브라우저나 curl로 확인한다.

```bash
curl http://<server-ip>/
```

Tailscale을 사용한다면:

```bash
curl http://<tailscale-ip>/
```

또는 브라우저에서:

```text
http://<server-ip>/
http://<tailscale-ip>/
```

### 기대 결과

nginx 기본 페이지가 보인다.

```text
Welcome to nginx!
```

### 문제가 생기면 볼 것

80 포트가 기존 nginx/apache/caddy에 잡혀 있으면 Traefik이 외부 요청을 받지 못할 수 있다.

```bash
ss -tlnp | grep -E ':80|:443'
kubectl get pods -A | grep -i traefik
kubectl logs -n kube-system deploy/traefik
```

Ingress 자체 상태도 확인한다.

```bash
kubectl describe ingress webapp
```

만약 외부 노출을 피하고 싶다면 Ingress 실습은 건너뛰고 `port-forward`만 사용한다.

### 정리 명령

Ingress만 삭제하려면:

```bash
kubectl delete ingress webapp
rm webapp-ingress.yaml
```

## Replica 수 변경하기

### 목적

Kubernetes가 원하는 Pod 개수를 유지하는 방식을 확인한다.

### 실행 명령

nginx Pod를 3개로 늘린다.

```bash
kubectl scale deployment webapp --replicas=3  # webapp Pod를 3개로 유지하도록 변경
```

상태 확인:

```bash
kubectl get deployment webapp # webapp Deployment의 replica 상태 확인
kubectl get pods -o wide      # Pod가 어느 노드에서 도는지까지 확인
```

다시 1개로 줄인다.

```bash
kubectl scale deployment webapp --replicas=1  # webapp Pod 수를 다시 1개로 축소
```

### 기대 결과

replica 수를 3으로 늘리면 Pod가 3개가 되고, 1로 줄이면 Pod가 1개만 남는다.

Kubernetes는 사용자가 선언한 replica 수를 맞추려고 한다.

### 문제가 생기면 볼 것

Pod가 `Pending`이면 노드 리소스 부족이나 스케줄링 문제일 수 있다.

```bash
kubectl describe pod <pod-name>
kubectl describe node
```

## 롤링 업데이트 실습

### 목적

서버에 직접 들어가 컨테이너를 하나씩 바꾸지 않고, Kubernetes Deployment를 통해 앱 버전을 교체하는 흐름을 확인한다.

### 실행 명령

현재 이미지 확인:

```bash
kubectl describe deployment webapp | grep Image
```

이미지를 특정 버전으로 바꾼다.

```bash
kubectl set image deployment/webapp nginx=nginx:1.27  # Deployment의 nginx 컨테이너 이미지를 새 버전으로 변경
```

롤아웃 상태를 본다.

```bash
kubectl rollout status deployment/webapp  # 롤링 업데이트가 완료될 때까지 상태 확인
```

Pod 상태를 확인한다.

```bash
kubectl get pods
kubectl describe deployment webapp | grep Image
```

### 기대 결과

Kubernetes가 기존 Pod를 새 이미지의 Pod로 점진적으로 교체한다.

중요한 점은 서버에 직접 SSH로 들어가 `docker stop`, `docker run`을 반복하지 않는다는 것이다. Deployment의 원하는 상태를 바꾸면 Kubernetes가 실제 상태를 맞춘다.

### 문제가 생기면 볼 것

롤아웃 기록과 상태를 확인한다.

```bash
kubectl rollout history deployment/webapp
kubectl rollout status deployment/webapp
kubectl describe deployment webapp
```

## 롤백 실습

### 목적

잘못된 배포를 이전 상태로 되돌리는 흐름을 확인한다.

### 실행 명령

일부러 존재하지 않는 이미지로 바꿔본다.

```bash
kubectl set image deployment/webapp nginx=nginx:not-a-real-tag  # 일부러 없는 이미지 태그를 지정해 실패 상황 만들기
```

상태 확인:

```bash
kubectl rollout status deployment/webapp --timeout=30s  # 30초 동안 롤아웃 성공 여부 확인
kubectl get pods                                      # 실패한 새 Pod 상태 확인
```

실패 상태를 확인한다.

```bash
kubectl describe deployment webapp
kubectl describe pod <문제있는-pod-name>
```

이전 정상 버전으로 롤백한다.

```bash
kubectl rollout undo deployment/webapp  # Deployment를 직전 정상 버전으로 되돌림
```

롤백 상태 확인:

```bash
kubectl rollout status deployment/webapp  # 롤백이 완료될 때까지 상태 확인
kubectl get pods                          # 정상 Pod가 다시 떠 있는지 확인
```

### 기대 결과

잘못된 이미지는 `ImagePullBackOff` 같은 상태를 만들 수 있다. `rollout undo`를 실행하면 이전 정상 이미지로 되돌아간다.

### 문제가 생기면 볼 것

롤아웃 히스토리를 확인한다.

```bash
kubectl rollout history deployment/webapp
```

Deployment 상세 확인:

```bash
kubectl describe deployment webapp
```

## 실습 리소스 삭제

### 목적

실습으로 만든 Kubernetes 리소스를 정리한다.

### 실행 명령

Ingress, Service, Deployment를 삭제한다.

```bash
kubectl delete ingress webapp --ignore-not-found      # webapp Ingress 삭제, 없어도 에러 무시
kubectl delete service webapp --ignore-not-found      # webapp Service 삭제, 없어도 에러 무시
kubectl delete deployment webapp --ignore-not-found   # webapp Deployment와 그 Pod 삭제, 없어도 에러 무시
```

확인:

```bash
kubectl get pods,svc,deployments,ingress
```

### 기대 결과

`webapp` 관련 리소스가 모두 사라진다.

기본 `kubernetes` Service는 남아 있을 수 있다. 이것은 정상이다.

```text
service/kubernetes
```

### 문제가 생기면 볼 것

삭제가 오래 걸리면 리소스 이름과 namespace를 확인한다.

```bash
kubectl get all
kubectl get ingress
```

## k3s 완전 제거

### 목적

실습 후 서버를 원래 상태에 가깝게 되돌린다.

주의: 이 명령은 k3s와 로컬 클러스터 데이터를 삭제한다. 실습이 끝났고 보존할 Kubernetes 리소스가 없을 때만 실행한다.

### 실행 명령

server node에서 k3s를 제거한다.

```bash
sudo /usr/local/bin/k3s-uninstall.sh  # k3s server와 로컬 클러스터 데이터 제거
```

제거 후 확인:

```bash
systemctl status k3s --no-pager || true
kubectl get nodes || true
```

### 기대 결과

`k3s` 서비스가 사라지고, `kubectl`은 더 이상 클러스터에 연결되지 않는다.

### 문제가 생기면 볼 것

k3s 제거 후 남은 프로세스나 포트를 확인한다.

```bash
ps aux | grep k3s | grep -v grep || true
ss -tlnp | grep 6443 || true
```

## Worker Node 확장 맛보기

### 목적

단일 서버 실습 이후, 여러 서버를 하나의 Kubernetes 클러스터로 묶는 감각을 이해한다.

이번 문서에서는 실제 worker node 추가를 필수로 하지 않고 개념과 명령 형태만 소개한다.

### Control Plane 서버에서 토큰 확인

```bash
sudo cat /var/lib/rancher/k3s/server/node-token  # worker node가 join할 때 사용할 클러스터 토큰 확인
```

### Worker 서버에서 join

다른 서버에서 다음 형식으로 실행한다.

```bash
curl -sfL https://get.k3s.io | K3S_URL=https://<control-plane-ip>:6443 K3S_TOKEN=<node-token> sh -  # worker node를 기존 k3s 클러스터에 join
```

### 확인

Control plane 서버에서:

```bash
kubectl get nodes
```

### 기대 결과

여러 노드가 하나의 클러스터에 보인다.

```text
NAME       STATUS   ROLES
server-1   Ready    control-plane,master
worker-1   Ready    <none>
```

### Worker 제거

worker node에서:

```bash
sudo /usr/local/bin/k3s-agent-uninstall.sh  # worker node에서 k3s agent 제거
```

## 자주 겪는 문제

### 80/443 포트 충돌

증상:

- Ingress로 접속했는데 nginx 실습 페이지가 안 보인다.
- 기존 웹 서버 페이지가 계속 보인다.
- Traefik 관련 Pod는 떠 있는데 외부 접근이 안 된다.

확인:

```bash
ss -tlnp | grep -E ':80|:443'
kubectl get pods -A | grep -i traefik
```

해결:

- 기존 nginx/apache/caddy를 중지한다.
- 또는 Ingress 실습을 건너뛰고 `port-forward`만 사용한다.

### kubectl 권한 문제

증상:

```text
permission denied: /etc/rancher/k3s/k3s.yaml
```

원인:

- `/etc/rancher/k3s/k3s.yaml`은 root 소유의 관리자 kubeconfig다.
- k3s에 포함된 `kubectl`은 기본값으로 이 파일을 보려고 할 수 있다.
- 일반 사용자로 실행하면 해당 파일을 읽지 못해 권한 에러가 난다.

해결:

사용자 전용 kubeconfig를 만들고, 현재 터미널에서 그 경로를 사용하도록 설정한다.

```bash
mkdir -p ~/.kube                                      # kubectl 기본 설정 폴더 생성
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config      # k3s 관리자 kubeconfig를 사용자 홈으로 복사
sudo chown $USER:$USER ~/.kube/config                 # 현재 사용자가 읽을 수 있게 소유권 변경
chmod 600 ~/.kube/config                              # kubeconfig 권한을 사용자만 읽고 쓰게 제한
export KUBECONFIG=$HOME/.kube/config                  # 현재 터미널에서 사용자 홈 kubeconfig 사용
```

새 터미널에서도 계속 적용하려면 한 번만 추가한다.

```bash
echo 'export KUBECONFIG=$HOME/.kube/config' >> ~/.bashrc  # bash 시작 시 자동 설정
source ~/.bashrc                                          # 현재 터미널에 바로 반영
```

### Pod가 Running이 되지 않음

확인:

```bash
kubectl get pods -A
kubectl describe pod <pod-name> -n <namespace>
```

주로 볼 것:

- image pull 실패
- 네트워크 문제
- 리소스 부족
- 방화벽 문제

## 실습 후 이해해야 할 것

이 실습을 끝내면 다음 질문에 답할 수 있어야 한다.

- k3s는 무엇을 설치하는가?
- `kubectl get nodes`와 `kubectl get pods -A`는 무엇을 보여주는가?
- Deployment와 Pod는 어떤 관계인가?
- Service는 왜 필요한가?
- Ingress는 Service와 무엇이 다른가?
- 서버마다 직접 컨테이너를 업데이트하지 않고 Deployment를 업데이트한다는 것이 무슨 뜻인가?
- 롤링 업데이트와 롤백은 어떤 문제를 줄여주는가?
