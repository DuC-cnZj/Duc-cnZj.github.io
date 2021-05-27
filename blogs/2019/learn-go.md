---
title: Go 语言学习
date: '2021-05-12 12:54:14'
sidebar: false
categories:
 - go
tags:
 - 语言学习
publish: true
---


## 迎接 GO2 的到来

<iframe width="100%" height="496" src="https://www.youtube.com/embed/6wIP3rO6On8" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>


1. init不是普通函数，可以定义有多个，所以不能被其它函数调用

## tcp

```go
// client.go
package main

import (
	"bufio"
	"fmt"
	"net"
	"os"
	"strings"
)

func arrHas(arr []string, item string) bool {
	for _, value := range arr {
		if value == item{
			return true
		}
	}

	return false
}

func main() {
	arr := []string{"exit", "q", "quit"}
	conn, err := net.Dial("tcp", ":6789")
	if err != nil {
		fmt.Println("Dial err", err)
	}

	defer conn.Close()
	input := bufio.NewReader(os.Stdin)
	for {
		str, err := input.ReadString('\n')
		str = strings.Trim(str, "\n")

		if arrHas(arr, str) {
			fmt.Println("接收到退出标识符->", str)
			return
		}

		if err != nil {
			fmt.Println("ReadString err", err)
		}

		_, err = conn.Write([]byte(str))

		if err != nil {
			fmt.Println("w err", err)
		}
	}
}

```

```go
// server.go
package main

import (
	"fmt"
	"io"
	"net"
)

func process(conn net.Conn) {
	defer conn.Close()
	for {
		b := make([]byte, 1024)
		n, err := conn.Read(b)
		if err == io.EOF {
			fmt.Printf("客户端：%s 关闭了请求\n", conn.RemoteAddr().String())
			return
		}
		fmt.Printf("收到来自客户端：%s 的请求, 内容为： %s \n",conn.RemoteAddr().String(), string(b[:n]))
	}
}
func main() {
	listener, err := net.Listen("tcp", ":6789")

	if err != nil {
		fmt.Println("listen err", err)
	}
	for {
		conn, err := listener.Accept()
		if err != nil {
			fmt.Println("Accept err", err)
		}

		go process(conn)
	}
}

```
