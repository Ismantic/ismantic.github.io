+++
date = '2026-07-18T09:00:00+08:00'
draft = false
title = '低成本实践：训练BERT（三）'
+++

{{< rawhtml >}}
<p style="text-align: right; font-style: italic; color: var(--secondary); margin: 1.5rem 0 2.5rem;">
若未自由故，两者皆可抛
</p>
{{< /rawhtml >}}

再来看下蒸馏，其实就是用BERT模型生成训练数据，再用来训练CRF。

F1=98.01, P=97.88, R=98.14，这是用PeopleDaily测试集的效果。不过只用PeopleDaily本身训练的话，F1也能到0.97了，似乎说明收益不大。其实这是因为PeopleDaily的多样性不够，模型很快就学到头了，要是改用蒸馏数据集的分布的测试的话，F1仅有0.92，而用蒸馏数据的话，F1=97.95，这个数值上就能体现出差异了。

低成本实践目前我主要就是做了这几个，还是该补上GPT的预训练，不然是不够完整的，进一步的，应该要试试强化学习让GPT写代码，这个就留给后面做了。