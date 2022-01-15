---
title:  Etcd 备份恢复
date: '2021-07-05 12:03:24'
sidebar: false
categories:
 - 技术
tags:
 - etcd
 - k8s
publish: true
---

## 2022-01-15 更新

恢复的时候节点名称千万不要和之前一样！！！会报错的 ID mismatch！

---

> 参考 http://www.mydlq.club/article/74/
>
> https://zhuanlan.zhihu.com/p/281742226


   ```bash
   ## 查看健康状态
   PKI_DIR=/etc/kubernetes/pki
   ECTD_API=3 etcdctl  --cacert=$PKI_DIR/etcd/ca.crt  --cert=$PKI_DIR/etcd/server.crt --key=$PKI_DIR/etcd/server.key  --endpoints=https://127.0.0.1:2379 endpoint health
   
   ## 创建备份（在一台etcd节点上）
   PKI_DIR=/etc/kubernetes/pki
   ECTD_API=3 etcdctl  --cacert=$PKI_DIR/etcd/ca.crt  --cert=$PKI_DIR/etcd/server.crt --key=$PKI_DIR/etcd/server.key  --endpoints=https://127.0.0.1:2379 snapshot save  /tmp/etcd-snapshot-`date +%Y%m%d`.db
   
   ## 把备份拷贝到每个master节点上
   scp /tmp/etcd-snapshot-`date +%Y%m%d`.db master01:/tmp/etcd-snapshot-`date +%Y%m%d`
   scp /tmp/etcd-snapshot-`date +%Y%m%d`.db master02:/tmp/etcd-snapshot-`date +%Y%m%d`
   scp /tmp/etcd-snapshot-`date +%Y%m%d`.db master03:/tmp/etcd-snapshot-`date +%Y%m%d`
   
   ## 开始备份恢复
   
   ## 关停 apiserver 和 etcd
   
   ## 移除且备份 /etc/kubernetes/manifests 目录
   $ mv /etc/kubernetes/manifests{,.bak}
   
   ## 查看 kube-apiserver、etcd 镜像是否停止
   $ docker ps|grep etcd && docker ps|grep kube-apiserver
   
   ## 备份现有 Etcd 数据
   $ mv /var/lib/etcd{,.bak}
   
   ## 恢复数据
   $ ECTD_API=3 etcdctl  --cacert=$PKI_DIR/etcd/ca.crt  --cert=$PKI_DIR/etcd/server.crt --key=$PKI_DIR/etcd/server.key  --endpoints=https://127.0.0.1:2379 snapshot restore /tmp/etcd-snapshot-`date +%Y%m%d`.db;mv /default.etcd/member/ /var/lib/etcd/
   
   ## 恢复 apiserver 和 etcd
   $ mv /etc/kubernetes/manifests.bak /etc/kubernetes/manifests
   ```






