---
title: golang Ratelimit
date: '2019-10-16 17:40:01'
sidebar: false
categories:
 - 技术
tags:
 - golang
 - ratelimit
publish: true
---


> https://chai2010.cn/advanced-go-programming-book/ch5-web/ch5-06-ratelimit.html

```golang
// https://chai2010.cn/advanced-go-programming-book/ch5-web/ch5-06-ratelimit.html
// https://chai2010.cn/advanced-go-programming-book/ch5-web/ch5-06-ratelimit.html
package main

import (
	"fmt"
	"time"
)

func main() {
	ch := make(chan struct{}, 10)
	t := time.NewTicker(time.Second)
	
	defer t.Stop()

	go func(ch chan struct{}) {
		for {
			select {
			case <-ch:
				fmt.Println("拿到一个请求")
			}
		}
	}(ch)

	for {
		select {
		case <-t.C:
			ch<- struct{}{}
			fmt.Println("进入一个请求, 长度是=", len(ch))
		}
	}
}


```