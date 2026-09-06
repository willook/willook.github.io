---
title: "딥러닝 모델 최적화 이론 — 2. Model Pruning"
description: "비정형·구조적 pruning부터 magnitude, activation, gradient·sensitivity 기준과 2:4 sparsity, one-shot·iterative 전략까지 정리합니다."
date: 2026-09-06 00:00:00 +0900
categories: [Machine Learning, Model Optimization]
tags: [pruning, sparsity, model-compression, model-optimization, tensorrt]
math: true
toc: false
---

## 1. pruning이란

pruning이라는 딥러닝 모델 최적화 이론에 대해 설명하고자 한다. pruning이란 가지치기를 의미하며 최적화 이론에서도 가지치기하듯 중요도가 낮은 부분을 제거하여 모델을 최적화하는 방법이다. 일반적으로 딥러닝에서 한 개 레이어는 다음과 같이 표현된다.

$$
Y = W \cdot X + b
$$

- $Y$: 출력
- $W$: 가중치
- $X$: 입력
- $b$: bias

마스킹 기반 pruning을 예로 들어보면 pruning은 다음과 같이 표현할 수 있다.

$$
W' = M \odot W,\qquad M_i \in \{0,1\}
$$

- $W'$: pruning된 가중치
- $M$: 마스크

참고로 $\odot$은 원소별 곱을 뜻하는 수학 기호이다.

만약 $M$의 값이 0인 위치에서는 대응하는 가중치가 0이 되어 추론할 때 기여하지 않게 된다. 그리고 이때 전체 원소 중 0인 비율을 sparsity(희소도), 0이 아닌 비율을 density(밀도)라고 한다.

## 2. pruning의 종류

pruning은 크게 비정형 pruning과 구조적 pruning으로 나뉜다.

### 2.1. 비정형 pruning

비정형 pruning은 앞서 소개한 마스킹 기반 pruning과 같이 개별적인 가중치 단위로 제거한다.

가중치를 독립적으로 제거한다. 예로 절대값이 일정 이하인 가중치를 0으로 만들거나, 다른 가중치와 연결이 거의 없는 가중치를 0으로 만들 수 있다. 다만 비정형 pruning 과정을 거쳐도 논리적인 tensor의 shape은 바뀌지 않는다.

비정형 pruning의 장점은 제거 단위가 매우 작기 때문에 영향력이 작은 가중치 위주로 제거할 수 있다는 점이다. 따라서 정확도 보존에 유리하다. 하지만 가중치 전체에서 건너뛸 가중치를 저장할 형식과 연산 커널이 없다면 속도 개선이 어려울 수 있다.

### 2.2. 구조적 pruning

구조적 pruning은 채널, 필터, attention head, block과 같이 구조 단위로 제거한다.

예로 다음과 같은 Conv2d가 있다고 하자.

$$
W \in \mathbb{R}^{C_{\mathrm{out}}\times C_{\mathrm{in}}\times k_H\times k_W}
$$

- $C_{out}$: 출력 채널 크기
- $C_{in}$: 입력 채널 크기
- $k_H$: 커널의 높이
- $k_W$: 커널의 너비

출력 채널을 줄인다면, $C_{out}$을 줄이고 이는 다음 레이어의 입력 크기가 줄어드는 것이기 때문에 다음 레이어의 입력도 조정해줘야한다. 만약 다음 레이어도 Conv2d라면 다음 레이어의 $C_{in}$ 또한 동일한 크기로 줄여야한다. 출력 채널을 줄인 레이어가 residual, concatenation으로 연결되어 있다면 연결된 tensor의 채널 크기도 동일하게 맞춰야한다.

구조적 pruning은 모델 구조 자체를 축소하므로 별도의 희소 저장 형식이나 연산 커널 없이도 성능 개선으로 연결하기 쉽다. 하지만 한 번에 비교적 많은 가중치를 제거하기 때문에 성능에 영향을 주기 쉽고, 구조적 제약이 있는 모델에서는 적용이 다소 어려울 수 있다는 점이 있다.

## 3. 가중치 제거 방법

pruning의 핵심은 중요도 점수(importance score)다. 낮은 점수를 가진 가중치나 구조를 제거한다. 중요도 점수를 어떻게 매기는가에 따라 작은 가중치를 없애거나, 결과에 영향을 적게 주는 가중치를 제거하거나, 정답을 추론하는 데 영향이 적은 가중치를 제거할 수 있다.

### 3.1. Magnitude 기준

Magnitude 기준 중요도 점수는 모델의 가중치의 크기를 중요도 점수로 측정하는 방법이다.

비정형 pruning에서는 가중치의 개별 값의 절댓값이 그대로 중요도 점수가 된다.

$$
s_i = |w_i|
$$

반면 구조적 pruning에서는 구조 단위로 제거하기 때문에 마찬가지로 구조 단위로 중요도 점수를 측정한다. 따라서 특정 가중치 구조의 Lp norm을 사용한다(보통 L1, L2 norm).

$$
s_c = \lVert W_c \rVert_p
$$

Magnitude 기반 pruning의 간략한 순서는 다음과 같다.

```text
1. 목표 희소도 비율 q를 결정
2. 단위별 magnitude 기반 중요도 점수 계산
3. 점수가 낮은 하위 q%를 제거
```

예로 다음과 같은 가중치를 갖는 모델에서 25% 비율로 magnitude 기반 비정형 pruning을 한다고 가정해보자.

```text
w = [0.9, -0.15, 0.3, -0.6]
```

이때 $w$의 중요도 점수는 다음과 같다.

```text
s = [0.9, 0.15, 0.3, 0.6]
```

이 중 하위 25%는 두 번째 값인 0.15이므로 적용되는 가중치는 다음과 같아진다.

```text
w' = [0.9, 0, 0.3, -0.6]
```

구조적 pruning의 경우도 크게 다르지 않다. 다음과 같이 4개의 출력 채널을 갖는 convolution 레이어가 있다고 하자.

```text
Convolution
   ├─ 채널 0
   ├─ 채널 1
   ├─ 채널 2
   └─ 채널 3
```

각 채널의 L2 norm 값을 측정해본 결과가 다음과 같다고 하자.

```text
채널 0의 L2 norm: 0.82
채널 1의 L2 norm: 0.47
채널 2의 L2 norm: 0.01  ← 해당 채널의 중요도 점수가 제일 낮음
채널 3의 L2 norm: 0.65
```

이 경우 하위 25% 채널은 채널 2이므로 해당 채널을 지우게 된다. 이때 출력 2번 출력 채널을 지울 경우 대응하는 다음 레이어의 2번 입력 채널도 함께 지워야한다.

#### 3.1.1. Magnitude 기반 2:4 Sparsity

N:M 희소 패턴은 인접한 M개의 가중치 중 N개의 가중치만 남기는 것을 의미한다. 일반적으로 사용하는 2:4 희소 패턴의 경우 인접한 4개의 가중치 중 2개만 사용하고 나머지는 비활성화한다. 예로 다음과 같은 가중치와 대응하는 중요도 점수가 있다고 하자.

```text
w = [0.9, -0.15, 0.3, -0.6], s = [0.9, 0.15, 0.3, 0.6]
```

해당 가중치에서 2:4 희소 패턴을 적용하면 다음과 같이 된다.

```text
w' = [0.9, 0, 0, -0.6]
```

2:4 패턴에서 중요한 점은 반드시 2개 이상의 가중치가 비활성화된다는 것이다. 따라서 이때 비활성화 패턴은 ${}_4C_2$으로 총 6개로 표현 가능하기 때문에 어느 위치가 살아 있는지를 작은 metadata로 표현할 수 있다.

즉 흥미롭게도 이 방법은 모델의 수학적 최적값이라서 2:4를 선택한 것이 아니라, 50%의 연산을 제거하면서도 위치 정보를 작고 규칙적으로 표현할 수 있도록 하드웨어가 지원하는 형식으로 선택하는 것이다. TensorRT에서는 가중치가 2:4 희소 패턴을 만족할 경우, 지원되는 GPU와 연산에서 희소 연산 사용을 활성화할 수 있다.

PyTorch의 BERT 예제에서는 F1이 dense `86.92`에서 sparse `86.48`로 소폭 감소했다고 한다.

### 3.2. Activation 기준

Activation 기준 중요도 점수는 모델의 가중치가 결과에 반영되는 영향의 크기를 중요도 점수로 사용한다. 구체적으로는 특정 채널이나 neuron의 activation 크기를 본다.

Convolution 레이어에서 activation 기준 중요도 점수 $s_c$는 다음과 같이 계산된다.

$$
A_c = \phi\left(\operatorname{Conv}(X, W_c) + b_c\right)
$$

$$
s_c = \operatorname{mean}\left(\lvert A_c \rvert\right)
$$

- $X$: 입력 feature map
- $W_c$: 출력 채널 $c$를 만드는 convolution filter
- $b_c$: 출력 채널 $c$의 bias
- $\phi$: ReLU 같은 activation 함수
- $A_c$: 최종 activation feature map
- $s_c$: Activation 기준 중요도 점수

가중치의 크기가 작더라도 들어오는 입력값이 큰 경우 이 가중치는 모델에 큰 영향을 줄 수 있고, 반대로 해당 가중치에 들어오는 입력값이 매우 작다면 가중치가 크더라도 중요하지 않다고 보는 것이다.

실제 데이터 분포에서 거의 반응하지 않는 단위를 찾을 수 있지만, calibration dataset이 필요하고, 또 데이터 편향의 위험이 있을 수 있다. 예로 calibration dataset에 특정 class나 특정 케이스가 빠져 있으면, 특정 상황에서만 활성화되는 채널을 불필요하다고 오판할 수 있다.

직관적인 예시로 입력 이미지와 이를 입력받는 convolution 레이어가 있다고 하자. 그리고 목표 희소 비율은 25%라고 하자.

```text
입력 이미지
   ↓
Convolution
   ├─ 채널 0의 feature map
   ├─ 채널 1의 feature map
   ├─ 채널 2의 feature map
   └─ 채널 3의 feature map
```

각 feature map의 절대값 평균이 다음과 같다고 하자.

```text
채널 0: 0.82
채널 1: 0.47
채널 2: 0.01  ← 거의 반응하지 않음(중요도 점수가 가장 낮음)
채널 3: 0.65
```

이 경우 채널 2를 지우고 마찬가지로 이에 대응하는 다음 레이어의 채널을 지운다.

### 3.3 Gradient·sensitivity 기준

Gradient·sensitivity 기준의 중요도 점수는 가중치를 제거했을 때 loss가 얼마나 변할지를 본다.

magnitude나 activation 기준보다 민감도를 직접 반영할 수 있지만 계산 비용과 구현 복잡도가 커지며 라벨을 포함한 calibration dataset이 필요하다.

Gradient·sensitivity 기준은 가중치나 채널을 제거했을 때 loss가 얼마나 변할지를 추정한다. 구체적으로는 특정 가중치를 0으로 만들 때의 loss 변화는 1차 Taylor 전개를 이용해 근사(gradient)할 수 있으며 더 나아가 2차 근사 등을 통해 곡률까지 반영할 수도 있다.

참고로 Sensitivity는 gradient보다 더 넓은 개념이다.

- **실제 제거 실험:** 후보를 하나씩 0으로 만들고 실제 loss 변화를 측정
- **1차 근사:** gradient를 이용해 loss 변화를 빠르게 추정
- **2차 근사:** Hessian이나 Fisher information까지 이용해 곡률을 반영

## 4. pruning 전략: One-shot과 Iterative

### 4.1. One-shot pruning

한 번에 목표한 희소도까지 제거한다. 이후 필요하다면 fine-tuning을 통해 손상된 성능을 회복시킨다. 빠르고 단순하지만, 높은 희소도를 목표하거나, 민감한 모델(regression)의 경우 성능 회복이 어려울 수 있다.

### 4.2. Iterative pruning

한 번에 목표한 희소도까지 가중치를 제거하지 않고, 조금 제거하고 fine-tuning을 통해 회복하는 과정을 반복하여 목표한 희소도에 도달한다.

1. 목표 희소도 $q$를 설정한다.
   1. 가중치의 중요도를 계산한다.
   2. 일부 가중치(비정형 pruning) 또는 구조(구조적 pruning)을 제거한다.
   3. 남은 가중치를 fine-tuning 한다.
   4. 목표 희소도 $q$에 도달하지 못했다면 위 a~d의 과정을 반복한다.

---

## 참고 자료

- [PyTorch — Pruning Tutorial](https://docs.pytorch.org/tutorials/intermediate/pruning_tutorial.html)
- [PyTorch — Accelerating neural network training with semi-structured 2:4 sparsity](https://docs.pytorch.org/tutorials/advanced/semi_structured_sparse.html)
- [NVIDIA TensorRT — Data formats and tensors: sparsity](https://docs.nvidia.com/deeplearning/tensorrt/latest/inference-library/data-formats-tensors.html)
