---
title: k8s bookmark 机制
date: '2022-01-19 15:55:40'
sidebar: false
categories:
 - 技术
tags:
 - k8s
publish: true
---

# k8s bookmark 机制
​
​
因为etcd会定期压缩版本，如果你k8s客户端watch的资源很久没有更新会导致一旦网络不稳定比如 ConnectionRefused 那么就会重新 watch 带上那个 version，因为被compacted了，watch报错，会退出方法重新执行ListAndWatch，重新list会带给apiserver压力。
​
​
​
代码 `func (r *Reflector) ListAndWatch(stopCh <-chan struct{})`。
10
​
​
​
## etcd 版本机制
​
> [etcd 中让人头大的 version, revision, createRevision, modRevision - 墨天轮](https://www.modb.pro/db/79479)
​
​

​## kubernetes - What k8s bookmark solves?

[kubernetes - What k8s bookmark solves? - Stack Overflow](https://stackoverflow.com/questions/66080942/what-k8s-bookmark-solves)
20
​
​
​