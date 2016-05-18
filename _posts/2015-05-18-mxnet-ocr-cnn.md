---
layout: post
title: 端到端的OCR：基于CNN的实现
categories: mxnet
tags: mxnet ocr cnn
---

OCR是一个古老的问题。这里我们考虑一类特殊的OCR问题，就是验证码的识别。传统做验证码的识别，需要经过如下步骤：

    1. 二值化
    2. 字符分割
    3. 字符识别

这里最难的就是分割。如果字符之间有粘连，那分割起来就无比痛苦了。

最近研究深度学习，发现有人做端到端的OCR。于是准备尝试一下。一般来说目前做基于深度学习的OCR大概有如下套路：

    1. 把OCR的问题当做一个多标签学习的问题。4个数字组成的验证码就相当于有4个标签的图片识别问题（这里的标签还是有序的），用CNN来解决。
    2. 把OCR的问题当做一个语音识别的问题，语音识别是把连续的音频转化为文本，验证码识别就是把连续的图片转化为文本，用CNN+LSTM+CTC来解决。

目前第1种方法可以做到90%多的准确率（4个都猜对了才算对），第二种方法我目前的实验还只能到20%多，还在研究中。所以这篇文章先介绍第一种方法。

我们以[python-captcha](https://pypi.python.org/pypi/captcha/0.1.1)验证码的识别为例来做验证码识别。

下图是一些这个验证码的例子：

![python-captcha](/static/img/captcha.png)

可以看到这里面有粘连，也有形变，噪音。所以我们可以看看用CNN识别这个验证码的效果。

首先，我们定义一个迭代器来输入数据，这里我们每次都直接调用python-captcha这个库来根据随机生成的label来生成相应的验证码图片。这样我们的训练集相当于是无穷大的。

    `
    class OCRIter(mx.io.DataIter):
    def __init__(self, count, batch_size, num_label, height, width):
        super(OCRIter, self).__init__()
        self.captcha = ImageCaptcha(fonts=['./data/OpenSans-Regular.ttf'])
        self.batch_size = batch_size
        self.count = count
        self.height = height
        self.width = width
        self.provide_data = [('data', (batch_size, 3, height, width))]
        self.provide_label = [('softmax_label', (self.batch_size, num_label))]

    def __iter__(self):
        for k in range(self.count / self.batch_size):
            data = []
            label = []
            for i in range(self.batch_size):
                # 生成一个四位数字的随机字符串
                num = gen_rand() 
                # 生成随机字符串对应的验证码图片
                img = self.captcha.generate(num)
                img = np.fromstring(img.getvalue(), dtype='uint8')
                img = cv2.imdecode(img, cv2.IMREAD_COLOR)
                img = cv2.resize(img, (self.width, self.height))
                cv2.imwrite("./tmp" + str(i % 10) + ".png", img)
                img = np.multiply(img, 1/255.0)
                img = img.transpose(2, 0, 1)
                data.append(img)
                label.append(get_label(num))

            data_all = [mx.nd.array(data)]
            label_all = [mx.nd.array(label)]
            data_names = ['data']
            label_names = ['softmax_label']

            data_batch = OCRBatch(data_names, data_all, label_names, label_all)
            yield data_batch

    def reset(self):
        pass
    `