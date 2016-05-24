---
layout: post
title: 离散特征和连续特征同时存在，同时解决回归和分类的问题
categories: mxnet
tags: mxnet regression classification
---

有些同学对于mxnet的自定义Iter不是很熟悉，对多输出也不熟悉，因此我用一个比较复杂的例子来说明这个问题：

    1. 特征中有连续特征和离散特征
    2. 同时要解决回归问题和分类问题

本着End-to-End的精神，我们不做特征工程，当然也就不能做离散化。于是，连续特征可以直接作为输入，而离散特征则通过Embeding的方式输入。如果要同时解决回归和分类问题，我们就需要两个Loss层。

我们虚构一个简单的二手车价格预估的问题。我们假设一辆车的价格只取决于两个因素，一个是车的品牌，一个是车的里程。不同品牌的车有不同的出厂价格，而车的行驶里程越长，价格就会越低。因此我们可以基于这个假设，用如下的代码构造一个数据集：

    #我们虚构了201个不同的品牌，并给每个品牌设置一个出场价格
    series = [1 + i for i in range(100)] + [101 - i for i in range(100)]

    for i in range(10000):
        k = random.randint(0, 199)
        #越贵的品牌，我们认为在数据集里出现的次数越少，因为它卖的少
        count = 1000 / series[k]
        for j in range(count):
            dis = random.random() * 10
            #实际的价格是品牌的出场价除以里程数的开方
            price = series[k] / math.sqrt(1.0 + dis)
            print str(price) + '\t' + str(dis) + '\t' + str(k)

这里，车的品牌是一个离散特征，而里程是个连续的特征。问题的目标是，给定品牌和里程，同时预测车的价格（回归问题），以及车的价格区间（分类问题）。我们用如下的网络来解决这个问题：

    # dis 是输入的里程
    dis = mx.symbol.Variable('dis')
    # price 是要预测的目标价格
    price = mx.symbol.Variable('price')
    # price_interval 是要预测的价格区间
    price_interval = mx.symbol.Variable('price_interval')
    # series 是输入的车的品牌
    series = mx.symbol.Variable('series')

    dis = mx.symbol.Flatten(data = dis, name = "dis_flatten")
    series = mx.symbol.Embedding(data = series, input_dim = 200,
                                 output_dim = 100, name = "series_embed")
    series = mx.symbol.Flatten(series, name = "series_flatten")

    net = mx.symbol.Concat(*[dis, series], dim = 1, name = "concat")
    net = mx.symbol.FullyConnected(data = net, num_hidden = 100, name = "fc1")
    net = mx.symbol.Activation(data = net, act_type="relu")
    net = mx.symbol.FullyConnected(data = net, num_hidden = 100, name = "fc2")
    net = mx.symbol.Activation(data = net, act_type="relu")
    net = mx.symbol.FullyConnected(data = net, num_hidden = 1, name = "fc3")
    # 这里最后为什么用relu呢？是因为价格一定是个正数
    net = mx.symbol.Activation(data = net, act_type="relu")
    net = mx.symbol.LinearRegressionOutput(data = net, label = price, name = "lro")

    net2 = mx.symbol.Concat(*[dis, series], dim = 1, name = "concat")
    net2 = mx.symbol.FullyConnected(data = net2, num_hidden = 100, name = "fc21")
    net2 = mx.symbol.Activation(data = net2, act_type="relu")
    net2 = mx.symbol.FullyConnected(data = net2, num_hidden = 100, name = "fc22")
    net2 = mx.symbol.Activation(data = net2, act_type="relu")
    net2 = mx.symbol.FullyConnected(data = net2, num_hidden = 8, name = "fc23")
    net2 = mx.symbol.Activation(data = net2, act_type="relu")
    net2 = mx.symbol.SoftmaxOutput(data = net2, label = price_interval, name="sf")

    ＃ 这里net预测price，net2预测price_interval, 最后group在一起返回
    return mx.symbol.Group([net, net2])

这个例子里，我们需要同时提供dis, series, price, price_interval 四个变量。常见的Iter似乎不支持这个功能，因此可以自己实现一个：

    class PriceIter(mx.io.DataIter):
        def __init__(self, fname, batch_size):
            super(PriceIter, self).__init__()
            self.batch_size = batch_size
            self.dis = []
            self.series = []
            self.price = []
            ＃ 这里预先从文件读入所有的数据存下来
            for line in file(fname):
                price, d, s = line.strip().split("\t")
                self.price.append(float(price))
                self.series.append(np.array([int(s)], dtype = np.int))
                self.dis.append(np.array([float(d) / 10.0]))

            ＃ 输入数据的shape
            self.provide_data = [('dis', (batch_size, 1)),
                                 ('series', (batch_size, 1))]
            # 输出数据的shape
            self.provide_label = [('price', (batch_size, )),
                                  ('price_interval', (batch_size,))]

        def __iter__(self):
            count = len(self.price)
            for i in range(count / self.batch_size):
                bdis = []
                bseries = []
                blabel = []
                blabel_interval = []
                for j in range(self.batch_size):
                    k = i * self.batch_size + j
                    bdis.append(self.dis[k])
                    bseries.append(self.series[k])
                    blabel.append(self.price[k])
                    blabel_interval.append(interval(self.price[k]))

                data_all = [mx.nd.array(bdis),
                            mx.nd.array(bseries)]
                label_all = [mx.nd.array(blabel), mx.nd.array(blabel_interval)]
                data_names = ['dis', 'series']
                label_names = ['price', 'price_interval']

                data_batch = Batch(data_names, data_all, label_names, label_all)
                yield data_batch

        def reset(self):
            pass

全部的例子见[这里](https://gist.github.com/xlvector/c304d74f9dd6a3b68a3387985482baac)