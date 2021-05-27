---
title:  课程名称：Redis5.0之12项新特性  
date: '2021-05-12 12:54:14'
sidebar: true
categories:
 - redis
tags:
 - redis
publish: true
---


> 转自 https://note.youdao.com/share/?id=2c0a6950be33827a851b9e44ef6e3a6a&type=note#/


课程地址：https://www.imooc.com/learn/1089  
简介：本次课程主要讲解 Redis5.0 的重要配置项，讲解 Stream 数据类型，新的 Sorted Set 命令，内存碎片整理以及用新的集群管理器搭建一个简单的集群，重点放在 Stream 数据类型上面，其它新特性会讲解理论知识和应用场景。  

## 第1章 课程介绍
介绍本次课程的主要学习内容，Redis5.0 的应用场景，重点内容以及课程安排。
### 1-1 课程介绍 (02:07)
#### 你将可以学到如下知识
1. redis的重要配置项(终点)
2. stream数据类型(重点)
3. help子命令
4. 更方便的搭建redis集群(重点)
5. 新的Sorted Set命令
6. 如何整理内存碎片和如何查看内存报告

tips——redis5搭建集群不再需要安装Ruby并且可以整理内存碎片了。
### 1-2 初识Redis5.0之12项新特性 (05:14)
1. 新的Stream数据类型  
    也就是说现在有string、list、hash、set、sort set、stream六种
2. 新的Redis模块API：Timers and Cluster API
3. RDB现在存储LFU和LRU信息
4. 集群管理从Ruby(redis-trib.rb)移植到C代码
5. 新的sorted set命令：ZPOPMIN/MAX的阻塞变种
6. 主动碎片整理V2
7. 增强HyperLogLog实现
8. 更好的内存统计报告
9. 许多带有子命令的命令现在都有一个HELP子命令
10. 客户经常连接和断开时性能更好
11. 错误修复和改进
12. Jemalloc升级到5.1(Jemalloc是一个内存分配器)

### 1-3 stream数据类型概述 (04:10)
什么是stream数据类型：
- stream翻译过来的意思是小河
- 本质是一个抽象日志
- redis现在又=有string、list、hash、set、sort set、stream六种数据类型

为什么要学习stream
1. 其它5中数据接口不能直接实现的需求，可直接用stream实现
2. 直接贴近业务需求，提升开发效率
3. 物联网，各种传感器产生时间序列数据，定位未来
## 第2章 Redis5.0配置文件
讲解Redis5.0 redis.conf 中的一些重要的配置项。
### 2-1 redis5.0的重要配置项 (15:24)
安装环境：
1. 操作系统：CentOS Linux release 7.4
2. redis版本：5.0-rc3，[下载地址](https://github.com/antirez/redis/archive/5.0-rc3.tar.gz)

安装方法：
1. 下载：
  `wget -O redis-5.0-rc3.tar.gz https://github.com/antirez/redis/archive/5.0-rc3.tar.gz`
2. 解压：
  `tar -zxvf redis-5.0-rc3.tar.gz -C /usr/local/src`
3. 编译并安装：make & make install
4. 修改配置文件`redis.conf`
5. 启动redis5.0：/usr/local/src/redis-5.0-rc3/src/redis-server /usr/local/src/redis-5.0-rc3/redis.conf
6. 关闭redis：redis-cli shutdown

### 配置讲解
```
daemonize yes # 守护进程(后台运行),设置为no的话，只是在当前终端下运行，ctrl+c会退出
protected-mode yes # 安全模式，不想外网访问就设置yes
# 下面三个是RDB相关
save 900 1  # 900秒内有1个key发生变化
save 300 10 # 300秒内有10个key发生变化
save 60 10000 # 60秒内有10000个key发生变化

# 下面是开启集群
cluster-enabled yes # 开启集群
cluster-config-file # 集群用到的配置文件，我们无法修改，redis用来保存集群状态用的
cluster-node-timeout 15000 # 集群节点失联最大时间，如果主节点失联超过这个时间，将会有个子节点升级成主节点
```

## 第3章 Stream数据类型详解
讲解Stream 的存储结构，并详细讲到 Consumer Group 和 Consumer 之间的关系以及数据的取用方法，以及讲解一些常用命令的使用方法。
### 3-1 图解stream数据类型 (02:51)
![stream-图解](https://note.youdao.com/yws/api/personal/file/186DBB4F03AC47DD9C5F43D1BAB60D23?method=download&shareKey=76ea0405dbe48b9b52bce6d46bb1b931)
### 3-2 XADD、XLEN和XDEL用法详解 (06:26)
#### XADD
作用：创建一个stream
用法：XADD key ID field string [field string ...]
ID：毫秒的unix时间戳-sequence(同一毫秒的序列号)组成
例子：
```
127.0.0.1:6379> XADD LOL * slain quadra_kill # *代表id默认
1539093800437-0
127.0.0.1:6379> XADD LOL * slain pent_kill
1539093831396-0
127.0.0.1:6379> XADD wangzhe 0-1 hero xiaoqiao # 自定义id
0-1 
```
#### XLEN
作用：返回stream中元素的个数
用法：XLEN key
```
127.0.0.1:6379> XLEN LOL
(integer) 2
127.0.0.1:6379> XLEN wangzhe
(integer) 1
```
####XDEL 
作用：删除一个ID
用法：XDEL ket ID
```
127.0.0.1:6379> XDEL LOL 1539093800437-0
(integer) 1
```
### 3-3 XRANGE和XREAD用法详解 (06:09)
#### XRANGE
作用：返回给定ID范围内的stream数据
用法：XRANGE ket start end [COUNT count]
特殊ID：`+`代表最大ID，`-`代表最小ID
例子：
```
127.0.0.1:6379> XADD LOL * slain double_kill
127.0.0.1:6379> XADD LOL * slain triple_kill
127.0.0.1:6379> XADD LOL * slain quadra_kill
127.0.0.1:6379> XRANGE LOL - +
127.0.0.1:6379> XRANGE LOL - + COUNT 1
127.0.0.1:6379> XRANGE LOL 1539093800437-0 1539093831396-0
```
#### XREAD
作用：从一个或多个stream读取数据
用法：XREAD [COUNT count] [BLOCK milliseconds] STREAMS key [key ...] ID [ID ...]
关键词：`BLOCK 0`永久阻塞；`$`获取最新的数据
例子：
```
127.0.0.1:6379> XREAD STREMS LOL 1539093800437-0 # 读取这个ID之后的数据
127.0.0.1:6379> XREAD COUNT 1 STREMS LOL 1539093800437-0 # 只读1条
127.0.0.1:6379> XREAD BLOCK 0 STREAMS LOL $ # 阻塞住不输出结果，直到新的一条记录进来马上打印出来
```

### 3-4 XGROUP和XREADGROUP用法详解 (06:18)
创建消息组：XGROUP
读取消息组：XREADGROUP
#### XGROUP
作用：创建一个Consumer Group
用法：XGROUP CREATE key groupname ID
例子：
```
127.0.0.1:6379> XADD wangzhe * hero luban
1539182362587-0
127.0.0.1:6379> XADD wangzhe * hero daji
1539182367086-0
127.0.0.1:6379> XADD wangzhe * hero xiaoqiao
1539182372217-0
127.0.0.1:6379> XADD wangzhe * hero diaochan
1539182377070-0
127.0.0.1:6379> XGROUP CREATE wangzhe fashi 1539182362587-0 # 1539182362587-0之后的数据，不包括1539182362587-0
```

#### XREADGROUP
作用：从Consumer Grouop中读取数据
用法：XREADGROUP GROUP groupname consumer [COUNT count] [BLOCK milliseconds] STREAMS key [key ...] ID [ID ...]
例子：
```
127.0.0.1:6379> XREAD GROUP fashi chuanrong STREAMS wangzhe > # 读取最新的消息
127.0.0.1:6379> XREAD GROUP fashi chuanrong STREAMS wangzhe > # 由于已经被读取过了，再次执行结果为空nil
127.0.0.1:6379> XREAD GROUP fashi chuanrong STREAMS wangzhe  1539182367086-0 # 读取指定的ID
127.0.0.1:6379> XREAD GROUP fashi chuanrong COUNT 1 STREAMS wangzhe 1539182367086-0 # 读取指定的ID #只读取一条
127.0.0.1:6379> XREAD GROUP fashi chuanrong BLOCK 0 STREAMS wangzhe # 阻塞读取最新数据
```

## 第4章 新的 Sorted Set 命令
用代码演示的方法讲解 ZPOPMAX、ZPOPMIN、BZPOPMAX 和 BZPOPMIN 四个命令的详细使用方法。
### 4-1 help子命令详解 (03:54)
```
127.0.0.1:6379> XINFO help # 查看stream相关命令帮助
127.0.0.1:6379> XINFO CONSUMERS wangzhe fashi
127.0.0.1:6379> PUBSUB help
127.0.0.1:6379> GROUP help
127.0.0.1:6379> XADD help # 会报错，因为XADD没有子命令
```

## 第5章 碎片整理和内存报告
讲解主动内存碎片整理Active defragmentation 升级到了Version 2，Jemalloc版本升级至5.1，以及查看更加详尽的内存报告。
### 5-1 搭建cluster集群 (18:24)
#### 集群模拟
搭建集群需要多台服务器，讲师这里用创建集群数据存储文件夹的方式模拟一哈
```
mkdir -p /usr/local/redis-cluster
cd /usr/local/redis-cluster
mkdir 5001 5002 5003 5004 5005 5006 # 集群最少都是三主三从，所以要6个
# 将配置文件分别复制到以上6个目录分别启动即可
cp /etc/redis/redis.conf 5001/redis.conf
cd 5001
vim redis.conf
# 修改端口
# 修改pidfile（比如改成/var/fun/redis_5001.pid）
# 修改日志文件logfile（比如/usr/local/redis-luster/5001/redis.log）
# 配置数据文件位置dir，把./改成/usr/local/redis-cluster/5001/
# 打开appendonly yes和appendfsync everysec
# 打开cluster-enabled yes
# 打开cluster-config-file nodes-5001.conf
# 打开cluster-node-timeout 15000
# 保存后启动
/usr/local/src/redis-5.0-rc3/src/redis-server /usr/local/redis-cluster/5001/redis.conf
# 查看
ps aux | grep redis
# 将这份配置文件复制到其它目录
cp redis.conf ../5002/redis.conf 
cp redis.conf ../5003/redis.conf 
cp redis.conf ../5004/redis.conf 
cp redis.conf ../5005/redis.conf 
cp redis.conf ../5006/redis.conf 
# 全局替换5001即可(vim的命令：`$s/5001/5002/g`  #全局替换5001为5002)
# 分别启动这些redis
/usr/local/src/redis-5.0-rc3/src/redis-server /usr/local/redis-cluster/5002/redis.conf
...
/usr/local/src/redis-5.0-rc3/src/redis-server /usr/local/redis-cluster/5006/redis.conf
# 查看
ps aux | grep redis
```
#### 创建六个子节点
1. ruby创建方法：redis-trib.rb create --replicas 1 192.168.4.147:500*
2. 新特性创建方法：redis-cli --cluster create 1 192.168.4.147:500* --cluster-replicas 1
3. 主/从=1：cluster-replicas 1 # 主从比例

创建：
```
cd /usr/local/src/redis-5.0-rc3/src/
redis-cli --cluster create 192.168.4.147:5001 192.168.4.147:5002 192.168.4.147:5003 192.168.4.147:5004 192.168.4.147:5005 192.168.4.147:5006 --cluster-replicas 1
```
查看效果：
```
redis-cli -c -h 192.168.4.147 -p 5001
192.168.4.147:5001> set name imooc 
-> Redirected to slot [5798] located at 192.168.4.147:5002
OK
192.168.4.147:5002> get name
"imooc"
192.168.4.147:5002> cluster info    # 查看集群信息
192.168.4.147:5002> cluster nodes    # 查看集群列表
```

### 5-2 cluster集群添加节点-动态扩容 (11:15)
#### 动态添加(从)节点
1. ruby添加节点：redis-trib.rb add-node 192.168.4.147:5007 192.168.4.147:5001
2. 新特性添加节点：redis-cli --cluster add-node 192.168.4.147:5007 192.168.4.147:5001
3. 添加从节点：redis-cli --cluster add-node 192.168.4.147:5008 192.168.4.147:5008

#### 分片(即添加主节点，将集群中已经存在的主节点slot分一些给新节点-因为新节点要作为主节点而非从节点)
1. ruby分片方法：redis-trib.rb reshard 192.168.4.147:5007
2. 新特性分片方法：redis-cli --cluster redhard 192.168.4.147:5007

#### 实际操作，添加两个节点5007、5008
```
cd /usr/local/redis-cluster/
mkdir 5007
mkdir 5008
cp 5001/redis.conf 5007/redis.conf
cp 5001/redis.conf 5008/redis.conf
# 同样地修改配置文件，将5001替换掉
# 启动
cd /usr/local/src/redis-5.0-rc3/src
redis-server /usr/local/redis-cluster/5007/redis.conf
redis-server /usr/local/redis-cluster/5008/redis.conf
# 开始添加节点到集群中
reidis-cli --cluster add-node 192.168.4.147:5007 192.168.4.147:5001 # 将5007添加到集群中,此时5007是没有数据槽slot的
# 下面进行分片，分一些slot给5007
redis-cli --cluster rehard 192.168.4.147:5007 # 回车后会询问需要分配多少个数据槽，讲师弄的500，之后一步一步进行
# 进入客户端查看效果
redis-cli -c -h 192.168.4.147 -p 5007
cluster nodes
# 此时，5007就是一个主节点了，下面来添加5008作为从节点
redis-cli --cluster add-node 192.168.4.147:5008 192.168.4.147:5001
redis-cli -c -h 192.168.4.147 -p 5008 # 进入看看效果
cluster nodes
cluster replicate {node_id} # 将5008指定给某个主节点
```

### 5-3 cluster集群删除节点 (06:25)
#### 删除从节点
1. ruby删除方法：redis-trib.rb del-node 192.168.4.147:5008
2. 新特性删除方法：redis-cli --cluster del-node 192.168.4.147:5008 {node_id}
#### 删除主节点(需要将数据槽移给其它主节点)
1. ruby删除方法：redis-trib.rb reshard 192.168.4.147:5007
2. 新特性删除方法：redis-cli --cluster redhard 192.168.4.147:5007
3. 然后再调用从节点的删除方法即可

例子：
```
redis-cli --cluster del-node 192.168.4.147:5008 {node_id} # 删除从节点
redis-cli --cluster reshard 192.168.4.147:5007 # 输入500(上面创建时拿过来的插槽数)，然后一步一步操作
redis-cli -c -h 192.168.4.147 -p 5001 # 进入集群
cluster nodes # 查看集群节点信息，可以发现5007没有了slot
redis-cli --cluster del-node 192.168.4.147:5007 {node_id} # 正式删除5007节点
```
## 第6章 C语言下的集群管理器
原来的集群管理器是用 Ruby 搭建的，现在改为 C 代码，讲解如何用redis-cli --cluster搭建集群以及此改方法的好处。
### 6-1 Sorted Set回顾及新增命令的应用场景 (02:57)
#### 新的sorted set命令
应用场景
1. 成绩排名：第一名和最后一名
2. 网站热搜：当下最火搜索

## ZPOPMAX
作用：删除返回集合中分值最高的元素
用法：ZPOPMAX key [count]
## ZPOPMIN
作用：删除返回集合中分值最低的元素
用法：ZPOPMIN key [count]
## BZPOPMAX
作用：ZPOPMAX的阻塞版
用法：BZPOPMAX key [key ...] timeout # timeout为0时表示永久阻塞
## BZPOPMIN
作用：ZPOPMIN的阻塞版
用法：BZPOPMIN key [key ...] timeout # timeout为0时表示永久阻塞


### 6-2 ZPOPMAX、ZPOPMIN、BZPOPMAX和BZPOPMIN用法详解 (04:47)
```
127.0.0.1:6379> ZADD exam 100 daji
127.0.0.1:6379> ZADD exam 99 xiaoqiao
127.0.0.1:6379> ZADD exam 98 zhenji
127.0.0.1:6379> ZADD exam 97 anqila
127.0.0.1:6379> ZRANGE exam 0 -1 WITHSCORES # 查看
127.0.0.1:6379> ZPOPMAX exam # 删除并返回score最大的元素
127.0.0.1:6379> ZRANGE exam 0 -1 WITHSCORES # 查看
```
## 第7章 Redis5.0其它新特性
讲解help子命令，版本功能改进及其它新特性的理论知识及应用场景。
### 7-1 碎片整理和内存报告 (06:32)
新增key——>redis分配内存，删除key——>redis并不一定会将内存返还

应用场景：
1. 在运行期间进行自动内存碎片清理，释放内存空间
2. 通过内存报告了解整个系统的内存使用情况

```
# 需要打开redis.conf的碎片整理功能
# 打开以下配置
activedefrag yes 
active-defrag-ignore-bytes 100mb
active-defrag-threshold-lower 10

# 启动redis
redis-server /etc/redis/redis/conf
# 进入redis
redis-cli
# 生成测试数据
127.0.0.1:6379> debug populate 300000 imooc 1000
127.0.0.1:6379> info memory # 查看内存使用情况
127.0.0.1:6379> MEMORY STATS # 这条也可以查看内存情况
127.0.0.1:6379> memory usage imooc:1 # 查看生成的第一个测试数据占用内存大小
```
## 第8章 课程回顾与总结
回顾和总结本次课程，详细说明本次课程的重难点以及以后的学习建议。
### 8-1 课程回顾与总结 (01:38)
- redis的重要配置项
- 讲解stream数据类型
- 讲解help子命令
- 更方便的搭建redis集群
- 新的sorted set命令
- 碎片整理和内存报告