---
title:  Etcd
date: '2021-07-27 13:18:59'
sidebar: false
categories:
 - 技术
tags:
 - etcd
 - k8s
publish: true
---


## Revision

[etcd 中让人头大的 version, revision, createRevision, modRevision - 墨天轮](https://www.modb.pro/db/79479)

实验部分不想看的，可以只看结论:

1. `Revision`
   : 作用域为集群，逻辑时间戳，全局单调递增，任何 key 修改都会使其自增

2. `CreateRevision`
   : 作用域为 key, 等于创建这个 key 时的 Revision, 直到删除前都保持不变

3. `ModRevision`
   : 作用域为 key, 等于修改这个 key 时的 Revision, 只要这个 key 更新都会改变

4. `Version`
   : 作用域为 key, 某一个 key 的修改次数(从创建到删除)，与以上三个 Revision 无关

关于 watch 哪个版本：

1. watch 某一个 key 时，想要从历史记录开始就用 CreateRevision，最新一条(这一条直接返回)开始就用 `ModRevision`  

2. watch 某个前缀，就必须使用 `Revision`



## 使用场景

> https://tonydeng.github.io/2015/10/19/etcd-application-scenarios/