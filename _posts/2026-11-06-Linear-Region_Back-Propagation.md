---
title: "왜 Deep일까? — Linear Region과 Backpropagation 직접 계산해보기"
# 대안 제목:
# - Shallow vs Deep Network, 그리고 Backpropagation 손계산
# - Deep Network의 표현력과 학습 원리 (Linear Region · Chain Rule · Gradient Descent)
date: 2026-11-06 22:30:00 +0900
categories: [AI, DL-Basics]
tags: [Deep Learning, Neural Network, ReLU, Backpropagation, Gradient Descent]
---

> 인공지능개론 (Deep Learning 2) 강의를 공부하며 정리한 글입니다.

## 목차

1. [Shallow vs Deep Neural Network](#1-shallow-vs-deep-neural-network)
2. [Backpropagation](#2-backpropagation)
3. [Gradient Descent](#3-gradient-descent)
4. [정리](#4-정리)

---

## 1. Shallow vs Deep Neural Network

Deep Network가 Shallow Network보다 유리한 이유를 한 문장으로 말하면 이렇습니다.

> **같은 수의 Parameter로 훨씬 다양한 모양의 함수를 표현할 수 있기 때문이다.**

이 말이 무슨 뜻인지, 입력 1개와 출력 1개짜리 가장 단순한 Network로 확인해 보겠습니다.

> 📷 **[이미지 추가]** Hidden layer가 1개인 Shallow Network와 여러 개인 Deep Network 구조 비교 — 추천 출처: 슬라이드 4 / 직접 그림

### 1.1 ReLU 하나는 "한 번 꺾이는 직선"

$$
\text{ReLU}(x) =
\begin{cases}
0 & (x \le 0) \\
x & (x > 0)
\end{cases}
$$

Hidden unit 하나는 $\text{ReLU}(wx + b)$ 꼴입니다. 이 함수는 $wx + b = 0$, 즉 $x = -b/w$에서 **한 번 꺾입니다**. 이 지점을 **Breakpoint**라고 합니다.

예를 들어 $w = [1, 1, 1]$, $b = [-1, -2, -3]$이면 Breakpoint는 $x = 1, 2, 3$에 생깁니다.

$$
y = \text{ReLU}(x-1) + \text{ReLU}(x-2) + \text{ReLU}(x-3)
$$

| 구간      | 켜진 unit 수 | 기울기 |
|-----------|--------------|--------|
| x < 1     | 0            | 0      |
| 1 < x < 2 | 1            | 1      |
| 2 < x < 3 | 2            | 2      |
| x > 3     | 3            | 3      |

> 📷 **[이미지 추가]** 위 식의 그래프 (x = 1, 2, 3에서 꺾이는 piecewise linear 함수) — 추천 출처: 슬라이드 4 / 직접 그림

각 Hidden unit은 **독립적으로** 계산됩니다. 하지만 모두 **같은 입력 $x$**를 받고, 출력에서 **가중합**됩니다. 그래서 어떤 unit이 켜지거나 꺼지는 지점에서만 출력 $y$의 기울기가 바뀝니다. Breakpoint 사이 구간에서는 켜진 unit 조합이 고정되어 있으므로 $y$가 직선이 됩니다. 이 구간을 **Linear region**이라고 합니다.

### 1.2 Parameter 수와 Linear region 수

입력 1개, 출력 1개, Hidden layer마다 Neuron이 $D$개라고 가정합니다.

| Network           | Parameters         | Max linear regions |
|-------------------|--------------------|--------------------|
| Shallow (1 layer) | $3D+1$             | $D+1$              |
| Deep (K layers)   | $3D+1+(K-1)D(D+1)$ | $(D+1)^K$          |

**Parameter 수 세기**

| 연결               | Weight      | Bias      |
|--------------------|-------------|-----------|
| 입력 → Hidden 1    | $D$         | $D$       |
| Hidden → Hidden    | $(K-1)D^2$  | $(K-1)D$  |
| Hidden K → 출력    | $D$         | $1$       |

모두 더하면 $3D + 1 + (K-1)D(D+1)$입니다. $K=1$을 넣으면 Shallow Network의 $3D+1$과 같아집니다.

> **Note** 출력층의 Weight $D$개를 빠뜨리기 쉽습니다. 저도 처음에 $2D+1+(K-1)(D^2+D)$로 잘못 셌습니다.

**Linear region 수**

- Shallow Network에서 Breakpoint는 최대 $D$개입니다. 수직선을 $D$번 자르면 최대 $D+1$개의 구간이 생깁니다.
- Deep Network에서는 각 layer가 앞 layer의 출력을 다시 접고 꺾습니다. 그래서 region 수가 layer마다 곱해져 최대 $(D+1)^K$개가 됩니다.

> **Note** 둘 다 **최대치(up to)**입니다. Breakpoint가 겹치거나, 입력 범위 밖에 있거나, unit이 항상 꺼져 있으면 region은 이보다 줄어듭니다. 반대로 입력이 다차원이거나 Activation function이 더 복잡하면 region은 더 많아질 수 있습니다.

### 1.3 같은 Parameter로 비교하면

Parameter를 22개로 맞추고 두 Network를 비교해 보겠습니다.

| Network           | D   | Parameters | Max regions |
|-------------------|-----|------------|-------------|
| Shallow (1 layer) | 7   | 22         | 8           |
| Deep (2 layers)   | 3   | 22         | 16          |

Parameter는 layer 수에 대해 **선형으로** 늘어나지만, region은 **지수적으로** 늘어납니다.

### 1.4 "다양한 그래프"가 왜 중요한가

Network가 표현하는 그래프는 곧 **입력 → 출력 관계(함수)**입니다. 학습은 이 그래프가 데이터에 맞도록 Weight와 Bias를 조정하는 과정입니다.

- 직선 조각이 많을수록 더 복잡한 곡선을 근사할 수 있습니다. 이것을 **Expressiveness(표현력)**라고 합니다.
- 표현력이 부족하면 데이터가 아무리 많아도 패턴을 잡지 못합니다. 이 상태를 **Underfitting**이라고 합니다.
- 반대로 표현력이 지나치면 노이즈까지 외울 수 있습니다. 이 상태를 **Overfitting**이라고 합니다.

> 📷 **[이미지 추가]** 같은 데이터에 직선 모델과 piecewise linear 모델을 맞춘 비교 그래프 — 추천 출처: 직접 그림

---

## 2. Backpropagation

표현력이 큰 Network를 만들었다면, 이제 Weight를 **어느 방향으로 얼마나** 바꿀지 알아야 합니다. 그 답이 **Gradient** $\partial L / \partial W$이고, 이것을 효율적으로 구하는 방법이 **Backpropagation**입니다.

### 2.1 Chain Rule

Network의 출력은 함수의 합성입니다.

$$
L = f(g(h(\mathbf{x})))
$$

- $h$: Weighted sum, $z = \sum w_i x_i$
- $g$: Activation function, $o = \sigma(z)$
- $f$: Loss function, $L = (\hat{y} - y)^2$

합성함수의 미분은 Chain rule을 따릅니다.

$$
(f \circ g \circ h)'(x) = f'(g(h(x))) \cdot g'(h(x)) \cdot h'(x)
$$

이를 Weight에 대한 미분으로 쓰면 다음과 같습니다.

$$
\frac{\partial L}{\partial w}
= \underbrace{\frac{\partial L}{\partial o}}_{2(\hat{y}-y)}
\times \underbrace{\frac{\partial o}{\partial z}}_{\sigma(z)(1-\sigma(z))}
\times \underbrace{\frac{\partial z}{\partial w}}_{\text{그 Weight에 곱해진 입력}}
$$

**앞쪽 layer의 Weight**를 구하려면 체인이 더 길어집니다. 중간에 $\partial z_2 / \partial o_1$이 끼는데, 이 값은 두 unit을 잇는 **Weight**입니다.

$$
\frac{\partial L}{\partial w_1}
= \frac{\partial L}{\partial o_2}
\times \frac{\partial o_2}{\partial z_2}
\times \underbrace{\frac{\partial z_2}{\partial o_1}}_{w_2}
\times \frac{\partial o_1}{\partial z_1}
\times \frac{\partial z_1}{\partial w_1}
$$

> 📷 **[이미지 추가]** $x \to z \to o \to L$ 연산 그래프와 Gradient가 거꾸로 흐르는 화살표 — 추천 출처: 슬라이드 15~16

### 2.2 직접 계산해보기

입력 2개, Hidden unit 2개, 출력 1개인 Network입니다. Activation은 Sigmoid이고 Bias는 없습니다. 정답은 $y = 0.5$입니다.

> 📷 **[이미지 추가]** 예시 Network 그림 ($x_0=0.35$, $x_1=0.7$, 각 Weight 값 표시) — 추천 출처: 슬라이드 12

여기서 $w_{l,i,j}$는 layer $l$에서 $i$번째 입력과 $j$번째 unit을 잇는 Weight입니다.

**Forward pass**

$$
\begin{aligned}
z_{1,0} &= 0.2 \times 0.35 + 0.2 \times 0.7 = 0.21 &&\Rightarrow\ o_{1,0} = \sigma(0.21) = 0.552 \\
z_{1,1} &= 0.3 \times 0.35 + 0.3 \times 0.7 = 0.315 &&\Rightarrow\ o_{1,1} = \sigma(0.315) = 0.578 \\
z_{2,0} &= 0.3 \times 0.552 + 0.9 \times 0.578 = 0.686 &&\Rightarrow\ \hat{y} = o_{2,0} = \sigma(0.686) = 0.665
\end{aligned}
$$

$$
L = (0.665 - 0.5)^2 \approx 0.027
$$

**Backward pass: 출력층 Weight**

$$
\begin{aligned}
\frac{\partial L}{\partial w_{2,0,0}}
&= 2(o_{2,0} - y) \times o_{2,0}(1 - o_{2,0}) \times o_{1,0} \\
&= 2(0.665 - 0.5) \times 0.665(1 - 0.665) \times 0.552 \\
&\approx 0.041
\end{aligned}
$$

같은 방법으로 $\partial L / \partial w_{2,1,0} \approx 0.042$입니다. 마지막 항만 $o_{1,1} = 0.578$로 바뀝니다.

**Backward pass: Hidden layer Weight**

$$
\begin{aligned}
\frac{\partial L}{\partial w_{1,0,0}}
&= 2(o_{2,0} - y) \times o_{2,0}(1 - o_{2,0}) \times w_{2,0,0} \times o_{1,0}(1 - o_{1,0}) \times x_0 \\
&= 2(0.665 - 0.5) \times 0.665(1 - 0.665) \times 0.3 \times 0.552(1 - 0.552) \times 0.35 \\
&\approx 0.002
\end{aligned}
$$

> 📷 **[이미지 추가]** 각 Weight 옆에 계산된 Gradient 값을 표시한 Network 그림 — 추천 출처: 슬라이드 17~19

> **Note** 출력층 Weight의 Gradient는 **0.041**이고, 한 층 앞의 Weight는 **0.002**입니다. 한 층만 내려왔는데 약 20배 작아졌습니다. 이 현상이 2편의 주제인 **Vanishing gradient**로 이어집니다.

---

## 3. Gradient Descent

Gradient를 구했으면 Loss가 줄어드는 방향, 즉 **Gradient의 반대 방향**으로 Parameter를 조금 이동시킵니다.

$$
W := W - \eta \frac{\partial L}{\partial W}
$$

- $W$: Parameter (Weight, Bias)
- $L$: Loss
- $\eta$ (eta): **Learning rate**, 한 번에 이동하는 보폭

> 📷 **[이미지 추가]** Loss–Parameter 곡선 위에서 Gradient 반대 방향으로 내려가는 그림 — 추천 출처: 슬라이드 20

**예시** ($\eta = 0.01$)

$$
\begin{aligned}
w_{2,0,0} &= 0.3 - 0.01 \times 0.041 = 0.29959 \\
w_{1,0,0} &= 0.2 - 0.01 \times 0.002 = 0.19998
\end{aligned}
$$

이처럼 Forward pass, Backward pass, Update를 계속 반복하는 것이 학습입니다.

### Optimizer 한눈에 보기

기본 Gradient Descent의 Update 식을 개선한 알고리즘들입니다.

- **GD**: 전체 데이터로 Gradient를 계산합니다. 정확하지만 느립니다.
- **SGD**: 샘플 하나나 Mini-batch로 Gradient를 근사합니다. 빠르지만 값이 흔들립니다.
- **AdaGrad**: 많이 업데이트된 Parameter일수록 Learning rate를 줄입니다.
- **RMSProp**: AdaGrad에서 Learning rate가 너무 빨리 줄어드는 문제를 이동 평균으로 보완합니다.
- **Adam**: Momentum과 RMSProp을 결합했습니다. 현재 가장 널리 쓰이는 기본값입니다.

---

## 4. 정리

- Hidden unit 하나(ReLU)는 Breakpoint 하나를 만듭니다. Shallow Network는 최대 $D+1$개의 Linear region을 가집니다.
- $K$개 layer를 쌓으면 Parameter는 $3D+1+(K-1)D(D+1)$로 선형으로 늘지만, region은 최대 $(D+1)^K$개로 지수적으로 늘어납니다. **이것이 Deep의 힘입니다.**
- Gradient는 Chain rule로 구하고, $W := W - \eta\,\partial L / \partial W$로 Parameter를 업데이트합니다.
- 앞쪽 layer로 갈수록 곱하는 항이 많아져 Gradient가 작아질 수 있습니다. 이 내용은 [2편](./week9_blog_2.md)에서 이어집니다. <!-- 블로그 URL로 교체 -->

## Reference

- Bishop, C. M., & Bishop, H. (2024). *Deep Learning: Foundations and Concepts*. Springer.
