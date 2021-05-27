---
title: google team 也会写这样的bug?
date: '2021-05-12 12:54:14'
sidebar: false
categories:
 - golang
tags:
 - bug
 - google
publish: true
---

> 在看 goole 发布的 github.com/google/exposure-notifications-server 的源码的时候发现了下面这个很奇怪，我觉得是bug，自己本地测的时候也测出来貌似bug，不知道是不是。

![2020_06_19_YpIKRyrQfm.jpg](../images/2020_06_19_YpIKRyrQfm.jpg)
可以看到他这里不等shutdown结束就直接主程序退出，根本走不到 closeIdleConns 这里，或者等不到 context done
```golang
func (srv *Server) Shutdown(ctx context.Context) error {
	atomic.StoreInt32(&srv.inShutdown, 1)

	srv.mu.Lock()
	lnerr := srv.closeListenersLocked()
	srv.closeDoneChanLocked()
	for _, f := range srv.onShutdown {
		go f()
	}
	srv.mu.Unlock()

	ticker := time.NewTicker(shutdownPollInterval)
	defer ticker.Stop()
	for {
		if srv.closeIdleConns() {
			return lnerr
		}
		select {
		case <-ctx.Done():
			return ctx.Err()
		case <-ticker.C:
		}
	}
}
```

不知道我想的对不对，还请路过的各位大佬提点下🧐
