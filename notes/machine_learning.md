# 前馈神经网络
## .1 神经元
- 神经元接收 $D$ 个输入 $x_1,x_2,\cdots,x_D$，令向量 $\boldsymbol{x}=\left[ x_1;x_2;\cdots ;x_D \right]$ 来表示这组输入，用 $z\in \mathbb{R}$ 表示一个神经元所获得的输入信号 $\boldsymbol x$ 的加权和
$$
z=\sum_{d=1}^D{w_dx_d+b}=\boldsymbol{w}^T\boldsymbol{x}+b
$$
其中 $\boldsymbol{w}=\left[ w_1;w_2;\cdots ;w_D \right] \in \mathbb{R} ^D$ 是 $D$ 维的权重向量，$b \in \mathbb {R}$ 是偏置
- 净输入 $z$ 在经过一个非线性函数 $f(\cdot)$ 后，得到神经元的<font color='red'>活性值</font>
$$
a=f(z)
$$
其中非线性函数 $f(\cdot)$ 称为激活函数
- 一个典型的神经元结构如下所示

![[neuron.svg]]

- <font color='red'>激活函数</font>在神经元中非常重要，需要具备以下几点性质
	- 连续并可导(允许少数点不可导)的非线性函数，可导的激活函数可以直接用数值优化的方式学习网络参数
	- 激活函数及其导函数要尽量简单，有助于提高网络计算效率
	- 激活函数的导函数的值域要在一个合适的区间内
### .1.1 Sigmoid 型函数
#### .1.1.1 Logistic 函数
$$
\sigma \left( x \right) =\frac{1}{1+\exp \left( -x \right)}
$$
#### .1.1.2 Tanh 函数
- `Tanh` 函数也是一种 `Sigmoid` 函数，其定义为
$$
\tanh \left( x \right) =\frac{\exp \left( x \right) -\exp \left( -x \right)}{\exp \left( x \right) +\exp \left( -x \right)}
$$
- `Tanh` 函数可以看作是放大并平移的 `Logistic` 函数，其值域是 $(-1,1)$
$$
\tanh(x)=2\sigma(2x)-1
$$
- 下图是 `Logistic` 函数和 `Tanh` 函数的形状，Tanh 函数的输出是<font color='red'>零中心化的</font>，而 Logistic 函数的输出恒大于 0，
