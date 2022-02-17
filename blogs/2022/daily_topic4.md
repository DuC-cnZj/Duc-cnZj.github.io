---
title: k8s 每日一问 - hpa 如何持续监控应用状态？
date: '2022-02-17 16:56:29'
sidebar: false
categories:
 - 技术
tags:
 - 每日一问
publish: true
---


```go

func (a *HorizontalController) processNextWorkItem(ctx context.Context) bool {
	key, quit := a.queue.Get()
	if quit {
		return false
	}
	defer a.queue.Done(key)

	deleted, err := a.reconcileKey(ctx, key.(string))
	if err != nil {
		utilruntime.HandleError(err)
	}

    // 如果 hpa 不是删除操作，那么重新加入队列进行监控, 周期是 15 s
	if !deleted {
		a.queue.AddRateLimited(key)
	}

	return true
}
```