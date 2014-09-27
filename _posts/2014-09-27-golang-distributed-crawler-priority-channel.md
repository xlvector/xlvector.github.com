---
layout: post
title: 基于Golang的分布式爬虫：优先级消息队列
categories: golang crawler
tags: golang crawler architecture channel priority
---

一个定向爬虫需要抓取不同优先级的任务，Golang的channel可以很容易的用来实现一个FIFO的消息队列，因此我们的任务就是基于channel实现一个优先级的消息队列。传统的优先级队列是基于[堆](http://en.wikipedia.org/wiki/Priority_queue)实现的。但如果我们要利用堆实现一个多生产者－单消费者模式的消息队列，不可避免的要实现一堆锁的逻辑，所以我们想尽量使用现有的channel。

爬虫的应用场景满足下面两个条件

1. 优先级是整数
2. 优先级的数量是有限的，一般也就不超过10个优先级

因此，我们可以通过一个channel的数组来实现一个优先级队列


	type PChanError struct {
		Code int
	}

	func NewPChanError(code int) *PChanError {
		return &PChanError{Code: code}
	}

	func (self *PChanError) Error() string {
		if self.Code == CHANNEL_FULL {
			return "channel is full"
		} else if self.Code == PRIORITY_OUT_OF_INDEX {
			return "priority out of index"
		}
		return ""
	}

	type PChan struct {
		chs      []chan interface{}
		sleepMS  time.Duration
		closeAll bool
		capacity int
	}

	func NewPChan(levels int, capacity int) *PChan {
		ret := PChan{}
		ret.chs = []chan interface{}{}
		ret.sleepMS = 1
		ret.closeAll = false
		ret.capacity = capacity
		for i := 0; i < levels; i++ {
			ret.chs = append(ret.chs, make(chan interface{}, capacity*(i+1)))
		}
		return &ret
	}

	func (self *PChan) Close() {
		for _, ch := range self.chs {
			close(ch)
		}
		self.closeAll = true
	}

	func (self *PChan) Push(priority int, val interface{}) error {
		if priority >= len(self.chs) || priority < 0 {
			return NewPChanError(PRIORITY_OUT_OF_INDEX)
		}
		idx := len(self.chs) - priority - 1
		if len(self.chs[idx]) == self.capacity*(priority+1) {
			return NewPChanError(CHANNEL_FULL)
		}
		self.chs[idx] <- val
		return nil
	}

	func (self *PChan) Pop() (interface{}, error) {
		for k, ch := range self.chs {
			if len(ch) > 0 {
				self.sleepMS = 1
				return <-ch, nil
			}
		}
		if self.closeAll {
			return nil, errors.New("channel is closed")
		}
		time.Sleep(self.sleepMS * time.Millisecond)
		self.sleepMS *= 2
		if self.sleepMS > 1000 {
			self.sleepMS = 1000
		}
		return nil, nil
	}

	func (self *PChan) QuickPop() (interface{}, error) {
		for k, ch := range self.chs {
			if len(ch) > 0 {
				self.sleepMS = 1
				return <-ch, nil
			}
		}
		if self.closeAll {
			return nil, errors.New("channel is closed")
		}
		return nil, nil
	}

