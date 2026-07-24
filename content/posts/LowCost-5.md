+++
date = '2026-07-17T09:00:00+08:00'
draft = true
title = '低成本实践：训练BERT（二）'
+++

{{< rawhtml >}}
<p style="text-align: right; font-style: italic; color: var(--secondary); margin: 1.5rem 0 2.5rem;">
安能摧眉折腰事权贵，使我不得开心颜
</p>
{{< /rawhtml >}}

BERT 的训练，其实比较像训练 W2V ，不过因为是 GPU 上训练，不能像 CPU 上那么灵活，什么负采样，什么霍夫曼树都不是很方便，因而就搞了个 15% Mask 的操作，这个训练任务也就被称之为完形填空。再应用到下游人物的时候，还需要把Head再去掉，继续训练。这点上其实不如GPT，GPT除了引入新的Token，就不需要引入新的模型组件了。

尝试训练了下165M和315M的BERT模型，用了中文字符/英文BPE的Tokenizer，模型结构上能简则简，训练时间上315M是165M的二倍以上，以CSC任务来看，F1 0.8308 -> 0.8346，多任务训练CWS/POS/NER来看，CWS F1 0.9836 -> 0.9840，POS 0.9753 -> 0.9800，NER 0.9632 -> 0.9660。四个任务确实也都有提高，但其实幅度很小了，这就是BERT的问题，因为只能做这样的任务，效果到天花板了，就不能Scale Up了。
