---
{"dg-publish":true,"permalink":"/CS/kafka/kafka 核心技术与实战/"}
---

#kafka

## 消息引擎
### 模型
- 点对点（消息队列）
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

> **Kafka只对“已提交”的消息（committed message）做有限度的持久化保证**


下面给出推荐（最佳）实践

### producer
- 使用带 callback 的 send，正确处理错误
- `acks = all`：尽量保证大于一个 broker 收到消息才视为正常提交
- retries 设置一个较大值，避免意外错误时自动重试

### broker
- `unclean.leader.election.enable = false`： 避免数据落后的broker成为 leader 导致消息丢失过多
- `replication.factor >= 3` ：增加副本数通过冗余避免故障丢失消息
- `min.insync.replicas > 1` ：控制消息被写到多个副本才算commit成功
- `replication.factor > min.insync.replicas`：避免因为某个副本挂机导致不可用？

### consumer
- `enable.auto.commit = false`：避免自动提交导致丢消息，保证消费后再提交offset

## 客户端拦截器

go sarama 库使用 Interceptor 示例（by ChatGPT）
```
package main

import (
	"fmt"
	"log"
	"os"
	"os/signal"
	"time"

	"github.com/Shopify/sarama"
)

type CustomInterceptor struct{}

func (ci *CustomInterceptor) OnSend(message *sarama.ProducerMessage) {
	fmt.Println("Interceptor: Message is being sent")
}

func (ci *CustomInterceptor) OnAck(message *sarama.ProducerMessage, ack *sarama.ProducerAck) {
	fmt.Println("Interceptor: Message sent successfully")
}

func (ci *CustomInterceptor) OnError(message *sarama.ProducerMessage, err error) {
	fmt.Printf("Interceptor: Message sending failed: %v\n", err)
}

func main() {
	config := sarama.NewConfig()
	config.Producer.RequiredAcks = sarama.WaitForAll
	config.Producer.Retry.Max = 5
	config.Producer.Return.Successes = true

	// TODO: Replace with your Kafka broker addresses
	brokerList := []string{"localhost:9092"}

	// Set custom interceptor
	config.Producer.Interceptors = []sarama.ProducerInterceptor{&CustomInterceptor{}}

	producer, err := sarama.NewSyncProducer(brokerList, config)
	if err != nil {
		log.Fatalln("Failed to start producer:", err)
	}
	defer producer.Close()

	topic := "your-topic-name"

	// Create a message
	message := &sarama.ProducerMessage{
		Topic: topic,
		Value: sarama.StringEncoder("Hello, Kafka!"),
	}

	// Send the message
	partition, offset, err := producer.SendMessage(message)
	if err != nil {
		log.Printf("Failed to send message: %v\n", err)
	} else {
		fmt.Printf("Message sent to partition %d at offset %d\n", partition, offset)
	}

	// Gracefully close the producer on interrupt
	signals := make(chan os.Signal, 1)
	signal.Notify(signals, os.Interrupt)
	<-signals
	fmt.Println("Shutting down...")
}

```

## 幂等与事务

### 常见交付可靠性保障
- 最多一次（at most once）：消息可能会丢失，但绝不会被重复发送。
- 至少一次（at least once）：消息不会丢失，但有可能被重复发送。
- 精确一次（exactly once）：消息不会丢失，也不会被重复发送。

### 幂等生产者
配置
将`config.Producer.Idempotent` 设置为 true 即可

原理
broker 端会留存冗余字段，当 producer 发生重复消息后，利用冗余字段去重

局限
- 只保证单分区上的幂等，即保证 topic 的单一分区不会出现重复消息
- 只保证单会话，当 producer 进程重启后，幂等性消失

### 事务
#### kafka 支持的ACID
- A：保证幂等写入多个分区
- I
- I：支持下列级别
	- read_uncommitted：未提交读（consumer 默认级别）
	- read_committed：已提交读
- D：producer 重启后也能保证发送消息的精确一次处理

配置
- producer 开始事务，使用事务api
	- InitTransactions
	- BeginTransaction
	- CommitTransaction
	- RollbackTransaction
- consumer 修改隔离级别（推荐read_committed）


## 消费者组 
为什么需要消费者组
传统消息模型有一下特性（弊端）
- 点对点（消息队列）：一旦消息被消费就被删除，无法被多个不同消费者消费
- 发布订阅：消费者需要订阅topic的全部分区

kafka 使用消费者组可以同时实现以上消息模型
如果所有实例都属于同一个Group，那么它实现的就是消息队列模型；如果所有实例分别属于不同的Group，那么它实现的就是发布/订阅模型。

### 特性
- 1. Consumer Group下可以有一个或多个Consumer实例。这里的实例可以是一个单独的进程，也可以是同一进程下的线程。在实际场景中，使用进程更为常见一些。
- Group ID是一个字符串，在一个Kafka集群中，它标识唯一的一个Consumer Group。
- Consumer Group下所有实例订阅的主题的单个分区，只能分配给组内的某个Consumer实例消费。这个分区当然也可以被其他的Group消费

### Consumer 数量选择
- 理想情况下，Consumer实例的数量应该等于该Group订阅主题的分区总数
- 当 Consumer 的实例数大于分区总数时，会有实例分配不到分区消费

### offset
- offset 本质是一组KV对，Key是分区，V对应Consumer消费该分区的最新位移
- 旧版 kafka 使用 zookeeper 管理offset，新版使用内部 topic ——__consumer_offsets 管理 offset

### Rebalance
Rebalance本质上是一种协议，规定了一个Consumer Group下的所有Consumer如何达成一致，来分配订阅Topic的每个分区

触发条件
1. 组成员数发生变更。比如有新的Consumer实例加入组或者离开组，抑或是有Consumer实例崩溃被“踢出”组。
2. 订阅主题数发生变更。Consumer Group可以使用正则表达式的方式订阅主题，比如consumer.subscribe(Pattern.compile(“t.*c”))就表明该Group订阅所有以字母t开头、字母c结尾的主题。在Consumer Group的运行过程中，你新创建了一个满足这样条件的主题，那么该Group就会发生Rebalance。
3. 订阅主题的分区数发生变更。Kafka当前只能允许增加一个主题的分区数。当分区数增加时，就会触发订阅该主题的所有Group开启Rebalance。
{ #0a048f}


弊端
1. STW：STW 期间，所有消费者都会停止消费
2. 目前Rebalance的设计是所有Consumer实例共同参与，全部重新分配所有分区。其实更高效的做法是尽量减少分配方案的变动。例如实例A之前负责消费分区1、2、3，那么Rebalance之后，如果可能的话，最好还是让实例A继续消费分区
3. 慢






ref
- [kafak核心技术与实战](https://learn.lianglianglee.com/%e4%b8%93%e6%a0%8f/Kafka%e6%a0%b8%e5%bf%83%e6%8a%80%e6%9c%af%e4%b8%8e%e5%ae%9e%e6%88%98/00%20%e5%bc%80%e7%af%87%e8%af%8d%20%e4%b8%ba%e4%bb%80%e4%b9%88%e8%a6%81%e5%ad%a6%e4%b9%a0Kafka%ef%bc%9f.md)