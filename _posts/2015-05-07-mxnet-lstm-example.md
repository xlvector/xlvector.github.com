---
layout: post
title: 用LSTM做排序
categories: mxnet
tags: mxnet lstm sort
---

LSTM是一种重要的用于序列分析的人工神经网络。它可以被用来做分词，机器翻译，文本摘要，语言生成，语音识别，OCR等等序列相关的问题。本文用一个很简单的例子来说明LSTM如何做排序。

假设我们有一个输入字符串：

WC2V9 EGONE I6V72 Q54NY ZO25W

排序之后就是

EGONE I6V72 Q54NY WC2V9 ZO25W

我们知道LSTM的训练需要给两个序列，一个是输入序列，一个是目标序列。那么在上面这个排序问题中，输入字符串就是：

WC2V9 EGONE I6V72 Q54NY ZO25W _____ _____ _____ _____ _____

而目标序列就是

_____ _____ _____ _____ _____ EGONE I6V72 Q54NY WC2V9 ZO25W

我们用mxnet的LSTM来训练这个模型。

首先我们用如下的代码生成了100行随机字符串，每行5个字符串。100万行中95万行用来训练，5万行用来测试。

    import string, random, sys

    N = int(sys.argv[1])

    vocab = []

    for i in range(200):
        vocab.append(''.join(random.choice(string.ascii_uppercase + string.digits) for _ in range(5)))

    for i in range(N):
        n = 5
        line = ''
        for j in range(n):
            line += vocab[random.randint(0, len(vocab) - 1)]
            line += ' '
        print line.strip()

然后，我们简单修改一下mxnet/examples/rnn/lstm_bucket.py 相关的代码。
只要修改bucket_io.py。这里原来的例子是用来训练语言模型的。所以需要修改一下从输入序列得到目标序列的过程。

    def default_text2id(sentence, the_vocab):
        words = sentence.split(' ')
        words = [the_vocab[w] for w in words if len(w) > 0]
        count = len(words)
        # 这里需要在原来的输入后面加上和输入等长的空白字符串，然后目标字符串就直接是输入字符串的排序（空白排在最前面）
        for i in range(count):
            words.append(0)
        return words

全部的代码在 https://github.com/xlvector/learning-dl/tree/master/mxnet/lstm_sort

下面是一些例子：

    29YNZ _
    23XE1 _
    EGONE _
    0BL70 _
    R5ENV _
    _  0BL70
    _  23XE1
    _  29YNZ
    _  EGONE
    _  SHVZG


    2S6AG _
    J4OOU _
    UKB7P _
    AW4ZJ _
    UEFHR _
    _ 2S6AG
    _ AW4ZJ
    _ J4OOU
    _ SHVZG
    _ UV5G9

这两个例子可以看到，输出都是排好序的，但是输出的字符串不一定都是输入的字符串。这说明LSTM并没有完全学到如何排序，但基本学会了。
