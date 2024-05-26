<!--Copyright © 适用于[License](https://github.com/chenzomi12/AISystem)版权许可-->

# 知识蒸馏

Knowledge Distillation(KD)最初由 Hinton 在文章“Distilling the Knowledge in a Neural Network”中提出。Hinton 在文章中提出了一个形象的比喻，将神经网络模型比作自然界中的昆虫，许多昆虫都有着幼虫形态和成虫形态，幼虫形态可以从环境中吸收养分，成虫形态则便于四处迁徙和繁殖。

神经网络模型在训练阶段学习大量数据中的知识，在这个阶段，可以使用更多的算力，而在模型的部署以及推理阶段，则对算力和时延有更严格的控制，需要更加轻量化的模型，以适应不同场景。

知识蒸馏算法由三部分组成，分别是知识（Knowledge）、蒸馏算法（Distillation algorithm）、师生架构（Teacher-student architecture）。一般的师生架构如下图所示。

![知识蒸馏架构](../03Slim/images/distill01.png)
===== 图片地址有错哦，另外图片统一改为 03Distillation01.pn 这种哈

通常，教师网络会比学生网络大，通过知识蒸馏的方法将教师网络的知识转移到学生网络，因此，蒸馏学习可以用于压缩模型，将大模型变成小模型。另外，知识蒸馏的过程需要数据集，这个数据集可以是用于教师模型预训练的数据集，也可以是额外的数据集。

======= 这里面的知识都较为传统，或者是 ZOMI 1 年前的理解了，可以加入最新的一些知识蒸馏相关技术哦！补充一大节二级目录

## 知识类型

知识的类型可以分为四类，主要有 Response-based、Feature-based、Relation-based 三种，而 Architecture-based 类型很少。

![知识类型](../03Slim/images/distill03.png)
===== 图片地址有错哦，另外图片统一改为 03Distillation01.pn 这种哈

### response-based knowledge

当知识蒸馏对这部分知识进行转移时，学生模型直接学习教师模型输出层的特征。通俗的说法就是老师充分学习知识后，直接将结论告诉学生。假设张量 $z_t$ 为教师模型的输出，张量 $z_s$ 为学生模型的输出，这里的输出都是指模型最后一层的输出，蒸馏学习的目标是让 $z_s$ 模仿 $z_t$，降低下图中的 distillation loss。

![Response-based knowledge](../03Slim/images/distill04.png)
===== 图片地址有错哦，另外图片统一改为 03Distillation01.pn 这种哈

### feature-based knowledge

上面一种方法学习目标非常直接，学生模型直接学习教师模型的最后预测结果。考虑到深度神经网络善于学习不同层级的特征，教师模型的中间层的特征激活也可以作为学生模型的学习目标，对 Response-based knowledge 形成补充。
下面是 Feature-based knowledge 的知识迁移过程。

![Feature-based knowledge](../03Slim/images/distill05.png)
===== 图片地址有错哦，另外图片统一改为 03Distillation01.pn 这种哈

虽然基于特征的知识转移为学生模型的学习提供了更多信息，但由于学生模型和教师模型的结构不一定相同，如何从教师模型中选择哪一层特征激活（提示层），从学生模型中选择哪一层（引导层）模仿教师模型的特征激活，是一个需要探究的问题。

另外，当提示层和引导层大小存在差异时，如何正确匹配教师与学生的特征表示也需要进一步探究，目前还没有成熟的方案。

### relation-based knowledge

上述两种方法都使用了教师模型中特定网络层中特征的输出，而基于关系的知识进一步探索了各网络层输出之间的关系或样本之间的关系。例如将教师模型中两层 feature maps 之间的 Gram 矩阵（网络层输出之间的关系）作为知识，或者将样本在教师模型上的特征表示的概率分布（样本之间的关系）作为知识。

![Feature-based knowledge](../03Slim/images/distill06.png)
===== 图片地址有错哦，另外图片统一改为 03Distillation01.pn 这种哈

## 知识蒸馏方式

知识蒸馏的方式一般分为三种：offline distillation，online distillation，self-distillation。

![知识蒸馏方式](../03Slim/images/distill07.png)
===== 图片地址有错哦，另外图片统一改为 03Distillation01.pn 这种哈

### offline distillation

这种方法是大部分知识蒸馏算法采用的方法，主要包含两个过程：1）蒸馏前教师模型预训练；2）蒸馏算法迁移知识。因此该方法主要侧重于知识迁移部分。教师模型通常参数量大，训练时间比较长，一些大模型会通过这种方式得到小模型，比如 BERT 通过蒸馏学习得到 tinyBERT。但这种方法的缺点是学生模型非常依赖教师模型。

### online distillation

这种方法要求教师模型和学生模型同时更新，主要针对参数量大、精度性能好的教师模型不可获得情况。而现有的方法往往难以获得在线环境下参数量大、精度性能好的教师模型。

### self-distillation

self-distillation 是 online distillation 的一种特例，教师模型和学生模型采用相同的网络模型。

用学习过程比喻，offline distillation 是知识渊博的老师向学生传授知识；online distillation 是老师和学生一起学习、共同进步；self-distillation 是学生自学成才。

## 经典算法解读

对于 Hinton 提出的 Distilling the Knowledge in a Neural Network 算法流程主要可以概括为以下 4 个流程步骤：

1. 训练教师模型
2. 教师模型的 logits 输出，在高温 T 下生成 soft target
3. 使用 $\mathcal{L}_{soft}$ 与 $\mathcal{L}_{hard}$ 同时训练学生模型
4. 将温度 T 调为 1，学生模型用于线上推理

### 详细算法过程

对于一般的神经网络模型，最后的输出向量 $z$ 被称为 logits，Hinton 等人之前，Caruana 等人已经提出了将大的集成的模型的知识迁移到单个小模型中，具体方法就是最小化小模型的 logits 输出和复杂模型的 logits 输出的平方差。

logits 经过 softmax 函数得到模型的类别预测概率，

$$q_i = \frac{\exp(z_i)}{\sum_j \exp(z_j)}$$

其中 $q_i$ 表示不同类别的预测概率。这个预测结果是 soft target，而真实目标是 hard target，一般机器学习的目标就是让 soft target 逼近 hard target。

Hinton 等人引入“蒸馏”的概念，在上式基础上添加一个温度系数 $T$，

$$q_i = \frac{\exp(z_i/T)}{\sum_j \exp(z_j/T)}$$

当 $T=1$ 时，就是标准的 softmax 函数。T 越大，分布的熵越大，负标签携带的信息会被放大，负标签的概率分布会对损失函数有更明显的影响，模型训练会更关注这部分信息。为什么要重视负标签的信息？

Hinton 举了一个例子，BMW 宝马被当做垃圾箱的概率很低，基本接近于 0，但还是比被当做胡萝卜的概率要高得多。当负标签的概率都很低时，负标签之间的概率差异仍然包含了一部分信息，而这部分信息往往被模型忽略（因为所有负标签概率接近于 0）。例如 MNIST 数据集中存在一个数字 2 的样本被预测为 3 的概率为 $10^{-6}$，被预测为 7 的概率为 $10^{-9}$，这部分负标签的信息就意味着这个数字 2 有可能与 3 和 7 有些相像。

通常，在蒸馏学习过程中，将 T 适当调高并保持不变，使得学生模型可以学习到负标签的信息，等学生模型训练完成后，将 T 设为 1，用于推理。

该蒸馏学习算法采用 offline distillation 的形式、教师-学生架构，其中教师是知识输出者，学生是知识接受者。算法过程分为两个部分：教师模型训练、学生模型蒸馏。教师模型特点是模型较为复杂，精度较高，对教师模型不做任何关于模型架构、参数量等方面的限制。

唯一要求就是对于输入 X，都能输出 Y，Y 经过 softmax 可以得到类别预测的概率分布。学生模型是参数量较小、结构相对简单的模型，同样对于输入 X，都能输出 Y，经过 softmax 可以得到概率分布。

论文中，Hinton 将问题限定为分类问题，即模型最终输出会经过 softmax 处理，得到一个概率分布。蒸馏过程中除了教师模型和学生模型，一个重要的部分是数据集，数据集可以是训练教师模型所用的数据集，也可以是其他的辅助数据集，可以是有标签的数据集，也可以是无标签的数据集。

如果蒸馏过程中使用的数据集有标签，则学生模型的训练目标有两个，一个是模仿教师模型的输出，另一个是接近真实标签，而一般前者是主要目标，后者是次要目标。目标函数可写为：

$$\mathcal{L} = \mathcal{L}_{soft} + \lambda \mathcal{L}_{hard}$$

$$\mathcal{L}_{soft} = -\sum_j p_j^T log(q_j^T)$$

$$\mathcal{L}_{hard} = -\sum_j c_j log(q_j)$$

其中 $p_j^T$ 表示教师模型在 $T$ 下（$T$ 通常大于 1）的预测结果，$q_j^T$ 表示学生模型在 $T$ 下的预测结果，$c_j$ 表示真实标签，$q_j$ 表示学生模型在 $T=1$ 时的预测结果。当数据集无标签时，只能用 $\mathcal{L}_{soft}$。

### 补充

1. 知识蒸馏与物理蒸馏的相似之处：

    - 知识蒸馏通过 T 系数控制模型输出的熵；物理蒸馏通过温度改变混合物的形态，影响物理系统的熵

    - 温度系数 T 训练时提高，最后变回 1；物理蒸馏时温度先上升使液体变为气体，气体再回到常温变回液体

2. 知识从何说起：在神经网络模型中，对模型中的知识是难以观察的，从更抽象的角度理解，模型的知识就是两个空间之间的映射关系。

## 小结

====== 加一段自己的理解哦

## 本节视频

<html>
<iframe src="https:&as_wide=1&high_quality=1&danmaku=0&t=30&autoplay=0" width="100%" height="500" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true"> </iframe>
</html>

## 参考文献

1. Jianping Gou et al. Knowledge Distillation: A Survey. https://doi.org/10.1007/s11263-021-01453-z
2. Hinton et al. Distilling the Knowledge in a Neural Network. http://arxiv.org/abs/1503.02531
3. Longhui Wei et al. Circumventing outlier of autoaugment with knowledge distillation.  https://doi.org/10.1007/978-3-030-58580-8_36
4. Caruana et al. Model compression. https://doi.org/10.1145/1150402.1150464
5. 模型压缩（上）--知识蒸馏（Distilling Knowledge）https://www.jianshu.com/p/a6d87b338bcf
6. DeiT：注意力也能蒸馏 https://www.cnblogs.com/ZOMI/p/16496326.html
