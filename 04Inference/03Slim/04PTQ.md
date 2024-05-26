<!--Copyright © 适用于[License](https://github.com/chenzomi12/AISystem)版权许可-->

# 训练后量化与部署

=========== 内容还可以在网上找一找，然后自己再总结提炼一下，目前还不够细。特别是端侧量化推理部署部分哈。

## 动态离线量化

动态离线量化(Post Training Quantization Dynamic, PTQ Dynamic)仅将模型中特定算子的权重从 FP32 类型映射成 INT8/16 类型，主要可以减小模型大小，对特定加载权重费时的模型可以起到一定加速效果。但是对于不同输入值，其缩放因子是动态计算。动态量化的权重是离线转换阶段量化，而激活是在运行阶段才进行量化。因此动态量化是几种量化方法中性能最差的。

不同的精度下的动态量化对模型的影响：

- 权重量化成 INT16 类型，模型精度不受影响，模型大小为原始的 1/2。
- 权重量化成 INT8 类型，模型精度会受到影响，模型大小为原始的 1/4。

![动态离线量化算法流程](./images/04PTQ01.png)

动态离线量化将模型中特定算子的权重从 FP32 类型量化成 INT8 等类型，该方式的量化有两种预测方式：

1. 反量化推理方式，即是首先将 INT8/FP16 类型的权重反量化成 FP32 类型，然后再使用 FP32 浮运算运算进行推理。
2. 量化推理方式，即是推理中动态计算量化算子输入的量化信息，基于量化的输入和权重进行 INT8 整形运算。

## 静态离线量化

静态离线量化（Post Training Quantization Static, PTQ Static）同时也称为校正量化或者数据集量化，使用少量无标签校准数据。其核心是计算量化比例因子，使用静态量化后的模型进行预测，在此过程中量化模型的缩放因子会根据输入数据的分布进行调整。相比量化训练，静态离线量化不需要重新训练，可以快速得到量化模型。

$$
uint8 = round(float/scale) - offset
$$

静态离线量化的目标是求取量化比例因子，主要通过对称量化、非对称量化方式来求，而找最大值或者阈值的方法又有 MinMax、KL 散度、ADMM、EQ，MSE 等方法。

静态离线量化的步骤如下：

1. 加载预训练的 FP32 模型，配置用于校准的数据加载器；
2. 读取小批量样本数据，执行模型的前向推理，保存更新待量化算子的量化 scale 等信息；
3. 将 FP32 模型转成 INT8 模型，进行保存。

![静态离线量化流程](./images/04PTQ02.png)

一些常用的计算量化 scale 的方法：

| 量化方法 | 方法详解                                                     |
| -------- | ------------------------------------------------------------ |
| $abs_{max}$  | 选取所有激活值的绝对值的最大值作为截断值α。此方法的计算最为简单，但是容易受到某些绝对值较大的极端值的影响，适用于几乎不存在极端值的情况。 |
| $KL$       | 使用参数在量化前后的 KL 散度作为量化损失的衡量指标。此方法是 TensorRT 所使用的方法。在大多数情况下，使用 KL 方法校准的表现要优于 abs_max 方法。 |
| $avg $     | 选取所有样本的激活值的绝对值最大值的平均数作为截断值α。此方法计算较为简单，可以在一定程度上消除不同数据样本的激活值的差异，抵消一些极端值影响，总体上优于 abs_max 方法。 |

## KL 散度校准法

下面以静态离线量化中的 KL 散度作为例子，看看静态离线量化的具体步骤。

### 原理

KL 散度校准法也叫相对熵，其中 p 表示真实分布，q 表示非真实分布或 p 的近似分布：

$$
𝐷_{KL} (P_f || Q_q)=\sum\limits^{N}_{i=1}P(i)*log_2\frac{P_f(i)}{Q_q(𝑖)}
$$

相对熵，用来衡量真实分布与非真实分布的差异大小。目的就是改变量化域，实则就是改变真实的分布，并使得修改后得真实分布在量化后与量化前相对熵越小越好。

### 流程和实现

1. 选取 validation 数据集中一部分具有代表的数据作为校准数据集 Calibration；
2. 对于校准数据进行 FP32 的推理，对于每一层：
     1. 收集 activation 的分布直方图
     2. 使用不同的 threshold 来生成一定数量的量化好的分布
     3. 计算量化好的分布与 FP32 分布的 KL divergence，并选取使 KL 最小的 threshold 作为 saturation 的阈值

主要注意的点：
- 需要准备小批量数据（500~1000 张图片）校准用的数据集；
- 使用校准数据集在 FP32 精度的网络下推理，并收集激活值的直方图；
- 不断调整阈值，并计算相对熵，得到最优解

通俗地理解，算法收集激活 Act 直方图，并生成一组具有不同阈值的 8 位表示法，选择具有最少 kl 散度的表示；此时的 kl 散度在参考分布（FP32 激活）和量化分布之间（即 8 位量化激活）之间。

KL 散度校准法的伪代码实现：

```python
Input: FP32 histogram H with 2048 bins: bin[0], … , bin[2047]

For i in range(128, 2048):
     reference distribution P = [bin[0], …, bin[i-1]]
     outliers count = sum(bin[i], bin[i+1], …, bin[2047])
     reference distribution P[i-1] += outliers count 
     P /= sum(P)
     candidate distribution Q = quantize [bin[0], …, bin[i-1]] into 128 levels
     expand candidate distribution Q to I bins
     Q /= sum(Q)
     divergence[i] = KL divergence(reference distribution P, candidate distribution Q)
End For

Find index m for which divergence[m] is minimal

threshold = (m+0.5) * (width of a bin)

```

## 端侧量化推理部署

### 推理结构

端侧量化推理的结构方式主要由 3 种，分别是下图 (a) Fp32 输入 Fp32 输出、(b) Fp32 输入 int8 输出、(c) int8 输入 int32 输出

![端侧量化推理方式](./images/04PTQ04.png)

INT8 卷积如下图所示，里面混合里三种不同的模式，因为不同的卷积通过不同的方式进行拼接。使用 INT8 进行 inference 时，由于数据是实时的，因此数据需要在线量化，量化的流程如图所示。数据量化涉及 Quantize，Dequantize 和 Requantize 等 3 种操作：

![INT8 卷积示意图](./images/04PTQ05.png)

### 量化过程

#### 量化

将 float32 数据量化为 int8。离线转换工具转换的过程之前，根据量化原理的计算出数据量化需要的 scale 和 offset：

#### 反量化

INT8 相乘、加之后的结果用 INT32 格式存储，如果下一 Operation 需要 float32 格式数据作为输入，则通过 Dequantize 反量化操作将 INT32 数据反量化为 float32。Dequantize 反量化推导过程如下：

#### 重量化

INT8 乘加之后的结果用 INT32 格式存储，如果下一层需要 INT8 格式数据作为输入，则通过 Requantize 重量化操作将 INT32 数据重量化为 INT8。重量化推导过程如下：

## 本节视频

<html>
<iframe src="https:&as_wide=1&high_quality=1&danmaku=0&t=30&autoplay=0" width="100%" height="500" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true"> </iframe>
</html>

## 参考

- 8-bit Inference with TensorRT https://on-demand.GPUtechconf.com/gtc/2017/presentation/s7310-8-bit-inference-with-tensorrt.pdf