# BERT 论文

[BERT: Pre-training of Deep Bidirectional Transformers for Language Understanding](https://arxiv.org/html/1810.04805v2)

[【BERT 论文逐段精读【论文精读】】](https://www.bilibili.com/video/BV1PL411M7eQ/?share_source=copy_web&vd_source=2261c9576a4b4cf4d5bae89613c8ab5a)

语言表示模型

### masked langue model （带掩码的模型）MLM

每一次随机选一些词元（token），将它盖住，目标函数是预测被盖住的字。类似于完形填空。

标准模型是从左看到右，MLM看左右信息。 双向的transformer模型。

附：下一个句子的预测。给两个句子，判断在原文中是不是相邻的或者是随机采样的两个句子。学习句子层的知识。

**双向信息的重要性。**

替换为 [MASK] BERT 训练时盖住 15% 的词来预测

### 算法
#### 预训练

在一个没有标号的数据上训练

#### 微调

使用一个 BERT 的模型。 权重初始化为在预训练中得到的权重。

所有的权重会被参与训练，用有标号的数据。

每一个下游任务是一个BERT模型，用最早哪个预训练好的BERT模型作为初始化，根据自己的数据训练好模型。