---
title: 读那些知名golang项目源码笔记
date: '2021-03-26 14:37:59'
sidebar: true
categories:
 - 技术
tags:
 - golang
 - coding
 - 源码解析
publish: true
---

## minio

`http.CanonicalHeaderKey` 返回规范的 http 头名称

```go
http.CanonicalHeaderKey("content-type") // Content-Type
http.CanonicalHeaderKey("etag") // Etag
http.CanonicalHeaderKey("cOntent-typE") // Content-Type
```



[编辑距离](https://en.wikipedia.org/wiki/Damerau-Levenshtein_distance)

> 是指两个字串之间，由一个转成另一个所需的最少编辑操作次数

```go
// from minio source code
package main

import "math"

// Returns the minimum value of a slice of integers
func minimum(integers []int) (minVal int) {
	minVal = math.MaxInt32
	for _, v := range integers {
		if v < minVal {
			minVal = v
		}
	}
	return
}

// DamerauLevenshteinDistance calculates distance between two strings using an algorithm
// described in https://en.wikipedia.org/wiki/Damerau-Levenshtein_distance
func DamerauLevenshteinDistance(a string, b string) int {
	var cost int
	d := make([][]int, len(a)+1)
	for i := 1; i <= len(a)+1; i++ {
		d[i-1] = make([]int, len(b)+1)
	}
	for i := 0; i <= len(a); i++ {
		d[i][0] = i
	}
	for j := 0; j <= len(b); j++ {
		d[0][j] = j
	}
	for i := 1; i <= len(a); i++ {
		for j := 1; j <= len(b); j++ {
			if a[i-1] == b[j-1] {
				cost = 0
			} else {
				cost = 1
			}
			d[i][j] = minimum([]int{
				d[i-1][j] + 1,
				d[i][j-1] + 1,
				d[i-1][j-1] + cost,
			})
			if i > 1 && j > 1 && a[i-1] == b[j-2] && a[i-2] == b[j-1] {
				d[i][j] = minimum([]int{d[i][j], d[i-2][j-2] + cost}) // transposition
			}
		}
	}
	return d[len(a)][len(b)]
}
```

```go
	log.Fatal(
		DamerauLevenshteinDistance("db:seed", "db:esed"),
		DamerauLevenshteinDistance("app:serve", "app:sreve"),
		DamerauLevenshteinDistance("migrate", "migarte"),
		DamerauLevenshteinDistance("test", "taet"),
	)
	//  1 1 1 2 数字越小越相似
```



`unicode.IsSpace`  判断是否有空格

```go
// IsSpace reports whether the rune is a space character as defined
// by Unicode's White Space property; in the Latin-1 space
// this is
//	'\t', '\n', '\v', '\f', '\r', ' ', U+0085 (NEL), U+00A0 (NBSP).
// Other definitions of spacing characters are set by category
// Z and property Pattern_White_Space.
func IsSpace(r rune) bool {
	// This property isn't the same as Z; special-case it.
	if uint32(r) <= MaxLatin1 {
		switch r {
		case '\t', '\n', '\v', '\f', '\r', ' ', 0x85, 0xA0:
			return true
		}
		return false
	}
	return isExcludingLatin(White_Space, r)
}
```



## Dapr

获取local ip的方法

```go
// GetHostAddress selects a valid outbound IP address for the host.
func GetHostAddress() (string, error) {
   if val, ok := os.LookupEnv(HostIPEnvVar); ok && val != "" {
      return val, nil
   }

   // Use udp so no handshake is made.
   // Any IP can be used, since connection is not established, but we used a known DNS IP.
   conn, err := net.Dial("udp", "8.8.8.8:80")
   if err != nil {
      // Could not find one via a  UDP connection, so we fallback to the "old" way: try first non-loopback IPv4:
      addrs, err := net.InterfaceAddrs()
      if err != nil {
         return "", errors.Wrap(err, "error getting interface IP addresses")
      }

      for _, addr := range addrs {
         if ipnet, ok := addr.(*net.IPNet); ok && !ipnet.IP.IsLoopback() {
            if ipnet.IP.To4() != nil {
               return ipnet.IP.String(), nil
            }
         }
      }

      return "", errors.New("could not determine host IP address")
   }

   defer conn.Close()
   return conn.LocalAddr().(*net.UDPAddr).IP.String(), nil
}
```



```go

// Context returns a context which will be canceled when either the SIGINT or
// SIGTERM signal is caught. It also returns a function that can be used to
// programmatically cancel the same context at any time. If either signal is
// caught a second time, the program is terminated immediately with exit code 1.
func Context() context.Context {
	ctx, cancel := context.WithCancel(context.Background())
	sigCh := make(chan os.Signal)
	signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
	go func() {
		sig := <-sigCh
		log.Infof(`Received signal "%s"; beginning shutdown`, sig)
		cancel()
		sig = <-sigCh
		log.Fatalf(
			`Received signal "%s" during shutdown; exiting immediately`,
			sig,
		)
	}()
	return ctx
}
```



### 指数退避算法

https://github.com/cenkalti/backoff



### 概率一致性，在差值小于3时，概率会出现不相等的情况

```go
// Returns a random value from the following interval:
// 	[currentInterval - randomizationFactor * currentInterval, currentInterval + randomizationFactor * currentInterval].
func getRandomValueFromInterval(randomizationFactor, random float64, currentInterval time.Duration) time.Duration {
	var delta = randomizationFactor * float64(currentInterval)
	var minInterval = float64(currentInterval) - delta
	var maxInterval = float64(currentInterval) + delta

	// Get a random value from the range [minInterval, maxInterval].
	// The formula used below has a +1 because if the minInterval is 1 and the maxInterval is 3 then
	// we want a 33% chance for selecting either 1, 2 or 3.
	return time.Duration(minInterval + (random * (maxInterval - minInterval + 1)))
}

// 保证了在差值极小情况下的概率相等 (33%), 差值大的情况下 1 可以忽略
0.33 * (3-2+1) 0.99 < 1 // => 0
0.34 * (3-2+1) 1.02 > 1 // => 1
```

**backoff  NewTicker 实现有点奇妙，之后多看看**

```go
// NewTicker returns a new Ticker containing a channel that will send
// the time at times specified by the BackOff argument. Ticker is
// guaranteed to tick at least once.  The channel is closed when Stop
// method is called or BackOff stops. It is not safe to manipulate the
// provided backoff policy (notably calling NextBackOff or Reset)
// while the ticker is running.

// NewTicker返回一个新的Ticker，其中包含将发送的频道
//由BackOff参数指定的时间。 股票代号是
//保证至少勾选一次。 停止时通道关闭
//方法被调用或BackOff停止。 操作
//提供的退避策略（特别是调用NextBackOff或Reset）
//当行情自动收报机运行时。
func NewTicker(b BackOff) *Ticker {
	return NewTickerWithTimer(b, &defaultTimer{})
}
```





```go
func AsyncCall() {
    ctx, cancel := context.WithTimeout(context.Background(), time.Duration(time.Millisecond*800))
    defer cancel()
    go func(ctx context.Context) {
        // 发送HTTP请求
    }()

    select {
    case <-ctx.Done():
        fmt.Println("call successfully!!!")
        return
    case <-time.After(time.Duration(time.Millisecond * 900)):
        fmt.Println("timeout!!!")
        return
    }
}
```


