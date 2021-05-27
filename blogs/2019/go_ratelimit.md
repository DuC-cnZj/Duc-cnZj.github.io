---
title: Go 限流之令牌桶
date: '2021-05-12 12:54:14'
sidebar: false
categories:
 - golang
tags:
 - golang
 - 限流
 - 微服务
publish: true
---


> https://github.com/juju/ratelimit



```go
func (tb *Bucket) take(now time.Time, count int64, maxWait time.Duration) (time.Duration, bool) {
	if count <= 0 {
		return 0, true
	}

  // 当前tick
	tick := tb.currentTick(now)
  // 调整桶的令牌数
	tb.adjustavailableTokens(tick)
	avail := tb.availableTokens - count
	if avail >= 0 {
		tb.availableTokens = avail
		return 0, true
	}
	
  // 如果当前没有可用的令牌，则计算当前请求会在哪个tick的时候变为可用，比如当前是tick 8，每个tick的流量是10，avail 为-10，那么这个请求将在tick为9的时候变为可用，它刚好是9的最后一个
	endTick := tick + (-avail+tb.quantum-1)/tb.quantum
  // 计算出可用的那一刻时间
	endTime := tb.startTime.Add(time.Duration(endTick) * tb.fillInterval)
  // 算出需要等待的时间
	waitTime := endTime.Sub(now)
  // 如果等待时间超过最大等待时间那么返回false
	if waitTime > maxWait {
		return 0, false
	}
//  此时的token为负数
	tb.availableTokens = avail
	return waitTime, true
}
```



```go
// 调整可以的令牌数量
func (tb *Bucket) adjustavailableTokens(tick int64) {
  // 保存上一次的标记位
	lastTick := tb.latestTick
  // 给予新的值
	tb.latestTick = tick
  // 如果当前的可用令牌总数大于桶的能力值，则返回
	if tb.availableTokens >= tb.capacity {
		return
	}
  // 如果不大于，则累加令牌，availableTokens = 原来已有的值+ （tick - lastTick）差值*每个tick的允许访问的容量
	tb.availableTokens += (tick - lastTick) * tb.quantum
  // 如果满了，溢出的扔掉
	if tb.availableTokens > tb.capacity {
		tb.availableTokens = tb.capacity
	}
	return
}
```



## 用令牌桶实现web请求限速的功能

```go
package main

import (
	"fmt"
	"github.com/juju/ratelimit"
	"log"
	"net/http"
	"sync/atomic"
	"time"
)

var reqs int64

type MiddlewareFunc func(http.HandlerFunc) http.HandlerFunc

type MiddlewareInterface interface {
	Through(...MiddlewareFunc) MiddlewareInterface
	Send(handler http.HandlerFunc) http.HandlerFunc
}

type MyMiddleware struct {
	middlewares []MiddlewareFunc
}

func (m *MyMiddleware) Through(funcs ...MiddlewareFunc) MiddlewareInterface {
	m.middlewares = append(m.middlewares, funcs...)
	return m
}

func (m *MyMiddleware) Send(handler http.HandlerFunc) http.HandlerFunc {
	var mHandler http.HandlerFunc

	for _, m := range m.middlewares {
		if mHandler != nil {
			mHandler = m(mHandler)
		} else {
			mHandler = m(handler)
		}
	}

	return mHandler
}

var (
	tb     = ratelimit.NewBucketWithQuantum(time.Second, 3, 1)
	middle = &MyMiddleware{middlewares: nil}
)

func main() {
	m1 := rateMiddleware()

	http.HandleFunc("/hello", middle.Through(m1).Send(handleHello()))

	log.Fatal(http.ListenAndServe(":8000", nil))
}

func rateMiddleware() func(fn http.HandlerFunc) http.HandlerFunc {
	m1 := func(fn http.HandlerFunc) http.HandlerFunc {
		return func(writer http.ResponseWriter, request *http.Request) {
			writer.Write([]byte("in: 1\n"))
			available := tb.TakeAvailable(1)
			log.Printf("tb.Available() = %d,tb.Capacity() = %d, tb.Rate() = %.2f\n", tb.Available(), tb.Capacity(), tb.Rate())
			if available <= 0 {
				fmt.Fprintf(writer, "你被限速了\n")
				return
			}
			fn(writer, request)
			writer.Write([]byte("out: 1\n"))
		}
	}
	return m1
}

func handleHello() http.HandlerFunc {
	return func(writer http.ResponseWriter, request *http.Request) {
		atomic.AddInt64(&reqs, 1)
		writer.Write([]byte(fmt.Sprintf("当前是第%d个请求\n", reqs)))
	}
}

```





## 漏桶算法比较简单，参考👇

https://github.com/yangwenmai/ratelimit