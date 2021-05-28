---
title: 记录博客前端页面api升级(php to go)
date: '2020-06-16 20:04:26'
sidebar: false
categories:
 - 技术
tags:
 - php 
 - blog 
 - golang
publish: true
---


>  最近一段时间用go重写了博客前端api

框架选用

- go 选用的是小而美的 gin
- php 选用的是优雅的 lumen 6.x

## 对比

### 打包大小

两者打包成docker镜像的大小一个 20M 一个 757M, 还是很明显的

```
registry.cn-hangzhou.aliyuncs.com/duc-cnzj/gin_blog                latest              1c22e2233dca        31 hours ago        20.2MB
registry.cn-hangzhou.aliyuncs.com/duc-cnzj/blog_app                caddy               58a1d8cc3ea5        6 days ago          757MB
```

### 响应速度

两个api地址

- https://go-api.whoops-cn.club/  go
- ../ php

#### 单纯的输出 logo 

php: `request_runtime: 4.5ms`

Go: `x-request-timing: 17.402µs`

一个ms  级别一个 us 级别，还是很明显的

#### 文章列表分页 [go](https://go-api.whoops-cn.club/articles) [php](../articles)

php: `request_runtime: 11.4ms`

Go：`x-request-timing: 1.545729ms`

差了10倍



### 文章详情 [go](https://go-api.whoops-cn.club/articles/1) [php](../articles/1)

php：`request_runtime: 7.2ms`

go: `x-request-timing: 420.859µs`



### Qps:

Go: 1.1w, 不记录日志和控制台输出 1.7w

Php: 几百



go貌似各方面都碾压了php😅



不过在写代码的难易程度上面 php 暴打 go，go要写一天php可能只要一小时‼️



在实际环境中还是推荐流量大的接口选用go，后台等流量小的接口选用php，不要为了用go而用go，都没几个人访问的后台怎么快怎么来，没必要用牛刀。



最后附上项目地址

go: https://github.com/DuC-cnZj/gin-blog

Php: https://github.com/DuC-cnZj/blog


