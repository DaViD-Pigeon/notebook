---
counter: True   
---

# Diffusion-LM：扩散语言模型改善可控文本生成

<div class="badges">
<span class="badge full-implement-badge">已完成</span>
</div>

!!! Brief
    ![](https://david-pigeon.github.io/notebook/images/Paper/Diffusion/Text_gen/diffulm_0.png)
    
    * Paper: [Diffusion-LM improves controllable text generation](https://arxiv.org/abs/2205.14217)
    * Code: [Diffusion-LM](https://github.com/XiangLi1999/Diffusion-LM)
    * 该工作已被NeurIPS 2022接收
    * 扩散模型在可控文本生成上的应用
    * 本文中的图片均来自论文。

## Abstract

文章提出了 Diffusion-LM,这是一种基于连续扩散的新型非自回归语言模型,旨在解决在不重新训练的情况下控制语言模型行为的挑战。关键要点如下:

- Diffusion-LM 通过迭代去噪一系列高斯向量来生成词向量,从而产生一个连续的分层潜在表示。这使得基于梯度的方法可以执行复杂的可控生成任务。

- Diffusion-LM 能够控制各种细粒度属性,从语义内容到句法结构,显著优于先前的即插即用方法,并与微调的先知或优于其性能。

- Diffusion-LM 还可以执行跨度锚定控制,如长度控制和填充,无需分类器,并优于先前的即插即用方法。

- Diffusion-LM 潜在表示的连续和分层性质是其在可控文本生成中取得成功的关键,与自回归语言模型形成对比。

## Introduction

DDPM(扩散模型)由于其空间上的连续性，在视觉和音频领域的生成任务上取得了极大的成功。但由于文本固有的离散性质，在之前并没有将扩散模型用于文本的相关工作。基于文本内容的离散性，Diffusion-LM在标准的扩散过程中添加了如下修改：

- 添加了嵌入步骤喝舍入步骤实现离散空间到连续空间的转换；

- 设计了一个学习嵌入的训练损失（目标）；

- 提出了改进舍入的相关技术；

![](https://david-pigeon.github.io/notebook/images/Paper/Diffusion/Text_gen/diffulm_1.png)

论文中使用基于梯度的方法来对 Diffusion-LM 进行控制，如上图所示。这样的方式能够将文本生成的过程转换为满足目标结构和语义控制的输出。我们迭代地对 Diffusion-LM 地连续隐变量进行梯度更新，来平衡流畅度和控制满意度。

## Related Work

- **扩散模型在文本生成中的应用**。论文指出,之前的工作研究了在离散状态空间上的文本扩散模型,通过定义离散数据的腐蚀过程来处理文本数据（每个token都有一定的概率被腐蚀为一个随机的token或者直接被吸收）。而本文则探索了连续扩散模型在文本生成中的应用,这是首次在连续潜在表示上研究文本扩散模型。连续潜在表示可以更好地支持基于梯度的可控文本生成方法。

- **自回归和非自回归语言模型**。 大多数预训练语言模型都是左到右的自回归模型,这限制了它们在可控文本生成任务中的灵活性,特别是那些需要同时控制左右文本上下文的任务,如填充和句法结构控制。为此,之前的工作提出了一些专门的训练和解码技术。而本文提出的扩散语言模型可以直接利用任意的分类器来控制复杂的全局属性。

- **即插即用的可控文本生成**。 即插即用的可控文本生成方法旨在保持语言模型不变,而通过潜在函数(如分类器)来引导生成过程。之前的工作提出了基于自回归语言模型的一些即插即用方法,如FUDGE、GeDi和DExperts等。

总的来说,Diffusion-LM提出的扩散语言模型在可控文本生成任务上取得了显著的进展,超越了之前的即插即用方法,并且可以处理更加复杂的控制需求。

## Problem Statement and Background

### 生成模型&文本可控合成

文本生成任务是指从经过训练的语言模型 $P_{\text{LM}}(w)$ 中采样 $w$ 的任务，其中 $w = [w_1 \circ w_2 \circ \cdots \circ w_n ]$ 为单词序列上的概率分布。可控文本生成是从条件分布 $p(w|c)$ 中采样 $w$ 的任务，其中 $c$ 代表控制变量。对于语法控制， $c$ 可以上图所示的目标语法树；而对于情感控制，$c$ 可能是期望的情感标签。可控生成的目标是生成满足控制目标 $c$ 的 $w$。

我们考虑即插即用的可控生成设置：在给定了一个从大量未标记文本数据中训练的语言模型 $P_{\text{LM}}(w)$，对于每个文本控制任务，我们得到了从少量标记文本数据训练的分类器 $p(c|w)$ （如对于语法控制，分类器是一个概率解析器）。我们的目标是利用这两个模型通过贝叶斯规则 $p(w|c)\propto p_{\text{LM}}\cdot p(c|w)$，从后验 $p(w|c)$ 来近似采样。这里我们可以如下理解我们的可控生成指标：$P_{\text{LM}}(w)$ 鼓励 $w$ 流畅，而 $p(c|w)$ 鼓励 $w$ 符合控制目标。

### 自回归语言模型

自回归的标准方法中语言建模因子 $P_{\text{LM}}$ 遵循从左向右的规则：
$$
P_{\text{LM}}(\textbf w) = p_{\text{LM}}(w_1)\cdot\prod_{i=2}^n p_{\text{LM}}(x_i|x_{<i})
$$

在这种情况下，我们可以考虑把文本生成简化为根据目前生成的部分序列重复预测下一个token的任务。而下一个token的预测 $p_{lm}(x_i | x_{<i})$ 通常由Transformer的体系结构来参数化。

### 连续域上的扩散模型

> 此部分为对论文的简单翻译，对于Diffusion Model(DDPM)更详细的介绍可以参考[DDPM](../AIGC/ddpm.md)

我们可以将扩散模型理解为将数据 $x_0\in \mathbb R ^d$ 建模为马尔可夫链 $x_T,\cdots, x_0$，$x_T$ 为高斯分布。扩散模型对隐变量序列 $x_{T:1}$ 去噪到服从目标数据分布的样本（如下图2所示）。扩散模型中，初始状态为 $p_\theta(\textbf{x}_T) \approx \mathcal N(0,\textbf{I})$，并且每一个去噪过程 $x_t \to x_{t-1}$ 都被参数化为模型 $p_\theta(\textbf{x}_{t-1} | \text{bf}_t) = \mathcal N (\textbf{x}_{t-1};\mu_\theta(\textbf{x}_t,t),\Sigma_\theta(\textbf{x}_t,t))$。通常来说， $\mu_\theta$ 和 $\sigma_\theta$ 可以通过 U-Net 或者 Transformer 来建模计算。

![](https://david-pigeon.github.io/notebook/images/Paper/Diffusion/Text_gen/diffulm_2.png)

为了训练扩散模型，我们定义了一个构造中间隐变量 $\mathbf x_{1:T}$ 的正向过程。该正向过程会逐步将高斯噪声添加到数据 $x_0$ 中，当扩散步骤达到一定数目 $T$ 时，样本 $\mathbf x_T$ 可以近似认为是高斯分布。我们对每个加噪过程 $\textbf x_{t-1}→\textbf x_t$ 通过 $q(\textbf x_t|\textbf x_{t-1})=\mathcal N(\textbf x_t;\sqrt{1-β_t}\textbf x_{t-1},β_t\textbf I)$ 进行参数化，其中超参数 $\beta_t$ 是在步骤 $t$ 中添加的噪声量。正向过程 $q$ 的参数化包含不可训练（也不用训练）的参数，允许我们定义一个训练目标，该目标包含有根据预定义的前向过程 $q$ 来生成噪声数据，并训练模型建模逆向的去噪过程以实现数据的重建。

训练扩散模型的目标是最大化数据的对数似然 $\mathbb E_{x_0  \sim p_{\text {data}}}[\log p_\theta(\mathbf x_0)]$，并且规范目标是 $\log p_\theta (\mathbf x_0)$ 的变分下界：

$$
\mathcal{L}_{vlb}(\mathbf{x}_0)=\underset{q(\mathbf{x}_{1:T}|\mathbf{x}_0)}{\operatorname*{\mathbb{E}}}\left[log\frac{q(\mathbf{x}_T|\mathbf{x}_0)}{p_{\boldsymbol{\theta}}(\mathbf{x}_T)}+\sum_{t=2}^Tlog\frac{q(\mathbf{x}_{t-1}|\mathbf{x}_0,\mathbf{x}_t)}{p_\theta(\mathbf{x}_{t-1}|\mathbf{x}_t)}-\log p_\theta(\mathbf{x}_0|\mathbf{x}_1)\right]
$$

但是，这个目标可能是不稳定的，需要许多优化技巧才能稳定。为了解决不容易收敛的问题，在后续的研究中提出了一个更加简单可替代的目标，该目标扩展并重新加权 $\mathcal L_{\text{vlb}}$ 中的每一个KL散度项，最终得到均方误差损失，如下所示：

$$
\mathcal{L}_{simple}(\mathbf{x}_0)=\sum_{t=1}^T\mathbb{E}_{q(\mathbf{x}_t|\mathbf{x}_0)}||\mu_\theta(\mathbf{x}_t,t)-\hat{\mu}(\mathbf{x}_t,\mathbf{x}_0)||^2
$$

其中 $\hat{\mu}(\mathbf{x}_t,\mathbf{x}_0)$ 是后验 $q(\mathbf x_{t-1}|\mathbf x_0,\mathbf x_t)$ 的均值，其接近高斯分布，$\mu_\theta(\mathbf{x}_t,t)$ 是由神经网络通过计算 $p_\theta(\mathbf x_{t-1} | \mathbf x_t)$ 得到的均值。尽管 $\mathcal{L}_{simple}$ 并不是变分下界，但在经验上发现，这能使得先验网络更容易训练并提高采样的质量。Diffusion-LM中也使用了类似地简化目标来稳定训练并提高采样质量。

## Diffusion-LM：Continuous Diffusion Language Modeling

构建Diffusion-LM需要对标准的扩散模型进行一些修改。首先，我们知道文本特征是离散的，因此我们必须定义一个embedding函数，将离散的文本映射到一个连续的空间。为了解决这个问题，作者提出了一个端到端的用于学习嵌入的训练目标（如4.1）。其次，我们需要一个函数将embedding空间中的向量映射回现实世界中的文本（单词）。为了更好的实现由连续空间（embedding）到离散空间（text）的映射，作者给出了一种训练和解码时间的方法来提升rounding的准确率。

### End-to-End Training

作者提出了一个embedding function $\textbf{EMB}(w_i)$ 用于将每个单词映射到一个 $\mathbb R^d$ 上的向量。特别的，对于一个长度为n的句子，我们的定义其嵌入为：

$$
\mathrm{EMB}(\mathbf{w})=[\mathrm{EMB}(w_1),\ldots,\mathrm{EMB}(w_n)]\in\mathbb{R}^{nd}.
$$

对于Diffusion Model的原始优化目标，我们通过联合学习扩散模型的参数和词嵌入来对该训练目标修改。在前置实验中，我们尝试了随机高斯嵌入，以及预训练的词嵌入。我们发现，与直接端到端的训练相比，使用固定嵌入方式对于Diffusion-LM而言是次优的（即表现不如直接end-to-end的训练）。当然需要指出的是，可训练的embedding方法在控制和生成任务中表现更好，但作者也在实验中发现，对于词汇的原型进行固定的embedding在优化困惑度时会有一定的帮助。

如上图所示，我们的方法在扩散模型前添加了一个将单词 $w$ 嵌入为 连续空间向量 $x_0$

