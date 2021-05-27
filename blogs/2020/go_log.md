---
title: go log 包学习意外收获
date: '2021-05-12 12:54:14'
sidebar: false
categories:
 - golang
tags:
 - log 
 - golang
publish: true
---


设置选项可在每条输出的文本前增加一些额外信息，如日期时间、文件名等。

```go
const (
	Ldate         = 1 << iota     // the date in the local time zone: 2009/01/23
	Ltime                         // the time in the local time zone: 01:23:23
	Lmicroseconds                 // microsecond resolution: 01:23:23.123123.  assumes Ltime.
	Llongfile                     // full file name and line number: /a/b/c/d.go:23
	Lshortfile                    // final file name element and line number: d.go:23. overrides Llongfile
	LUTC                          // if Ldate or Ltime is set, use UTC rather than the local time zone
	Lmsgprefix                    // move the "prefix" from the beginning of the line to before the message
	LstdFlags     = Ldate | Ltime // initial values for the standard logger
)
```

`log`库提供了 6 个选项：


- `Ldate`：输出当地时区的日期，如`2020/02/07`；
- `Ltime`：输出当地时区的时间，如`11:45:45`；
- `Lmicroseconds`：输出的时间精确到微秒，设置了该选项就不用设置`Ltime`了。如`11:45:45.123123`；
- `Llongfile`：输出长文件名+行号，含包名，如`github.com/darjun/go-daily-lib/log/flag/main.go:50`；
- `Lshortfile`：输出短文件名+行号，不含包名，如`main.go:50`；
- `LUTC`：如果设置了`Ldate`或`Ltime`，将输出 UTC 时间，而非当地时区。

调用`log.SetFlag`设置选项，可以一次设置多个：

```go
log.SetFlags(log.Lshortfile | log.Ldate | log.Lmicroseconds)
```

为什么这里需要 用 1<<iota 的方式来设置常量呢？为什么设置时可以用或运算呢？

可以得到上面的常量都是 2 的倍数

```go
const (
	Ldate         = 1 << iota     // 1*2^0 = 1      	0000 0001
	Ltime                         // 1*2^1 = 2				0000 0010
	Lmicroseconds                 // 1*2^2 = 4				0000 0100
	Llongfile                     // 1*2^3 = 8				0000 1000
	Lshortfile                    // 1*2^4 = 16				0001 0000
	LUTC                          // 1*2^5 = 32				0010 0000
	Lmsgprefix                    // 1*2^6 = 64				0100 0000
	LstdFlags     = Ldate | Ltime // 1 | 2 => 3
)
```

这样的做法是什么呢？任意常量或(|)操作的和与(&)本身都不等于0

```
1|2 & 1 != 0
1|2 & 2 != 0
1|2 & 3 == 0
```

底层通过&运算!=0来判断是否设置了该常量

```go
l.flag&(Ltime|Lmicroseconds) != 0
l.flag&Lmicroseconds != 0
l.flag&(Lshortfile|Llongfile) != 0
```

o(￣▽￣)ｄ