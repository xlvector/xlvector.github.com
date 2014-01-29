---
layout: post
title: Flume Spooldir 源的一些问题
categories: hadoop
tags: flume
---

最近在用Flume做数据的收集。用到了里面的Spooldir的源在使用中有如下的问题：

* 如果文件的某一行有乱码，不符合指定的编码规范，那么flume会抛出一个exception，然后就停在那儿了。
* spooldir指定的文件夹中的文件一旦被修改，flume就会抛出一个exception，然后停在那儿了。

其实，flume的最大问题就是不够鲁棒。一旦出现问题，不能跳过，只能死在那儿。不知道flume为什么要这么设计。理论上，它应该允许我们在配置文件中指定在遇到错误的行时，是停止还是跳过，不过它目前并不支持这个。所以，我们只能写一个自己的flume的插件了。

	https://github.com/xlvector/flume
	https://github.com/ponyma/flume

这个插件主要修复了前面提到的两个问题：

* 如果某一行有乱码，flume会忽略这一行
* flume只会check最近N分钟没有修改过的文件
