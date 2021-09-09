---
title: golang 插件机制
date: '2021-09-09 16:57:39'
sidebar: true
categories:
 - 技术
tags:
 - 插件
 - golang
publish: true
---

> 插件模式易于框架的扩展，是一个很有用的代码设计方法。

最近看到腾讯开源了北极星，看到其数据面采用**插件化**的方式实现，又联想到 go-micro 也是一款高度插件化的工具。因此想对比想两者的实现方式



## 北极星

先来看北极星的实现方式，下面是我自己仿照着写的简化版

1. 用一个全局变量  `pluginSet` 存放所有的插件
2. 通过 `RegisterPlugin` 注册插件，可以看到只要实现 `PluginInterface` 的三个方法就符合条件了

```go
// plugins/plugin.go
var (
	pluginSet = make(map[string]PluginInterface)
)

func RegisterPlugin(name string, pluginInterface PluginInterface)  {
	pluginSet[name] = pluginInterface
}

type PluginInterface interface {
	Name() string
	Initialize() error
	Destroy() error
}
```

hello 服务

1. 从`pluginSet`获取配置的插件，这里直接断言你注册时的插件符合了 `imp.HelloInterface` 接口，如果不实现会 panic，这里貌似也没有办法去判断，对比 micro 好像差了点

```go
// imp/hello_interface.go
type HelloInterface interface {
	SayHello()
}

```

```go
// plugins/hello.go
package plugins

import (
	"pluginstest/imp"
	"sync"
)

var hOnce = &sync.Once{}

func GetHelloWord(name string) imp.HelloInterface {
	pluginInterface, _ := pluginSet[name]
	hOnce.Do(func() {
		pluginInterface.Initialize()
	})

	return pluginInterface.(imp.HelloInterface)
}
```

自定义的 `myhello.go` 插件

通过程序启动时调用 init 实现注册

```go
// plugins/hello/myhello.go
// 实现了 PluginInterface, HelloInterface 这两个接口
package hello

import (
	"log"
	"pluginstest/plugins"
)

func init() {
	h := &Hello{}
	plugins.RegisterPlugin(h.Name(), h)
}

type Hello struct{}

func (h *Hello) Name() string {
	return "duc"
}

func (h *Hello) Initialize() error {
	log.Println("Initialize")
	return nil
}

func (h *Hello) Destroy() error {
	log.Println("Destroy")
	return nil
}

func (h *Hello) SayHello() {
	log.Println("duc saying...")
}

```

默认`hello`实现

```go
// default/default_hello.go
package _default

import (
	"log"
	"pluginstest/plugins"
)

func init() {
	dh:=&DefaultHello{}
	plugins.RegisterPlugin(dh.Name(), dh)
}

type DefaultHello struct {}

func (h *DefaultHello) Name() string {
	return "default_hello"
}

func (h *DefaultHello) Initialize() error {
	log.Println("default_hello Initialize")
	return nil
}

func (h *DefaultHello) Destroy() error {
	log.Println("default_hello Destroy")
	return nil
}

func (h *DefaultHello) SayHello()  {
	log.Println("default hello")
}

```

### **main.go**

>  最后只要在这里引入就可以实现注册

```go
package main

import (
	"pluginstest/plugins"

	_ "pluginstest/default"
	_ "pluginstest/plugins/hello"
)

func main() {
	plugins.GetHelloWord("duc").SayHello()
}
```





## go-micro



micro 通过直接定义全局变量的方式来定义不同的插件， 如下，不同的插件不同的全局变量，这样在注册时就能确定类型，避免了北极星的断言 panic

```go
DefaultRegistries = map[string]func(...registry.Option) registry.Registry{}

DefaultSelectors = map[string]func(...selector.Option) selector.Selector{}

DefaultServers = map[string]func(...server.Option) server.Server{}

DefaultTransports = map[string]func(...transport.Option) transport.Transport{}

DefaultRuntimes = map[string]func(...runtime.Option) runtime.Runtime{}
```

来看代码实现

```go
// global/global.go
package global

import (
	_default "pluginstest/default"
	"pluginstest/imp"
)

var HelloSvcs = map[string]func() imp.HelloInterface {
	"default": func() imp.HelloInterface {
		return &_default.DefaultHello{}
	},
}
```

因为`micro`直接通过全局变量，而不是通过结构体，所以没有 `Initialize` ，`Destroy` 等方法，这里采用 `func() imp.HelloInterface {}` 的方式来延迟初始化，直到 `Before` 函数执行, 详见 micro 的 `type Cmd interface `接口和 before 函数以及run方法

```go
// plugins/hello_plugin.go
package plugins

import (
	"log"
	"pluginstest/global"
	"pluginstest/imp"
)

func init() {
	global.HelloSvcs["duc"] = func() imp.HelloInterface {
		return &MyHello{}
	}
}

type MyHello struct {}

func (h *MyHello) SayHello()  {
	log.Println("my hello say...")
}
```

### **main.go**

```go
package main

import (
	"pluginstest/global"
	_ "pluginstest/plugins"
)

func main() {
	hello, _ := global.HelloSvcs["duc"]
	hello().SayHello()
}
```


