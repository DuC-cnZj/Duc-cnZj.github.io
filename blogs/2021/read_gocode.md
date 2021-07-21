---
title: 读那些知名golang项目源码笔记
date: '2021-03-26 14:37:59'
sidebar: true
sticky: 1
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

防止切片扩容，设置cap，设置len=0，防止Marshal出现空结构体, 算小的优化把，细节

```go
	activeActorsCount := make([]ActiveActorsCount, 0, len(actorCountMap))
	for actorType, count := range actorCountMap {
		activeActorsCount = append(activeActorsCount, ActiveActorsCount{Type: actorType, Count: count})
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



SingleFlight 是 Go 开发组提供的一个扩展并发原语。它的作用是，在处理多个 goroutine 同时调用同一个函数的时候，只让一个 goroutine 去调用这个函数，等到这个 goroutine 返回结果的时候，再把结果返回给这几个同时调用的 goroutine，这样可以减少并发调用的数量。（位于go src/internal）

```go

// call is an in-flight or completed singleflight.Do call
type call struct {
	wg sync.WaitGroup

	// These fields are written once before the WaitGroup is done
	// and are only read after the WaitGroup is done.
	val interface{}
	err error

	// These fields are read and written with the singleflight
	// mutex held before the WaitGroup is done, and are read but
	// not written after the WaitGroup is done.
	dups  int
	chans []chan<- Result
}

// Group represents a class of work and forms a namespace in
// which units of work can be executed with duplicate suppression.
type Group struct {
	mu sync.Mutex       // protects m
	m  map[string]*call // lazily initialized
}

// Result holds the results of Do, so they can be passed
// on a channel.
type Result struct {
	Val    interface{}
	Err    error
	Shared bool
}

// Do executes and returns the results of the given function, making
// sure that only one execution is in-flight for a given key at a
// time. If a duplicate comes in, the duplicate caller waits for the
// original to complete and receives the same results.
// The return value shared indicates whether v was given to multiple callers.
func (g *Group) Do(key string, fn func() (interface{}, error)) (v interface{}, err error, shared bool) {
	g.mu.Lock()
	if g.m == nil {
		g.m = make(map[string]*call)
	}
	if c, ok := g.m[key]; ok {
		c.dups++
		g.mu.Unlock()
		c.wg.Wait()
		return c.val, c.err, true
	}
	c := new(call)
	c.wg.Add(1)
	g.m[key] = c
	g.mu.Unlock()

	g.doCall(c, key, fn)
	return c.val, c.err, c.dups > 0
}

// DoChan is like Do but returns a channel that will receive the
// results when they are ready. The second result is true if the function
// will eventually be called, false if it will not (because there is
// a pending request with this key).
func (g *Group) DoChan(key string, fn func() (interface{}, error)) (<-chan Result, bool) {
	ch := make(chan Result, 1)
	g.mu.Lock()
	if g.m == nil {
		g.m = make(map[string]*call)
	}
	if c, ok := g.m[key]; ok {
		c.dups++
		c.chans = append(c.chans, ch)
		g.mu.Unlock()
		return ch, false
	}
	c := &call{chans: []chan<- Result{ch}}
	c.wg.Add(1)
	g.m[key] = c
	g.mu.Unlock()

	go g.doCall(c, key, fn)

	return ch, true
}

// doCall handles the single call for a key.
func (g *Group) doCall(c *call, key string, fn func() (interface{}, error)) {
	c.val, c.err = fn()
	c.wg.Done()

	g.mu.Lock()
	delete(g.m, key)
	for _, ch := range c.chans {
		ch <- Result{c.val, c.err, c.dups > 0}
	}
	g.mu.Unlock()
}

// ForgetUnshared tells the singleflight to forget about a key if it is not
// shared with any other goroutines. Future calls to Do for a forgotten key
// will call the function rather than waiting for an earlier call to complete.
// Returns whether the key was forgotten or unknown--that is, whether no
// other goroutines are waiting for the result.
func (g *Group) ForgetUnshared(key string) bool {
	g.mu.Lock()
	defer g.mu.Unlock()
	c, ok := g.m[key]
	if !ok {
		return true
	}
	if c.dups == 0 {
		delete(g.m, key)
		return true
	}
	return false
}

```

[信号量](https://time.geekbang.org/column/article/308399)

## 输出水分子

> [请求合并和循环栅栏](https://time.geekbang.org/column/article/310443)

```go

type H2O struct {
	H *semaphore.Weighted
	O *semaphore.Weighted
	b cyclicbarrier.CyclicBarrier
}

func NewH2O() *H2O {
	return &H2O{
		H: semaphore.NewWeighted(2),
		O: semaphore.NewWeighted(1),
		b: cyclicbarrier.New(3),
	}
}

func (a *H2O) GetH(ch chan<- string) {
	a.H.Acquire(context.Background(), 1)
	ch <- "h"
	a.b.Await(context.Background())
	a.H.Release(1)
}

func (a *H2O) GetO(ch chan<- string) {
	a.O.Acquire(context.Background(), 1)
	ch <- "o"
	a.b.Await(context.Background())
	a.O.Release(1)
}

func main() {
	var ch = make(chan string, 300)
	h20 := NewH2O()
	wg := sync.WaitGroup{}
	wg.Add(300)
	for i := 0; i < 200; i++ {
		go func(i int) {
			defer wg.Done()
			h20.GetH(ch)
		}(i)
	}
	for i := 0; i < 100; i++ {
		go func(i int) {
			defer wg.Done()
			h20.GetO(ch)
		}(i)
	}

	go func() {
		wg.Wait()
		close(ch)
	}()

	var i int
	for s := range ch {
		if i%3 == 0 {
			log.Println()
		}
		i++
		log.Printf("%s", s)
	}
}
```


