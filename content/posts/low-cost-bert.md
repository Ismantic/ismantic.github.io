+++
date = '2026-07-13T10:00:00+08:00'
draft = false
title = '低成本实践：训练BERT'
+++

{{< rawhtml >}}
<p style="text-align: right; font-style: italic; color: var(--secondary); margin: 1.5rem 0 2.5rem;">
山穷水尽疑无路，柳暗花明又一村
</p>
{{< /rawhtml >}}

预训练刚兴起的时候，有三个代表性的工作：ELMo、GPT 和 BERT。它们最初想解决的，其实都是 Context-Aware 的词表示问题。ELMo 使用 LSTM，GPT 和 BERT 则转向 Transformer。那时 GPT 在很多下游任务上还不如 BERT，BERT 之后又出现了 RoBERTa、ELECTRA、MacBERT 等一系列改进，当年也都跟进过。

之后大语言模型的发展远远超出了当时的想象。Decoder-Only 模型不仅可以生成，还能通过少量样本、指令训练和强化学习去完成各种任务。相比之下，BERT 只能编码，能做的事情少得多，模型继续变大以后，很多传统任务也早已接近天花板。

但“能做的事情少”不等于没有价值。在一些目标明确、调用频繁、对延迟敏感的任务上，BERT 仍然合适。中文错别字纠正是字符级预测，CWS、POS、NER 也是典型的字序列标注；如果每次请求都交给生成式大模型，成本和延迟都太高。于是想重新训练一次 BERT，看看把近几年大模型积累的架构和训练经验放回 Encoder-Only 模型，还能做到什么程度。

## BERTc：重新训练一个中文 BERT

实验代码放在 [BERTc](https://github.com/Ismantic/BERTc)。它不是在现有中文 BERT 上继续微调，而是从头训练的字符级中文模型。

Tokenizer 对中文采用“一字一个 Piece”，英文则使用 BPE，整个词表只有 12536。预训练任务仍然是 15% Masked Language Modeling，但 Whole Word Masking 所需的中文词边界由 Wapic 分词提供。也就是说，模型输入保持字符级，Mask 时却能够按词处理。

模型结构也没有照搬 2018 年的 BERT，而是吸收了一些后来被证明有效的做法：Pre-Norm、GeGLU、无 Bias LayerNorm、简化 MLM Head、Megatron 初始化、StableAdamW 和 Damped Cosine 学习率。训练数据约 17.65B Token，165M 和 316M 两个版本都可以在单张 RTX 4090 上训练。

最终主要评估两个方向：一是 CWS、POS、NER 联合训练，二是 SIGHAN-15 中文拼写纠错。

| 模型 | 参数量 | CWS F1 | POS F1 | NER F1 | 综合分 |
|---|---:|---:|---:|---:|---:|
| BERTc-165M | 165M | 0.9836 | 0.9753 | 0.9632 | 1.4689 |
| **BERTc-315M** | **316M** | 0.9840 | **0.9800** | 0.9660 | **1.4712** |
| MacBERT-Large | 326M | **0.9856** | 0.9629 | **0.9664** | 1.4677 |
| RoBERTa-wwm-ext | 102M | 0.9828 | 0.9562 | 0.9629 | 1.4623 |

| 模型 | CSC F1 | Precision | Recall |
|---|---:|---:|---:|
| **BERTc-315M-CSC** | **0.8346** | 0.9407 | 0.7480 |
| MacBERT4CSC | 0.8314 | 0.9274 | 0.7534 |
| MacBERT-Large CSC | 0.8309 | 0.9302 | 0.7507 |
| BERTc-165M-CSC | 0.8308 | **0.9516** | 0.7373 |

模型从 165M 增长到 316M，四个任务都有提升，但提升幅度并不算大。这个现象和我原来的判断一致：BERT 能解决的任务相对固定，很多指标也已经接近天花板，因此很难像 GPT 那样通过不断 Scale Up 获得新的能力。不过，小幅提升并不等于没有意义。315M 版本在多任务综合分和 CSC 上都超过了对比的 MacBERT-large，尤其 POS 提升比较明显。

## BERT 再回到 CRF

BERT 还有一个现实问题：这些字符级任务通常对延迟很敏感。即使模型不大，只用 CPU 推理仍然有些慢。于是可以让 BERT 充当 Teacher，生成大规模标注语料，再用这些数据训练 CRF。这样做有点像绕了一圈：先用更强的模型获得泛化能力，再把能力压回一个简单、快速、容易部署的模型里。

[Wapic](https://github.com/Ismantic/Wapic) 就是这条路线的产物。它是一个 C++17 线性链 CRF 工具，支持中文分词训练和 BMES 解码。BERTc 与 Wapic 之间形成了一个有意思的闭环：

1. Wapic 为 BERTc 的 Whole Word Masking 提供中文分词；
2. 训练好的 BERT 模型再生成分词语料；
3. Wapic 用蒸馏语料重新训练，得到低延迟的发布模型。

发布数据分为两个阶段：第一阶段约 950 万句蒸馏语料，第二阶段约 439 万句针对 Bad Case 的增强语料。为了避免训练集与测试集重合，最终指标使用了去污染后的测试集：

| Test Set | F1 |
|---|---:|
| PD-1998 | **98.01** |
| News | **97.95** |

如果只使用人民日报 1998 的 1—5 月数据训练，再用 6 月测试，几分钟也能得到约 97.4 的 F1。蒸馏真正带来的收益，不一定能在同分布的小测试集上完全体现；当测试分布更广、人物和新词更多时，大规模蒸馏数据的价值才会明显。

## BERT 还有没有价值

大语言模型当然可以完成分词、词性标注、实体识别和错别字纠正，甚至不需要专门训练。但当任务已经确定，需要离线运行，需要每秒处理大量文本，或者只允许很小的延迟时，BERT 和 CRF 仍然有各自的位置。

这次实践最后不是简单地证明“BERT 还没过时”。更准确地说，是把不同模型放回它们适合的约束里：BERT 负责学习更复杂的上下文，CRF 负责把结果快速、稳定地交付。模型新不新并不是唯一标准，能不能解决真实问题才是。

## 项目与模型

- GitHub：[BERTc](https://github.com/Ismantic/BERTc)
- GitHub：[Wapic](https://github.com/Ismantic/Wapic)
- Hugging Face：[BERTc-165M](https://huggingface.co/Ismantic/BERTc-165M)
- Hugging Face：[BERTc-315M](https://huggingface.co/Ismantic/BERTc-315M)
- Hugging Face：[BERTc-165M-MT](https://huggingface.co/Ismantic/BERTc-165M-MT)
- Hugging Face：[BERTc-315M-MT](https://huggingface.co/Ismantic/BERTc-315M-MT)
- Hugging Face：[BERTc-165M-CSC](https://huggingface.co/Ismantic/BERTc-165M-CSC)
- Hugging Face：[BERTc-315M-CSC](https://huggingface.co/Ismantic/BERTc-315M-CSC)
- Hugging Face：[Wapic-CWS](https://huggingface.co/Ismantic/Wapic-CWS)
- Hugging Face：[Wapic-CWS-Data](https://huggingface.co/datasets/Ismantic/Wapic-CWS-Data)
