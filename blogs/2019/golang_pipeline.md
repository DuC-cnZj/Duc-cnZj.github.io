---
title: golang pipeline
date: '2019-10-18 16:22:25'
sidebar: false
categories:
 - 技术
tags:
 - golang
 - pipeline
publish: true
---

> https://mp.weixin.qq.com/s/D4hgb_bw6OdAvcZHfQ5UKw

```golang
package pipe

type myFunc func(...interface{})

type middleware func(myFunc) myFunc

type Pipeline struct {
	middlewares []middleware
}

func (p *Pipeline) through(m ...middleware) *Pipeline {
	p.middlewares = append(p.middlewares, m...)

	return p
}

func (p *Pipeline) send(handler myFunc) myFunc {
	length := len(p.middlewares)

	for i := length; i > 0; i-- {
		handler = p.middlewares[i-1](handler)
	}

	return handler
}
```

```go
package pipe

import (
	"fmt"
)

func ExamplePipeline() {
	p := &Pipeline{
		middlewares: nil,
	}

	p.through(func(m myFunc) myFunc {
		return func(i ...interface{}) {
			//defer func(t time.Time) {fmt.Println("请求的执行时间是", time.Since(t))}(time.Now())

			fmt.Println("in")
			m(i...)
			fmt.Println("out")
		}
	}).send(func(i ...interface{}) {
		fmt.Printf("核心逻辑：%s，收到的参数：%v\n", "core", i)
	})(1,2,3)

	// Output:
	// in
	// 核心逻辑：core，收到的参数：[1 2 3]
	// out
}
```