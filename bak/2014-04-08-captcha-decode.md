---
layout: post
title: CAPTCHA 验证码识别
categories: captcha
tags: captcha image cv
---

验证码是一个著名的图灵测试问题，它的目的就是将人和机器区分开。不过既然有图灵测试，就有试图攻破测试的程序。于是最近研究了一下验证码的识别。研究中发现这个问题可以做为计算机视觉课的大作业，要系统的试图解决这个问题，需要用到计算机视觉从底层视觉到高层语义的几乎所有的技术。

不过，在解决验证码识别问题前，我们需要讨论一下如何写出一个正确的验证码程序。我们发现，有不少有验证码的网站，其实不用输入正确的验证码，就可以绕过验证码。这种验证码程序就是形同虚设。

首先，好的验证码是
1. 人很容易识别
2. 机器很难识别