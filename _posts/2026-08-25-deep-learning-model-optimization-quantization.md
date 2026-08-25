---
title: "딥러닝 모델 최적화 이론 — 1. Quantization"
description: "Float32·Float16·Float8과 INT8의 표현 방식부터 Affine Quantization, PTQ, QAT, Fake Quantization과 STE까지 정리합니다."
date: 2026-08-25 00:00:00 +0900
categories: [Machine Learning, Model Optimization]
tags: [quantization, ptq, qat, mixed-precision, model-optimization]
math: true
---

## 1. 들어가기에 앞서

딥러닝 모델 추론은 다른 방식에 비해 비용이 매우 크고 이를 최적화하기 위한 많은 기법들이 있다. 실무에서는 Quantization, Graph Optimization을 자주 접하게 된다. 주로 모델을 배포할 때 최적화를 위해 PyTorch, TensorFlow 모델을 그대로 배포하기보다는 ONNX나 TensorRT와 같은 모델로 변환하여 배포하게 된다. 이렇게 모델을 ONNX, Plan(TensorRT로 빌드)로 변환하는 과정에 Graph Optimization이 적용된다. 또한 이때 Float16이나 Int8과 같은 낮은 정밀도를 적용하는 옵션인 Quantization도 함께 사용할 수 있다. 이외에도 다양한 기법들이 있어서 이번 글에서는 대표적인 최적화 기법들의 개념에 대해 하나씩 정리해 보고자 한다. 이번 포스팅에서 Quantization에 대해 정리한다.

## 2. Quantization

Quantization(양자화)는 ONNX, TensorRT와 같이 모델을 변환하다 보면 자주 접하게 되는 개념이다. 일반적인 딥러닝 프레임워크(PyTorch, TensorFlow)를 통해 모델을 학습하는 경우 대부분 모델의 파라미터가 Float32(모델의 정밀도를 Float32로 사용한다고 표현함)를 사용하여 저장하게 된다. 양자화는 여기서 파라미터의 일부 소수점을 버림으로 모델의 크기를 줄이고 연산 속도와 메모리 효율을 개선한다.

### 2.1. Quantization 정밀도

정밀도는 보통 Float16, Int8을 사용하게 된다. Float의 경우 비트를 지수와 가수로 사용하며, 정밀도가 낮아질수록 표현할 수 있는 숫자의 범위와 유효 십진 자릿수가 줄어들게 된다.

| 자료형 | 유한값 범위 | 유효 십진 자릿수 | 최소 표현 가능한 간격 | 일반적인 표현 |
|---|---:|---:|---:|---|
| Float32 | 약 ±3.4 × 10³⁸ | 약 7자리 | 0.000000119 | 단정밀도 |
| Float16 | ±65,504 | 약 3~4자리 | 0.0009766 | 반정밀도 |
| Float8[^fp8-e4m3] | ±448 | 약 1~2자리 | 0.125 | - |

[^fp8-e4m3]: 본 표의 Float8 수치는 E4M3 형식을 기준으로 한다.

정밀도로 Int를 사용하는 경우 Float 값을 정수 범위로 Quantization한다. 모델 그래프에서는 필요에 따라 Dequantization을 거치며, 실제 런타임에서는 지원되는 연산을 정수 연산으로 실행할 수 있다. Int로 양자화를 하는 대표적인 방법으로 affine quantization이 있다.

Affine Quantization:

$$
q = \operatorname{clip}\left(\operatorname{round}(x/s) + z, q_{min}, q_{max}\right)
$$

Dequantization:

$$
\hat{x} = s(q-z)
$$

출처: [Jacob et al., Quantization and Training of Neural Networks for Efficient Integer-Arithmetic-Only Inference (CVPR 2018), Eq. (1)](https://openaccess.thecvf.com/content_cvpr_2018/html/Jacob_Quantization_and_Training_CVPR_2018_paper.html) · [ONNX QuantizeLinear](https://onnx.ai/onnx/operators/onnx__QuantizeLinear.html) · [ONNX DequantizeLinear](https://onnx.ai/onnx/operators/onnx__DequantizeLinear.html)

- $x$: 원래 floating-point 값
- $q$: 양자화된 정수값
- $s$: scale
- $z$: zero point
- $\hat{x}$: dequantization으로 복원한 근사값

round(•)는 반올림 연산자, clip(x, x_min, x_max)은 x값이 x_min ~ x_max 범위를 벗어나는 경우 주어진 x_min이나 x_max값으로 고정하는 연산자이다. 여기서 주의해볼 부분 중에 하나는 scale과 zero point가 Dequantization에도 필요하다는 것이다. 따라서 Int를 사용하는 경우 scale과 zero point를 별도로 저장하고 있어야한다는 것이다.

보통 scale과 zero point는 값마다 저장하지 않고 per-tensor, per-channel 또는 per-block 단위로 저장한다. 따라서 이에 따른 metadata overhead는 모델 파라미터에 비해 매우 작다. 또한 이 두 값을 공유함으로써 연산량을 줄일 수 있다. 입력과 가중치는 다음과 같이 표현되는데,

$$
x \approx s_x \times (q_x - z_x), \\
w \approx s_w \times (q_w - z_w)
$$

이를 내적으로 다음처럼 계산할 수 있다.

$$
x \cdot w \approx s_x \times s_w \times \sum((q_x - z_x)(q_w - z_w))
$$

### 2.2. Post-Training Quantization

Quantization은 크게 Post-Training Quantization(PTQ)과 Quantization-Aware Training(QAT)이 있다. Post-Training Quantization(PTQ)는 이미 학습된 모델의 파라미터의 정밀도를 바꾸는 방식으로 학습이 필요 없고 해당 기능을 지원하는 경우가 많아 빠르게 적용해 보기 좋다는 장점이 있다. 일반적으로 Quantization이라고 하면 PTQ를 칭하는 경우가 많다.

개인적으로 실무에서 PTQ를 적용했을 때 수치상 성능이 약간 하락하는 경우가 있는데 대부분 정성적으로 판단해보면 결과 자체는 유의미하게 다르지 않은 경우가 많고 이 경우에는 대체로 PTQ를 그대로 적용하는 경우가 많았다. 다만 그럼에도 classification이 아닌 regression 문제의 경우 지표상으로나 정성적으로나 차이가 나는 경우가 종종 있기 때문에 이 경우에는 QAT나 다른 기법을 적용하는 것을 검토해 보는 것을 추천한다.

### 2.3. Quantization-Aware Training

예전에 기상청에서 날씨를 예측하는 모델에서 소수점 한 자리를 없앴더니 아예 다른 결과가 나왔다는 얘기가 있었다. 딥러닝의 경우도 이미 학습된 모델에 소수점 자리를 버리고 추론을 하면 결과가 달라지고 정확도가 하락하는 경우가 있다. 이를 완화하기 위한 방법으로 Quantization-Aware Training(QAT)가 있다.

QAT는 학습 중에 미리 양자화를 할 것을 감안하여 학습하는 방법이다. Float 계열로 Quantization을 하는 경우에는 Mixed Precision Training 기법을 많이 사용하는데, 이는 모델에서 주요한 Conv, Linear와 같이 주요한 layer를 사전에 Float16으로 캐스팅한 후 학습한다. 참고로 PyTorch는 학습 중 autocast와 같은 함수를 통한 형변환을 지원한다.

반면 Int 계열의 경우 약간 방법이 다른데 Quantization에서 발생하는 rounding과 clipping 오차를 fake quantization으로 시뮬레이션하고, 모델이 이 오차에 미리 적응하도록 weight를 업데이트한다. Fake Quantization은 forward에서 quantization과 dequantization을 연속으로 적용한다.

$$
\begin{aligned}
q &= \operatorname{clip}\left(\operatorname{round}\left(\frac{x}{s}+z\right),q_{\min},q_{\max}\right) \\
\tilde{x} &= s(q-z)
\end{aligned}
$$

중간의 정수값 q는 정수 범위로 제한되지만, 다음 연산으로 전달되는 근사값은 양자화 오차가 반영된 floating-point 값이다. 따라서 실제 INT8로 학습하지 않고도 추론 시 발생할 rounding과 clipping의 영향을 forward에 포함할 수 있다. 학습 중 입력 $x$ 대신 affine quantization을 적용한 $\tilde{x}$를 사용해서 학습한다. 그리고 이 값을 통해 발생한 loss를 다시 $x$에 전파한다. 다만 round 연산은 미분할 수 없으므로 backward에서는 일반적으로 Straight-Through Estimator(STE)를 사용해 gradient를 전달한다.

$$
\tilde{x}=s\left[\operatorname{clip}\left(\operatorname{round}\left(\frac{x}{s}+z\right),q_{\min},q_{\max}\right)-z\right]
$$

출처: [PyTorch FakeQuantize 공식 문서](https://docs.pytorch.org/docs/stable/generated/torch.ao.quantization.fake_quantize.FakeQuantize.html) · [TensorFlow Model Optimization — Quantization-Aware Training](https://www.tensorflow.org/model_optimization/guide/quantization/training)

### 2.4. PTQ와 QAT 비교

| 구분 | PTQ | QAT |
|---|---|---|
| 적용 시점 | 학습 완료 후 | 학습 또는 fine-tuning 중 |
| weight 업데이트 | 없음 | quantization error를 사전에 업데이트 |
| 필요 데이터 | 없음<br>(calibration data가 필요한 경우 있음) | 일반적으로 학습 데이터와 label |
| 정확도 | 약간 하락이 있을 수 있음<br>regression 모델은 더 큰 하락이 있을 수 있음 | PTQ보다 정확도 하락폭이 낮을 수 있음<br>regression 모델에 유리 |
| 적용 난이도 | 쉬움 - 별도 구현 필요 없음 | 어려움 - 별도 구현이 요구될 수 있음 |

---

## 참고 자료

- [Distilling the Knowledge in a Neural Network — Hinton, Vinyals, Dean](https://arxiv.org/abs/1503.02531)
- [Learning both Weights and Connections for Efficient Neural Networks — Han et al.](https://arxiv.org/abs/1506.02626)
- [Quantization and Training of Neural Networks for Efficient Integer-Arithmetic-Only Inference — Jacob et al., CVPR 2018](https://openaccess.thecvf.com/content_cvpr_2018/html/Jacob_Quantization_and_Training_CVPR_2018_paper.html)
- [A White Paper on Neural Network Quantization — Nagel et al.](https://arxiv.org/abs/2106.08295)
- [PyTorch FakeQuantize 공식 문서](https://docs.pytorch.org/docs/stable/generated/torch.ao.quantization.fake_quantize.FakeQuantize.html)
- [TensorFlow Model Optimization — Quantization-Aware Training](https://www.tensorflow.org/model_optimization/guide/quantization/training)
- [ONNX QuantizeLinear 공식 연산 명세](https://onnx.ai/onnx/operators/onnx__QuantizeLinear.html)
- [ONNX DequantizeLinear 공식 연산 명세](https://onnx.ai/onnx/operators/onnx__DequantizeLinear.html)
