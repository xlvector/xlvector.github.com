---
layout: post
title: Golang, too many opened files
categories: golang
tags: golang
---

如果用Golang写服务器，运行时间长了有可能系统会卡死，然后看exception，会发现是由于打开太多文件了，超过了操作系统的限制。这当然是因为程序里面打开了很多文件，忘记close了，随着时间的推延，总会超过操作系统的限制(Linux 一个进程默认最多打开1024个文件)。解决这个问题有2个思路：

* 提高操作系统的限制，比如从1024变成2048：这个解决方案治标不治本
* 及时的关闭打开的文件：这个问题的关键是找到程序中哪儿忘记关闭文件了

如果要查看那些文件被打开了，可以执行下面的命令：

	ls -l /proc/{pid}/fd

这里，{pid}是你关心的进程的ID。

一般来说，有几种文件比较容易忘记关闭。

## HTTP 请求

有的时候，我们的程序需要向其他HTTP服务发送一个请求。某些时候，我们不太关心对方服务器的返回是什么，所以我们发送之后就不管了。在Golang，这会造成问题。所以正确的做法应该是下面的代码：

	func PostHTTPRequest(host string, data map[string]string) string {
		post := url.Values{}
		for key, value := range data {
			post.Set(key, value)
		}
		resp, err := http.PostForm(host, post)

		if err != nil {
			log.Println(err)
		}
		if resp != nil && resp.Body != nil {
			defer resp.Body.Close()
			output, err := ioutil.ReadAll(resp.Body)
			if err != nil {
				log.Println(err)
			}
			return string(output)
		}
		return ""
	}

这里，resp里面的Body，无论我们要不要用到，都得记住关闭。