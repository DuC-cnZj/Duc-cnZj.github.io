---
title: 使用 go generate
date: '2020-06-10 16:16:39'
sidebar: false
categories:
 - 技术
tags:
 - go generate 
 - golang
publish: true
---


> 参考一 https://learnku.com/go/t/45620
> 参考二 https://juejin.im/post/5d5ff2f45188256dad113236
> 参考三 http://c.biancheng.net/view/4442.html

## 注意
1. 该特殊注释必须在 .go 源码文件中；
2. 每个源码文件可以包含多个 generate 特殊注释；
3. 运行go generate命令时，才会执行特殊注释后面的命令；
4. 当go generate命令执行出错时，将终止程序的运行；
5. 特殊注释必须以//go:generate开头，双斜线后面没有空格。

## 在下面这些场景下，我们会使用go generate命令：
1. yacc：从 .y 文件生成 .go 文件；
2. protobufs：从 protocol buffer 定义文件（.proto）生成 .pb.go 文件；
3. Unicode：从 UnicodeData.txt 生成 Unicode 表；
4. HTML：将 HTML 文件嵌入到 go 源码；
5. bindata：将形如 JPEG 这样的文件转成 go 代码中的字节数组。

再比如：
1. string 方法：为类似枚举常量这样的类型生成 String() 方法；
2. 宏：为既定的泛型包生成特定的实现，比如用于 ints 的 sort.Ints。