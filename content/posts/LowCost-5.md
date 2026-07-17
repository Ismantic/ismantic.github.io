+++
date = '2026-07-17T09:00:00+08:00'
draft = false
title = '低成本实践：训练BERT（二）'
+++

{{< rawhtml >}}
<p style="text-align: right; font-style: italic; color: var(--secondary); margin: 1.5rem 0 2.5rem;">
安能摧眉折腰事权贵，使我不得开心颜
</p>
{{< /rawhtml >}}

BERT 的训练，其实比较像训练 W2V ，不过因为是 GPU 上训练，不能像 CPU 上那么灵活，什么负采样，什么霍夫曼树都不是很方便，因而就搞了个 15% Mask 的操作，这个训练任务也就被称之为完形填空。再应用到下游人物的时候，还需要把Head再去掉，继续训练。这点上其实不如GPT，GPT除了引入新的Token，就不需要引入新的模型组件了。

