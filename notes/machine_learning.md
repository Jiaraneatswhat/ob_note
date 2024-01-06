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


