---
layout: post
title: Sqoop， HBase和Elastic Search的增量索引
categories: hadoop
tags: sqoop, hbase, elasticsearch
---

假设我们遇到一个问题，我们有一张MySQL的表，我们要实现对这个表里面的每个字段的搜索。

这里面最简单的是用SQL，比如说

	select * from table where column like '%abc%'

就不说这种方案无法实现abc的模糊匹配了，就性能来说，如果column没有索引，肯定就很慢了。如果要解决这个问题，就得动用到搜索引擎了。

开源的搜索引擎有很多，比如Sphinx，Luence， Elastic Search。这里我们选择了Elastic Search。ES是建立在luence之上的，它有很多比较好的特性。比如他支持对JSON的索引，而且支持非常友好的JSON查询。在这个基础上，我们甚至可以很容易的实现Knowledge Graph中的搜索（Freebase之前自己弄了一个基于knowledge graph的搜索，但效果一般）。缺点就是性能比Sphinx要差，不过相比于之前的MySQL解决方案，已经是非常快了，所以综合考虑，用了ES。

选择了搜索引擎，下一个问题是如何将mysql的每一行转化为document，发送给ES进行索引。这里我们选择用Sqoop将MySQL的表导入到HBase中，然后将HBase的每一行转化为document，进行索引。

这里增加了HBase这一层主要是基于以下的考虑：

1. 我们要索引的表可能存在MySQL上，也可能存储在Oracle上，或者DB2上。我们通过HBase将数据库和搜索引擎解耦。
2. HBase支持版本。对于每一个值都会纪录最近的几个版本。这使得HBase到搜索引擎的增量索引很容易实现。因为我们只需要索引版本发生变化的值。
3. HBase可以存储稀疏的数据，对于数据库中为NULL的列可以不存储。
4. 有的时候我们需要索引的文档来自很多表join之后的结果。直接在数据库上做大表的join会导致严重的性能问题。而HBase可以通过MapReduce 进行Join。

为了将不同数据库中的数据导入到HBase，我们用到了Sqoop。

所以整个系统的流程是

1. Sqoop将数据从关系型数据库导入到HBase
2. 定期跑MapReduce 任务，对不同的表进行Join，生成文档
3. 将文档发送给ES进行索引

好，下面总结一下解决这个问题的过程中可能遇到的坑

1. 如果用Sqoop导入表到HBase中，必须得指定一个MySQL的主键作为HBase的row key。但是默认情况下这个主键作为row key后就不会同时存储到value中。
2. 如果用MapReduce对HBase的表进行Join时，因为我们要拿到上个版本和当前版本的数据，需要设置 scan.setMaxVersions(2) 这个代表取最近的2个版本的数据。