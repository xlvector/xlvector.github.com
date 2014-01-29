---
layout: post
title: CentOS install Graphite and Statsd
categories: golang
tags: golang
---

Graphite是一个时序序列的画图软件，经常用来监控各种指标随时间的变化趋势。Statsd基于Graphite提供更强大的统计能力。下面的脚本包括了在CentOS上安装这两个工具的全部命令。
	
	＃ Graphite由3个部分组成: whisper, carbon, graphite-web
	# Whisper提供了一个存储时间序列的数据库
	# Carbon提供了一个缓存服务，它接受数据请求，然后将数据写到Whisper中。因为Carbon服务需要比较高的并发能力，所以他是基于Twisted的
	# Graphite web提供了一个可视化的画图界面，它通过carbon从whisper中拿到数据，然后暂时出来。这个网站是基于Dijango的

	yum install -y pycairo mod_python Django python-ldap python-memcached python-sqlite2  bitmap bitmap-fonts python-devel python-crypto pyOpenSSL gcc python-zope-filesystem python-zope-interface git gcc-c++ zlib-static

	wget https://pypi.python.org/packages/source/s/setuptools/setuptools-2.0.2.tar.gz --no-check-certificate
	tar zxvf setuptools-2.0.2.tar.gz
	cd setuptools-2.0.2
	python setup.py install
	cd ..

	easy_install pip
	pip install 'Twisted<12.0'

	wget https://django-tagging.googlecode.com/files/django-tagging-0.3.1.tar.gz --no-check-certificate
	tar zxvf django-tagging-0.3.1.tar.gz
	cd django-tagging-0.3.1
	python setup.py install
	cd ..

	wget https://github.com/downloads/graphite-project/graphite-web/graphite-web-0.9.10.tar.gz
	wget https://github.com/downloads/graphite-project/carbon/carbon-0.9.10.tar.gz
	wget https://github.com/downloads/graphite-project/whisper/whisper-0.9.10.tar.gz

	tar zxvf whisper-0.9.10.tar.gz
	cd whisper-0.9.10
	python setup.py install
	cd ..

	tar zxvf carbon-0.9.10.tar.gz
	cd carbon-0.9.10
	python setup.py install
	cd ..

	tar zxvf graphite-web-0.9.10.tar.gz
	cd graphite-web-0.9.10
	python check-dependencies.py
	python setup.py install
	cd ..

	cd /opt/graphite/webapp/graphite
	cp local_settings.py.example local_settings.py
	python manage.py syncdb

	cd /opt/graphite/conf/
	cp carbon.conf.example carbon.conf
	cp storage-schemas.conf.example storage-schemas.conf

	cd /opt/
	git clone https://github.com/joyent/node.git
	cd node
	./configure
	make
	make install

	cd /opt/
	git clone https://github.com/ktmud/statsd.git
	cd statsd
	echo "{graphitePort:2003,graphiteHost:\"127.0.0.1\",port:8125}" > localConfig.js

	## Start Service

	/opt/graphite/bin/carbon-cache.py start
	nohup /opt/graphite/bin/run-graphite-devel-server.py /opt/graphite > log &

	cd /opt/statsd/
	node stats.js localConfig.js