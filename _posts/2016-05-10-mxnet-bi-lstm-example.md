---
layout: post
title: 用LSTM做排序 - Bidirectional LSTM
categories: mxnet
tags: mxnet lstm sort
---

上次提到用LSTM做排序。有一个问题是普通的LSTM需要看到完整的序列后才能进行排序。这就导致我们设计时不得不输入2倍于原字符串长度的字符串。前一半是原字符串，后一半是空格。然后后一半的空格会输出排序好的字符串。

但如果改成Bidirectional LSTM，就没有这个问题了（多谢yiwang的提醒）。BiLSTM总是在读完整个字符串之后才开始解码。关于它的具体介绍网上可以找到。

折腾了2天，在mxnet原来的lstm的基础上实现了一个bi-lstm。相关的代码见：[https://github.com/xlvector/mxnet/tree/master/example/bi-lstm-sort](https://github.com/xlvector/mxnet/tree/master/example/bi-lstm-sort)