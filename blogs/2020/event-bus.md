---
title: 微服务消息总线
date: '2020-11-22 21:06:21'
sidebar: false
categories:
 - 技术
tags:
 - event-bus
 - 微服务消息总线
publish: true
---

> [项目地址](https://github.com/DuC-cnZj/event-bus)

## feature

1. 秒级延迟队列
2. grpc 任意客户端语言接入
3. 消息重试机制
4. 分布式锁
5. 高性能： 生产消费的瓶颈 = `rabbitmq` 本身生产消费的瓶颈
6. 断线重连(60s)

## v1 架构

![2020_12_01_Wi2TilUlxl.png](../images/2020_12_01_Wi2TilUlxl.png)

## v2 架构

![2020_12_01_aZ8LlB1cQQ.png](../images/2020_12_01_aZ8LlB1cQQ.png)

## Usage

```shell script
 dk run --rm -v $(pwd)/config.yaml:/config.yaml registry.cn-hangzhou.aliyuncs.com/duc-cnzj/event-bus
```

## Configuration

```yaml
Debug: false
PrintConfig: false

# 重试次数
RetryTimes: 3

# 分布式锁加锁时长
DLMExpiration: 1800

# 每个job最大执行时间
MaxJobRunningSeconds: 1800

# nackd 之后重新推送的延迟时间
NackdJobNextRunDelaySeconds: 600

# nack 以及 未消费队列重新推送定时任务(cronjob)
CronRepublishEnabled: true

# 延迟队列定时任务(cronjob)
CronDelayPublishEnabled: true

# 是否开启后台 event_bus_ack_queue 和 event_bus_confirm_queue 消费队列, 保持默认 `true` 就行
BackgroundConsumerEnabled: true

# 后台 event_bus_ack_queue  event_bus_confirm_queue 定时任务 的协程数量
# 默认 10 个就行了，根据数据库处理能力定, BackgroundConsumerEnabled 为 false 不执行
BackConsumerGoroutineNum: 20

AmqpUrl: amqp://guest:guest@localhost:5672/
PrefetchCount: 0
HttpPort: 7878
RpcPort: 9091

# redis
RedisAddr: localhost:6379
RedisDB: 0
RedisUsername:
RedisPassword:

# DATABASE
DB_HOST: 127.0.0.1
DB_PORT: 3306
DB_DATABASE: mq_v2
DB_USERNAME: root
DB_PASSWORD:

# 每个队列的消费者数量
EachQueueConsumerNum: 5

# 每个队列的生产者数量
EachQueueProducerNum: 2
```

## SDK

php laravel, 配置 env 例如：`MQ_HOST=localhost:9091`
```shell script
composer require duc_cnzj/event-bus-sdk
```

golang
```shell script
go get -u github.com/DuC-cnZj/event-bus-proto
```

## 坑

fiber 通过 `ctx.Query("queue", "test_queue")` 拿出来的 string 被底层改过，都指向同一个地址，你前一个请求的值和会被后一个请求的值更改！！！

## TODO

1. 关闭长时间没用的连接

## test case

1. 普通队列/延迟队列：持续压测关注 qos 情况
2. 消费时重启 mq、重启 event-bus，查看数据是否有丢失
3. 后台 job: `republish` `delay publish` 的处理速率
4. 后台 `event_bus_ack_queue` 、 `event_bus_confirm_queue` 、`event_bus_delay_queue` 处理速率，这些操作都是入库操作，和数据库本身性能有关系

### 重点关注

1. 100w 数据量时队列延迟情况
2. 各种有可能丢数据的骚操作，会不会真的丢数据
3. 是否存在内存泄露的情况
4. 数据库可能出现连接过多的问题，注意数据库的最大连接数

## 生产过快导致消费缓慢

https://www.rabbitmq.com/blog/2011/09/24/sizing-your-rabbits/

## 注意事项

1. 关注数据库的最大连接数，防止 `too many connections` 错误导致入库失败 

## 4核16G机器测试

10 个container
纯ping接口压测
![2020_12_01_mk4YyRiJga.png](../images/2020_12_01_mk4YyRiJga.png)
往mq推消息 压测10s
![2020_12_01_jIbYpPUKDv.png](../images/2020_12_01_jIbYpPUKDv.png)
![2020_12_01_4bA94glvPq.png](../images/2020_12_01_4bA94glvPq.png)

往mq推消息 压测30s

![2020_12_01_X3flaF0rOC.png](../images/2020_12_01_X3flaF0rOC.png)
![2020_12_01_k3xN0WkMVe.png](../images/2020_12_01_k3xN0WkMVe.png)

5个 container
![2020_12_01_3rr6T5BVgW.png](../images/2020_12_01_3rr6T5BVgW.png)
![2020_12_01_HdI2FTlIwL.png](../images/2020_12_01_HdI2FTlIwL.png)
![2020_12_01_5jFqlgbmpD.png](../images/2020_12_01_5jFqlgbmpD.png)

生产3w
![2020_12_01_whWirE5IEo.png](../images/2020_12_01_whWirE5IEo.png)
![2020_12_01_nUsSNMS1xl.png](../images/2020_12_01_nUsSNMS1xl.png)
![2020_12_01_3GvOLZMRUC.png](../images/2020_12_01_3GvOLZMRUC.png)

消费(40个)+ack
![2020_12_01_ZhtnPOdo7M.png](../images/2020_12_01_ZhtnPOdo7M.png)

## 实测处理100w条数据

>  生产 100w 条数据执行的时间是 1m2.173512229s


平均每秒 1.6w 条数据

不含业务情况下 consume + ack  100w 数据的时间是 8分钟