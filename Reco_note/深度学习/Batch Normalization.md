### 1. 前言

- 在学习GoogLeNet时，见到了BN层的概念，在ResNet网络中，也有BN层，所以查阅了网上几篇博客来学习，首先大致看一下BN的作用是什么？

- GoogLeNet_InceptionV2 在 V1 的基础上引入了**BN 层**， 减少了**Internal Covariate Shift(内部神经元的数据分布差异)**， 使得每一层的输出都可以规范化到一个标准高斯分布上。

- 机器学习领域有个很重要的假设：**IID独立同分布假设**，就是**假设训练数据和测试数据是满足相同分布**的，这是通过训练数据获得的模型能够在测试集获得好的效果的一个基本保障。那**BatchNorm的作用是什么**呢？BatchNorm就是**在深度神经网络训练过程中使得每一层神经网络的输入保持相同分布**的。
- 接下来一步一步的理解什么是BN：为什么深度神经网络**随着网络深度加深，训练起来越困难，收敛越来越慢？**这是个在DL领域很接近本质的好问题。很多论文都是解决这个问题的，比如ReLU激活函数，再比如Residual Network，BN本质上也是解释并从某个不同的角度来解决这个问题的。

### 2. Internal Covariate Shift

- BN是用来解决“Internal Covariate Shift”问题的，那么首先得理解什么是“Internal Covariate Shift”？
- **covariate shift的概念**：**如果ML系统实例集合<X,Y>中的输入值X的分布老是变，这不符合IID假设**，网络模型很难**稳定的学规律**。
- 对于深度学习这种包含很多隐层的网络结构，在训练过程中，因为各层参数不停在变化，所以每个隐层都会面临covariate shift的问题，也就是**在训练过程中，隐层的输入分布老是变来变去，这就是所谓的“Internal Covariate Shift”，Internal指的是深层网络的隐层，是发生在网络内部的事情，而不是covariate shift问题只发生在输入层。**

- BatchNorm的基本思想：能不能**让每个隐层节点的激活输入分布固定下来呢**？这样就避免了“Internal Covariate Shift”问题了。

### 3. BN的基本思想

#### 3.1 BN综述

- 深层神经网络在做非线性变换前的**激活输入值(**就是那个input=Wx+b，x是输入)，**随着网络深度加深或者在训练过程中，其分布逐渐发生偏移或者变动**，之所以训练收敛慢，一般是整体分布逐渐往激活函数的取值区间的上下限两端靠近，导致反向传播时底层神经网络的梯度消失，这就是**深层神经网络收敛越来越慢**的原因
- **BN就是通过规范化手段，把每层神经网络任意神经元输入值值的分布强行拉回到均值为0方差为1的标准正态分布，使得激活输入值落在中间比较敏感的区域的概率较大，让梯度变大，避免梯度消失问题，梯度变大意味着学习收敛速度快，加快训练速度。**
- 就像激活函数层、卷积层、全连接层、池化层一样，BN(Batch Normalization)也属于网络的一层。事实上，paper的**算法本质原理**就是：在网络的每一层输入的时候，又插入了一个归一化层，也就是先做一个归一化处理，然后再进入网络的下一层。不过这个归一化层，可不像我们想象的那么简单，它**是一个可学习、有参数的网络层**。

#### 3.2 数据预处理

- 预处理公式如下：

$$
\widehat{x}^{(k)}=\frac{x^{(k)}-\mathrm{E}\left[x^{(k)}\right]}{\sqrt{\operatorname{Var}\left[x^{(k)}\right]}}
$$

- 使用上面这个公式，对某一个层网络的输入数据做一个归一化处理。需要注意的是，我们训练过程中采用batch 随机梯度下降，上面的E(xk)指的是**每一批训练数据神经元 $x^{k}$ 的平均值；然后分母就是每一批数据神经元 $x^k$ 激活度的一个标准差**了。

### 4. BN算法实现

#### 4.1 算法概述

- 经过前面的学习，好像就是在网络中间层数据做一个归一化处理，也不是很难。但是在实现起来并不容易。
- 如果是仅仅使用上面的归一化公式，对网络某一层A的输出数据做归一化，然后送入网络下一层B，这样是会**影响到本层网络A所学习到的特征的**。

- 打个比方，比如我网络中间某一层学习到特征数据本身就分布在S型激活函数的两侧，你却把它归一化处理、标准差也限制在了1，把数据变换成分布于s函数的中间部分，这样就相当于我这一层网络所学习到的特征分布被破坏了，这可怎么办？于是在paper中使用了一个方法：**变换重构**，**引入了可学习参数γ、β**，这就是算法关键之处：

$$
y^{(k)}=\gamma^{(k)} \widehat{x}^{(k)}+\beta^{(k)}
$$

- 每一个神经元$x^{(k)}$都有一对参数γ、β。
- 观察上式，当$\gamma^{(k)} =\sqrt{\operatorname{Var}\left[x^{(k)}\right]} ,\beta^{(k)} = \mathrm{E}\left[x^{(k)}\right]$ 时，再结合上面的归一化公式，是可以恢复出原始的某一层所学到的特征的。
- 因此我们引入了这个可学习重构参数γ、β，让**网络可以学习恢复出原始网络所要学习的特征分布**。最后Batch Normalization网络层的前向传导过程公式就是：

$$
\begin{array}{ll}{\mu_{\mathcal{B}} \leftarrow \frac{1}{m} \sum_{i=1}^{m} x_{i}} & {\text { 均值 }} \\ {\sigma_{\mathcal{B}}^{2} \leftarrow \frac{1}{m} \sum_{i=1}^{m}\left(x_{i}-\mu_{\mathcal{B}}\right)^{2}} & {\text { 方差 }} \\ {\widehat{x}_{i} \leftarrow \frac{x_{i}-\mu_{\mathcal{B}}}{\sqrt{\sigma_{\mathcal{B}}^{2}+\epsilon}}} & {\text { 归一化 }} \\ {y_{i} \leftarrow \gamma \widehat{x}_{i}+\beta \equiv \mathrm{B} \mathrm{N}_{\gamma, \beta}\left(x_{i}\right)} & {\text { 变换重构 }}\end{array}
$$

- 上面的公式中m指的是mini-batch size。

### 5. BN在CNN中的使用

- 通过上面的学习，我们知道BN层是对于每个神经元做归一化处理，甚至只需要对某一个神经元进行归一化，而不是对一整层网络的神经元进行归一化。
- 既然BN是对单个神经元的运算，那么在CNN中卷积层上要怎么搞？假如某一层卷积层有6个特征图，每个特征图的大小是100 x 100，这样就相当于这一层网络有6 x 100 x 100个神经元，如果采用BN，就会有6 x 100 x 100个参数γ、β，这样就太多了。
- 因此卷积层上的BN使用，其实也是使用了类似权值共享的策略，把一整张特征图当做一个神经元进行处理。
- 卷积神经网络经过卷积后得到的是一系列的特征图，如果min-batch sizes为m，那么网络某一层输入数据可以表示为四维矩阵(m,f,p,q)，m为min-batch sizes，f为特征图个数，p、q分别为特征图的宽高。
- 在cnn中我们可以**把每个特征图看成是一个特征处理（一个神经元）**，因此在使用Batch Normalization，mini-batch size 的大小就是：m x p x q，于是对于每个特征图都只有一对可学习参数：γ、β。就是相当于**求取所有样本所对应的一个特征图的所有神经元的平均值、方差**，然后对这个特征图神经元做归一化。
  

