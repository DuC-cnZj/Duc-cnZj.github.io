---
title: go get 实践
date: '2020-06-18 17:45:47'
sidebar: false
categories:
 - 技术
tags:
 - go get 
 - golang
publish: true
---

> [参考煎鱼大佬的go mod 终极指南](https://mp.weixin.qq.com/s?__biz=MzUxMDI4MDc1NA==&mid=2247483713&idx=1&sn=817ffef56f8bc5ca09a325c9744e00c7&source=41#wechat_redirect) 



go get 来拉取外部依赖会自动下载并安装到`$GOPATH`目录(如果该第三方包有main文件 会自动安装到$GOPATH/bin下)

go get : git clone + go install

go build : 编译出可执行文件
go install : go build + 把编译后的可执行文件放到GOPATH/bin目录下



使用go get的时候还是挺迷惑的，自己弄了个包实践了下。https://github.com/DuC-cnZj/hello_golang



如果要从 v1.0.0 更新到 v1.0.2 必须使用 go get -u=patch github.com/xxx/xxx@v1

如果要使用未发布的最新的提交必须使用 go get -u github.com/xxx/xxx@master

如果要更新到最新的发布的版本必须使用 go get -u github.com/xxx/xxx@latest

这上面2个默认了v1



如果包有多个version v1 v2

如果要使用未发布的最新的v2提交必须使用 go get -u github.com/xxx/xxx/v2@master

如果要更新到最新的发布的v2版本必须使用 go get -u github.com/xxx/xxx/v2@latest



注意这里的操作可能会和预期的不一样，原因可能在于go mod做了本地缓存，所以最好事先清除下本地缓存`go clean -modcache `, 也有可能有延迟，导致你最新发布的版本一时间没办法获取，多等等既可。