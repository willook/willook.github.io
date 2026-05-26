---
title: "Ubuntu에서 NVIDIA 드라이버 · CUDA Toolkit 버전 최신화하기"
date: 2026-05-26 14:30:00 +0900
categories: [Infra, GPU]
tags: [nvidia, cuda, ubuntu, gpu, driver, kernel]
---

GPU 서버를 운영하다 보면 신형 GPU 장착 또는 라이브러리 업데이트와 같은 이유로 NVIDIA 드라이버와 CUDA Toolkit을 최신화해야하는 경우가 종종 있습니다.
도커 컨테이너를 사용하는 경우 도커 CUDA 런타임을 사용하기 때문에 CUDA toolkit을 업데이트 해줄 필요는 없는데요, 그럼에도 **NVIDIA 드라이버만큼은 호스트 OS의 것을 그대로 사용**합니다. 따라서 도커 컨테이너를 사용하더라도 경우 호스트 OS의 NVIDIA 드라이버를 업데이트 해줘야합니다.
참고로 호스트 OS란 도커를 설치하고 컨테이너를 띄우는 OS를 의미합니다.
이 글에서는 Ubuntu 환경에서 NVIDIA 드라이버와 CUDA Toolkit을 업데이트하면서 겪었던 여러 시행착오를 토대로 업데이트 하는 과정을 정리했습니다.

## 1. 현재 설치된 버전 확인

드라이버 버전 확인:

```shell
nvidia-smi
```
만약 위 명령어가 없다면 NVIDIA 드라이버가 설치되어있지 않거나 전역으로 명령어를 사용할수 있도록 설정되어있지 않는 것 입니다.

CUDA Toolkit 버전 확인:

```shell
nvcc --version
```

> `nvidia-smi`에 표시되는 CUDA 버전은 **현재 드라이버가 호환 가능한 가장 최신 버전**이지, 실제로 설치된 CUDA Toolkit 버전이 아닙니다. 실제 설치 버전은 반드시 `nvcc --version`으로 확인하세요.
{: .prompt-warning }

## 2. 기존 드라이버 · Toolkit 제거

CUDA Toolkit 제거:

```shell
sudo apt remove --autoremove nvidia-cuda-toolkit
sudo apt autoremove
sudo apt clean
```

NVIDIA 드라이버 제거:

```shell
sudo apt remove --purge '^nvidia-.*'
sudo apt autoremove
sudo apt autoclean
sudo reboot
```

드라이버를 제거하는 과정에서 CUDA Toolkit도 함께 제거됩니다. 재부팅 후 아래 명령으로 잔여 패키지가 없는지 확인하세요 (아무것도 출력되지 않아야 정상):

```shell
dpkg -l | grep nvidia
dpkg -l | grep cuda
```

## 3. 호환 버전 확인

설치하기 전에 **드라이버 / CUDA / cuDNN / 커널**의 호환 조합을 반드시 확인해야 합니다. NVIDIA가 제공하는 Support Matrix가 가장 정확했습니다.

- [Support Matrix — NVIDIA cuDNN](https://docs.nvidia.com/deeplearning/cudnn/latest/reference/support-matrix.html)

특히 **최신 드라이버는 최신 커널을 요구**하는 경우가 많으므로, 커널 호환 여부를 먼저 점검하세요 (아래 7번 항목 참고).
**주의**: 호환되지 않는 커널의 NVIDIA 드라이버를 설치를 시도하면 드라이버 설치 과정에서 커널을 자동으로 업데이트 하도록 시도합니다. 이때 높은 확률로 커널이 꺠지고 서버가 부팅 되지 않을 수 있으니 반드시 원하는 NVIDIA 드라이버에 맞는 커널 버전(ubuntu-drivers devices 명령어로 조회 가능)을 먼저 확인해주세요. 커널 업데이트가 필요한 경우 **## 7. 커널 업데이트 (HWE 커널)**을 먼저 참고해주세요. 


## 4. NVIDIA 드라이버 설치

현재 GPU에 사용 가능한 드라이버 목록 확인:

```shell
ubuntu-drivers devices
```

설치:

```shell
sudo apt update

# 자동으로 적절한 드라이버 버전 선택 후 설치 (ubuntu-drivers devices에서 recommaneded로 뜬 버젼으로 설치됨)
sudo ubuntu-drivers autoinstall
```

특정 버전을 지정하여 설치하는 방법도 있으나, 제 경우에는 버전을 지정해서 설치하면 드라이버가 정상적으로 설치되지 않았습니다.

**주의**: 드라이버 설치 완료 후 reboot까지 해야 드라이버가 정상적으로 동작합니다.
```shell
sudo reboot
```

재부팅 후 정상 동작 확인:

```shell
nvidia-smi
```

## 5. CUDA Toolkit 설치

```shell
# 빌드용 gcc 설치
sudo apt install gcc -y

# CUDA Toolkit 설치
sudo apt install nvidia-cuda-toolkit

sudo reboot

# 설치 확인
nvcc -V
```

## 6. (옵션) Docker에서 NVIDIA 런타임 사용하기

도커 컨테이너에서 NVIDIA GPU를 쓰려면 `nvidia-container-toolkit`이 필요합니다.

```shell
# 1. nvidia-container-toolkit 설치 (이미 설치돼 있어도 재실행 안전)
sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit

# 2. 도커 런타임 설정 (자동 구성)
sudo nvidia-ctk runtime configure --runtime=docker

# 3. 도커 재시작
sudo systemctl restart docker
```

## 7. 커널 업데이트 (HWE 커널)

최신 드라이버가 현재 커널을 지원하지 않으면 커널부터 올려야 합니다.

현재 커널 확인:

```shell
uname -r
```

OS 버전 확인:

```shell
lsb_release -a
```

OS 버전에 맞는 HWE(Hardware Enablement) 커널 메타패키지는 다음과 같습니다:

| OS 버전 | HWE 커널 메타패키지 |
|---|---|
| Ubuntu 20.04 | `linux-generic-hwe-20.04` |
| Ubuntu 22.04 | `linux-generic-hwe-22.04` |
| Ubuntu 24.04 | `linux-generic-hwe-24.04` |

설치 (Ubuntu 24.04 예시):

```shell
sudo apt update
sudo apt install --install-recommends linux-generic-hwe-24.04
```

참고: [Ubuntu 커널 라이프사이클 가이드](https://ubuntu.com/kernel/lifecycle)
