---
title: go还能这样写!
date: '2020-06-12 11:01:34'
sidebar: false
categories:
 - 技术
tags:
 - design 
 - golang
publish: true
---


读go设计模式`Bounded Parallelism Pattern`的时候发现一个写法，读到 wg.wait 就纳闷了，这样写有啥用，难道不会先执行wg.wait，然后导致协程退出吗

> 并行（Parallelism）:完成大量的独立任务；
>
> 有界并行模式 (Bounded Parallelism Pattern):完成大量资源限制的独立任务；

```go
func MD5All(root string) (map[string][md5.Size]byte, error) {
	// MD5All closes the done channel when it returns; it may do so before
	// receiving all the values from c and errc.
	done := make(chan struct{})
	defer close(done)

	paths, errc := walkFiles(done, root)

	// Start a fixed number of goroutines to read and digest files.
	c := make(chan result) // HLc
	var wg sync.WaitGroup
	const numDigesters = 20
	wg.Add(numDigesters)
	for i := 0; i < numDigesters; i++ {
		go func() {
			digester(done, paths, c) // HLc
			wg.Done()
		}()
	}
  
  // 
	go func() {
		wg.Wait()
		close(c) // HLc
	}()
	// End of pipeline. OMIT

	m := make(map[string][md5.Size]byte)
	for r := range c {
		if r.err != nil {
			return nil, r.err
		}
		m[r.path] = r.sum
	}
	// Check whether the Walk failed.
	if err := <-errc; err != nil { // HLerrc
		return nil, err
	}
	return m, nil
}
```

分析：

平常我们多线程执行任务的时候一般会这样写, 总是先等到协程执行结束, 才执行打印操作，这样就会浪费时间去等待所有的任务结束

```go
func main() {
	var wg = &sync.WaitGroup{}
	var ch = make(chan int, 3)
	
	wg.Add(3)

	go func() {
		defer wg.Done()
		ch <- 1
	}()
	go func() {
		defer wg.Done()
		ch <- 2
	}()
	go func() {
		defer wg.Done()
		ch <- 3
	}()


	wg.Wait()
  close(ch)


	for i := range ch {
		fmt.Println(i)
	}
}
```

有没有什么方法可以不等待任务结束，在任务执行的时候就打印结果呢？

我们都知道 for range 一个未关闭的管道会 deadlock。那要怎么做的，很简单，包个go，让该协程去等待所有任务结束并且关闭管道。真的很妙！

```go
func main() {
	var wg = &sync.WaitGroup{}
	var ch = make(chan int, 3)
	
	wg.Add(3)

	go func() {
		defer wg.Done()
		ch <- 1
	}()
	go func() {
		defer wg.Done()
		ch <- 2
	}()
	go func() {
		defer wg.Done()
		ch <- 3
	}()


  go func {
    wg.Wait()
	  close(ch)
  }()


	for i := range ch {
		fmt.Println(i)
	}
}
```



上面是有界的，既控制了并发协程的总数20，对使用的资源做了限制。



还有一种无界的，既对资源不做限制，有多少个路径就有多少个协程

```go
package main

import (
	"crypto/md5"
	"errors"
	"fmt"
	"io/ioutil"
	"os"
	"path/filepath"
	"sort"
	"sync"
)

// A result is the product of reading and summing a file using MD5.
type result struct {
	path string
	sum  [md5.Size]byte
	err  error
}

// sumFiles starts goroutines to walk the directory tree at root and digest each
// regular file.  These goroutines send the results of the digests on the result
// channel and send the result of the walk on the error channel.  If done is
// closed, sumFiles abandons its work.
func sumFiles(done <-chan struct{}, root string) (<-chan result, <-chan error) {
	// For each regular file, start a goroutine that sums the file and sends
	// the result on c.  Send the result of the walk on errc.
	c := make(chan result)
	errc := make(chan error, 1)
	go func() { // HL
		var wg sync.WaitGroup
		err := filepath.Walk(root, func(path string, info os.FileInfo, err error) error {
			if err != nil {
				return err
			}
			if !info.Mode().IsRegular() {
				return nil
			}
			wg.Add(1)
			go func() { // HL
				data, err := ioutil.ReadFile(path)
				select {
				case c <- result{path, md5.Sum(data), err}: // HL
				case <-done: // HL
				}
				wg.Done()
			}()
			// Abort the walk if done is closed.
			select {
			case <-done: // HL
				return errors.New("walk canceled")
			default:
				return nil
			}
		})
		// Walk has returned, so all calls to wg.Add are done.  Start a
		// goroutine to close c once all the sends are done.
		go func() { // HL
			wg.Wait()
			close(c) // HL
		}()
		// No select needed here, since errc is buffered.
		errc <- err // HL
	}()
	return c, errc
}

// MD5All reads all the files in the file tree rooted at root and returns a map
// from file path to the MD5 sum of the file's contents.  If the directory walk
// fails or any read operation fails, MD5All returns an error.  In that case,
// MD5All does not wait for inflight read operations to complete.
func MD5All(root string) (map[string][md5.Size]byte, error) {
	// MD5All closes the done channel when it returns; it may do so before
	// receiving all the values from c and errc.
	done := make(chan struct{}) // HLdone
	defer close(done)           // HLdone

	c, errc := sumFiles(done, root) // HLdone

	m := make(map[string][md5.Size]byte)
	for r := range c { // HLrange
		if r.err != nil {
			return nil, r.err
		}
		m[r.path] = r.sum
	}
	if err := <-errc; err != nil {
		return nil, err
	}
	return m, nil
}

func main() {
	// Calculate the MD5 sum of all files under the specified directory,
	// then print the results sorted by path name.
	m, err := MD5All(os.Args[1])
	if err != nil {
		fmt.Println(err)
		return
	}
	var paths []string
	for path := range m {
		paths = append(paths, path)
	}
	sort.Strings(paths)
	for _, path := range paths {
		fmt.Printf("%x  %s\n", m[path], path)
	}
}

```


