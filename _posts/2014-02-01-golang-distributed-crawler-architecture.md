---
layout: post
title: 基于Golang的分布式爬虫框架
categories: golang crawler
tags: golang crawler architecture
---

爬虫是数据搜集系统中的一个重要工具，主要用于从Web上搜集数据。一般一个爬虫的流程如下：

	func Crawler(seed_url):
		Q = Queue()
		Q.append(seed_url)
		while !Q.empty() :
			link = Q.pop()
			html = download(link)
			sub_links = extract_links(html)
			for link in sub_links:
				Q.append(link)

如果我们要设计一个分布式爬虫，可以将上述程序分成3个部分：

1. downloader : 输入一个url，返回这个url对应的HTML
2. link extractor : 输入一个HTML，返回这个HTML中的链接
3. redirector : 负责接收link extractor提取的链接，并且将这些链接转发给downloader

不过，因为downloader主要消耗网络资源，而link extractor主要消耗CPU资源，因此我们可以将1，2部分合并在一个程序中。我们把合并后的程序也称为downloader。downloader集群通过nginx做负载均衡和外面通信。而downloader集群的鲁棒性可以通过nginx的health check实现。

## Downloader

downloader的任务就是
1. 给定一个url，将他对应的HTML下载下来，写到磁盘上
2. 从HTML中提取链接，并将提取出来的链接发送给redirector

下载网页这一步是需要解决一下的问题：
1. 如何通过HTTP GET下载网页（这个是最简单的）
2. 如何使用HTTP 代理
3. 如何通过HTTP POST下载网页，这个涉及到表单的自动post
4. 如何处理页面的Javascript，获得Ajax的调用内容
5. 如何处理非HTML网页，比如PDF，Excel等等

## Redirector

redirector的任务是
1. 控制每个不同网站的爬取速率
2. 控制网站爬取的优先级
3. 控制网页的更新速率（同一个网页，什么时候再爬一次）

所以，redirector从downloader接受到新的链接后，要对链接进行排序，整理，再发送给downloader下载。从而redirector和downloader之间形成闭环。

## Golang

golang在实现上述逻辑时，可以充分发挥它强大的channel特性。

比如，如何控制不同网站的爬取速率？

可以给每个域名建立一个channel，每个域名的网页都进入到各自的channel。然后channel的消费者按照一定的速率从channel中取出链接进行消费。

不过使用channel也有一个问题。就是channel是阻塞的，如果channel满了，一定要消费者消费一个才能往里面放东西。而因为我们整个系统是一个环状系统：

1. downloader 有一个input的queue A
2. redirector 有一个input的queue B
3. redirector将B中的链接放入A
4. download消费A中的链接，得到新链接，放入到B

这时候，如果A，B都满了，redirector和downloader就会出现死锁，因为他们都需要别人先动一下，自己才能动。

因此，我们就需要设计一个逻辑，让所有的queue都不可能满。这样就只能利用文件系统了。

1. 如果queue满了，将链接写文件
2. 如果queue没有满，将链接写入queue中
3. 如果queue空闲了，将文件中的链接写入queue
