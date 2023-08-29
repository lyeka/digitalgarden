---
{"dg-publish":true,"permalink":"/CS/kafka/kafka 核心技术与实战/"}
---

#kafka

## 消息引擎
### 模型
- 点对点
- 发布订阅

### 作用
- 削峰填谷

### 选型
- Kafka
- ActiveMQ
- RabbitMQ
- Pulsar

## kafka 基本概念
### broker

### producer & consumer

### topic

### partition

### offset

### rebalance

![](/img/user/CS/kafka/attachments/Pasted image 20230829133438.png)


## 版本

- 0.7版本：只提供了最基础的消息队列功能。
- 0.8版本：引入了副本机制，至此Kafka成为了一个真正意义上完备的分布式高可靠消息队列解决方案。
- 0.9.0.0版本：增加了基础的安全认证/权限功能；使用Java重写了新版本消费者API；引入了Kafka Connect组件。
- 0.10.0.0版本：引入了Kafka Streams, 正式升级成分布式流处理平台。
- 0.11.0.0版本：提供了幂等性Producer API以及事务API；对Kafka消息格式做了重构。
- 1.0和2.0版本：主要还是Kafka Streams的各种改进。

## 集群参数配置
### broker

### topic

### JVM

### 操作系统

![](/img/user/CS/kafka/attachments/Pasted image 20230829133541.png)

## 生产者分区机制

### partition 分区作用
提供负载均衡的能力，提高系统的伸缩性

### 顺序问题
kafka 只保证消息在分区的顺序性

### 分区算法
- 轮询 round-robin（默认）
- 随机
- 按 key 分区
- 自定义

## 消息压缩





ref
- [kafak核心技术与实战](https://learn.lianglianglee.com/%e4%b8%93%e6%a0%8f/Kafka%e6%a0%b8%e5%bf%83%e6%8a%80%e6%9c%af%e4%b8%8e%e5%ae%9e%e6%88%98/00%20%e5%bc%80%e7%af%87%e8%af%8d%20%e4%b8%ba%e4%bb%80%e4%b9%88%e8%a6%81%e5%ad%a6%e4%b9%a0Kafka%ef%bc%9f.md)

### 消息格式（todo）
在kafka 中，消息分为两层：消息集合（message set）以及消息（message）。一个消息集合中包含若干条日志项（record item），而日志项才是真正封装消息的地方。

### 解压缩流程
生产者压缩；broker保持；消费者解压
- 压缩
	- 生产者压缩
	- broker压缩（与 生产者压缩不对等的话需要解压在压缩）
- 解压

### 压缩算法
- GZIP
- Snappy
- LZ4
- zstd

对比
- 吞吐量：LZ4 > Snappy > zstd=GZIP；
- 压缩比：zstd > LZ4 > GZIP > Snappy；
- 物理资源：使用Snappy算法占用的网络带宽最多，zstd最少；
- CPU使用率：差不多，只是在压缩时Snappy算法使用的CPU较多一些，而在解压缩时GZIP算法则可能使用更多的CPU；

## 如何保证消息不丢
下面给出推荐（最佳）实践

### producer
- 使用带 callback 的 send，正确处理错误
- acks = all：尽量保证大于一个 broker 收到消息才视为正常提交
- 