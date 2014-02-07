---
layout: post
title: Parse Excel with Python
categories: python
tags: python
---

Excel一般要通过微软的工具比如C#才可以解析，不过其实Python也可以解析，而且不仅仅windows平台的python可以，mac和linux的python都可以。python解析主要通过一个叫做xlrd的库，可以按照如下方法安装：

	easy_install xlrd

可以通过这个地方 https://secure.simplistix.co.uk/svn/xlrd/trunk/xlrd/doc/xlrd.html?p=4966 看到它的相关文档。

其他关于python操作excel的方法可以参考 http://www.python-excel.org/

下面这段代码是一个xlrd的例子：

	import xlrd
	import sys

	fname = sys.argv[1]

	workbook = xlrd.open_workbook(fname)
	sheets = workbook.sheets()

	print 'sheet count : ', len(sheets)

	for sheet in sheets:
	    for i in range(sheet.nrows):
	        print '\t'.join([unicode(x).encode('utf-8') for x in sheet.row_values(i)])