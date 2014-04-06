---
layout: post
title: 基于Phantomjs和Casperjs的AJAX爬虫
categories: ajax crawler
tags: ajax crawler
---

很多网站用Ajax来展示内容，这样的好处是前端和后端开发的耦合更低，但也给搜索引擎的数据抓取带来的一定的困难。目前解决AJAX抓取，基本上的思路有2个：
1. 使用浏览器内核，比如WebKit
2. 使用前端的自动化测试工具，比如WebUnit，Selenium。

不过之前的解决方案在速度和可靠性上都不是特别的好。最近发现了一个叫做Phantomjs的东西，它的速度和稳定性都比较好。

直接用Phantomjs来写抓取的代码还是稍微有些麻烦。于是一个叫做Casperjs的东西基于Phantomjs写了一个更容易用的框架。比如下面是一个模拟人人网登陆的程序：

	var casper = require('casper').create({
	    verbose: false,
	    logLevel: 'debug'
	});

	casper.start('http://www.renren.com/');

	casper.wait(500, function(){
	    this.capture("start.png");
	});

	casper.wait(500, function(){
	    this.fill('form#loginForm', {email: 'your@email.com', password: 'your password' }, false);
	});

	casper.then(function(){
	    this.click("#login");
	});

	casper.wait(500, function(){
	    this.capture("after.png");
	    this.exit();
	});

	casper.run();
