---
layout: post
title: Hive Hbase的集成
categories: hadoop
tags: hive hbase
---

Hive和HBase是Hadoop生态圈的两个重要工具。Hive是个数据仓库，主要是用来做统计报表等BI的功能的。而HBase是一个分布式的key value存储引擎。我们考虑用将这两个著名工具结合起来主要是考虑如下的使用场景：

1. 我们有很多数据存储在HBase上，他们会被按照key查询
2. 我们希望支持对这些数据的统计分析，而又不想自己写map reduce
3. 我们想支持查询里面的部分数据，而不想自己写map reduce

因此，我们选择用hive来支持我们的两个需求：统计数据和方便的拿出部分数据

下面来说具体如何配置。

假设我们有一个HBase表users。其中rowkey是user_id，column family只有一个，是f。user有name和gender两个属性。

然后，我们可以通过如下语句建立一个hive的表

	hive --auxpath /usr/lib/hive/lib/hive-hbase-handler-0.10.0-cdh4.5.0.jar,/usr/lib/hive/lib/hbase.jar

	> CREATE external TABLE users(id bigint, name string, gender string)
	STORED BY 'org.apache.hadoop.hive.hbase.HBaseStorageHandler'
	WITH SERDEPROPERTIES ('hbase.columns.mapping' = ':key,f:name,f:gender')
	TBLPROPERTIES ('hbase.table.name' = 'users');

	> select count(*) from users;

这里，hive要使用hbase，需要用到两个jar包hive-hbase-handler-0.10.0-cdh4.5.0.jar, hbase.jar。

不过，Hive使用Hbase，是将sql转化成map reduce，速度不是非常的快。还有一个选择，就是使用Apache Phoenix。这个工具将sql转化成Hbase的Scan，而不使用Map Reduce。

关于HBase和Hive的集成，可以参考下面的文章：

1. [Hbase via Hive part 1](http://hortonworks.com/blog/hbase-via-hive-part-1/)
2. [Hbase via Hive part 2](http://hortonworks.com/blog/using-hive-to-interact-with-hbase-part-2/)

