---
layout: post
title: PHP file_get_contents Permission Denied
categories: php
tags: php
---

如果你在PHP用file_get_contents失败了，查看日志发现是Permission Denied，请尝试下面的命令：

	/usr/sbin/setsebool httpd_can_network_connect=1

在我这儿就OK了。

来自 http://www.php.net/manual/en/function.fopen.php#56551