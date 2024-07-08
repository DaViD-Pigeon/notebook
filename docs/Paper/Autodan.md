---
counter: True   
---

# AutoDAN：基于遗传算法的LLM越狱攻击

<div class="badges">
<span class="badge not-implement-badge">未完成</span>
</div>

!!! Brief
    ![](https://david-pigeon.github.io/notebook/images/Paper/Jail/Autodan_0.png)
    
    * Paper: [AutoDAN: Generating Stealthy Jailbreak Prompts on Aligned Large Language Models](https://arxiv.org/abs/2310.04451)
    * Code: [AutoDAN](https://github.com/SheltonLiu-N/AutoDAN)
    * 该工作已被ICLR 2024接收
    * GCG的优化：生成可读性较强的对抗性攻击后缀
    * 本文中的图片均来自论文。

## Abstract

文件介绍了一种名为 AutoDAN 的新型越狱攻击,它可以自动生成隐蔽的提示来绕过对齐的大型语言模型(LLMs)的安全功能。关键要点如下:

- 介绍了 AutoDAN,这是一种针对对齐 LLMs 的新型越狱攻击,可以自动生成隐蔽的提示。
- 提出了一种针对结构化离散数据(如提示文本)的分层遗传算法,以解决从手工制作的提示初始化的细粒度空间进行搜索的挑战。
- 证明了 AutoDAN 在攻击强度、可迁移性和针对 LLMs 的普遍性方面优于基线方法。
- 表明 AutoDAN 可以有效绕过基于困惑度的防御,而之前的基于token级别的越狱攻击做不到。