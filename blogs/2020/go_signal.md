---
title: signal 信号之平滑退出
date: '2020-01-08 14:34:05'
sidebar: false
categories:
 - 技术
tags:
 - golang
 - signal
publish: true
---


> 学习笔记
>
> 参考 https://colobu.com/2015/10/09/Linux-Signals/
>
> https://github.com/unknwon/building-web-applications-in-go/blob/master/articles/01.md
>
> https://www.jianshu.com/p/ae72ad58ecb6

### 信号类型

个平台的信号定义或许有些不同。下面列出了POSIX中定义的信号。
Linux 使用34-64信号用作实时系统中。
命令`man 7 signal`提供了官方的信号介绍。

在POSIX.1-1990标准中定义的信号列表

| 信号    | 值       | 动作 | 说明                                                         |
| :------ | :------- | :--- | :----------------------------------------------------------- |
| SIGHUP  | 1        | Term | 终端控制进程结束(终端连接断开)                               |
| SIGINT  | 2        | Term | 用户发送INTR字符(Ctrl+C)触发                                 |
| SIGQUIT | 3        | Core | 用户发送QUIT字符(Ctrl+/)触发                                 |
| SIGILL  | 4        | Core | 非法指令(程序错误、试图执行数据段、栈溢出等)                 |
| SIGABRT | 6        | Core | 调用abort函数触发                                            |
| SIGFPE  | 8        | Core | 算术运行错误(浮点运算错误、除数为零等)                       |
| SIGKILL | 9        | Term | 无条件结束程序(不能被捕获、阻塞或忽略)                       |
| SIGSEGV | 11       | Core | 无效内存引用(试图访问不属于自己的内存空间、对只读内存空间进行写操作) |
| SIGPIPE | 13       | Term | 消息管道损坏(FIFO/Socket通信时，管道未打开而进行写操作)      |
| SIGALRM | 14       | Term | 时钟定时信号                                                 |
| SIGTERM | 15       | Term | 结束程序(可以被捕获、阻塞或忽略)                             |
| SIGUSR1 | 30,10,16 | Term | 用户保留                                                     |
| SIGUSR2 | 31,12,17 | Term | 用户保留                                                     |
| SIGCHLD | 20,17,18 | Ign  | 子进程结束(由父进程接收)                                     |
| SIGCONT | 19,18,25 | Cont | 继续执行已经停止的进程(不能被阻塞)                           |
| SIGSTOP | 17,19,23 | Stop | 停止进程(不能被捕获、阻塞或忽略)                             |
| SIGTSTP | 18,20,24 | Stop | 停止进程(可以被捕获、阻塞或忽略)                             |
| SIGTTIN | 21,21,26 | Stop | 后台程序从终端中读取数据时触发                               |
| SIGTTOU | 22,22,27 | Stop | 后台程序向终端中写数据时触发                                 |

在SUSv2和POSIX.1-2001标准中的信号列表:

| 信号      | 值       | 动作 | 说明                                              |
| :-------- | :------- | :--- | :------------------------------------------------ |
| SIGTRAP   | 5        | Core | Trap指令触发(如断点，在调试器中使用)              |
| SIGBUS    | 0,7,10   | Core | 非法地址(内存地址对齐错误)                        |
| SIGPOLL   |          | Term | Pollable event (Sys V). Synonym for SIGIO         |
| SIGPROF   | 27,27,29 | Term | 性能时钟信号(包含系统调用时间和进程占用CPU的时间) |
| SIGSYS    | 12,31,12 | Core | 无效的系统调用(SVr4)                              |
| SIGURG    | 16,23,21 | Ign  | 有紧急数据到达Socket(4.2BSD)                      |
| SIGVTALRM | 26,26,28 | Term | 虚拟时钟信号(进程占用CPU的时间)(4.2BSD)           |
| SIGXCPU   | 24,24,30 | Core | 超过CPU时间资源限制(4.2BSD)                       |
| SIGXFSZ   | 25,25,31 | Core | 超过文件大小资源限制(4.2BSD)                      |

Windows中没有SIGUSR1,可以用SIGBREAK或者SIGINT代替。



> 有两种信号不能被拦截和处理: `SIGKILL`和`SIGSTOP`



## Signal 五个方法

> 位运算看的头昏昏，直接看源码测试的文件或许对怎么使用该包会有收获

```go
// signal 包内部handlers的结构
var handlers struct {
   sync.Mutex
  
   m map[chan<- os.Signal]*handler

   ref [numSig]int64
   // Map channels to signals while the channel is being stopped.
   // Not a map because entries live here only very briefly.
   // We need a separate container because we need m to correspond to ref
   // at all times, and we also need to keep track of the *handler
   // value for a channel being stopped. See the Stop function.
   stopping []stopping
}

type handler struct {
	mask [(numSig + 31) / 32]uint32
}

numSig = 65 // 所有系统信号k'n最大值

// 每一个 Notify 的 ch 都有一个 handler

```


- Notify 注册一个接受信号的 chan
- Ignore 忽略某个信号
- Ignored 某个信号是否被忽略
- Reset 重置 handlers
- Stop 关闭



### Notify

如果第二个参数不传，那么默认是所有的信号

```go
func Notify(c chan<- os.Signal, sig ...os.Signal) {
	if c == nil {
		panic("os/signal: Notify using nil channel")
	}

	handlers.Lock()
	defer handlers.Unlock()

	h := handlers.m[c]
	if h == nil {
		if handlers.m == nil {
			handlers.m = make(map[chan<- os.Signal]*handler)
		}
		h = new(handler)
		handlers.m[c] = h
	}

	add := func(n int) {
		if n < 0 {
			return
		}
		if !h.want(n) {
			h.set(n)
			if handlers.ref[n] == 0 {
				enableSignal(n)
			}
			handlers.ref[n]++
		}
	}

  // 这里
	if len(sig) == 0 {
		for n := 0; n < numSig; n++ {
			add(n)
		}
	} else {
		for _, s := range sig {
			add(signum(s))
		}
	}
}
```

Handler 

```go
type handler struct {
	mask [(numSig + 31) / 32]uint32
}

func (h *handler) want(sig int) bool {
	return (h.mask[sig/32]>>uint(sig&31))&1 != 0
}

func (h *handler) set(sig int) {
	h.mask[sig/32] |= 1 << uint(sig&31)
}

func (h *handler) clear(sig int) {
	h.mask[sig/32] &^= 1 << uint(sig&31)
}

// 为什么是 32 目前不清楚

set(2) => h.mask[0] = 4
want(2) => true
clear(2) => h.mask[0] = 0
```



## golang web 平滑退出

```go
func main() {
	mux := &http.ServeMux{}

	mux.HandleFunc("/", func(writer http.ResponseWriter, request *http.Request) {
		addr := request.RemoteAddr
		writer.Write([]byte("hello" + addr))
	})

	s := http.Server{
		Addr:    ":8888",
		Handler: mux,
	}

	c := make(chan os.Signal)
	ctx, _ := context.WithCancel(context.Background())
	
	signal.Notify(c, os.Interrupt, os.Kill, syscall.SIGUSR1, syscall.SIGUSR2, syscall.SIGTERM)
	
	go func() {
		select {
		case <-c:
			log.Println(s.Shutdown(ctx))
			log.Println("平滑退出 graceful exit")
		}
	}()

	log.Fatal(s.ListenAndServe())
}
```


